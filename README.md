# claude-quota-exporter

Standalone Prometheus exporter for Anthropic Claude OAuth API usage
limits. Polls `GET https://api.anthropic.com/api/oauth/usage` on a TTL,
caches the last response in memory, and exposes the buckets as
Prometheus gauges on `GET /metrics`.

Designed to run as a Kubernetes pod with the OAuth credentials mounted
from a `Secret`.

## Overview

What it solves: surfaces the five-hour and seven-day Claude OAuth quota
buckets to Prometheus so dashboards and alerting know how close a
process is to the rate limit before requests start failing.

Key features:

- Scrape-driven refresh — `/metrics` triggers a throttled upstream
  fetch; no background poller needed.
- TTL gating + exponential backoff guards against hot-looping the API
  on a failing token or upstream outage.
- Bucket discovery is dynamic — every top-level dict in the API
  response carrying `utilization` and `resets_at` becomes a labeled
  series (no code change required when Anthropic adds a new bucket).
- Stateless single process. No DB, no broadcast, no WebSocket — one
  pod, one token, one scrape endpoint.
- Kubernetes-native: ships a Deployment, Service, ConfigMap, and
  Secret example with restricted-pod security defaults.

Pipeline fit: Prometheus scrapes the exporter; Grafana / Alertmanager
consume the gauges and trigger before quota exhaustion impacts the
Claude consumer (CI runner, agent, IDE, etc.).

## Endpoints

| Path       | Purpose                                                                 |
|------------|-------------------------------------------------------------------------|
| `/metrics` | Prometheus exposition. Triggers a throttled fetch on scrape.            |
| `/healthz` | Liveness — `200` while the process is up.                               |
| `/readyz`  | Readiness — `200` once the credentials file is loaded; `503` until then.|

## Authentication

The exporter consumes a long-lived OAuth bearer token minted via
`claude setup-token` (requires an active Claude Pro/Max subscription).
Tokens are valid for approximately 1 year. The exporter does NOT
refresh tokens — rotation is a manual operator action signalled by
the `claude_quota_access_token_expires_at_seconds` gauge.

### Mint a token

```sh
make mint-creds                                 # writes ./mint/.credentials.json
make mint-creds MINT_DIR=/secure/path           # override location
make mint-creds MINT_TTL_DAYS=180               # override expiry
```

The script wraps `claude setup-token` (interactive — paste the URL
into your browser, paste the returned code back into the terminal)
and writes a `credentials.json` of the shape:

```json
{
  "claudeAiOauth": {
    "accessToken": "sk-ant-oat01-...",
    "expiresAt": 1811116800000
  }
}
```

`expiresAt` is informational only — it records when you minted the
token plus `MINT_TTL_DAYS` (default 365). The exporter does not act
on it beyond emitting the gauge. Anthropic determines actual expiry
server-side.

### Rotate

Re-run `make mint-creds` and restart the exporter. Recommended alert:
fire when `claude_quota_access_token_expires_at_seconds - time() <
30 * 86400` (30 days from expiry).

### Why no refresh

The first iteration of the exporter implemented the full OAuth
refresh-token grant. It was over-engineered for this use case — the
long-lived bearer is simpler, dodges the per-refresh-token rate-limit
bucket on `/v1/oauth/token`, and matches Anthropic's headless-tool
guidance (the `CLAUDE_CODE_OAUTH_TOKEN` env var pattern). The full
refresh design is preserved in
[docs/superpowers/specs/2026-05-24-oauth-refresh-design.md](docs/superpowers/specs/2026-05-24-oauth-refresh-design.md)
for reference.

## Metrics

| Name                                          | Type    | Labels    | Notes                                       |
|-----------------------------------------------|---------|-----------|---------------------------------------------|
| `claude_quota_utilization`                    | gauge   | `bucket`  | Ratio in `0..1` (API percentage / 100).     |
| `claude_quota_resets_at_seconds`              | gauge   | `bucket`  | Unix epoch seconds when the bucket resets.  |
| `claude_quota_fetch_success_total`            | counter | —         | Successful upstream fetches.                |
| `claude_quota_fetch_failure_total`            | counter | —         | Failed upstream fetches.                    |
| `claude_quota_last_fetch_timestamp_seconds`   | gauge   | —         | Unix epoch of the last successful fetch.    |
| `claude_quota_access_token_expires_at_seconds`| gauge   | —         | Unix epoch when the bearer token expires (informational; omitted when creds absent or expiresAt not set). |
| `claude_quota_credentials_present`            | gauge   | —         | `1` if a valid credentials file is loaded, else `0`. |

Buckets are discovered dynamically — every top-level dict in the API
response containing `utilization` and `resets_at` is emitted as a
labeled series. Known names today: `five_hour`, `seven_day`,
`seven_day_opus`.

## Installation

### Development

```sh
git clone https://github.com/jewzaam/claude-quota-exporter.git
cd claude-quota-exporter
make install-dev
```

### From Git

```sh
pip install git+https://github.com/jewzaam/claude-quota-exporter.git
```

## Usage

```sh
python -m claude_quota_exporter --config <path-to-config.json>
```

Options:

| Flag             | Type | Default | Purpose                                              |
|------------------|------|---------|------------------------------------------------------|
| `--config PATH`  | path | —       | required path to JSON config file                    |
| `--debug`        | flag | off     | force DEBUG log level (overrides config)             |
| `--quiet` / `-q` | flag | off     | force WARNING log level (overrides config)           |
| `--log-file PATH`| path | none    | append log records to file in addition to stderr     |
| `--dryrun`       | flag | off     | resolve config, log planned listener, do not bind    |
| `--version`      | flag | —       | print version and exit                               |

## Configuration

The exporter reads JSON config from `--config <path>`. All keys
optional except as noted. Copy `config.example.json` to a local
working file before editing — `config.local.json` is gitignored.

```json
{
  "host": "127.0.0.1",
  "port": 9180,
  "credentials_path": "/var/run/claude/credentials.json",
  "fetch_ttl_seconds": 120,
  "fetch_timeout_seconds": 10,
  "max_backoff_seconds": 3600,
  "log_level": "INFO"
}
```

`credentials_path` must point at a file with the bearer-token shape:

```json
{"claudeAiOauth": {"accessToken": "sk-ant-oat01-...", "expiresAt": 1811116800000}}
```

`expiresAt` is optional. When absent, `claude_quota_access_token_expires_at_seconds`
is not emitted.

## Fetch throttling

- TTL gates how often a fetch attempt fires (default 120s).
- On consecutive failures, exponential backoff of
  `min((2^failures) * TTL, max_backoff_seconds)` suppresses retries.
- `/metrics` scrapes attempt a refresh, then serve from the in-memory
  cache. Stale cache is preserved across failures so Prometheus keeps
  receiving the last known values.

## Local development

```sh
cp config.example.json config.local.json   # edit credentials_path / overrides
make install-dev
make check          # format + lint + typecheck + unit + coverage
make run            # uses ./config.example.json by default
```

## Container

```sh
make build-image
docker run --rm -p 9180:9180 \
  -v /path/to/credentials.json:/var/run/claude/credentials.json:ro \
  -v $(pwd)/config.example.json:/etc/claude-quota-exporter/config.json:ro \
  claude-quota-exporter:0.1.0
```

## Kubernetes

Manifests under [`deploy/k8s/`](deploy/k8s) are namespace-agnostic — pass
`-n <ns>` at apply time. Apply order:

```sh
NS=<your-namespace>
kubectl -n $NS apply -f deploy/k8s/configmap.yaml
kubectl -n $NS apply -f deploy/k8s/secret-example.yaml   # replace with real creds first
kubectl -n $NS apply -f deploy/k8s/deployment.yaml
kubectl -n $NS apply -f deploy/k8s/service.yaml
kubectl -n $NS apply -f deploy/k8s/networkpolicy.yaml    # edit scraper selector first
```

The Service carries `prometheus.io/scrape` annotations for
annotation-based discovery; switch to a `ServiceMonitor` if you run
the Prometheus Operator.

### NetworkPolicy

[`networkpolicy.yaml`](deploy/k8s/networkpolicy.yaml) ships a
default-deny ingress with a placeholder allow rule. The placeholder
selector matches nothing on purpose so an unconfigured policy fails
closed. **Edit the `from:` block** to match your Prometheus pod
selector (see comments in the file for common patterns) before
applying.

### Config / Secret reload

The exporter reads its ConfigMap and Secret once at process start.
Trigger a manual rollout when the mounted content changes:

```sh
kubectl -n $NS rollout restart deployment/claude-quota-exporter
```
