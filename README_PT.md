[English](README.md)

---

# Claude Code — Stack de Monitoramento

> [English version](README.md)

> Essa documentação foi gerada com IA

Stack completa de observabilidade para o Claude Code (métricas, logs/eventos e traces) baseado em **Grafana Alloy** (coletor OpenTelemetry), **Grafana Mimir** (métricas), **Loki**, **Tempo** e **Grafana** — stack **LGTM** completa.

```
┌────────────┐    OTLP     ┌──────────┐    ┌──────────────┐    ┌─────────┐
│ Claude Code│ ──────────▶ │  Alloy   │ ─▶ │ Mimir        │ ─▶ │ Grafana │
│  (CLI/SDK) │  4317/4318  │ (OTel    │ ─▶ │ Loki         │ ─▶ │  3000   │
└────────────┘             │ Collector)│ ─▶ │ Tempo        │    └─────────┘
                           └──────────┘    └──────────────┘
```

## Componentes

| Serviço | Porta | Função |
|---|---|---|
| **Alloy** | 4317 (gRPC), 4318 (HTTP), 12345 (UI) | Recebe OTLP do Claude e roteia para Mimir/Loki/Tempo |
| **Mimir** | 9009 | Métricas (cost, tokens, sessions) — API compatível com Prometheus (PromQL) |
| **Loki** | 3100 | Logs/eventos (`api_request`, `user_prompt`, `tool_result`...) |
| **Tempo** | 3200 | Distributed traces (spans `claude_code.interaction`, etc.) |
| **Grafana** | 3000 | Dashboards e correlação log↔trace↔métrica (admin/admin) |

## Subir a stack

```bash
docker compose up -d
```

Acesse o Grafana em http://localhost:3000 (login `admin` / `admin`). O dashboard **Claude Code — ROI & Usage** já vem provisionado.

---

# Configurando o Claude Code

Existem **duas formas** de configurar as variáveis OpenTelemetry no Claude Code. A forma recomendada para uso pessoal/desenvolvimento é via **shell** (rápido, isolado por sessão). A recomendada para uso corporativo é via **`settings.json` gerenciado** (centralizado, sobrepõe variáveis do shell).

## Opção 1 — `settings.json` (configuração persistente / admin)

O Claude Code lê configurações de vários arquivos `settings.json` em ordem de precedência. Variáveis em `env` desses arquivos **sobrepõem variáveis do shell** quando o arquivo é gerenciado.

### Locais dos arquivos (ordem de precedência, do menor ao maior)

| Escopo | Caminho | Quem edita |
|---|---|---|
| Usuário | `~/.claude/settings.json` | Você (pessoal, máquina inteira) |
| Projeto compartilhado | `<repo>/.claude/settings.json` | Time (versionado no git) |
| Projeto local | `<repo>/.claude/settings.local.json` | Você (não versionado) |
| **Gerenciado (admin)** | `/etc/claude-code/managed-settings.json` (Linux/macOS) ou `C:\ProgramData\ClaudeCode\managed-settings.json` (Windows) | Administrador via MDM — **prevalece sobre tudo** |

### Configuração via UI do Claude Code

1. No terminal com Claude Code aberto, rode `/config`
2. Escolha o escopo (User / Project / Local)
3. Edite o JSON gerado — o painel abre o arquivo no editor padrão

### Exemplo completo de `settings.json` para esta stack

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

> ⚠️ Todos os valores em `env` são **strings**, mesmo `"1"` ou números.

### Distribuição corporativa via MDM

Para forçar telemetria em toda a organização sem que usuários possam desabilitar, coloque o arquivo em:

- **macOS / Linux**: `/etc/claude-code/managed-settings.json`
- **Windows**: `C:\ProgramData\ClaudeCode\managed-settings.json`

Distribua via Jamf, Intune, GPO ou similar. Variáveis nesse arquivo têm precedência máxima — o usuário não consegue sobrepor com shell ou settings pessoais.

---

## Opção 2 — Variáveis manuais via shell

Para uso pessoal, basta `export` no shell. Cole no `~/.bashrc` ou `~/.zshrc` para persistir entre sessões.

```bash
## ── Geral ───────────────────────────────────────────────────────────
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318

## ── Métricas ────────────────────────────────────────────────────────
export OTEL_METRICS_EXPORTER=otlp
export OTEL_METRIC_EXPORT_INTERVAL=10000          # 10s; default 60s

## ── Logs / Eventos ──────────────────────────────────────────────────
export OTEL_LOGS_EXPORTER=otlp
export OTEL_LOGS_EXPORT_INTERVAL=5000             # default já é 5s
export OTEL_LOG_USER_PROMPTS=1                    # texto dos prompts (sensível)
export OTEL_LOG_TOOL_DETAILS=1                    # args de Bash/Read/Edit/Skill/MCP
export OTEL_LOG_TOOL_CONTENT=1                    # input/output de tools (requer tracing)
export OTEL_LOG_RAW_API_BODIES=1                  # body da Messages API (muito verboso)

## ── Traces (beta) ───────────────────────────────────────────────────
export CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1
export OTEL_TRACES_EXPORTER=otlp
export OTEL_TRACES_EXPORT_INTERVAL=1000
```

Depois rode `claude` em um shell **novo** (ou `source ~/.zshrc`).

---

# Referência completa de variáveis

## Variáveis core (obrigatórias)

| Variável | Valores válidos | Descrição |
|---|---|---|
| `CLAUDE_CODE_ENABLE_TELEMETRY` | `1` | **Liga toda a telemetria.** Sem isso, nada é exportado. |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | `grpc` / `http/protobuf` / `http/json` | Protocolo OTLP. Para esta stack use `http/protobuf` (porta 4318) ou `grpc` (porta 4317). |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `http://localhost:4318` | Endpoint base do coletor. O SDK anexa `/v1/metrics`, `/v1/logs`, `/v1/traces` automaticamente. |

## Métricas

| Variável | Valores válidos | Descrição |
|---|---|---|
| `OTEL_METRICS_EXPORTER` | `otlp` / `prometheus` / `console` / `none` | Tipo do exporter de métricas. |
| `OTEL_METRIC_EXPORT_INTERVAL` | ms (default `60000`) | Intervalo de envio. Para debug local use `10000` (10s). |
| `OTEL_EXPORTER_OTLP_METRICS_PROTOCOL` | mesmos valores do geral | Sobrepõe `OTEL_EXPORTER_OTLP_PROTOCOL` só para métricas. |
| `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` | URL completa | Sobrepõe endpoint só para métricas. |
| `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE` | `delta` (default) / `cumulative` | Use `cumulative` se seu backend não suporta delta. **Esta stack usa `delta` + `deltatocumulative` no Alloy.** |

### Métricas exportadas pelo Claude Code

| Métrica | Tipo | Descrição |
|---|---|---|
| `claude_code.session.count` | counter | Sessões CLI iniciadas |
| `claude_code.cost.usage` | counter | Custo em USD da sessão |
| `claude_code.token.usage` | counter | Tokens usados (label `type` = `input` / `output` / `cacheRead` / `cacheCreation`) |
| `claude_code.lines_of_code.count` | counter | Linhas de código modificadas |
| `claude_code.commit.count` | counter | Commits git criados |
| `claude_code.pull_request.count` | counter | PRs criados |
| `claude_code.code_edit_tool.decision` | counter | Decisões de permissão de edit |
| `claude_code.active_time.total` | counter | Tempo ativo total em segundos |

## Logs / Eventos

| Variável | Valores válidos | Descrição |
|---|---|---|
| `OTEL_LOGS_EXPORTER` | `otlp` / `console` / `none` | Tipo do exporter de logs. **Sem isso nada de log é exportado.** |
| `OTEL_LOGS_EXPORT_INTERVAL` | ms (default `5000`) | Intervalo de envio dos eventos. |
| `OTEL_EXPORTER_OTLP_LOGS_PROTOCOL` | mesmos valores do geral | Sobrepõe `OTEL_EXPORTER_OTLP_PROTOCOL` só para logs. |
| `OTEL_EXPORTER_OTLP_LOGS_ENDPOINT` | URL completa | Sobrepõe endpoint só para logs. |
| `OTEL_LOG_USER_PROMPTS` | `1` | **Inclui texto dos prompts no evento `user_prompt`.** Por padrão vai como `<REDACTED>`. ⚠️ Sensível. |
| `OTEL_LOG_TOOL_DETAILS` | `1` | Inclui args de tools: comandos Bash, paths Read/Edit/Write, nomes de skills/MCP, parâmetros. |
| `OTEL_LOG_TOOL_CONTENT` | `1` | Inclui input/output completo das tools nos span events. **Requer tracing**. Trunca em 60 KB. |
| `OTEL_LOG_RAW_API_BODIES` | `1` ou `file:<dir>` | Emite o JSON completo de request/response da Messages API. **Implica consentimento de tudo acima.** Inclui histórico inteiro da conversa. |

### Eventos exportados (sem flags adicionais)

- `claude_code.user_prompt` — Cada prompt do usuário
- `claude_code.api_request` — Cada chamada à Messages API (com `cost_usd`, `duration_ms`, tokens)
- `claude_code.api_error` — Erros da API
- `claude_code.tool_result` — Resultado de tool calls (com nome, sucesso, duration)
- `claude_code.tool_decision` — Decisões de permissão
- `claude_code.api_request_body` / `claude_code.api_response_body` — apenas com `OTEL_LOG_RAW_API_BODIES=1`

## Traces (beta)

| Variável | Valores válidos | Descrição |
|---|---|---|
| `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA` | `1` | **Liga distributed tracing.** Sem isso, spans não são gerados mesmo com `OTEL_TRACES_EXPORTER` setado. |
| `OTEL_TRACES_EXPORTER` | `otlp` / `console` / `none` | Tipo do exporter de traces. |
| `OTEL_TRACES_EXPORT_INTERVAL` | ms (default `5000`) | Intervalo de envio. Use `1000` para feedback rápido em debug. |
| `OTEL_EXPORTER_OTLP_TRACES_PROTOCOL` | mesmos valores do geral | Sobrepõe protocolo só para traces. |
| `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` | URL completa | Sobrepõe endpoint só para traces. |

### Hierarquia de spans

```
claude_code.interaction (root, 1 por user prompt)
├── claude_code.llm_request (Anthropic Messages API)
└── claude_code.tool (Bash, Read, Edit, Skill, Task, MCP...)
    ├── span filho: aguardando permissão
    └── span filho: execução
```

## Auth & headers

| Variável | Descrição |
|---|---|
| `OTEL_EXPORTER_OTLP_HEADERS` | Headers HTTP no formato `key1=val1,key2=val2`. Útil para `Authorization=Bearer ...`. |
| `OTEL_EXPORTER_OTLP_METRICS_CLIENT_KEY` | Path para chave mTLS. |
| `OTEL_EXPORTER_OTLP_METRICS_CLIENT_CERTIFICATE` | Path para cert mTLS. |
| `CLAUDE_CODE_OTEL_HEADERS_HELPER_DEBOUNCE_MS` | Intervalo de refresh do script `otelHeadersHelper` (default `1740000` = 29 min). |

## Atributos customizados

| Variável | Descrição |
|---|---|
| `OTEL_RESOURCE_ATTRIBUTES` | Adiciona atributos a TODOS os sinais. Ex: `department=engineering,team.id=platform,cost_center=eng-123`. |

---

# Validação

Após configurar, valide cada sinal:

| Sinal | Como verificar |
|---|---|
| Métricas | http://localhost:9009/prometheus → query `claude_code_token_usage_tokens_total` |
| Logs | http://localhost:3000 → Explore → Loki → `{service_name="claude-code"}` |
| Traces | http://localhost:3000 → Explore → Tempo → Search → `service.name="claude-code"` |
| Pipeline Alloy | http://localhost:12345 → aba **Graph** mostra fluxo entre componentes |

# Dashboard ROI

Acesse: **http://localhost:3000/d/claude-roi**

Inclui custo total, sessões, tokens, cache hit rate, breakdown por modelo/usuário/source, eventos por minuto e log explorer integrado. Filtros multi-select por usuário, modelo e source.

# Avisos de privacidade

`OTEL_LOG_USER_PROMPTS`, `OTEL_LOG_TOOL_DETAILS`, `OTEL_LOG_TOOL_CONTENT` e `OTEL_LOG_RAW_API_BODIES` enviam **conteúdo bruto** (prompts, código, comandos, conversas inteiras) ao Loki/Grafana. Apropriado para debug local — **não** para ambientes compartilhados sem controle de acesso ao Grafana e ao Loki.

---

# MCP Grafana (acesso conversacional ao Grafana)

O projeto inclui o **MCP server oficial do Grafana** em [.mcp.json](.mcp.json), permitindo que o Claude Code consulte dashboards, execute PromQL/LogQL e crie/edite painéis diretamente via conversa.

## Setup do token

1. Login no Grafana (admin/admin) → menu lateral **Administration → Users and access → Service accounts**
2. **Add service account** → nome `claude-mcp` → role `Admin`
3. **Add service account token** → copie o token (`glsa_...`)
4. Edite `.mcp.json` substituindo o valor de `GRAFANA_SERVICE_ACCOUNT_TOKEN`

> ⚠️ Adicione `.mcp.json` ao `.gitignore` se for commitar — token Admin tem acesso completo.

## Ativação

Ao abrir o projeto pela primeira vez, o Claude Code detecta `.mcp.json` e pede permissão. Aceite. Verifique:

```
/mcp
```

Deve listar `grafana` como `connected`.

## Exemplos de uso

- "Liste meus datasources e dashboards"
- "Mostre as métricas `claude_code_token_usage_tokens_total` agrupadas por modelo na última hora"
- "Busque no Loki por logs de erro nas últimas 24h: `{service_name=\"claude-code\"} |= \"error\"`"
- "Liste os traces no Tempo onde `span.prompt.id = X`"
- "Adicione um painel ao dashboard ROI mostrando custo projetado mensal"

---

# Troubleshooting

## "No data" no dashboard ROI

**Causa**: O Claude Code emite métricas com **AggregationTemporality=Delta**, mas o Mimir (compatível com Prometheus) só aceita **Cumulative**. O processor `otelcol.processor.deltatocumulative` no Alloy converte. Porém:

- Cada `session_id` cria uma série no Mimir com **valor cumulativo final fixo** (não cresce após o último delta)
- Após `max_stale = 5m` sem novos deltas, a série fica stale
- `rate()` e `increase()` retornam **0 ou nada** em séries flat/stale

**Solução**: o dashboard usa `last_over_time()` para totais e queries Loki (que têm os deltas originais nos eventos `api_request`) para gráficos temporais. Ver Painel ROI.

## Loki não recebe logs

**Verifique**:
- `OTEL_LOGS_EXPORTER=otlp` está setado (junto com `CLAUDE_CODE_ENABLE_TELEMETRY=1`)
- Pipeline do Alloy: http://localhost:12345 → aba **Graph** deve mostrar tráfego no caminho `OTLP → batch → otelcol.exporter.loki → loki.write`
- Endpoint `loki.write` está com path correto: `http://loki:3100/loki/api/v1/push`

## Tempo não recebe traces

**Verifique**:
- Ambos `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1` **E** `OTEL_TRACES_EXPORTER=otlp` estão setados
- O Alloy tem `--stability.level=experimental` no command (necessário para `otelcol.exporter.debug`)
- O exporter OTLP no Alloy aponta para `tempo:4317` (não localhost, pois é dentro da rede docker)

## Alloy não inicia — erro `unknown flag --log.level`

No Alloy o nível de log é configurado via bloco no config, **não** flag CLI. Use:

```alloy
logging {
  level  = "debug"
  format = "logfmt"
}
```

## Componentes experimentais bloqueados

Erro `component is at stability level "experimental"` → adicione `--stability.level=experimental` ao command do Alloy no [docker-compose.yaml](docker-compose.yaml).

## Métrica `claude_code_cost_usage_USD_total` retorna 0

A query `sum(increase(claude_code_cost_usage_USD_total[24h]))` pode retornar 0 ou nada porque o cumulativo fica stale. Use:

```promql
sum(last_over_time(claude_code_cost_usage_USD_total[24h]))
```

Ou consulte os eventos `api_request` no Loki, que carregam `cost_usd` em cada delta:

```logql
sum(sum_over_time({service_name="claude-code"} | json | attributes_event_name=`api_request` | unwrap attributes_cost_usd [24h]))
```

## Validar fluxo end-to-end

```bash
# 1. Stack está up?
docker compose ps

# 2. Alloy está recebendo OTLP?
docker logs alloy --tail 50 | grep -i "ResourceMetrics\|ResourceLogs\|ResourceSpans"

# 3. Mimir tem as métricas?
curl -s 'http://localhost:9009/prometheus/api/v1/label/__name__/values' | jq '.data[] | select(startswith("claude"))'

# 4. Loki tem logs?
curl -s 'http://localhost:3100/loki/api/v1/labels' | jq

# 5. Tempo tem traces?
curl -s 'http://localhost:3200/api/search?tags=service.name%3Dclaude-code' | jq
```

---

# Arquitetura de pipelines no Alloy

Pipeline configurado em [alloy-config/config.alloy](alloy-config/config.alloy):

```
                            ┌─▶ deltatocumulative ─▶ exporter.prometheus ─▶ remote_write ─▶ Mimir
                            │
OTLP receiver ─▶ memory ─▶ batch ─▶ exporter.debug (stdout)
                  limiter         │
                                  ├─▶ exporter.loki ─▶ loki.write ─▶ Loki
                                  │
                                  └─▶ exporter.otlp(tempo:4317) ─▶ Tempo
```

# Correlação log ↔ trace ↔ métrica no Grafana

Configurada em [grafana-config/provisioning/datasources/datasources.yaml](grafana-config/provisioning/datasources/datasources.yaml):

| Origem → Destino | Mecanismo |
|---|---|
| **Logs (Loki) → Trace (Tempo)** | `derivedFields` extrai `prompt.id` do JSON e abre Tempo via TraceQL `{ span.prompt.id = "..." }` |
| **Trace (Tempo) → Logs (Loki)** | `tracesToLogsV2` filtra Loki por `session.id` do span (`attributes_session_id`) |
| **Trace (Tempo) → Metrics (Mimir)** | `tracesToMetrics` abre RED metrics (rate/error/duration) via `traces_spanmetrics_*` |
| **Metrics (Mimir) → Trace (Tempo)** | `exemplarTraceIdDestinations` — exemplars clicáveis no gráfico |
| **Service Graph** | Tempo `metrics_generator` gera span-metrics + service-graphs no Mimir |

> **Nota crítica**: Claude Code **não propaga `trace_id` nos events** — events e spans são pipelines OTel independentes. A chave de correlação real é `session.id` (presente em ambos como `attributes_session_id` nos logs e `span.session.id` nos traces).

