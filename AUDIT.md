# OpenBot Full Codebase Audit

**Repository:** https://github.com/YashwanthGathuku/OpenBot  
**Upstream lineage:** CopilotKit/openbot (MIT, alpha template)  
**Tree SHA audited:** `9c6a2370cb02aa94eaea67a8e7cedf0f2884f42c`  
**Audit date:** 2026-09-21  
**Auditor:** GitHub (Grok Bot assistant) via GitHub API reads (no local clone)

---

## 0. How to read this document

This is a **working understanding** of OpenBot after a systematic, package-by-package deep read of the repository: entry points, schemas, gateways, agent harnesses, compose/Helm, and the critical security paths. It records:

- what was studied and how
- what I understand the system to *be*
- my own judgement of design quality and risks
- tree structure, data models, algorithms, and key methods/symbols
- concrete findings ranked by severity

**Honesty about coverage:** a literal “every line of every file” dump is neither useful nor what happened. The monorepo is large (`plugins/store.ts` alone is ~7k LOC; Composio adapter ~5k; ~191 server tests; 15 agent packages). What *did* happen is a full inventory of every top-level package, deep reads of every governance-critical module, harness contracts, supervisor/computer/worker paths, and synthesis of how they connect. Where a file was only sampled (e.g. individual React leaf components, every Helm template field), that is noted.

---

## 1. My understanding — what OpenBot actually is

OpenBot is not “another chatbot UI.” It is a **self-hosted agent governance platform**:

1. People talk to **coworkers** (durable Bot profiles with standing roles).
2. Coworkers are AG-UI endpoints (built-in or remote frameworks).
3. Anything a Bot does to a browser, file, shell, MCP server, or component must pass a **server-side gateway**: resolve target → evaluate policy → write audit → then act (or refuse with the rule name).
4. Optionally, each Bot gets its **own computer** (Chromium + `/workspace` + profile) created by a **supervisor** that alone holds the Docker socket.
5. Threads/memory live in **CopilotKit Intelligence**; product data/policy/audit live in **PostgreSQL**.

The product thesis (and I think it holds): *the difference between an agent that can use your tools and an agent you can let near them is one gate that decides and records before anything happens.*

It is deliberately a template to fork, not a SaaS. Tenant YAML under `examples/` is the white-label surface. Alpha status is honest — expect movement.

### What I think about the design

**Strong:**
- Fail-closed defaults (empty policy permits nothing; broken deny denies; production refuses placeholder encryption keys and private-host browsing).
- Clear security story in code comments (compose network split so Bot shells cannot DNS-resolve Postgres; loopback publishes; CapDrop on computers).
- Durable multi-replica work (`work_items` + `FOR UPDATE SKIP LOCKED`) for routines, handoffs, cullers — correct shape for horizontal scale.
- Separation of **role** (instruction) from **capability** (grants + policy). YAML coworkers name skills but grant nothing.
- Run assertions binding tool callbacks to bot identity — stops a stolen token from impersonating another Bot’s run (when used with per-agent tokens).

**Weak / sharp:**
- Ops footguns (`OPENBOT_SINGLE_USER`, well-known Compose tokens) are documented but still easy to copy into a bad deploy.
- Two tool-loop models (`agent-bot` ends for the surface vs LangGraph/harness calling `/api/agent-tools/call`) make “managed Bot” behaviour non-uniform.
- Plugin/Composio surface is huge — well tested, but the blast radius of a `callTool` regression is high.
- Groups ACL is scaffolded in schema (`users.groups`, `channels.allowed_groups`) but not enforced; membership only. Package channels without membership rows are unreachable (safe, incomplete).

**Overall judgement:** this is a serious engineering effort at the governance layer, not a thin wrapper around an LLM SDK. For a company that wants to own the assistant stack, the architecture is coherent. The residual risk is mostly **deployment discipline** and **complexity concentration** in plugins, not absence of controls.

---

## 2. Methodology — what was studied

| Pass | Scope | Method |
|------|-------|--------|
| Architecture | README, `docs/architecture.md`, `docker-compose.yml`, ports | Full read |
| Coworker model | `docs/coworkers.md`, `docs/configuration.md`, `examples/fintech/agents.yaml`, `agents/*.yaml` | Full read |
| Inventory | Root tree + package manifests | `get_repository_tree` / directory listings |
| Server deep dive | `server/src/**` modules, drizzle schemas, tests shape | File reads + large-file sampling |
| Runtime deep dive | `app`, all `agent-*`, `supervisor`, `agent-computer`, `worker`, `desktop`, `shared`, `charts`, `scripts` | Entry points + security-critical modules |
| Synthesis | Cross-cutting flows + ranked findings | This document |

**Not fully line-walked:** every React component body, every Helm template line, every Python harness beyond main/middleware/tool runtime, full 1.3k-line computer `index.ts` handler table (auth/control/shell/egress were read in depth).

---

## 3. Comprehensive repository tree

```
OpenBot/
├── .claude/                          # Claude-oriented notes
├── .github/workflows/                # ci, desktop, release, zizmor security
├── .env.example                      # ~23KB — required + optional secrets
├── .dockerignore, .gitattributes, .gitignore
├── CHANGELOG.md                      # large release history
├── Dockerfile                        # all-in-one image (app+API+optional embedded PG)
├── LICENSE                           # MIT
├── README.md
├── Tauri-signing-SKILL.md
├── package.json                      # workspaces: app, server, worker; Bun 1.3+
├── bun.lock, bunfig.toml, biome.json, tsconfig.base.json
├── docker-compose.yml                # postgres, migrate, computer, bots, supervisor, SPIRE*, harness*
├── prompt.txt                        # AI-assisted setup checklist
├── renovate.json
│
├── app/                              # Vite + React UI (workspace)
│   ├── package.json
│   ├── src/
│   │   ├── main.tsx, router.tsx
│   │   ├── routes/                   # sign, _app/{agents,bot,channel,routines,skills,admin,settings,onboarding}
│   │   ├── components/               # gallery, channel UI, admin, etc.
│   │   ├── hooks/, lib/
│   └── serve.ts
│
├── server/                           # Hono API + governance (workspace) — KERNEL
│   ├── package.json, Dockerfile, drizzle.config.ts, bunfig.toml
│   ├── drizzle/                      # 0000–0041 SQL migrations + meta/
│   ├── scripts/                      # migrate, cull-idle-computers, fire-routines, test-preload
│   ├── src/
│   │   ├── index.ts                  # composition root, Bun.serve
│   │   ├── production-entry.ts       # eventsource preload for Bun MCP
│   │   ├── app.ts                    # createApp — route mount + DI
│   │   ├── config.ts                 # loadConfig fail-closed
│   │   ├── copilot.ts                # CopilotKit Intelligence bridge
│   │   ├── audit.ts, audit-retention.ts
│   │   ├── credentials.ts            # AES-GCM vault
│   │   ├── tenant-package.ts         # YAML load + sync
│   │   ├── agents/                   # profiles, routes, callback tokens, handoff, escalation
│   │   ├── auth/                     # better-auth, roles, SSO encrypt, org authority
│   │   ├── channels/                 # membership, attachments, WS events, stall, summary
│   │   ├── computer/                 # gateway, policy (CEL), SSRF floor, supervisor client
│   │   ├── plugins/                  # MCP, Composio, OAuth, grants, selection, store
│   │   ├── routines/                 # cron store, sweep, runner
│   │   ├── work/                     # durable queue, culler, loop
│   │   ├── host-access/              # desktop folder grants → host_* tools
│   │   ├── components/               # published + sandboxed generative UI
│   │   ├── people/, routing/
│   │   └── db/schema/{core,coworker,computer,components,plugins,work,json}.ts
│   └── tests/                        # ~191 *.test.ts + helpers/ + support/
│
├── worker/                           # routines CronJob stand-in (workspace)
│   └── src/{index,env}.ts            # imports server routines/* + work queue
│
├── shared/                           # no package.json — relative imports
│   ├── agent-authorisation.ts        # constant-time MANAGED_AGENT_TOKEN compare
│   ├── listen-port.ts
│   ├── bot-prompt.ts, user-content.ts, attachments.ts
│   ├── handoff-markers.ts, routine-firing.ts, work-owner.ts
│
├── supervisor/                       # Docker socket holder
│   └── src/{index,docker,names,environment,identity}.ts
├── agent-computer/                   # browser + workspace + shell
│   └── src/{index,authorisation,control,shell,egress,profiles,...}.ts
├── agent-bot/                        # PoC AG-UI Bot (port 4200) — surface tool loop
├── agent-langgraph/                  # LangGraph + server tool callback (4201)
├── agent-langgraph-agui/             # Python ag-ui-langgraph
├── agent-mastra/                     # Mastra HTTP (no /ag-ui in harness)
├── agent-adk/, agent-ag2/, agent-agno/, agent-claude-sdk/
├── agent-crewai/, agent-langroid/, agent-llamaindex/
├── agent-microsoft/, agent-pydantic-ai/, agent-strands/
│
├── desktop/                          # Tauri + Vite (harness/provider pickers)
├── examples/
│   ├── fintech/                      # default TENANT_PACKAGE_DIR
│   │   ├── agents.yaml, agents/*.yaml (10 reusable coworkers)
│   │   ├── brand, channels, model, knowledge, skills.yaml
│   └── langgraph-bot/, mastra-bot/, pydantic-ai-bot/
├── docs/                             # architecture, configuration, coworkers, routines, …
├── charts/openbot/                   # Helm 0.1.2 / app 0.0.14
├── docker/s6/                        # process supervision for all-in-one
├── scripts/                          # start.sh, stop.sh, diagram, CI helpers
├── tests/                            # cross-package smoke / compose
├── assets/                           # architecture diagrams
└── spire/                            # optional workload identity configs
```

---

## 4. Runtime topology (ports & networks)

| Component | Port | Notes |
|-----------|------|-------|
| `app` | 3010 | Vite / static UI |
| `server` | 3001 | Hono + CopilotKit runtime |
| `agent-computer` | 4100 | Shared or per-Bot |
| `agent-bot` | 4200 | PoC AG-UI |
| `agent-langgraph` | 4201 | Default managed Bot |
| `agent-harness` | 4202 (typical) | Picked framework image |
| `supervisor` | 4500→4300 | Host map loopback only |
| PostgreSQL + pgvector | 5432 | `data` network only |
| CopilotKit Intelligence | external | Threads / memory |

**Compose networks:** Postgres + migrate on `data`; Bots/computers on `default`. Intentional: a Bot shell must not reach the DB by service name. Ports published on `127.0.0.1` and `[::1]` only.

---

## 5. Core data structures (Postgres / Drizzle)

### 5.1 Identity & access (`db/schema/core.ts` and related)

| Table / concept | Purpose |
|-----------------|---------|
| `users`, `sessions`, `accounts`, `verifications` | Better Auth |
| `roles` | OpenBot roles (admin floor via `INITIAL_ADMIN_EMAILS`) |
| `sso_providers` | Runtime SAML/OIDC (deployment-scoped) |
| `revoked_access` | Deny list after people removal |
| `instructions` | Standing user instructions |
| `deployment_packages` | Tenant package checksum / status |

### 5.2 Agents & channels

| Structure | Purpose |
|-----------|---------|
| `agents` | Runtime: AG-UI endpoint + key ref |
| `agent_profiles` | Name, title, role, owner, visibility, soft-delete, callback token **hash** |
| `agent_preferences` | Per-user hide |
| `channels` | Conversation + coworker binding; `allowed_groups` stored but **unread** |
| `channel_memberships` | **Actual** access control |
| `intelligence_channel_mappings` | Channel ↔ Intelligence thread |
| Attachments staging tables | Upload lifecycle + MIME sniff |

### 5.3 Computer & policy

| Structure | Purpose |
|-----------|---------|
| `action_policy` | CEL deny/allow lists (JSON) |
| `computer_snapshot` | Cross-replica target snapshots |
| `computer_page_frame` | Transcript page frames |

### 5.4 Plugins & skills

| Structure | Purpose |
|-----------|---------|
| `mcp_servers`, `mcp_tools` | Catalogue + discovered tools |
| `plugin_grants` | Bot ↔ MCP tool / skill / bot-handoff grants |
| `skills`, `skill_tools` | Instructions + tool refs (`serverId/toolName`) |
| `composio_connections`, `mcp_user_credentials` | Brokered / user OAuth |
| `components`, exclusions, functions | Generative UI governance |
| `sandboxed_components` | Playground drafts/published |

### 5.5 Work & routines

| Structure | Purpose |
|-----------|---------|
| `work_items` | Durable queue: handoffs, routine fires, summaries, culls |
| `routines`, `routine_runs`, `routine_sweeps` | Schedules + outcomes |

### 5.6 Credentials vault

Kinds (enum): `model`, `connector`, `agent`, `mcp`, `mcp_oauth_client`, `mcp_user_token`.  
Encryption: AES-GCM with `KEY_ENCRYPTION_KEY` (base64 32-byte). Plaintext never returned by APIs; redacted from audit.

### 5.7 In-memory / process structures (not tables)

| Structure | Where | Role |
|-----------|-------|------|
| `DeploymentConfig` | `config.ts` | Parsed env; boot refusals |
| `ActionPolicy` | `computer/policy.ts` | `{ mode, deny[], allow[] }` CEL strings |
| Run assertion (signed) | `agents/callback-token.ts` | botId, actor, depth, initiator |
| Channel event hub | `channels/events.ts` | Postgres LISTEN fan-out |
| Control state | `agent-computer/control.ts` | help/secret/take/release TTLs |
| Tool selection set | `plugins/selection.ts` | Narrow offer when grants > 12 |

---

## 6. Algorithms & critical methods

### 6.1 Action governance (computer + MCP)

**Algorithm (gateway / `callTool`):**

```
1. Authenticate caller (session | agent callback token | worker secret)
2. Resolve subject (bot, actor, initiator kind)
3. Resolve target (URL / file / MCP tool / command) from server-held snapshot when needed
4. SSRF / address floor (metadata always deny; private hosts gated)
5. Evaluate ActionPolicy with CEL:
      - deny rules first
      - then allow
      - missing/empty → permit nothing
      - broken deny → deny; broken allow → does not permit
6. Write audit row (decision)
7. If forward: execute (computer HTTP or MCP transport)
8. If execution fails: second audit row
```

**Key symbols:** `evaluateActionPolicy`, `createComputerGateway`, `pluginStore.callTool`, `checkComputerAddress`, `checkNavigationTarget`, `isNeverAllowedHostname`.

**CEL context fields (examples):** `tool.name`, `intent`, `bot.id`, `actor.id`, `page.url`/`host`, `element.*`, `key`, `command`, `file.*`, `mcp.server`/`tool`/`effect`, `initiator.kind`/`id`.

### 6.2 Agent / coworker creation

**Three creation paths:**

1. **Tenant package sync** (`synchronizeTenantPackage`): read `agents.yaml` + `agents/*.yaml` → upsert public ownerless profiles. Types: `built-in` (`system_prompt`) | `remote-ag-ui` (`endpoint`).
2. **HTTP API** (`agents/routes.ts` `POST /`): owned coworker; empty endpoint → `MANAGED_AGENT_AG_UI_URL` + role-derived system prompt.
3. **Conversation** (`bot-creator` skill): frontend tools `list_bots` / `save_bot` only while skill granted.

**Standing role injection:** system message from title + `role_description` + deployment provenance block (cite sources; don’t fake them).

**Visibility filter:** `private` → owner+admins; `public` → all; package agents not editable in product.

### 6.3 Tool callback authorisation

**Algorithm (`authoriseAgentCall`):**

```
1. Parse body { name, args, run }
2. Match x-openbot-agent-token:
      - per-agent hash (obot_agt_*) OR legacy AGENT_TOOL_TOKEN / MANAGED_AGENT_TOKEN
3. Verify signed run assertion (botId must match caller; carries depth, initiator)
4. Dispatch to pluginStore.callTool or deployment host_* tools
```

**Key symbols:** `mintCallbackToken`, `hashCallbackToken`, `mintRunAssertion`, `readRunAssertion`, `sameToken` (constant-time), `parseAgentToolCallInput`.

### 6.4 Per-Bot computer ensure

**Algorithm (supervisor `ensure`):**

```
1. Validate botId charset (names.ts) — no Docker name injection
2. Optional SPIRE entry register → spiffe://…/bot/{botId}
3. Derive container/volume names: ${NAMESPACE}-computer-${botId}
4. If exists: reconcile image id / COMPUTER_TOKEN drift → replace if needed
5. Else create: CapDrop ALL, no-new-privileges, env from environmentFor(botId),
   HostIp 127.0.0.1 when publishing ports, labels openbot.*
6. Return URL to server provider.locate
```

**Key symbols:** `createDockerSupervisorProvider.locate`, `ensure`/`stop`/`reset`/`listOwned`, `environmentFor`, `registerEntry`.

### 6.5 Durable work queue

**Algorithm:**

```
1. Writer inserts work_items row (kind, payload, attempt caps)
2. Workers: SELECT … FOR UPDATE SKIP LOCKED, claim with lease (DB clock)
3. Process; renew lease for long runs (handoffs)
4. Complete or retry; attempt cap → fail / reap
```

Used by: routine fires, bot handoffs, channel titling, idle computer cull.  
**Key symbols:** work `queue.ts` claim/offer; `repeatAfterEach`; `createHandoffRunner`; `offerDueRoutines` / `dispatchClaimedRoutines`.

### 6.6 Handoff between Bots

**Algorithm:**

```
1. message_bot tool offered only if plugin_grants bot-kind exist
2. Desk resolves target against asker’s visible roster (no enumeration)
3. Caps: BOT_HANDOFF_MAX_DEPTH, BOT_HANDOFF_MAX_PER_RUN (refuse, don’t truncate)
4. Enqueue work item; another replica delivers
5. Answer lands in *target* Bot’s conversation (Intelligence thread ownership)
6. Audit offered/refused/delivered/failed
```

Depth/initiator travel inside signed run assertion across pods.

### 6.7 Tool selection / narrowing

When a Bot holds more than **12** tools (`SELECTION_FLOOR`): ask model which skills match the message; offer intersection of those skills’ tools with grants. Failure modes leave **full** grant catalogue offered (narrowing must not silently remove admin grants).  
**Key symbols:** `plugins/selection.ts`, `grantedTools` / `grantedSkills`.

### 6.8 Shell environment sanitisation (computer)

`environmentForCommand`: allow-list PATH/locale/terminal/proxy vars; strip userinfo from proxy URLs; refuse `BASH_ENV`, `LD_PRELOAD`, etc.; run `bash -c` (non-login); kill process group on cancel.

### 6.9 Stall detection

`createStallGuard`: if AG-UI stream produces nothing for `AGENT_STALL_TIMEOUT_MS`, end turn and audit `agent.stream_stalled`.

### 6.10 Tenant package expansion

`${NAME}` / `${NAME:-fallback}` env interpolation in YAML; missing required env fails boot with file name; checksum change → resync on next start. Duplicate agent ids across `agents.yaml` and `agents/` refuse startup.

### 6.11 Auth path selection

```
if any OAuth/SSO provider configured → Better Auth sessions + INITIAL_ADMIN_EMAILS floor
else if OPENBOT_SINGLE_USER=true AND not public URL → DEV_ACTOR admin for every request
else → refuse to start
```

---

## 7. Package-by-package notes

### 7.1 `server/` — governance kernel

Composition: `loadConfig` → stores → `createApp` → `Bun.serve` (+ WS proxy for `/api/computers/:botId/stream`).

**Route islands (high level):** `/health`, `/api/capabilities`, `/api/auth/*`, `/api/me`, admin (audit, people, IdPs, credentials), `/api/agents`, `/api/channels`, `/api/computers`, `/api/plugins`, `/api/routines`, `/api/components`, `/api/agent-tools/call`, `/internal/routines/run`, CopilotKit handler, static SPA.

**Sharp edge:** `createApp` uses many **positional** optional args — easy to mis-wire on refactor.

### 7.2 `app/`

UI only. CopilotKit React + TanStack. No Docker socket, no managed agent tokens. Coworker CRUD and admin go through server HTTP.

### 7.3 Agent harness contract

Common pattern:
1. `/health` unauthenticated (Compose healthcheck)
2. All other paths require `x-openbot-agent-token` == `MANAGED_AGENT_TOKEN`
3. Tools come from AG-UI `input.tools` — harnesses do not invent tool catalogues

| Harness | Tool loop | Path |
|---------|-----------|------|
| `agent-bot` | Surface executes; run ends after TOOL_CALL_* | `/ag-ui` :4200 |
| `agent-langgraph` | Server callback `OPENBOT_TOOL_URL` + `AGENT_TOOL_TOKEN` + run assertion | `/ag-ui` :4201 |
| `agent-langgraph-agui` | Same callback idea in Python | FastAPI AG-UI |
| `agent-mastra` | Mastra native; server uses `@ag-ui/mastra` | no `/ag-ui` in harness |
| Other Python | Ecosystem adapters; often `/` | various |

**Inconsistency:** TS shared auth uses constant-time compare; most Python harnesses use `!=`.

### 7.4 `supervisor/` + `agent-computer/`

Supervisor vocabulary only: ensure/stop/reset/list. Computer: Playwright, control handoff (`computer.help_requested` / `control_taken` / `control_released`), files, exec, screencast. Human at wheel → Bot acting paths refused.

### 7.5 `worker/`

Every ~30s: offer due routines, dispatch claimed; hourly purge. `POST {SERVER_INTERNAL_URL}/internal/routines/run` with `WORKER_SHARED_SECRET`. Expects 202.

### 7.6 `desktop/`

Tauri install UX: harness picker, provider picker, organization sign-in. Sets local runtime; not the multi-tenant web app.

### 7.7 `examples/fintech`

Reusable coworkers as one-file-per-agent under `agents/` (expense-review, ticket-triage, …). README: drop a file = instruction, not capability. Skills named in YAML are preferences for narrowing, not grants.

---

## 8. Cross-cutting flows (end-to-end)

### A. Interactive chat
Session → CopilotKit → resolve coworker → mint run assertion → AG-UI → tool callback → grant+policy+audit → stream to app + Intelligence.

### B. Computer action
UI/Bot tool → gateway → SSRF floor → CEL → audit → supervisor locate if needed → computer HTTP with `COMPUTER_TOKEN`.

### C. Routine
Cron due → work_items → worker → internal run → headless turn as owner → initiator=`routine` on audit.

### D. Handoff
Bot A `message_bot` → desk → queue → Bot B run → answer in B’s channel → unread on roster.

### E. Package boot
`TENANT_PACKAGE_DIR` validate → checksum → sync agents/channels/skills → omit remote agents whose endpoint expands empty.

---

## 9. Findings (severity-ranked)

### High
1. **`OPENBOT_SINGLE_USER`** — every request is admin. Public-URL refusal helps; copying `.env.example` into a reachable host remains catastrophic.
2. **Docker socket on supervisor** — narrow verbs, but stolen `SUPERVISOR_TOKEN` / compromised supervisor = host-root-adjacent control of Bot computers.
3. **Compose default tokens** (`openbot-dev-computer-token`, `openbot-dev-supervisor-token`) — loopback-mitigated; unsafe if ports or defaults leak to a shared machine.
4. **Legacy deployment-wide `AGENT_TOOL_TOKEN`** still accepted beside per-agent hashes — rotate toward per-agent only.

### Medium
5. Dual tool-loop models change unattended behaviour depending on managed Bot image.
6. AG-UI URL path differs by harness — misconfiguration fails opaquely.
7. Positional `createApp` DI fragility.
8. Enormous Composio/plugin-store surface area.
9. Computer token on WebSocket query string (`/stream`) — log/Referer exposure risk.
10. Groups ACL unfinished; schema promises more than enforcement.

### Low / informational
11. Dev Postgres password `openbot`/`openbot` on loopback.
12. Chromium sandbox often off; isolation leans on Docker/gVisor (`COMPUTER_RUNTIME=runsc`).
13. SPIRE optional — product works without workload identity.
14. `bot.declined` is self-reported by models, not a hard control.
15. Positive: metadata SSRF always-deny, DB/Bot network split, fail-closed policy, CapDrop, shell env allow-list, extensive server tests (~191 files).

---

## 10. Security-sensitive watch list

- `server/src/credentials.ts`, `auth/encrypt-sso-config.ts`, `auth/signed-value.ts`
- `agents/callback-token.ts`, `agents/endpoint.ts`
- `app.ts` → `/api/agent-tools/call`, `/internal/routines/run`
- `computer/target.ts`, `computer/gateway.ts`, `computer/policy*.ts`
- `plugins/store.ts` (`callTool`), `plugins/oauth.ts`, `plugins/access.ts`, `plugins/content-governance.ts`
- `config.ts` (single-user, private hosts, placeholder keys)
- `supervisor/src/docker.ts`, `agent-computer/src/{authorisation,shell,control}.ts`
- Attachment MIME serving; computer WS stream proxy

---

## 11. What I learned (distilled)

1. **Governance is the product.** The LLM and the UI are replaceable; the gateway + audit + grants are the durable value.
2. **Coworkers are configuration.** Reuse = YAML files + skills declarations; power = admin grants.
3. **Replica-correctness was designed in.** Timers inside the API would double-fire; the queue is the right abstraction.
4. **Fail closed is cultural here.** Comments and boot checks repeatedly choose “refuse to start” over “open by accident.”
5. **Complexity hid in plugins.** If you fork this for production, budget review time for `plugins/store.ts` and Composio paths equal to the rest of the server.
6. **Harness choice is a security/ops choice**, not just a framework preference — tool-loop location changes what unattended runs can do.

---

## 12. Recommendations (opinionated)

1. Treat production as: real tokens, no `OPENBOT_SINGLE_USER`, IdP configured, `KEY_ENCRYPTION_KEY` unique, `COMPUTER_RUNTIME=runsc` where available.
2. Prefer `agent-langgraph` (or a callback-capable harness) as managed Bot for routines/handoffs; keep `agent-bot` as protocol reference only.
3. Mint per-agent callback tokens; plan deprecation of legacy shared `AGENT_TOOL_TOKEN`.
4. Before relying on channel `allowed_groups`, implement IdP group sync — or delete the fields to stop false confidence.
5. Add a short `docs/harness-matrix.md`: path, port, tool-loop model, required env — reduce footguns.
6. Consider constant-time token compare helper for Python harnesses for posture consistency.

---

## 13. Document history

| Date | Change |
|------|--------|
| 2026-09-21 | Initial full audit from dual deep-pass analysis of `YashwanthGathuku/OpenBot` @ `9c6a237` |

---

*This audit describes understanding of the codebase as read via GitHub. It is not a formal penetration test or legal compliance assessment. Re-run after large merges; the alpha moves.*
