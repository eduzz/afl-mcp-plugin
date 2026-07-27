---
name: afl-hub
description: >-
  Operate the Agents for Life (AFL) hub via its MCP server (`mcp__afl__*` tools):
  talk to AFL agents, read data connected to them (Jira, HubSpot, Notion,
  knowledge base, databases), write/act through them (create/edit records, with
  confirmation for destructive ops), manage your agents and skills (create/edit/delete,
  and enable/disable a skill on an agent), and create/run org squads and automations. Use
  whenever the user wants to ask/act through their AFL agents, search an AFL
  agent's knowledge base, look up or mutate Jira/HubSpot/Notion or database data
  that lives in AFL, create/edit an agent or a skill, create/trigger a squad or automation,
  or mentions "AFL", "Agents for
  Life", "hub", "meu agente"/"my agent", or an agent by name. Requires the `afl`
  MCP server configured (`claude mcp list` → afl ✔ Connected).
---

# AFL Hub — operating the MCP at maximum capability

The `afl` MCP server exposes AFL's capabilities as **tools**, plus MCP **resources**
(skill discovery — browse the platform skill catalog and an agent's enabled skills)
and MCP **prompts** (ready-made instructions to drive an agent to apply a skill).
**Everything runs in the context of a real AGENT you choose** (the service-agent
model): the chosen agent carries the organization, connected data sources, and OAuth
credentials used to execute. There is no generic/global execution — always pick an
agent first.

## Golden workflow (always)

1. **`mcp__afl__list_agents`** (no args) → returns `{ agents: [{ id, name, description }] }`.
2. **Pick the agent whose `description` matches the domain** of the task (e.g. a
   "Financeiro" agent for finance, a "Jira"/"Labzz" agent for Jira, an "Assistente"
   for general/Microsoft tasks). When ambiguous, ask the user which agent to use.
3. Pass that agent's **`id`** as `agentId` to every other tool.

Never guess an `agentId`. If a call returns `agent not accessible`, the agent isn't
yours (or your token's org) — pick another from `list_agents`.

## Which tool to use

Two modes — choose deliberately:

- **Let the agent orchestrate** → `mcp__afl__chat_with_agent`
  `{ agentId, message, conversationId? }`. Runs the agent's FULL loop (RAG over its
  knowledge base + its native tools + reasoning) and returns a natural-language
  answer. Use for open-ended questions, analysis, summaries, multi-step tasks, or
  when you don't know which underlying source has the answer. For a follow-up in the
  same thread, reuse the `conversationId` returned in the result `_meta`.

- **Deterministic structured data** → the per-provider read tools (return raw JSON,
  no LLM in the middle, cheaper/faster):
  - `mcp__afl__search_knowledge_base` `{ agentId, query }` — hybrid semantic+keyword
    search over the agent's indexed KB. **Use specific, literal terms** (names,
    numbers, section titles); generic queries fall below the similarity threshold and
    return `[]`.
  - `mcp__afl__jira_search` `{ agentId, query, project?, maxRows? }` — `query` is
    **JQL** (e.g. `ORDER BY created DESC`, `project = AV AND status = "In Progress"`).
    Never pass natural language as JQL.
  - `mcp__afl__jira_descobrir` `{ agentId, kind, project_key?, board_id?, issue_key?, query? }`
    — real Jira metadata: `projects | issue_types | transitions | priorities | boards |
    sprints | components | versions | users | fields`. Call it BEFORE a Jira write to
    get the real transition name, issue type, sprint/board id, assignee accountId or
    `customfield_NNNNN` — never guess them, and don't burn a `chat_with_agent` for it.
  - `mcp__afl__hubspot_search` `{ agentId, objectType, query }` — objectType is
    `contacts | companies | deals | tickets`.
  - **Per-provider reads, named after the native tool** (`tools:read`): `jira_ler_issue`,
    `jira_ler_comentarios`, `jira_exportar_anexo`, `hubspot_crm_activities`,
    `hubspot_crm_files`, `notion_pages_export`, `google_gmail_read`,
    `google_calendar_read`, `google_drive_read`, `google_sheets_read`,
    `microsoft_mail_read`, `microsoft_calendar_read`, `microsoft_onedrive_read`,
    `github_search`. Use these to **read the state before writing** instead of burning a
    `chat_with_agent`. The export ones (plus `google_drive_read`/`microsoft_onedrive_read`
    with `action: "export"`) return a **`file_key`** — feed it straight into
    `gerenciar_documentos` (knowledge base), `jira_anexar_arquivo` or `hubspot_crm_attach`
    without downloading anything. Drive/OneDrive reads are scoped to the folders
    configured in the agent's source, so a `list`/`search` won't walk the whole drive.
  - `mcp__afl__notion_query` `{ agentId, databaseId?, query? }` — pass `databaseId`
    (the Notion database id) to query a specific DB; otherwise `query` does a
    workspace search.
  - `mcp__afl__query_database` `{ agentId, dataSourceId?, query }` — natural-language
    or SELECT-style question over a connected DB. `dataSourceId` is optional and
    auto-resolved when the agent has a single database source.

The read tools **auto-resolve** the provider's source from the agent's connections.
If an agent has more than one source of a provider, the tool returns an error asking
you to specify the source — surface that to the user.

### Write & action tools (Fase 3)

The hub is **no longer read-only**. Beyond the reads above there are now write and
orchestration tools, each gated by its own scope (`missing scope ...` = the session
lacks it — surface verbatim):

- **Write (~46 tools, scope `tools:write`)** — one MCP tool per native write executor
  (e.g. create/update issues, send messages, mutate provider records). Each takes an
  `agentId` plus the native tool's params. **Destructive actions return a
  `confirmationId` instead of executing** — call `mcp__afl__confirm_action`
  `{ confirmationId }` to actually run it. Per-source `allow_agent_write` still gates
  whether a source accepts writes. Don't enumerate all of them from memory; discover them
  with `listTools` (or `/mcp`) and consult the handbook.
- **Feed the knowledge base** → `mcp__afl__gerenciar_documentos` (`tools:write`)
  `{ agentId, op: "adicionar", titulo, conteudo? | file_key? | file_url?, nome_arquivo?,
  is_critical? }`. `conteudo` is already-extracted text; **`file_key`** takes a file
  exported from another integration (same interchangeable key as `jira_anexar_arquivo`)
  and **`file_url`** an http(s) file. PDF/DOCX/DOC/TXT/MD are extracted, audio/video
  (MP3/WAV/M4A/OGG, MP4/WEBM/MOV) is transcribed — then vectorized. It is
  **asynchronous**: the result comes back `processingStatus: "pending"` and the content
  is only searchable once processing finishes (check with `op: "listar"`). The format
  is inferred from the file name, so pass `nome_arquivo` (e.g. `"ata.pdf"`) when the
  key/URL has no extension. Unsupported formats (e.g. `.xlsx`) are rejected — read the
  data with the integration's own tool and pass the text as `conteudo`. `op: "remover"`
  is destructive (goes through `confirm_action`).
- **`mcp__afl__execute_tool`** (scope-gated) — agent-less **org** tool call: run a
  named tool directly in the token's organization context without picking an agent.
  Use only when you have no suitable agent carrier and know the exact `tool_name`.
- **`mcp__afl__execute_in_background`** (`tools:write`) → returns a `task_id`; fetch it
  later with **`mcp__afl__get_task_result`** `{ task_id }` (`tools:read`). Use for
  long/multi-step work so you don't block.
- **Squads** — `mcp__afl__run_squad` (scope `squads:run`) fires an org squad
  asynchronously → returns a `run_id`; poll **`mcp__afl__get_squad_run`**
  `{ run_id }` (scope `squads:read`). `mcp__afl__list_squads` (`squads:read`) lists
  the org squads you can trigger — pass `include_all: true` to also see **drafts**,
  which is how you find the id of a squad you just created. **`mcp__afl__create_squad`**
  (scope `squads:write`) creates a squad (DAG of steps): pass `name`, `steps[]` and
  `edges[]` — build steps from `list_agents` (agent-type step `config: { agentId }`);
  each step needs an `id` you generate so edges can wire them. Born as a **draft**
  (`is_active=false`) for review in the builder unless you pass `is_active: true`.
  Squads created through the hub are **agent-triggerable by default** (`allow_agent_trigger`
  defaults to `true`), so the full loop is just `create_squad {… is_active:true}` →
  `run_squad`; pass `allow_agent_trigger: false` to opt out. (Squads made in the UI default to
  the trigger off, so `run_squad` rejects until you enable it.) To edit one:
  **`mcp__afl__get_squad`** `{ squad_id }` (`squads:read`) returns
  the full definition, then **`mcp__afl__update_squad`** (`squads:write`) applies changes
  — omitted fields keep their value, but `steps`/`edges` are a **full REPLACE**, so
  resend the whole DAG; `is_active` activates a draft or deactivates a squad, and
  `allow_agent_trigger` toggles whether `run_squad` may fire it (so
  `update_squad { squad_id, allow_agent_trigger: true }` unblocks an existing squad). Squad
  tools require a token bound to an organization.
  **Scheduling (cron) is configurable from the hub** — no need to open the AFL builder.
  Both `create_squad` and `update_squad` accept `schedule_enabled`, `schedule_frequency`
  (`realtime` = every 10min · `every_15_minutes` · `hourly` = minute 0 · `every_6_hours` ·
  `daily` = 09:00 · `weekly` = Monday 09:00 · `custom`), plus `custom_schedule_days`
  (`0`=Sunday … `6`=Saturday) and `custom_schedule_time` (`HH:MM`, server timezone) —
  the last two are **required** with `custom` and ignored otherwise. Passing
  `schedule_frequency` without `schedule_enabled` turns the schedule **on** (the response
  says so in `warnings`). A scheduled squad only fires while `is_active: true`, so a
  scheduled draft warns you. The tools reject combinations that would never fire
  (enabled without a frequency, `custom` without days/time); on `update_squad` the check
  is made against the **merged** state, so `{ squad_id, custom_schedule_time: "18:45" }`
  works on a squad that is already `custom`. Scheduled runs impersonate the squad's
  creator. `list_squads` reports each squad's schedule state (`scheduleEnabled`,
  `scheduleFrequency`, `customScheduleDays`, `customScheduleTime`).
- **Automations** — `mcp__afl__run_automation` (scope `automations:run`) fires an
  automation (fire-and-forget) → `{ queued, correlationId }`; read history with
  **`mcp__afl__get_automation_result`** (scope `automations:read`).
  `mcp__afl__list_automations` (`automations:read`) lists the visible automations.

### Manage agents and skills

CRUD of the user's own agents and skills — separate from `chat_with_agent` (which
*uses* an agent). These reuse AFL's CQRS commands directly (no LLM in the middle).

- **Agents (org-aware)** — the CRUD tools route automatically between **personal**
  agents and **organization** agents (which live in the b2b service). `mcp__afl__get_agent`
  `{ agent_id }` (scope `agents:read`) returns the full config for either.
  **`mcp__afl__create_agent`** (scope `agents:write`): without `organization_id` it creates
  a **personal** agent (only `name` ≥3 chars required); with `organization_id` it creates an
  **org** agent (requires `name` + `prompt`, and the user must be **admin/owner** of the
  org). Other optionals: `description`, `prompt` (system instructions), `level` (personal
  only), `llm_model`, `temperature` (0–2), `category`/`target_audience` (personal only),
  `avatar_icon`, `avatar_color`. **`mcp__afl__update_agent`** `{ agent_id, ... }`
  (`agents:write`) patches fields (omitted = preserved; on an org agent `category`/`is_active`
  are ignored). **`mcp__afl__delete_agent`** `{ agent_id }` (`agents:write`) soft-deletes.
  An org agent shows `type: "organizational"` + `organizationId` in `list_agents`; writing
  to it needs org admin/owner (enforced server-side).
- **Skills** — `mcp__afl__list_skills` `{ type?, category?, search?, organization_id? }`
  and `mcp__afl__get_skill` `{ skill_id }` (scope `skills:read`).
  **`mcp__afl__create_skill`** (scope `skills:write`) — required `slug`
  (`^[a-z0-9][a-z0-9-]*$`), `name`, `description`, `type` (`prompt|tool|composite`);
  optional `organization_id`, `category`, `prompt_injection`, `tool_definitions[]`,
  `execution_config`, `parameters_schema`, `default_parameters`. Visibility is **derived
  from the target org**: `organization_id` (or the org of an org-scoped token) → skill of
  that **organization** (caller must be a member); no org → **personal**. The response
  echoes `visibility` + `organizationId`. **`mcp__afl__update_skill`**
  `{ skill_id, ... }` (`skills:write`) patches (slug/type/visibility are immutable);
  **`mcp__afl__delete_skill`** `{ skill_id }` (`skills:write`).
- **Enable a skill on an agent (opt-in)** — `mcp__afl__list_agent_skills` `{ agent_id }`
  (`agents:read`) lists the skills enabled on an agent, each with an `agent_skill_id`
  (requires access to the agent: yours, or one of your organizations').
  **`mcp__afl__enable_agent_skill`** `{ agent_id, skill_id, config? }` (`agents:write`)
  turns a skill on for that agent; **`mcp__afl__disable_agent_skill`**
  `{ agent_id, agent_skill_id }` (`agents:write`) removes it — use the `agent_skill_id`
  from `list_agent_skills`, not the `skill_id`.

### Manage data sources

Data sources (Jira/Notion/Google/DB/API…) are a **separate** subsystem from skills —
they live in `user_data_sources` and link to agents via `agent_data_connections`.
**Prerequisite:** for a business integration (Jira, HubSpot, Microsoft, Notion) the
integration must already be **connected via OAuth** (in the AFL UI); its `integration_uuid`
comes from there. Find sources/ids with `list_data_sources`.

- `mcp__afl__list_data_sources` (`datasources:read`) → `{ personal, organization }`.
  `mcp__afl__get_data_source` `{ data_source_id, organization_id? }` (`datasources:read`)
  reads one — looks in the personal scope first, then in the organization (membership
  required); the result carries `scope: "personal" | "organization"`.
- **`mcp__afl__create_data_source`** (`datasources:write`):
  `{ source_type, name, organization_id?, integration_uuid?, description?, config? }`.
  With `organization_id` (or an org-scoped token) the source belongs to the
  **organization** — caller must be org **admin/owner**, and it is created org-wide (no
  group scoping; use the org UI when the source must be scoped to a group). Without an
  org it is **personal**. `source_type` is canonical (business integrations carry the
  `_data` suffix: `jira_data`, `hubspot_data`, `notion_database`, `microsoft_365_data`,
  `google_sheet`, `api_endpoint`, `database_table`, `mcp_server`, …; aliases like
  `jira`/`notion` are normalized). Provider specifics go in `config` (e.g. Jira →
  `{ jiraProjectKeys, jiraJqlFilter }`; Notion → `{ notionDatabaseId }`; API →
  `{ apiEndpoint, apiMethod }`).
- **`mcp__afl__connect_agent_data_source`** `{ agent_id, data_source_id, sync_frequency? }`
  and **`mcp__afl__disconnect_agent_data_source`** `{ agent_id, data_source_id }`
  (`datasources:write`) attach/detach a source — both **org-aware** (org agent → b2b path,
  caller must be org admin/owner; else the personal connection).

**Confirm before destructive writes:** when a write returns a `confirmationId`, tell
the user what will change and only call `confirm_action` after they agree (or the
user's request was already an explicit, unambiguous instruction to do it).

## Resources & prompts (skill discovery)

Besides tools, the `afl` server exposes MCP **resources** and **prompts** to discover
and drive **skills** (modular, opt-in agent capabilities). No new scopes: platform
resources/prompts need only a valid token; per-agent ones need access to that agent.
Sampling is not supported.

**Resources** (enumerate with `listResources`, fetch with `readResource <uri>`):

- `afl://skills/platform` — the platform skill catalog **index** (curated, admin-side
  skills, e.g. the native tools exposed as skills). Read this first to discover what
  the system can do.
- `afl://skills/platform/{slug}` — the **full definition** of one platform skill:
  `description`, `promptInjection`, `toolDefinitions`, `parametersSchema`, and
  `execution.kind`. Use a `slug` from the index above.
- `afl://skills/agent/{agentId}` — the skills **enabled on that specific agent**.
  Needs access to the agent — get the `agentId` from `mcp__afl__list_agents`.

**Prompts** (enumerate with `listPrompts`, invoke with `getPrompt`):

- `use_skill` `{ agentId, skill, request? }` — builds an instruction to apply a
  specific skill through the agent. A fast way to make an agent use one skill: feed the
  returned message to `mcp__afl__chat_with_agent`. `skill` is a slug; `request` adds
  the concrete ask.
- `discover_agent_skills` `{ agentId }` — lists the agent's skills and helps pick the
  right one for a task.

**Guidance:** read `afl://skills/platform` to learn what capabilities exist; read
`afl://skills/agent/{id}` to see a given agent's enabled skills; then use the
`use_skill` prompt to invoke one via `chat_with_agent`.

## Getting the most out of it

- **Ground, then synthesize:** `list_agents` → `search_knowledge_base` (or a provider
  read) to pull facts → feed them into your answer, or hand the task to
  `chat_with_agent` for the agent to reason over its own tools.
- **Prefer the specific read tool** when the user wants exact records/rows; prefer
  `chat_with_agent` when they want an interpreted answer ("resume", "analise", "o que
  mudou").
- **Iterate queries:** if `search_knowledge_base` returns `[]`, retry with more
  specific/literal terms before concluding there's no data.
- **Multi-turn:** keep the `conversationId` to preserve context across
  `chat_with_agent` calls.
- **Report tool errors verbatim** to the user (don't silently swallow) — they usually
  say exactly what to fix.

## Auth, scopes, ownership

- Auth is OAuth (browser login on first use; token stored in the OS keychain). The
  AFL is the OIDC Authorization Server (discovery + PKCE + Dynamic Client
  Registration), so `claude mcp add --transport http afl <hub-url>` **without**
  `--header` triggers the browser login flow — no token to paste. A static API key
  (`afl_live_...`) via `claude mcp add --header "Authorization: Bearer afl_live_..."`
  is the fallback for headless/backend-to-backend use.
- **Scopes gate each tool** (`missing scope ...` = the session lacks it). The
  discovery advertises the full set and the consent screen offers all of them:
  `agents:chat`, `agents:read`, `agents:write`, `tools:read`, `tools:write`,
  `skills:read`, `skills:write`, `datasources:read`, `datasources:write`, `squads:read`,
  `squads:run`, `squads:write`, `automations:read`, `automations:run` (or `*`). Reads/chat
  need `tools:read`/`agents:chat`; writes need `tools:write`; agent CRUD needs
  `agents:read`/`agents:write`; skill CRUD needs `skills:read`/`skills:write`; data-source
  CRUD needs `datasources:read`/`datasources:write`; squads need their own (`squads:read`
  to list/read, `squads:run` to fire, `squads:write` to create/edit); automations likewise.
  `list_agents` / `list_organizations` need no scope. If a call returns `missing scope <x>`,
  re-authorize (or mint an API key) with that scope selected — **existing tokens must
  re-consent to gain the new `agents:*`/`skills:*`/`datasources:*` scopes**.
- You can only use agents you own or that belong to your session's organization. One
  session = one org context (+ agents you created).

## Known limitations (current)

- **Writes are opt-in per source** — a write tool only executes if the target data
  source has `allow_agent_write` enabled; otherwise it's rejected. Destructive actions
  always route through `confirm_action`.
- **Squad/automation tools need an org-bound token** (`squads:read` returns nothing
  useful for a purely personal token).
- No path versioning yet; the contract may evolve.

## Reference

Full handbook: `docs/afl-hub-mcp-handbook.md` in the labzz-afl repo (endpoint,
token creation, service-agent model, per-tool args, troubleshooting).
