# claude-quota-exporter: OAuth refresh-token support

Status: approved design, ready for implementation plan.
Date: 2026-05-24.

## Summary

Today the exporter reads only `claudeAiOauth.accessToken` from the
credentials file and dies when that bearer expires (Anthropic's
access tokens last ~8 hours; the exporter has no way to recover).
This design adds:

- Reading and writing the full credential set (`accessToken`,
  `refreshToken`, `expiresAt`).
- Proactive refresh via the OAuth `refresh_token` grant against
  `https://console.anthropic.com/v1/oauth/token` before the access
  token expires.
- Expiry-gated reactive refresh when the usage endpoint returns 401
  and our own expiry math agrees the token is past its skew window.
- Atomic write-back so a process restart does not lose the rotated
  token pair.
- New metrics for refresh outcomes, token expiry, credential
  loadability, and credential writability.
- A new `/readyz` contract: ready iff credentials are loaded.

## Goals

1. The exporter survives access-token expiry without operator
   intervention.
2. Refreshed credentials are persisted to the same file the operator
   originally mounted, preserving every field we did not touch.
3. The refresh path never leaks token material to logs.
4. Refresh failures and write-back failures are observable in
   Prometheus as distinct signals.
5. Operators can distinguish "exporter is misconfigured" from
   "exporter is up but its upstream is sad" via probes and metrics
   independently.

## Non-goals

1. **No multi-token support.** One credentials file, one token pair.
   Anthropic-side multi-tenancy is out of scope; spawn one Pod per
   token.
2. **No PKCE / initial OAuth flow.** The exporter consumes
   credentials produced by Claude Code (or any compatible source).
   It never opens a browser, never speaks to
   `claude.ai/oauth/authorize`, never invents a `client_secret`.
3. **No refresh-token expiry metric.** Anthropic's refresh endpoint
   does not expose refresh-token expiry. We document the gap and
   emit `claude_quota_refresh_token_issued_at_seconds` (when *we*
   first observed the current refresh token) as the closest proxy.
4. **No Pod-restart on credentials change.** If the operator
   replaces the Secret, they restart the Pod. (Kubernetes Secret
   mounts auto-reload except under `subPath`; the exporter does not
   try to harmonize the two cases.)
5. **No mid-process refresh-URL discovery.** The list of endpoints
   is config; if Anthropic adds a host, the operator updates config
   or upgrades the exporter.

## Verified facts (do not invent)

Verified via five independent third-party implementations (cedws
gist, NousResearch/hermes-agent, mastra-ai/mastra, plandex-ai/plandex,
1jehuang/jcode):

- **Refresh endpoint URL.** `POST https://console.anthropic.com/v1/oauth/token`
  is universal. `https://platform.claude.com/v1/oauth/token` appears
  in hermes-agent as a preferred-first host. We expose both as config;
  default list is `[platform.claude.com, console.anthropic.com]`,
  tried in order.
- **Method.** POST.
- **Content type.** `application/json` (also accepted:
  `application/x-www-form-urlencoded`; we use JSON).
- **Request body.**
  ```json
  {
    "grant_type": "refresh_token",
    "refresh_token": "<the stored refresh token>",
    "client_id": "9d1c250a-e61b-44d9-88ed-5944d1962f5e"
  }
  ```
  The `client_id` is the well-known Claude Code public client
  identifier.
- **Response body.** Contains `access_token` (required),
  `refresh_token` (rotates on every successful refresh — write back
  or the next refresh fails), `expires_in` (seconds; default 3600
  if absent).
- **Refresh-token expiry is NOT in the response.** Only the
  access-token lifetime is returned. This gap is the reason
  `claude_quota_refresh_token_expires_at_seconds` is omitted from
  the metric set.
- **Refresh tokens rotate on every successful call.** Write-back is
  mandatory for surviving a process restart after any refresh.
- **HTTP 401 from the usage endpoint** typically indicates an
  expired or revoked access token. The response body shape for
  Anthropic's `/api/oauth/usage` is not publicly documented; we do
  not parse the body, only the status code.

## Architecture

### Module split

```
src/claude_quota_exporter/
├── auth.py        NEW: Credentials dataclass + read/refresh/write functions
├── config.py      EXTEND: new knobs for refresh
├── fetcher.py     EXTEND: consume auth, hold refresh state, expiry-gated 401
├── metrics.py     EXTEND: new gauges/counters in QuotaCollector
├── server.py      EXTEND: /readyz semantic change
└── __main__.py    UNCHANGED (one-line init delta)
```

`auth.py` is functional decomposition: a small immutable
`Credentials` dataclass plus pure functions that touch disk and
network. State and the single lock stay in `Fetcher`. No new
long-lived stateful class.

### Data flow on a scrape

```
1. Prometheus scrapes /metrics
2. QuotaCollector.collect()
3.   Fetcher.refresh_if_due()
       acquire _lock
       compute now, expiry_gap, is_due
       if not is_due: release lock, return False
       if _creds is None: try auth.read_credentials(); set _creds
       if _creds is None still: release lock, return False
       if expiry_gap < settings.refresh_skew_seconds:
         _proactive_refresh()
       raw = auth.fetch_usage(_creds.access_token, ...)
       if status == 401 AND expiry_gap <= skew AND
          (now - _last_refresh_at) > skew:
         _reactive_refresh()
         raw = auth.fetch_usage(_creds.access_token, ...)  (one retry)
       update cache + counters
       release _lock
4.   Yield labeled gauges + counters from cache
```

The whole sequence runs under one `Fetcher._lock`. Worst-case scrape
latency = refresh round-trip (~hundreds of ms once every ~8 hours)
plus a usage round-trip. Acceptable; matches the existing design.

## Credentials data model

### On-disk format (unchanged: matches Claude Code)

```json
{
  "claudeAiOauth": {
    "accessToken": "sk-ant-oat01-...",
    "refreshToken": "sk-ant-ort01-...",
    "expiresAt": 1779580800694,
    "scopes": ["user:profile", "user:inference"],
    "subscriptionType": "max"
  }
}
```

- `expiresAt` is milliseconds-since-epoch (Claude Code convention).
  `read_credentials` also accepts ISO-8601 string form (some
  community implementations write that shape) and converts both
  to internal epoch-seconds float.
- `scopes`, `subscriptionType`, and any other sibling fields are
  preserved verbatim on write-back. The exporter touches only the
  three fields it owns.

### In-memory model

```python
@dataclass(frozen=True)
class Credentials:
    access_token: str
    refresh_token: str
    expires_at: float          # epoch seconds (not ms)
    issued_at: float           # epoch seconds; when WE first observed
                               # this refresh_token in our cache
```

`read_credentials(path: Path) -> tuple[Credentials, dict] | None`
returns the parsed Credentials plus the raw dict (used at write-back
time to preserve unknown fields).

`refresh(...) -> Credentials | None` returns a new Credentials with
`issued_at = time.time()` on success.

`write_back(path, creds, raw_dict) -> bool` writes the merged
state atomically, preserves source-file mode, never raises.

## Refresh trigger semantics

### Proactive

Fired during `refresh_if_due()` when:

```python
expiry_gap = _creds.expires_at - time.time()
if expiry_gap < settings.refresh_skew_seconds:
    new_creds = auth.refresh(...)
```

`settings.refresh_skew_seconds` defaults to **300** (5 minutes).
Anthropic access tokens last ~8 hours; this gives a wide margin and
matches Claude Code's documented "refresh 5 minutes early" behavior.

### Reactive (expiry-gated, single-retry)

Fired only when the usage endpoint returns HTTP 401 AND both:

- `(_creds.expires_at - now) <= settings.refresh_skew_seconds` —
  our own expiry math agrees the token is past its skew window.
- `(now - _last_refresh_at) > settings.refresh_skew_seconds` — we
  have not refreshed in the last skew window. Prevents loop on a
  refresh-then-401 cycle that refresh would not fix anyway.

If both gates pass, refresh once, retry the usage call once. The
retry's success or failure feeds the existing `fetch_success_total`
or `fetch_failure_total`. Refresh accounting is always separate
(`refresh_success_total` / `refresh_failure_total`).

All other 401s flow through `fetch_failure_total` and the existing
exponential backoff. We do not parse the 401 body — Anthropic's
error shape for `/api/oauth/usage` is undocumented.

### Cold start

`Fetcher.__init__` calls `auth.read_credentials(path)` once. If it
returns `None`:

- `_creds` is `None`.
- `claude_quota_credentials_present` gauge = 0.
- `claude_quota_credentials_writable` gauge = 0.
- `/readyz` returns 503.
- `refresh_if_due()` short-circuits without log spam on every tick.

Recovery: operator fixes the credentials file, then restarts the
Pod (or the host process). The exporter does not re-probe the file
during its lifetime — Secret mount auto-reload is unreliable across
`subPath` configurations and we keep the contract uniform.

## Write-back and degraded mode

### Atomic write

```python
def write_back(path: Path, creds: Credentials, raw: dict) -> bool:
    raw.setdefault("claudeAiOauth", {}).update({
        "accessToken": creds.access_token,
        "refreshToken": creds.refresh_token,
        "expiresAt": int(creds.expires_at * 1000),
    })
    try:
        src_mode = path.stat().st_mode & 0o777
    except OSError:
        return False
    tmp = path.with_suffix(path.suffix + ".tmp")
    try:
        with tmp.open("w", encoding="utf-8") as f:
            json.dump(raw, f, indent=2)
            f.flush()
            os.fsync(f.fileno())
        os.replace(tmp, path)
        try:
            os.chmod(path, src_mode)
        except OSError:
            pass  # Windows; non-fatal
        return True
    except OSError:
        tmp.unlink(missing_ok=True)
        return False
```

### Degraded mode (write-back disabled)

Detected once at startup via a no-op probe in `auth.is_writable`:
create a sibling temp file, delete it. If either fails,
`claude_quota_credentials_writable = 0` for the life of the
process; refresh continues to work in-memory; rotated tokens are
lost on Pod restart.

If write-back ever fails mid-run (filesystem flips to read-only,
quota hit, etc.), the gauge latches to 0, a single WARNING is
logged, and `write_back` is not retried during the process
lifetime. Operator fixes the mount and restarts the Pod.

## /readyz semantic change

| State | /healthz | /readyz |
|---|---|---|
| Process up | 200 | 200 |
| Creds loaded (`_creds is not None`) | 200 | 200 |
| Creds missing / malformed | 200 | 503 |
| Token currently invalid (401 streak) | 200 | 200 (deps in metrics) |
| `credentials_writable == 0` | 200 | 200 (soft degrade) |
| First fetch hasn't happened yet | 200 | 200 |

Rationale: misconfiguration ("operator forgot to mount the
Secret") earns a loud Kubernetes signal — Pod NotReady, kube state
metrics, alerting. Runtime dependency sadness lives in metrics
because flapping /readyz hides the signal under ServiceMonitor-style
scrapes (Pod gets dropped from endpoints). Operators using pod-level
scrape (`role: pod`) still see metrics during cold start.

`server.py` change: drop the `if fetcher.last_success_at > 0.0`
branch in the `/readyz` handler. Replace with a check against
`fetcher.has_credentials()` (new accessor that returns `_creds is
not None`).

## Metrics surface

Naming follows the new
[observability/metric-naming.md](https://github.com/jewzaam/standards/blob/main/observability/metric-naming.md)
standard. Absolute-time suffix policy: `_seconds` is acceptable when
the name itself reads as a timestamp (`_at_seconds`, `_resets_at_seconds`);
`_timestamp_seconds` is required otherwise.

| Name | Type | Labels | Source / notes |
|---|---|---|---|
| `claude_quota_utilization` | gauge | `bucket` | Existing. `extract_buckets`. Ratio 0..1. HELP states the range. |
| `claude_quota_resets_at_seconds` | gauge | `bucket` | Existing. `extract_buckets`. Unix epoch. |
| `claude_quota_fetch_success_total` | counter | — | Existing. |
| `claude_quota_fetch_failure_total` | counter | — | Existing. |
| `claude_quota_last_fetch_timestamp_seconds` | gauge | — | Existing. `_timestamp_seconds` because name does not contain `_at`. |
| `claude_quota_access_token_expires_at_seconds` | gauge | — | NEW. `_creds.expires_at`. Omitted when `_creds is None`. |
| `claude_quota_refresh_token_issued_at_seconds` | gauge | — | NEW. `_creds.issued_at`. Omitted when `_creds is None`. |
| `claude_quota_last_refresh_timestamp_seconds` | gauge | — | NEW. `_last_refresh_at`. |
| `claude_quota_credentials_present` | gauge | — | NEW. `1` if `_creds is not None`, else `0`. |
| `claude_quota_credentials_writable` | gauge | — | NEW. `1` if writable, `0` if degraded. |
| `claude_quota_refresh_success_total` | counter | — | NEW. Increments on each successful refresh (proactive or reactive). |
| `claude_quota_refresh_failure_total` | counter | — | NEW. Increments when ALL configured refresh URLs fail in one cycle. |

**Refresh-token expiry is not surfaced.** Anthropic's response
omits it. Documented in README.

**Failure counting under endpoint failover.** When
`settings.refresh_urls = ["A", "B"]` and A returns 5xx while B
returns 2xx, this counts as ONE success. Operators see the URL-1
outage in the WARNING log line (`oauth refresh failed url=A
status=503 error=unknown`) but not in the metric.

## Config knobs

| Key | Type | Default | Purpose |
|---|---|---|---|
| `refresh_skew_seconds` | int | `300` | Refresh proactively when `expires_at - now < skew`. Also gates reactive refresh. |
| `refresh_urls` | list[str] | `["https://platform.claude.com/v1/oauth/token", "https://console.anthropic.com/v1/oauth/token"]` | Tried in order until 2xx. |
| `oauth_client_id` | str | `"9d1c250a-e61b-44d9-88ed-5944d1962f5e"` | Public Claude Code client ID. |
| `refresh_user_agent` | str | `"claude-quota-exporter/<version>"` | UA header on the refresh POST. |

Reuses existing `fetch_timeout_seconds` for the refresh POST
timeout. No separate `refresh_timeout_seconds` unless asked.

`config.example.json` ships `refresh_skew_seconds: 300` (the only
new knob most operators will see) and a comment explaining the
others are usually fine as defaults.

## Logging discipline

- `auth.refresh()` success: INFO `oauth refresh succeeded url=<which> attempts=<n>`. No token, no prefix, no hash.
- `auth.refresh()` failure: WARNING `oauth refresh failed url=<which> status=<code> error=<oauth-error-code>`. `oauth-error-code` is the RFC-6749 `error` field from the response body when present (e.g., `invalid_grant`), else `unknown`. Body content beyond that field is ignored.
- `auth.write_back()` first success in process: INFO `credentials written back`.
- `auth.write_back()` first failure: WARNING `credentials write-back failed; running in-memory only`. Latches `credentials_writable=0`.
- `auth.read_credentials()` failure at startup: ERROR `credentials_path not loadable: <reason>`. Subsequent ticks suppress repeated logs.
- No log line in any module emits `access_token` or `refresh_token` content. Review checklist: `grep -E "access_token|refresh_token" src/claude_quota_exporter/auth.py` must not show either as a `%s` argument to a logger.

## Error handling rollup

| Failure | Effect |
|---|---|
| Refresh URL N fails (network error, timeout, non-2xx, non-JSON body, or 2xx with missing `access_token`) | Try URL N+1. If all configured URLs fail, `_refresh_failure_count++`, log WARNING per attempt, keep old creds, fall through to fetch with old token. |
| Refresh 401 / `invalid_grant` on the last URL | Same as above. In practice operator must re-authenticate via Claude Code; the exporter cannot self-recover. |
| Usage 401, expiry math says fresh | NO refresh; counts as fetch failure. |
| Usage 401, expiry math says stale + no recent refresh | Reactive refresh + 1 retry. |
| Write-back failure at startup probe | `credentials_writable=0` from start; never attempted during process lifetime. |
| Write-back failure mid-run | Latch `credentials_writable=0`, log once, no retry. |
| Credentials file missing/malformed at startup | `_creds=None`, `credentials_present=0`, `/readyz=503`. |

## Testing approach

Test isolation rules from `tests/conftest.py` (autouse
`urllib.request.urlopen` block) extend naturally. The refresh path
uses the same `urllib.request.urlopen` patch pattern as the existing
fetch tests.

### New test files

**`tests/test_auth.py`** — pure-function tests of `auth.py`:

- `read_credentials` happy path, missing file, malformed JSON, missing `claudeAiOauth` block, missing required sub-field (`accessToken`, `refreshToken`, `expiresAt`), `expiresAt` as int vs string, preservation of unknown sibling fields.
- `refresh` happy path (single URL), failover (URL 1 5xx → URL 2 2xx), all URLs fail, missing `access_token`, missing `refresh_token` (reuse old), missing `expires_in` (fall back to 3600).
- `write_back` happy path, mode preservation on POSIX (skip the chmod assertion on Windows), atomic-replace via tmp file, write failure rolls back tmp file, returns False without raising.
- `is_writable` true / false paths via tmp-dir permission tweaks (skip the POSIX-only chmod test on Windows).

### Extended test files

**`tests/test_fetcher.py`** — new cases:

- Proactive refresh fires when `expires_at - now < skew`.
- Proactive refresh does NOT fire when token has plenty of life.
- 401 reactive refresh fires when token is expired AND no recent refresh; retries usage call once; success counted in both `fetch_success_total` and `refresh_success_total`.
- 401 reactive refresh DOES NOT fire when token has plenty of life.
- 401 reactive refresh DOES NOT fire when we already refreshed in the last skew window.
- Refresh failure: `_refresh_failure_count++`, old `_creds` preserved, no write-back attempted.
- Mid-run write-back failure latches `credentials_writable=0` and never retries.
- Startup with missing creds: `_creds is None`, `credentials_present=0`, no refresh attempted on subsequent ticks.

**`tests/test_server.py`** — new cases:

- `/readyz` returns 503 when `_creds is None`.
- `/readyz` returns 200 when `_creds is not None` regardless of fetch state.
- `/healthz` returns 200 in all states tested.

**`tests/test_metrics.py`** — new cases:

- All new gauges/counters render with the right name + help + value.
- When `_creds is None`, `access_token_expires_at_seconds` and `refresh_token_issued_at_seconds` series are absent (only `# HELP` / `# TYPE` lines).
- `credentials_present` and `credentials_writable` toggle as expected.

Coverage gate stays at 80%. `__main__.py` coverage holds at ~97%
(only `_configure_logging` and arg parsing are touched).

## Documentation deliverables in this PR

- **README.md** — new "Authentication and token refresh" section
  covering the refresh model, the writability contract, the
  missing-refresh-token-expiry gap, and the metric set. Updated
  `## Configuration` for new knobs.
- **CLAUDE.md** — two new gotchas: "no token in logs ever" and
  "credentials_path must be writable to survive token rotation".
- **config.example.json** — `refresh_skew_seconds: 300`. Comment
  notes the other knobs are usually fine.
- **deploy/k8s/secret-example.yaml** — example payload now includes
  `refreshToken` and `expiresAt`.
- **~/source/standards/observability/metric-naming.md** — new
  cross-project standard. Committed separately to the standards
  repo.

## Out of scope

1. Initial OAuth login (PKCE flow). Out: use Claude Code.
2. Multi-credential / multi-token support. Out: one Pod per token.
3. Refresh-token expiry metric. Out: Anthropic does not expose it.
4. Hot-reload of the credentials file without Pod restart. Out: rely on the operator and `kubectl rollout restart`.
5. Volume / Secret design beyond "give me a writable path". Out: operator concern.

## Open questions (none blocking implementation)

None at writing time. The two endpoint-URL hosts
(`console.anthropic.com` vs `platform.claude.com`) are encoded as a
config list so future Anthropic-side changes do not require a code
release.

## Implementation order (for writing-plans)

1. `auth.py` skeleton + `Credentials` dataclass + `read_credentials` + unit tests.
2. `auth.refresh` + unit tests (mocked `urllib.request.urlopen`).
3. `auth.write_back` + `auth.is_writable` + unit tests.
4. `config.py` — add new knobs with defaults.
5. `fetcher.py` — adopt `Credentials`, add refresh state, proactive refresh path, write-back integration.
6. `fetcher.py` — reactive 401 path.
7. `metrics.py` — new gauges/counters.
8. `server.py` — /readyz semantic change.
9. `__main__.py` — wire Settings → Fetcher (minimal delta).
10. Docs: README, CLAUDE.md, config.example.json, secret-example.yaml.
11. Standards file: observability/metric-naming.md (committed to the standards repo separately).
12. `make check` green; tag release `v0.2.0`.
