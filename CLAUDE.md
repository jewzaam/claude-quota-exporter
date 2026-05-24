# claude-quota-exporter

Prometheus exporter for Anthropic Claude OAuth API usage limits. One
pod, one token, scrape-driven.

## Conventions and gotchas

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
- No OAuth refresh. The exporter takes a long-lived bearer minted
  via `claude setup-token` (1-year lifetime). Rotation is a manual
  operator action. The `claude_quota_access_token_expires_at_seconds`
  gauge is the alert signal — fire when value − now drops below
  ~30 days. Re-mint via `make mint-creds`, restart the exporter.
- Mint exporter-scoped credentials with `make mint-creds`. Wraps
  [scripts/mint-creds.sh](scripts/mint-creds.sh) which calls
  `claude setup-token` (interactive — paste URL into browser, paste
  returned code back) and captures the `sk-ant-oat01-*` bearer via
  `claude setup-token 2>&1 | grep sk-ant-oat01 | sed 's/[[:space:]]//g'`.
  Writes `./mint/.credentials.json` with `expiresAt = now + MINT_TTL_DAYS*86400*1000`
  (default 365 days). The `expiresAt` is informational only — feeds
  the `claude_quota_access_token_expires_at_seconds` gauge so
  operators can alert on rotation deadline. Anthropic decides actual
  expiry server-side.
- [.claude/settings.json](.claude/settings.json) denies
  `Read(./mint/**)` / `Glob(./mint/**)` / `Grep(./mint/**)` plus
  shell variants for every Claude Code agent working in this repo —
  do not lift those denies. The exporter consumes
  `./mint/.credentials.json` directly; never copy its contents into
  the repo or into chat.

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
