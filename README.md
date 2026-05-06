[Português](README_PT.md)

---

# Claude Code — Monitoring Stack

> [Versão em Português](README_PT.md)

> This documentation was AI-generated

Complete observability stack for Claude Code (metrics, logs/events and traces) based on **Grafana Alloy** (OpenTelemetry collector), **Grafana Mimir** (metrics), **Loki**, **Tempo** and **Grafana** — full **LGTM** stack.
Metrics:
<img width="1914" height="893" alt="image" src="https://github.com/user-attachments/assets/e7067c08-9044-40ca-9636-af09f91c05c6" />

Tracing
<img width="1914" height="893" alt="image" src="https://github.com/user-attachments/assets/4e9af412-9dd6-44f2-9a60-0cc80b803f21" />

Logging
<img width="1914" height="893" alt="image" src="https://github.com/user-attachments/assets/9505660e-1dcf-4421-ac00-6309ed0802c2" />
```
┌────────────┐    OTLP     ┌──────────┐    ┌──────────────┐    ┌─────────┐
│ Claude Code│ ──────────▶ │  Alloy   │ ─▶ │ Mimir        │ ─▶ │ Grafana │
│  (CLI/SDK) │  4317/4318  │ (OTel    │ ─▶ │ Loki         │ ─▶ │  3000   │
└────────────┘             │ Collector)│ ─▶ │ Tempo        │    └─────────┘
                           └──────────┘    └──────────────┘
```

## Components

| Service | Port | Function |
|---|---|---|
| **Alloy** | 4317 (gRPC), 4318 (HTTP), 12345 (UI) | Receives OTLP from Claude and routes to Mimir/Loki/Tempo |
| **Mimir** | 9009 | Metrics (cost, tokens, sessions) — Prometheus-compatible API (PromQL) |
| **Loki** | 3100 | Logs/events (`api_request`, `user_prompt`, `tool_result`...) |
| **Tempo** | 3200 | Distributed traces (spans `claude_code.interaction`, etc.) |
| **Grafana** | 3000 | Dashboards and log↔trace↔metric correlation (admin/admin) |

## Starting the stack

```bash
docker compose up -d
```

Access Grafana at http://localhost:3000 (login `admin` / `admin`). The **Claude Code — ROI & Usage** dashboard comes pre-provisioned.

---

# Configuring Claude Code

There are **two ways** to configure OpenTelemetry variables in Claude Code. The recommended approach for personal/development use is via **shell** (quick, session-isolated). For corporate use the recommended approach is via a **managed `settings.json`** (centralized, overrides shell variables).

## Option 1 — `settings.json` (persistent configuration / admin)

Claude Code reads configuration from multiple `settings.json` files in order of precedence. Variables in the `env` field of these files **override shell variables** when the file is managed.

### File locations (precedence order, lowest to highest)

| Scope | Path | Who edits |
|---|---|---|
| User | `~/.claude/settings.json` | You (personal, whole machine) |
| Shared project | `<repo>/.claude/settings.json` | Team (git-versioned) |
| Local project | `<repo>/.claude/settings.local.json` | You (not versioned) |
| **Managed (admin)** | `/etc/claude-code/managed-settings.json` (Linux/macOS) or `C:\ProgramData\ClaudeCode\managed-settings.json` (Windows) | Administrator via MDM — **overrides everything** |

### Configuration via Claude Code UI

1. In a terminal with Claude Code open, run `/config`
2. Choose the scope (User / Project / Local)
3. Edit the generated JSON — the panel opens the file in your default editor

### Full `settings.json` example for this stack

```json
{
  "env": {
    "CLAUDE_CODE_ENABLE_TELEMETRY": "1",

    "OTEL_EXPORTER_OTLP_PROTOCOL": "http/protobuf",
    "OTEL_EXPORTER_OTLP_ENDPOINT": "http://localhost:4318",

    "OTEL_METRICS_EXPORTER": "otlp",
    "OTEL_METRIC_EXPORT_INTERVAL": "10000",

    "OTEL_LOGS_EXPORTER": "otlp",
    "OTEL_LOGS_EXPORT_INTERVAL": "5000",
    "OTEL_LOG_USER_PROMPTS": "1",
    "OTEL_LOG_TOOL_DETAILS": "1",
    "OTEL_LOG_TOOL_CONTENT": "1",
    "OTEL_LOG_RAW_API_BODIES": "1",

    "CLAUDE_CODE_ENHANCED_TELEMETRY_BETA": "1",
    "OTEL_TRACES_EXPORTER": "otlp",
    "OTEL_TRACES_EXPORT_INTERVAL": "1000"
  }
}
```

> ⚠️ All values in `env` are **strings**, even `"1"` or numbers.

### Corporate distribution via MDM

To enforce telemetry across the organization without users being able to disable it, place the file at:

- **macOS / Linux**: `/etc/claude-code/managed-settings.json`
- **Windows**: `C:\ProgramData\ClaudeCode\managed-settings.json`

Distribute via Jamf, Intune, GPO or similar. Variables in this file have the highest precedence — users cannot override them with shell or personal settings.

---

## Option 2 — Manual variables via shell

For personal use, simply `export` in the shell. Add to `~/.bashrc` or `~/.zshrc` to persist across sessions.

```bash
## ── General ─────────────────────────────────────────────────────────
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318

## ── Metrics ─────────────────────────────────────────────────────────
export OTEL_METRICS_EXPORTER=otlp
export OTEL_METRIC_EXPORT_INTERVAL=10000          # 10s; default 60s

## ── Logs / Events ───────────────────────────────────────────────────
export OTEL_LOGS_EXPORTER=otlp
export OTEL_LOGS_EXPORT_INTERVAL=5000             # default is already 5s
export OTEL_LOG_USER_PROMPTS=1                    # prompt text (sensitive)
export OTEL_LOG_TOOL_DETAILS=1                    # Bash/Read/Edit/Skill/MCP args
export OTEL_LOG_TOOL_CONTENT=1                    # tool input/output (requires tracing)
export OTEL_LOG_RAW_API_BODIES=1                  # Messages API body (very verbose)

## ── Traces (beta) ───────────────────────────────────────────────────
export CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1
export OTEL_TRACES_EXPORTER=otlp
export OTEL_TRACES_EXPORT_INTERVAL=1000
```

Then run `claude` in a **new** shell (or `source ~/.zshrc`).

---

# Full variable reference

## Core variables (required)

| Variable | Valid values | Description |
|---|---|---|
| `CLAUDE_CODE_ENABLE_TELEMETRY` | `1` | **Enables all telemetry.** Without this, nothing is exported. |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | `grpc` / `http/protobuf` / `http/json` | OTLP protocol. For this stack use `http/protobuf` (port 4318) or `grpc` (port 4317). |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `http://localhost:4318` | Collector base endpoint. The SDK appends `/v1/metrics`, `/v1/logs`, `/v1/traces` automatically. |

## Metrics

| Variable | Valid values | Description |
|---|---|---|
| `OTEL_METRICS_EXPORTER` | `otlp` / `prometheus` / `console` / `none` | Metrics exporter type. |
| `OTEL_METRIC_EXPORT_INTERVAL` | ms (default `60000`) | Export interval. Use `10000` (10s) for local debugging. |
| `OTEL_EXPORTER_OTLP_METRICS_PROTOCOL` | same as general | Overrides `OTEL_EXPORTER_OTLP_PROTOCOL` for metrics only. |
| `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` | full URL | Overrides endpoint for metrics only. |
| `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE` | `delta` (default) / `cumulative` | Use `cumulative` if your backend does not support delta. **This stack uses `delta` + `deltatocumulative` in Alloy.** |

### Metrics exported by Claude Code

| Metric | Type | Description |
|---|---|---|
| `claude_code.session.count` | counter | CLI sessions started |
| `claude_code.cost.usage` | counter | Session cost in USD |
| `claude_code.token.usage` | counter | Tokens used (label `type` = `input` / `output` / `cacheRead` / `cacheCreation`) |
| `claude_code.lines_of_code.count` | counter | Lines of code modified |
| `claude_code.commit.count` | counter | Git commits created |
| `claude_code.pull_request.count` | counter | PRs created |
| `claude_code.code_edit_tool.decision` | counter | Edit permission decisions |
| `claude_code.active_time.total` | counter | Total active time in seconds |

## Logs / Events

| Variable | Valid values | Description |
|---|---|---|
| `OTEL_LOGS_EXPORTER` | `otlp` / `console` / `none` | Log exporter type. **Without this no logs are exported.** |
| `OTEL_LOGS_EXPORT_INTERVAL` | ms (default `5000`) | Event export interval. |
| `OTEL_EXPORTER_OTLP_LOGS_PROTOCOL` | same as general | Overrides `OTEL_EXPORTER_OTLP_PROTOCOL` for logs only. |
| `OTEL_EXPORTER_OTLP_LOGS_ENDPOINT` | full URL | Overrides endpoint for logs only. |
| `OTEL_LOG_USER_PROMPTS` | `1` | **Includes prompt text in the `user_prompt` event.** Defaults to `<REDACTED>`. ⚠️ Sensitive. |
| `OTEL_LOG_TOOL_DETAILS` | `1` | Includes tool args: Bash commands, Read/Edit/Write paths, skill/MCP names, parameters. |
| `OTEL_LOG_TOOL_CONTENT` | `1` | Includes full tool input/output in span events. **Requires tracing**. Truncates at 60 KB. |
| `OTEL_LOG_RAW_API_BODIES` | `1` or `file:<dir>` | Emits the full request/response JSON from the Messages API. **Implies consent to all of the above.** Includes the entire conversation history. |

### Events exported (without additional flags)

- `claude_code.user_prompt` — Each user prompt
- `claude_code.api_request` — Each Messages API call (with `cost_usd`, `duration_ms`, tokens)
- `claude_code.api_error` — API errors
- `claude_code.tool_result` — Tool call results (with name, success, duration)
- `claude_code.tool_decision` — Permission decisions
- `claude_code.api_request_body` / `claude_code.api_response_body` — only with `OTEL_LOG_RAW_API_BODIES=1`

## Traces (beta)

| Variable | Valid values | Description |
|---|---|---|
| `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA` | `1` | **Enables distributed tracing.** Without this, spans are not generated even with `OTEL_TRACES_EXPORTER` set. |
| `OTEL_TRACES_EXPORTER` | `otlp` / `console` / `none` | Trace exporter type. |
| `OTEL_TRACES_EXPORT_INTERVAL` | ms (default `5000`) | Export interval. Use `1000` for quick feedback during debugging. |
| `OTEL_EXPORTER_OTLP_TRACES_PROTOCOL` | same as general | Overrides protocol for traces only. |
| `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` | full URL | Overrides endpoint for traces only. |

### Span hierarchy

```
claude_code.interaction (root, 1 per user prompt)
├── claude_code.llm_request (Anthropic Messages API)
└── claude_code.tool (Bash, Read, Edit, Skill, Task, MCP...)
    ├── child span: waiting for permission
    └── child span: execution
```

## Auth & headers

| Variable | Description |
|---|---|
| `OTEL_EXPORTER_OTLP_HEADERS` | HTTP headers in `key1=val1,key2=val2` format. Useful for `Authorization=Bearer ...`. |
| `OTEL_EXPORTER_OTLP_METRICS_CLIENT_KEY` | Path to mTLS key. |
| `OTEL_EXPORTER_OTLP_METRICS_CLIENT_CERTIFICATE` | Path to mTLS certificate. |
| `CLAUDE_CODE_OTEL_HEADERS_HELPER_DEBOUNCE_MS` | Refresh interval for the `otelHeadersHelper` script (default `1740000` = 29 min). |

## Custom attributes

| Variable | Description |
|---|---|
| `OTEL_RESOURCE_ATTRIBUTES` | Adds attributes to ALL signals. E.g.: `department=engineering,team.id=platform,cost_center=eng-123`. |

---

# Validation

After configuring, validate each signal:

| Signal | How to verify |
|---|---|
| Metrics | http://localhost:9009/prometheus → query `claude_code_token_usage_tokens_total` |
| Logs | http://localhost:3000 → Explore → Loki → `{service_name="claude-code"}` |
| Traces | http://localhost:3000 → Explore → Tempo → Search → `service.name="claude-code"` |
| Alloy pipeline | http://localhost:12345 → **Graph** tab shows flow between components |

# ROI Dashboard

Access: **http://localhost:3000/d/claude-roi**

Includes total cost, sessions, tokens, cache hit rate, breakdown by model/user/source, events per minute and integrated log explorer. Multi-select filters by user, model and source.

# Privacy notices

`OTEL_LOG_USER_PROMPTS`, `OTEL_LOG_TOOL_DETAILS`, `OTEL_LOG_TOOL_CONTENT` and `OTEL_LOG_RAW_API_BODIES` send **raw content** (prompts, code, commands, entire conversations) to Loki/Grafana. Appropriate for local debugging — **not** for shared environments without access control on Grafana and Loki.

---

# MCP Grafana (conversational access to Grafana)

The project includes the **official Grafana MCP server** in [.mcp.json](.mcp.json), allowing Claude Code to query dashboards, run PromQL/LogQL and create/edit panels directly via conversation.

## Token setup

1. Log in to Grafana (admin/admin) → sidebar **Administration → Users and access → Service accounts**
2. **Add service account** → name `claude-mcp` → role `Admin`
3. **Add service account token** → copy the token (`glsa_...`)
4. Edit `.mcp.json` replacing the value of `GRAFANA_SERVICE_ACCOUNT_TOKEN`

> ⚠️ Add `.mcp.json` to `.gitignore` if you plan to commit — Admin token has full access.

## Activation

When opening the project for the first time, Claude Code detects `.mcp.json` and asks for permission. Accept. Verify:

```
/mcp
```

Should list `grafana` as `connected`.

## Usage examples

- "List my datasources and dashboards"
- "Show `claude_code_token_usage_tokens_total` metrics grouped by model over the last hour"
- "Search Loki for error logs in the last 24h: `{service_name=\"claude-code\"} |= \"error\"`"
- "List traces in Tempo where `span.prompt.id = X`"
- "Add a panel to the ROI dashboard showing projected monthly cost"

---

# Troubleshooting

## "No data" in the ROI dashboard

**Cause**: Claude Code emits metrics with **AggregationTemporality=Delta**, but Mimir (Prometheus-compatible) only supports **Cumulative**. The `otelcol.processor.deltatocumulative` processor in Alloy converts them. However:

- Each `session_id` creates a Mimir series with a **fixed final cumulative value** (does not grow after the last delta)
- After `max_stale = 5m` without new deltas, the series goes stale
- `rate()` and `increase()` return **0 or nothing** on flat/stale series

**Solution**: the dashboard uses `last_over_time()` for totals and Loki queries (which carry the original deltas in `api_request` events) for time-series charts. See ROI Panel.

## Loki not receiving logs

**Check**:
- `OTEL_LOGS_EXPORTER=otlp` is set (along with `CLAUDE_CODE_ENABLE_TELEMETRY=1`)
- Alloy pipeline: http://localhost:12345 → **Graph** tab should show traffic on the `OTLP → batch → otelcol.exporter.loki → loki.write` path
- The `loki.write` endpoint has the correct path: `http://loki:3100/loki/api/v1/push`

## Tempo not receiving traces

**Check**:
- Both `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1` **AND** `OTEL_TRACES_EXPORTER=otlp` are set
- Alloy has `--stability.level=experimental` in the command (required for `otelcol.exporter.debug`)
- The OTLP exporter in Alloy points to `tempo:4317` (not localhost, since it is inside the docker network)

## Alloy fails to start — `unknown flag --log.level` error

In Alloy, log level is configured via a block in the config, **not** a CLI flag. Use:

```alloy
logging {
  level  = "debug"
  format = "logfmt"
}
```

## Experimental components blocked

Error `component is at stability level "experimental"` → add `--stability.level=experimental` to the Alloy command in [docker-compose.yaml](docker-compose.yaml).

## Metric `claude_code_cost_usage_USD_total` returns 0

The query `sum(increase(claude_code_cost_usage_USD_total[24h]))` may return 0 or nothing because the cumulative goes stale. Use:

```promql
sum(last_over_time(claude_code_cost_usage_USD_total[24h]))
```

Or query the `api_request` events in Loki, which carry `cost_usd` in each delta:

```logql
sum(sum_over_time({service_name="claude-code"} | json | attributes_event_name=`api_request` | unwrap attributes_cost_usd [24h]))
```

## Validate end-to-end flow

```bash
# 1. Is the stack up?
docker compose ps

# 2. Is Alloy receiving OTLP?
docker logs alloy --tail 50 | grep -i "ResourceMetrics\|ResourceLogs\|ResourceSpans"

# 3. Does Mimir have the metrics?
curl -s 'http://localhost:9009/prometheus/api/v1/label/__name__/values' | jq '.data[] | select(startswith("claude"))'

# 4. Does Loki have logs?
curl -s 'http://localhost:3100/loki/api/v1/labels' | jq

# 5. Does Tempo have traces?
curl -s 'http://localhost:3200/api/search?tags=service.name%3Dclaude-code' | jq
```

---

# Pipeline architecture in Alloy

Pipeline configured in [alloy-config/config.alloy](alloy-config/config.alloy):

```
                            ┌─▶ deltatocumulative ─▶ exporter.prometheus ─▶ remote_write ─▶ Mimir
                            │
OTLP receiver ─▶ memory ─▶ batch ─▶ exporter.debug (stdout)
                  limiter         │
                                  ├─▶ exporter.loki ─▶ loki.write ─▶ Loki
                                  │
                                  └─▶ exporter.otlp(tempo:4317) ─▶ Tempo
```

# Log ↔ trace ↔ metric correlation in Grafana

Configured in [grafana-config/provisioning/datasources/datasources.yaml](grafana-config/provisioning/datasources/datasources.yaml):

| Source → Destination | Mechanism |
|---|---|
| **Logs (Loki) → Trace (Tempo)** | `derivedFields` extracts `prompt.id` from JSON and opens Tempo via TraceQL `{ span.prompt.id = "..." }` |
| **Trace (Tempo) → Logs (Loki)** | `tracesToLogsV2` filters Loki by the span's `session.id` (`attributes_session_id`) |
| **Trace (Tempo) → Metrics (Mimir)** | `tracesToMetrics` opens RED metrics (rate/error/duration) via `traces_spanmetrics_*` |
| **Metrics (Mimir) → Trace (Tempo)** | `exemplarTraceIdDestinations` — clickable exemplars on the chart |
| **Service Graph** | Tempo `metrics_generator` generates span-metrics + service-graphs in Mimir |

> **Critical note**: Claude Code **does not propagate `trace_id` in events** — events and spans are independent OTel pipelines. The real correlation key is `session.id` (present in both as `attributes_session_id` in logs and `span.session.id` in traces).
