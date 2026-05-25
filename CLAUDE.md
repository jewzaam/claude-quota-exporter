# claude-quota-exporter

Prometheus exporter for Anthropic Claude OAuth API usage limits. One
pod, one token, scrape-driven.

## Conventions and gotchas

- **Exporter is a dumb credential consumer.** Re-reads the credentials
  file from disk on every fetch attempt (each `/metrics` scrape that
  crosses TTL) and on every `has_credentials()` call (readyz +
  credentials_present metric). No refresh, no write-back, no
  Kubernetes-API access. Rotation is OWNED BY THE ROTATOR — a
  separate workload (operator's manual `kubectl apply`, a CronJob,
  External Secrets Operator, whatever) that updates the mounted
  Secret. Because the fetcher re-reads each cycle, the next scrape
  picks up the new bearer with no pod restart needed — Stakater
  Reloader is no longer required and was removed from the Deployment
  in 0.2.0. Do not add refresh logic, write-back logic, or k8s-API
  code to this exporter — that complexity belongs in the rotator
  workload, not here.
- **Live credential re-read invariants** (since 0.2.0). Every code
  path that touches credential state goes through
  `Fetcher._load_current_credentials()`, called under `self._lock`.
  That helper logs at WARNING when crossing the good→bad boundary
  (or when creds are unloadable on the very first check) and at
  INFO when crossing bad→good or on the first successful load.
  [src/claude_quota_exporter/auth.py](src/claude_quota_exporter/auth.py)
  `read_credentials()` logs the specific parse/IO failure at WARNING
  on every call — keep it at WARNING even though the function is now
  polled. The failure modes (missing file, dir-not-file, malformed
  JSON, missing accessToken) are critical startup signals and must
  surface in pod logs. Steady-bad state is intentionally loud; the
  alternative (DEBUG) would hide the root cause from operators
  diagnosing a never-started exporter. Fetcher transition logs sit
  on top to add the "fetch disabled / restored" operator context
  that auth.py cannot infer on its own.
- **Bearer source must be PKCE-login-derived.** The required input
  is an `accessToken` from a `~/.claude/.credentials.json` shape
  (interactive `claude /login` flow). The 245c8e9 design path used
  the refresh-token grant against this same shape; we sidestepped
  the entire refresh feature in 815c164. The design in
  [docs/superpowers/specs/2026-05-24-oauth-refresh-design.md](docs/superpowers/specs/2026-05-24-oauth-refresh-design.md)
  is preserved as historical context only.
- Token-vs-endpoint compatibility matrix. The `sk-ant-oat01-` prefix
  is shared across multiple Anthropic OAuth token types; same prefix
  does NOT mean same scope. Empirical matrix (verified this session):
  PKCE-login access token (from interactive `claude /login` →
  `~/.claude/.credentials.json`) returns 200 on `/api/oauth/usage`;
  `claude setup-token` output (intended for `CLAUDE_CODE_OAUTH_TOKEN`
  env var, designed for `/v1/messages`) returns 403 on
  `/api/oauth/usage`. For this exporter only PKCE-login access
  tokens are valid input — setup-token output is the wrong credential
  entirely despite the matching prefix.
- Anthropic OAuth error envelopes are NOT RFC-6749 shape.
  `/v1/oauth/token` returns nested errors like
  `{"error":{"type":"rate_limit_error","message":"Rate limited. Please try again later."}}`,
  not the RFC-6749 flat
  `{"error":"rate_limit"}`. Any parser in
  [src/claude_quota_exporter/auth.py](src/claude_quota_exporter/auth.py)
  must read `.error.type`, not `.error`. The `_extract_oauth_error()`
  helper in commit 245c8e9 had this bug (looked for top-level `error`
  as a string only) — re-introduce it carefully when restoring the
  refresh design.
- Anthropic `/v1/oauth/token` 429 responses include NO `Retry-After`
  header (verified via `curl -D -` 2026-05-24 22:08:12 GMT). The
  Cloudflare `__cf_bm` cookie's `Expires` is bot-management noise,
  not a rate-limit hint. There is no clean look-but-don't-touch probe
  to detect when the limit clears — any probe that triggers the gate
  consumes whatever bucket budget applies. See
  [scripts/wait-for-rate-limit-clear.sh](scripts/wait-for-rate-limit-clear.sh)
  for the chosen approach (exponential-backoff sleep + real refresh,
  accepting that a successful probe consumes a rotation).
- Refresh-token invalidation policy after rotation is UNVERIFIED.
  Not the exporter's concern (rotator owns refresh), but a
  consideration for whoever builds the rotator workload. OAuth 2.1
  § 4.13 recommends single-use rotation with invalidation as a
  theft-detection defense; whether Anthropic enforces it is unknown
  (the 2-call test was blocked by rate limit).
- K8s Secret mounts are tmpfs projections. Kubelet re-projects when
  the Secret content changes; the file under
  `/var/run/claude/credentials.json` swaps atomically. The fetcher
  re-reads on every quota-fetch attempt, so the next scrape after
  re-projection picks up the new bearer with no pod restart and no
  Deployment rollout. The exporter has zero in-process refresh and
  no longer carries a Reloader annotation (removed in 0.2.0).
- Do not run `claude auth login` inside a bare `node:22-slim`
  container expecting `.credentials.json` to land on disk. Claude
  Code on Linux targets the OS keychain (libsecret / `secret-tool` /
  `gnome-keyring-daemon`); a slim container has no keyring service.
  The login flow runs and creates supporting directories under
  `~/.claude` (e.g. `backups/`) but writes no `.credentials.json`.
  Run `claude /login` on a workstation that has a keyring instead,
  then copy the resulting bearer into the exporter's mounted Secret.
- Bind host defaults to loopback. `DEFAULT_HOST = "127.0.0.1"` lives at
  [src/claude_quota_exporter/config.py:11](src/claude_quota_exporter/config.py).
  Override to `0.0.0.0` only inside a pod-network scrape boundary; the
  override site is [deploy/k8s/deployment.yaml](deploy/k8s/deployment.yaml)
  (non-root container behind a ClusterIP Service). Backing: subprocess
  security Rule 2 (CVE-2025-49596).
- `make check` runs format, lint, typecheck, unit tests, AND coverage.
  The 80% floor is enforced by the `test-coverage` target in
  [make/test.mk](make/test.mk). Do not regress `__main__.py` coverage —
  the gate will fail.
- HTTP is blocked in tests. [tests/conftest.py](tests/conftest.py)
  autouse-blocks `urllib.request.urlopen` so the fetcher cannot reach
  the real Anthropic API. Server tests need real localhost sockets, so
  [tests/test_server.py](tests/test_server.py) uses raw
  `socket.create_connection` against its own running
  `ThreadingHTTPServer`. Do not "fix" server tests by routing through
  urllib — that path is intentionally blocked.
- Logging level precedence in
  [src/claude_quota_exporter/__main__.py](src/claude_quota_exporter/__main__.py)
  is `--debug` > `--quiet/-q` > `settings.log_level`, resolved in
  `_resolve_log_level()`. Convert a config-file string to an int with
  `logging.getLevelNamesMapping().get(name.upper(), logging.INFO)`.
  `logging.getLevelName()` returns `Any` per mypy stubs and fails
  `disallow_untyped_defs`.
- Bucket extraction is name-agnostic.
  [src/claude_quota_exporter/fetcher.py](src/claude_quota_exporter/fetcher.py)
  `extract_buckets` treats any top-level dict with both `utilization`
  and `resets_at` as a bucket, divides the API's `0..100` percentage by
  100, and emits a `0..1` ratio. `QuotaCollector` surfaces every bucket
  as `claude_quota_utilization{bucket="<name>"}` and
  `claude_quota_resets_at_seconds{bucket="<name>"}`. New bucket names
  (e.g. a future `seven_day_haiku`) need zero code changes.
- Windows venv bootstrap. [Makefile](Makefile) sets `PY_SYS ?= py -3`
  in the Windows arm of the OS branch. Do not change it back to
  `python3` for Windows: on a bare Windows install `python3` triggers
  the Microsoft Store app execution alias and opens the Store instead
  of running Python. POSIX still uses `python3`.
- Token contents never appear in logs.
  [src/claude_quota_exporter/auth.py](src/claude_quota_exporter/auth.py)
  `read_credentials()` and the fetcher log only outcomes (path,
  status, success/failure). Review checklist before merging changes
  to `auth.py` or `fetcher.py`: grep both files for `access_token`
  as a `%s` argument to a logger — should return nothing.
- GHCR push requires a CLASSIC PAT with `write:packages` scope. The
  `make push-image` target pushes to
  `ghcr.io/jewzaam/claude-quota-exporter`. Fine-grained Personal
  Access Tokens DO NOT support GHCR write — there is no Packages
  permission in the fine-grained token UI (verified against
  https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens
  which lists no Packages permission; community feature request open
  in https://github.com/orgs/community/discussions/36441). Classic
  PATs cannot be scoped to a single repository — the token can push
  to ANY of your user-owned packages. For per-repo scoping, the only
  option is a GitHub Actions workflow using the auto-issued
  `GITHUB_TOKEN` (scoped to the repo's own packages, requires
  `permissions: packages: write` in the workflow). Manual workstation
  push: classic PAT with short expiry, treat as broad blast radius.
- Dockerfile carries
  `LABEL org.opencontainers.image.source="https://github.com/jewzaam/claude-quota-exporter"`
  (see [deploy/Dockerfile](deploy/Dockerfile)). GHCR reads this label
  after first push and auto-links the published package to the source
  repo, surfacing it in the repo's Packages sidebar and enabling
  provenance attestation. Do not remove the LABEL; do not point it at
  a different URL.
- `$(abspath ...)` on container bind-mount paths in the
  [Makefile](Makefile). The `run-image` target wraps both mount
  sources in `$(abspath $(CREDS_FILE))` /
  `$(abspath $(IMAGE_CONFIG_FILE))`. This defends against Git Bash
  MSYS2 colon-mangling on Windows: a relative path passed to
  `podman run -v ./path:/container/path:ro` gets rewritten to
  `\Program Files\Git\path;ro` by MSYS2's bash-to-Windows path
  conversion, corrupting the mount spec. The fix is making the
  host-side path absolute before `podman` sees it. Do not regress to
  raw `$(CREDS_FILE)` etc. in any new container-running Makefile
  target — wrap in `$(abspath ...)`.

## Entry points

- [README.md](README.md) — user-facing overview, endpoints, metrics,
  install, container, Kubernetes.
- [pyproject.toml](pyproject.toml) — package metadata, dependencies.
- [Makefile](Makefile) — quality gate and run targets.

## Source

- [src/claude_quota_exporter/__main__.py](src/claude_quota_exporter/__main__.py) — CLI entry, argument parsing, logging setup.
- [src/claude_quota_exporter/config.py](src/claude_quota_exporter/config.py) — Settings dataclass and JSON loader.
- [src/claude_quota_exporter/fetcher.py](src/claude_quota_exporter/fetcher.py) — throttled upstream fetcher + bucket extraction.
- [src/claude_quota_exporter/metrics.py](src/claude_quota_exporter/metrics.py) — Prometheus Collector that emits labeled gauges.
- [src/claude_quota_exporter/server.py](src/claude_quota_exporter/server.py) — stdlib HTTP server (`/metrics`, `/healthz`, `/readyz`).

## Tests

- [tests/conftest.py](tests/conftest.py) — autouse safety guards (no subprocess, no real HTTP, no out-of-tmp writes).
- [tests/test_config.py](tests/test_config.py)
- [tests/test_fetcher.py](tests/test_fetcher.py)
- [tests/test_metrics.py](tests/test_metrics.py)
- [tests/test_server.py](tests/test_server.py)

## Deploy

- [deploy/Dockerfile](deploy/Dockerfile) — multi-stage, non-root uid 10001.
- [deploy/k8s/configmap.yaml](deploy/k8s/configmap.yaml) — exporter config.
- [deploy/k8s/secret-example.yaml](deploy/k8s/secret-example.yaml) — credentials Secret shape (placeholder).
- [deploy/k8s/deployment.yaml](deploy/k8s/deployment.yaml) — restricted-pod Deployment.
- [deploy/k8s/service.yaml](deploy/k8s/service.yaml) — ClusterIP + scrape annotations.
- [deploy/k8s/networkpolicy.yaml](deploy/k8s/networkpolicy.yaml) — default-deny ingress + placeholder allow rule; operator must edit the scraper selector before apply.

## Config

- [config.example.json](config.example.json) — template; copy to a gitignored working file before editing.

## CI

- [.github/workflows/test.yml](.github/workflows/test.yml) — pytest + coverage matrix (3.11/3.12/3.13).
- [.github/workflows/quality.yml](.github/workflows/quality.yml) — black / flake8 / mypy gate.

## Findings / reports

- [Findings-standards.json](Findings-standards.json) / [Findings-standards.md](Findings-standards.md) / [Findings-standards-supplementary.md](Findings-standards-supplementary.md) — most recent standards audit.
