# How to Put PentAGI to Work

This guide is the "what do I actually do with this thing" companion to [Installing and Configuring PentAGI](installation_configuration.md) and the [README](../../README.md). It covers the usage patterns PentAGI was built for, the knobs that decide how well it performs, and the failure modes worth knowing before your first engagement.

Read this after the stack is up and you have signed in. If the stack is not running yet, start with the [installation guide](installation_configuration.md).

## What PentAGI actually is

A self-hosted multi-agent system for autonomous penetration testing. You describe a target and a goal in plain language; a team of specialized AI agents plans the work, runs 20+ professional tools (nmap, sqlmap, Metasploit, nikto, and friends) inside an isolated Docker sandbox, remembers what it learned, and hands you a report.

Everything happens inside a per-flow container named `pentagi-terminal-<flow-id>`. The backend spawns it, the agents talk to it through the `terminal` tool, and it dies with the flow. Nothing touches your host.

The execution hierarchy is:

| Layer | What it is | Who drives it |
| --- | --- | --- |
| Flow | A complete testing session, persisted | You |
| Task | An objective inside a flow | You or the `primary_agent` |
| Subtask | An auto-decomposed step (max 15 per task) | `generator` / `refiner` |
| Action | One tool call | The specialist agent |

Specialist agents each own one job: `pentester` (exploitation), `coder` (write and fix code), `searcher` (external research), `memorist` (long-term memory), `enricher` (multi-source synthesis), `adviser` (expert guidance), `reflector` (corrects agents that forget to call tools), `reporter` (writes the final report), `installer` (environment setup).

## Usage pattern 1 - Interactive assessment

The fastest way to get value. Open **Flows** → **New Flow**, pick a mode, describe the target, submit.

**Automation** runs the assessment end to end without you. Use it when you have a clear, bounded goal and want the answer in the morning.

**Assistant** is interactive. You talk to the flow, inspect state, redirect it mid-run. Use it when the goal is fuzzy or you want to steer priorities as findings come in.

A good prompt names four things: the target, the type of assessment, the scope boundaries, and the expected output.

```text
Assess https://target.example for common web application vulnerabilities.
Focus on authentication, file handling, and injection issues.
Stay within the provided target only and summarize confirmed findings
with reproduction steps.
```

The stronger the scope statement, the better the result. An agent told "stay within this target" behaves measurably differently from one told "find vulnerabilities" — the latter will wander into whatever it discovers.

Two prompt templates ship in the repository and are worth adapting rather than writing from scratch:

- [`examples/prompts/base_web_pentest.md`](../prompts/base_web_pentest.md) — a Web application checklist covering path traversal, CSRF, XSS, SQL injection, command injection, SSRF, XXE, and unsafe file upload. Each entry tells the agent what to check and how.
- [`examples/prompts/scope_of_work_pentest.md`](../prompts/scope_of_work_pentest.md) — an authorization-scope template with allowed targets, forbidden actions, rate limits, stop conditions, and a scope-check protocol the agent applies before every network action.

The scope template is the more valuable of the two for anything resembling a real engagement. It gives the agent an explicit protocol for deciding whether a host is in bounds, and explicit stop conditions for when to ask a human instead of guessing.

### Steering a running flow

Every flow carries an **Assistant** view that shares the flow's containers and provider settings but keeps its own log tables. This is a real control path, not a background queue:

- `get_flow_status` — query flow state, tasks, subtasks, and recent agent messages
- `wait_flow_completion` — block until the running automation task finishes, or the timeout expires. Returns immediately with an informational message if nothing is running, so confirm the flow state with `get_flow_status` first.
- `stop_flow` — cancel the currently running task
- `submit_flow_input` — answer an `ask` checkpoint, or create a new task
- `patch_flow_subtasks` — rewrite the planned subtask list with add / remove / modify / reorder. Errors if a task is currently running — cancel it first.

Reach for this when the autonomous run finds something interesting and you want to redirect without losing the accumulated context. Restarting a flow throws away the memory of what was already tried.

## Usage pattern 2 - Security team / recurring engagements

The intended audience: security engineers, penetration testers, researchers, red teams testing their own infrastructure.

Save a flow as a **template** to reuse for recurring assessments. The new-flow form has a template picker that prefills the message box, so a weekly external perimeter check or a pre-release validation pass becomes a two-click operation with the scope already written.

**Files** are scoped per flow and live at `{dataDir}/flow-{id}-data/` on the host:

- `uploads/` — files you attach from the UI, pushed into the running container at `/work/uploads/`
- `resources/` — files pulled from your reusable resource library, pushed in at `/work/resources/`
- `container/` — read-only snapshots pulled out of the running agent container

The agent sees uploads and resources automatically, via the `{{.UserFiles}}` template variable in its system prompt. You do not paste file contents into the chat. Container snapshots are visible in the UI but are not injected back into the prompt.

Practical uses: upload a list of endpoints to test, attach a target's source for white-box review, attach credentials for authenticated testing, pull the agent's working directory out to inspect the evidence it gathered.

Two limits worth knowing: uploads cap at 300 MB per file, 1000 files and 2 GB per request; and deleting a flow removes the record and its memory entries but leaves the `flow-{id}-data/` directory on disk — clean it up yourself.

## Usage pattern 3 - CI/CD automation

REST and GraphQL both accept a Bearer token, and the whole flow lifecycle is scriptable.

Generate a token in **Settings → API Tokens**, or over REST at `/api/v1/tokens`. The value is shown exactly once. Each token inherits its owner's role and permissions.

```bash
# List flows
curl https://pentagi.example.com:8443/api/v1/flows \
  -H "Authorization: Bearer $PENTAGI_TOKEN"

# GraphQL
curl -X POST https://pentagi.example.com:8443/api/v1/graphql \
  -H "Authorization: Bearer $PENTAGI_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ flows { id title status } }"}'
```

The REST surface covers the full lifecycle: create a flow, read status, pull tasks and subtasks, list tool calls, terminal logs, agent logs, screenshots, search logs, and vector-store logs, download files, and delete the flow.

Interactive docs:

- Swagger UI at `/api/v1/swagger/index.html`
- GraphQL Playground at `/api/v1/graphql/playground`

For a pipeline, the shape is: create a flow against the deployment target, poll until it reaches a terminal state, fetch the report, and fail the build on the severity threshold you care about. Because a flow is stateful and its results are persisted, a failed run is inspectable rather than lost — you can read the terminal logs and tool calls to see exactly which command did what.

Type-safe clients can be generated from the shipped schemas:

- GraphQL — `backend/pkg/graph/schema.graphqls`, or download it from Settings
- REST — the OpenAPI document in `backend/pkg/server/docs/`

## Usage pattern 4 - Extending the platform

### Adding an LLM provider

Order matters: types before the door, because the door's constructor names them. Copy the last provider added (`mistral` or `xai`) rather than an old one.

1. **Types** — `pkg/providers/provider/provider.go`: the `Provider<Name>` and `DefaultProviderName<Name>` constants, the entry in `AllProviderTypes`, the classification in `ReasoningProvider()`.
2. **Config** — `<NAME>_API_KEY` / `<NAME>_SERVER_URL` / `<NAME>_PROVIDER` in `pkg/config/config.go`, mirrored in `.env.example`, `docker-compose.yml`, and the `clearConfigEnv` list in `config_test.go`. Add every credential to `GetSecretPatterns()` — values listed there are masked out of logs and agent transcripts.
3. **Migration** — a goose migration rebuilding the `PROVIDER_TYPE` enum with the new value and re-pointing `providers.type`, `flows.model_provider_type`, and `assistants.model_provider_type`. Postgres will not extend that enum in place.
4. **Catalogue and door** — `pkg/providers/<name>/` with `config.yml`, `models.yml`, and a `<name>.go` exporting `DefaultProviderConfig`, `BuildProviderConfig`, `DefaultModels`, and `New`. An OpenAI-shaped vendor does not implement the interface by hand — `New` returns `openaicompat.New(openaicompat.Spec{…})`.
5. **Three tables, three edits** — `providerRegistry` (`registry.go`), `bundledCatalogs` (`catalog.go`), `doorEndpoints` (`routing.go`). Miss one and a guard test in `pkg/providers` fails.
6. **Installer** — screen id, menu row, screen construction, and strings under `cmd/installer/wizard/`. Nothing fails the build if you skip it; the wizard simply never offers the provider.
7. **GraphQL** — the `ProviderType` enum value plus fields in `ProvidersModelsList`, `ProvidersReadinessStatus`, and `DefaultProvidersConfig`; re-run gqlgen and extend the per-type switches in `schema.resolvers.go`.
8. **Frontend** — `providerLabels` in `pages/settings/settings-providers.tsx` (a `Record<ProviderType, …>`, so it will not compile without the new key), the icon in `components/icons/` registered in `provider-icon.tsx`, the fields in `frontend/graphql-schema.graphql`, then `pnpm run graphql:generate`, and the e2e cassette in `e2e/mocks/cassettes/settings-providers.ts`.
9. **Docs** — the section every other provider has in `README.md`, `backend/docs/config.md`, and `backend/docs/database.md`. `TestCatalog_ReadmeProviderSectionsMatchTheShippedConfig` in `pkg/providers/catalog_test.go` reads `README.md` — a provider or model missing from it turns that test red.

Embeddings are a separate axis (`EMBEDDING_PROVIDER`, `pkg/providers/embeddings/`). Adding a chat door does not add an embedder.

### Adding a search engine

Engines are primitives under `backend/pkg/tools/searchers/`, orchestrated by the single `web_search` tool. Agents never call an engine directly — they call `web_search` with an intent `mode` (`links`, `answer`, `research`, `exploit`), and the orchestrator picks the engine, retries transient failures, and falls back across providers.

1. `searchers/<name>.go` implementing `searchers.Searcher`: constructor, `IsAvailable()`, `Engine()`, and a `Handle(ctx, Request)` returning **typed** errors (`searchers.Retryable` / `searchers.Fatal` / `searchers.ErrNotConfigured`, or `searchers.ClassifyHTTPStatus`). Never swallow an error into a result string. `searchers` must not import `pkg/tools`.
2. Config fields in `pkg/config/config.go`, plus `.env.example`, `docker-compose.yml`, the `envDefault` assertions in `config_test.go`, and `GetSecretPatterns()` for every credential.
3. Construct it in `buildSearchEngines` and place its id in the `fallbackStrategy` chains in `web_search.go` — that table is the only place per-mode priority lives. An id in a chain with no entry in the engines map is skipped silently at runtime, not at compile time.
4. A new `SearchengineType` needs a goose migration rebuilding the enum, the constant in `pkg/database/models.go`, **and** the value in both the constant block and the `Valid()` switch in `pkg/server/models/searchlogs.go`. Reusing an existing value needs none of it.
5. `<name>_test.go` in `searchers/` — the shared MITM proxy harness lives in `fixtures_test.go`.
6. The installer wizard entry, and `cmd/ftester` if you want to drive the engine in isolation.
7. The prose that enumerates engines: `pkg/templates/prompts/searcher.tmpl` (what the agent is told exists), `backend/docs/config.md`, and `README.md`.

No frontend change is needed — the frontend treats `SearchLog.engine` as an opaque string.

### Customizing prompts

Three levels of increasing effort:

1. **Web UI** — Settings → Prompts. System, human, and tool prompt templates. The quickest loop, no rebuild.
2. **Files** — `backend/pkg/templates/prompts/*.tmpl`, embedded into the binary. One per agent role: `primary_agent`, `pentester`, `coder`, `searcher`, `memorist`, `adviser`, `reflector`, `reporter`, `generator`, `refiner`, and the `question_*` variants for assistant mode.
3. **Provider configs** — `*.provider.yml` files map a model to each agent role. Mount your own and point the matching `*_SERVER_CONFIG_PATH` at it. The `examples/configs/` directory has working examples, including a vLLM setup where the adviser uses the same base model with maximum reasoning.

Editing a prompt file requires a rebuild — the templates are embedded. There is a validator that checks a user-edited prompt still renders (`pkg/templates/validator/`).

## Usage pattern 5 - Fully local / air-gapped inference

Every LLM provider has a local path. Options, in increasing order of effort:

- **Ollama** — `OLLAMA_SERVER_URL` pointing at a local daemon. Simplest local option.
- **Custom OpenAI-compatible endpoint** — `LLM_SERVER_URL` / `LLM_SERVER_KEY` / `LLM_SERVER_MODEL`. Covers vLLM, SGLang, llama.cpp, and LiteLLM proxies. These variables are experimental and may change.
- **vLLM + Qwen3.5-27B-FP8** — the production-grade reference setup, documented in [`examples/guides/vllm-qwen35-27b-fp8.md`](vllm-qwen35-27b-fp8.md). Reported at ~13,000 TPS prompt processing and ~650 TPS completion on 4× RTX 5090, supporting 12+ concurrent flows with no cloud dependency.

### The small-model rule

If you run a model below 32B parameters, turn on both supervision features:

```bash
EXECUTION_MONITOR_ENABLED=true
AGENT_PLANNING_STEP_ENABLED=true
```

Measured on Qwen3.5-27B-FP8, both together produce roughly **2× better results** than the baseline, because they attack the specific way small models fail.

- **Execution monitoring** detects loops (identical tool calls, default threshold 5; total tool calls, default threshold 10) and invokes an adviser agent to recommend a different strategy. Tune with `EXECUTION_MONITOR_SAME_TOOL_LIMIT` and `EXECUTION_MONITOR_TOTAL_TOOL_LIMIT`.
- **Intelligent task planning** has a planner agent decompose the request into 3–7 concrete steps before specialists start, which prevents scope creep and redundant work.

The cost is 2–3× more tokens and 2–3× more wall-clock time. That trade is worth making on small models and is usually not worth making on frontier models that plan well already.

One high-leverage refinement: give the **adviser** agent a stronger configuration than the rest. The bundled `vllm-qwen3.5-27b-fp8.provider.yml` does exactly this — same base model, maximum reasoning mode for the adviser — and it makes the planner's output substantially better without adding a second model to the fleet.

Tool call limits are always active and cap runaway execution: `MAX_GENERAL_AGENT_TOOL_CALLS` (default 100) for the primary, pentester, coder, and installer agents, `MAX_LIMITED_AGENT_TOOL_CALLS` (default 20) for searcher, enricher, memorist, reporter, adviser, reflector, and planner. A reflector agent guides the others to terminate cleanly as they approach the limit.

### Choosing a provider

The bundled catalogues list per-role model assignments. MiniMax, for example, ships `MiniMax-M3` (~1M context) as the catalogue default and uses `MiniMax-M2.7` for roles where conformance testing showed more reliable context retention:

```bash
MINIMAX_API_KEY=your_minimax_api_key
MINIMAX_SERVER_URL=https://api.minimax.io/v1
```

For a LiteLLM proxy in front of it, set `MINIMAX_PROVIDER=minimax` to enable model-name prefixing.

The full provider list is `providerRegistry` in `backend/pkg/providers/registry.go` — check it before enumerating providers anywhere else.

## Usage pattern 6 - Production deployment

A single node is fine for development. For production, especially security-sensitive environments, run the **two-node architecture**: the main node keeps the UI, API, and database; a separate worker node runs the sandbox containers.

What this buys:

- **Isolated execution** — worker containers run on dedicated hardware
- **Network isolation** — separate network boundaries for the testing traffic
- **Security boundary** — Docker-in-Docker with TLS authentication
- **Out-of-band attack support** — dedicated port ranges for callback techniques

Full walkthrough in [`examples/guides/worker_node.md`](worker_node.md).

### The Docker socket trap

Many pentest workflows need `docker` inside the agent sandbox. There are two ways to provide it, and they differ sharply in risk.

**Recommended — point sandboxes at a hardened dind daemon over TLS:**

```bash
DOCKER_INSIDE=true
DOCKER_SOCKET=                                          # mount no socket
DOCKER_INSIDE_HOST=tcp://10.0.0.5:3376                  # hardened dind endpoint
DOCKER_INSIDE_TLS_VERIFY=1
DOCKER_INSIDE_CERT_PATH=/etc/docker/dind/certs/client
DOCKER_INSIDE_POLICY_TESTS=true                         # prove it on every worker
```

PentAGI injects these into every worker container as `DOCKER_HOST`, `DOCKER_TLS_VERIFY`, and `DOCKER_CERT_PATH` — the `_INSIDE_` segment is dropped — and bind-mounts the certificate directory read-only at the same path.

**Not recommended — bind-mounting a Docker socket (`DOCKER_SOCKET`).** Two failure modes:

- **Boot-order race** — a bind-mount source that does not exist yet is created by Docker as a *directory*. After a worker-node reboot, a worker container can start before dind has recreated its socket; Docker then puts a directory where the socket belongs, and dind cannot start until it is removed by hand.
- **Blast radius** — the race is only reliably avoided when the mounted socket is the *host* daemon's. But that grants an autonomous agent the host Docker API: it can start a privileged container, mount `/`, and compromise the entire node, PentAGI included.

Use `DOCKER_SOCKET` only on single-node development setups where the host daemon is already trusted.

### Multiple instances on shared infrastructure

If several PentAGI installations share a PostgreSQL server, a worker node, a Neo4j instance, or a Langfuse, set `TENANT_ID` on each. Every instance numbers its flows from `1`, so without a tenant they collide on container names, database rows, knowledge-graph namespaces, and session cookies.

```bash
TENANT_ID=acme   # must match ^[a-z][a-z0-9_]{0,31}$; an invalid value aborts startup
```

| Area | Effect when `TENANT_ID=acme` |
| --- | --- |
| PostgreSQL | Works inside schema `acme` instead of `public` |
| Worker containers | `acme-pentagi-terminal-<flow>` instead of `pentagi-terminal-<flow>` |
| Knowledge graph | Graphiti/Neo4j group ids become `acme-flow-<id>` |
| Auth | Cookie and API token keys derive from `COOKIE_SIGNING_SALT` **plus** the tenant; the session cookie is renamed |
| Telemetry | Langfuse traces carry the tenant as `environment` and a `tenant:acme` tag |

Some things stay per-instance because they are host resources rather than names: `DATA_DIR` (two instances sharing it will overwrite each other's flow data), `DOCKER_PORTS_BASE`, the published ports, and `INSTALLATION_ID`. Effective values print at startup under *Instance identity*.

### Ports and outbound access

Two separate concerns:

- The **UI** binds to `127.0.0.1` by default. Set `PENTAGI_LISTEN_IP=0.0.0.0` and update `PUBLIC_URL` and `CORS_ORIGINS` to reach it externally. Use the real hostname or IP in `PUBLIC_URL`, never `0.0.0.0`.
- **Flow sandboxes are not bound to localhost.** Each flow publishes two TCP ports for out-of-band callbacks such as reverse shells, from a 2000-port window starting at `DOCKER_PORTS_BASE` (`28000`–`29999` by default), on `DOCKER_PUBLIC_IP` — `0.0.0.0` by default. Docker routes published ports around `ufw` and `firewalld` rules, so restrict this range with a firewall in front of the host, or with rules in the `DOCKER-USER` chain under the default iptables backend. With `DOCKER_NETWORK=host`, sandbox listeners are exposed without any port mapping at all.

This is the most common way to accidentally expose a testing machine to the internet. Check it before the first assessment.

### Proxy behavior

`PROXY_URL` routes the backend's own HTTP calls through a proxy: every request to LLM and embedding providers, to search engines — including self-hosted Ollama and SearXNG, as there is no `NO_PROXY` exemption — and the update check. Sandbox containers, the scraper, the Graphiti service, OAuth sign-in, and Langfuse/OpenTelemetry export do **not** use it. For air-gapped isolation, restrict outbound traffic at the network level.

## Observability

Three optional stacks, each brought up with an additional compose file:

```bash
docker compose -f docker-compose.yml -f docker-compose-observability.yml up -d  # Grafana stack
docker compose -f docker-compose.yml -f docker-compose-langfuse.yml up -d       # LLM analytics
docker compose -f docker-compose.yml -f docker-compose-graphiti.yml up -d       # knowledge graph
```

Run `docker-compose.yml` first so the shared networks exist.

- **Observability** — OpenTelemetry to VictoriaMetrics (metrics), Loki (logs), Jaeger (traces), with Grafana dashboards on top.
- **Langfuse** — LLM-specific analytics: prompt costs, latency, per-agent behavior. The fastest way to find out which agent is burning your tokens.
- **Graphiti + Neo4j** — a knowledge graph of hosts, services, and findings, giving the agents semantic relationships across flows. Worth enabling when you run many related assessments and want the system to carry context between them.

## Security checklist before real work

- [ ] Default password changed (`admin@pentagi.com` / `admin` → something strong)
- [ ] `COOKIE_SIGNING_SALT` set to a random value
- [ ] `PENTAGI_POSTGRES_PASSWORD` and `NEO4J_PASSWORD` changed from defaults
- [ ] UI still bound to localhost, or `PUBLIC_URL` / `CORS_ORIGINS` set to real values and a firewall in place
- [ ] `DOCKER_PORTS_BASE` range firewalled, or `DOCKER_PUBLIC_IP` narrowed
- [ ] `DOCKER_SOCKET` **not** mounted into sandboxes on anything reachable from a network
- [ ] Worker node separated from the main node for production engagements
- [ ] Every target in scope documented and authorized in writing

## What PentAGI is not

Worth being explicit, because the marketing shape invites the wrong expectations:

- It is **not** a CALDERA-style Breach and Attack Simulation product. There are no predefined campaigns or attack plans to replay. BAS-like agent-authored attack scripts are conceptual work, not a shipped feature.
- It is **not** a replacement for human judgment on scope, authorization, or severity. The report is a well-researched starting point, not a compliance artifact.
- Flow report export is web view, clipboard, Markdown download, and PDF download. JSON export is not a supported output format today.

## Troubleshooting

**The provider answers but tool calls fail.** Run `ctester` — it runs the conformance suite against one provider and reports which agent roles and groups fail.

```bash
docker exec -it pentagi /opt/pentagi/bin/ctester -verbose
docker exec -it pentagi /opt/pentagi/bin/ctester -verbose -type custom
```

Tool-call failures on a custom, llama.cpp, vLLM, or SGLang backend usually point at the backend's tool-call parser, not at PentAGI.

**Memory search returns nothing.** Check the embedding provider separately with `etester`. If you changed the embedding provider or model after the fact, flush and reindex the knowledge base — `etester flush` and `etester reindex` are destructive and act on the real database.

**The agent loops on one approach.** You are on a small model without supervision enabled. Turn on `EXECUTION_MONITOR_ENABLED` and `AGENT_PLANNING_STEP_ENABLED`, and give the adviser a stronger model configuration.

**The agent wanders outside the target.** The prompt lacked scope boundaries. Use the scope-of-work template and name allowed targets explicitly.

**Container will not start after a worker reboot.** You almost certainly have the Docker socket race. A directory exists where the socket should be; remove it by hand and switch to the dind-over-TLS configuration.

**Tool calls stop mid-flow.** The agent hit its tool call limit. Check `MAX_GENERAL_AGENT_TOOL_CALLS` and `MAX_LIMITED_AGENT_TOOL_CALLS` — the reflector agent should have guided a clean termination, but a limit reached mid-task means the task was larger than the budget.

## Where to go next

- [Installing and Configuring PentAGI](installation_configuration.md) — the deployment walkthrough
- [Worker Node Setup](worker_node.md) — distributed production architecture
- [vLLM + Qwen3.5-27B-FP8](vllm-qwen35-27b-fp8.md) — production local inference
- [OpenVAS custom image](openvas-custom-image.md) — adding a vulnerability scanner image
- [`examples/prompts/`](../prompts/) — prompt templates to adapt
- [`examples/reports/`](../reports/) — sample reports from real runs
- [`examples/tests/`](../tests/) — provider conformance reports
- [`examples/proposals/`](../proposals/) — design proposals for MCP, Kubernetes, OSINT, evidence chains
- [EULA](../../EULA.md) — acceptable use requirements
