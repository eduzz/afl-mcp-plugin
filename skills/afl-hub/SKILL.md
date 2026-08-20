---
name: afl-hub
description: >-
  Operate the Agents for Life (AFL) hub via its MCP server (`mcp__afl__*` tools):
  talk to AFL agents, read data connected to them (Jira, HubSpot, Notion,
  knowledge base, databases), write/act through them (create/edit records, with
  confirmation for destructive ops), manage your agents and skills (create/edit/delete,
  put an org agent in a group, and enable/disable a skill on an agent), and create/run org
  squads and automations. Use
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

This skill answers **how to call** the hub. When the question is **what to create** —
should this be an agent, a skill, a data source, a knowledge document, a squad or a
web app, in which order, with which scope — read **`afl-design`** instead, and come
back here for the signatures.

## Golden workflow (always)

1. **`mcp__afl__list_agents`** (no args) → returns
   `{ agents: [{ id, name, description, capabilitySummary, dataSources }] }`.
2. **Pick the agent by what it is CONNECTED to, not by its name.** `dataSources`
   lists each connected source as `{ name, sourceType, allowAgentWrite }`, and Jira
   sources also carry `jiraProjects` — the project keys that source is scoped to.
   That is the field that answers "which agent can write in project LAB": a write
   outside a Jira source's project scope is **denied**, so guessing from the name
   costs a failed call. `description` and `capabilitySummary` are hints, not
   evidence — both are empty on plenty of org agents. When still ambiguous, ask.
3. Pass that agent's **`id`** as `agentId` to every other tool.

Never guess an `agentId`. If a call returns `agent not accessible`, the agent isn't
yours (or your token's org) — pick another from `list_agents`.

## Rules that make the first attempt fail

Short list, high cost. Every item below is a failure that actually happened in
production use — not a hypothetical. Read it before your first write call.

**1. Array arguments must be real arrays — never a JSON string.**
A parameter declared `array` in the schema expects `["a","b"]`, not `"[\"a\",\"b\"]"`.
Serializing it as text is the single most common failure in this hub. The server now
coerces the common shapes, but coercion is a safety net, not a contract: for
parameters that decide **scope** (a search filter) or that **replace a whole set**
(labels on an update, attendees on a calendar update, rows of a spreadsheet write),
an unreadable value is **rejected instead of guessed** — deliberately, because
silently dropping a filter returns the *wrong, larger* result set and silently
emptying a set *deletes* what was there. Send the array.

**2. Read `data.url` — do not parse the text marker.**
Every file-producing tool returns a display marker in `message`
(`[AFL_FILE_URL:<url>|<name>]`) **and** a structured `data` carrying the canonical
**`data.url`** (plus `filename`/`format`). Use `data.url`. The marker's
`<url>|<name>` shape is a trap: both halves look plausible and picking the wrong one
fails silently — which is exactly how a reported bug produced a filename where a URL
belonged. Some tools also keep legacy aliases (`imageUrl`, `file_url`); they are kept
for compatibility, `url` is the contract.

**3. Some tools answer before the work is done.**
`execute_in_background` obviously, but also `criar_app_web` and `editar_app_web`, which
**are hub tools now** (`tools:write`, since 2026-08-07). They validate the contract
synchronously — a refusal comes back whole, with the full list of problems — and then
dispatch the page write, which takes minutes, returning
`{ "dispatched": true, "task_id": "..." }`. **A `task_id` means nothing has been
produced yet**: fetch the outcome with `get_task_result`, and watch the app with
`get_app_web`. (Observed before this: 4m17s between the reply and the app existing.)
The row, though, is **reserved at dispatch**: the app shows up in
**`mcp__afl__listar_apps`** (`agents:read`) from the first instant, as
`status: "gerando"`. So a missing reply is answerable instead of a guess — confirm
there before retrying. The list is newest-first with `status`
(`gerando`/`rascunho`/`publicado`/`revogado`/`expirado`/`falhou`) and takes
`created_within_minutes`, `search` and `app_id`. `gerando` = the page is being
written right now: **wait and list again, do not re-call** (that creates a second
app; `openUrl` is withheld while there is no page). **A draft is not a failure.**
`falhou` means the generation died — `failedReason` says why, and that one you may
recreate. Only *absence from the list* justifies calling `criar_app_web` again. The
list does not carry the manifest — to read what an app actually authorizes, use
**`mcp__afl__get_app_web`** (see "Reading an app's contract" below).

**4. `chat_with_agent` inside a squad step ≠ `chat_with_agent` directly.**
A squad step's `timeoutSeconds` reaches 1800, but the turn the agent answers **inline**
is cut at **245s** — the HTTP leg under it stops at 270s and the model needs the
difference to write its final answer. The same prompt that finishes in a direct call —
where your client can move it to background — blows the deadline inside a step, and the
tools dispatched in the last act come back with `durationMs: 1` and "the task deadline
was reached before this tool finished". Work that does not fit in 245s must go to
**background** (`executar_em_background`, ~20 min, collected by polling), and only then
does a `timeoutSeconds` above 245 buy you anything. Do not calibrate from the number the
field accepts: `get_squad` publishes the arithmetic per agent step in **`turnBudget`**
(`configuredTimeoutSeconds`, `effectiveInlineTurnSeconds`, `inlineTurnCapSeconds`), and
`create_squad`/`update_squad` warn — without refusing, since the background path
legitimately uses the larger number.

**5. Judge a running step by its heartbeat.** See `get_squad_run` below: fresh
heartbeat with `elapsedSeconds` climbing = working. `overdue` is anomalous and
short-lived — not seeing it proves nothing.

**6. Writes are opt-in per data source.** A write tool only executes if the target
source has `allow_agent_write` enabled. The refusal names the source and the fix.

**7. A Jira source's project scope binds EVERY write, not just issue creation.**
Commenting, transitioning, updating a field and creating an issue are all denied
when the target project isn't in the scope of a writable Jira source of that agent
(`dataSources[].jiraProjects` in `list_agents`). It used to bind only
`jira_criar_issue`, so an agent scoped to `AV` could comment on and close `LAB`
cards while the create call claimed the project "was not configured for this
agent" — a message that contradicted what the same agent had just done. Now the
rule is one rule; a source with no project scope declared is still unrestricted.
The refusal names the requested project, the scope that denied it, and which of the
agent's sources reach what.

**8. A `✅` from `jira_criar_issue` is not "everything was saved".** Check
`fieldsDropped` in the returned `data`: any field you asked for that did NOT make it
into the card is listed there (and `parentApplied` / `priorityApplied` say whether
the hierarchy and priority took). A card can be created with the project's default
priority if the value you passed doesn't exist in that instance — discover the real
ones with `jira_descobrir { kind: "priorities" }` and pass the name or the id.

**9. The prose of `chat_with_agent` is NOT evidence that anything ran.** The reply
text comes out of an LLM, and *fabricated* success reads exactly like real success:
an agent reported "Ferramenta `google_calendar_read` — Status: ✓ Sucesso — Nenhum
evento encontrado" for a call that **never happened**, and that became a project
decision ("the Calendar is connected, the agenda is empty") until something else
disproved it. **Cross the narrative with the turn record that comes as the SECOND
text block of the reply** (`── AFL · registro verificável do turno ──`) before
treating any claim of execution as fact. If the prose says a tool ran and it is not
there with status `ok`, it did not run; and `tools: NENHUMA` means the answer came
from the prompt alone. See "Reading the turn record" below.

**10. A Google data source must declare WHICH Google service it is.** The
`source_type`s `google_drive_file`, `google_drive_folder` and `google_services_data`
are shared by Gmail, Calendar and Drive, so they say nothing on their own.
`config.googleSourceType` is **mandatory** on those three (`gmail | drive | calendar
| meet | sheets | docs | slides | forms`) — without it `create_data_source` is
**refused**, naming the field. That refusal is the feature: it used to be accepted,
and the source then showed `isActive: true` in the listing, connected to the agent
without complaint, and **every read failed** with "Integração Google não configurada"
— an error pointing at the integration (which was fine) instead of at the missing
field. See "Manage data sources" below.

## Which tool to use

Two modes — choose deliberately. The dividing line is **fact vs judgement**: for a
FACT use the deterministic tool (`google_calendar_read`, `google_gmail_read`,
`jira_search`, `query_database` — and `mcp_call_tool` for an `mcp_server` source — raw
JSON, no model in the middle); for a JUDGEMENT
(synthesis, analysis, a decision) use the agent.

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
    return `[]`. **Its write counterpart is `create_knowledge_document`** (`tools:write`) —
    the base is not read-only from here, and believing it was cost one migration four of
    its five documents. **An empty result now comes with a census instead of a shrug**:
    `knowledgeBase: { total, critical, awaitingIndexing, processing }` plus a diagnostic
    sentence. Read it before concluding anything, because the two cases need opposite
    fixes: `total: 0` means **nothing was ever written** ("not found" is not even the
    question); `total > 0` with an empty result means the document is there and your query
    did not match it. `critical` documents are injected, never indexed, so they can never
    appear in a search; `awaitingIndexing`/`processing` mean **wait and search again**.
  - `mcp__afl__jira_search` `{ agentId, query, project?, maxRows?, verbosity? }` —
    `query` is **JQL** (e.g. `ORDER BY created DESC`, `project = AV AND status = "In
    Progress"`). Never pass natural language as JQL. Each issue carries `key`, `id`
    (the numeric one, needed by custom fields that take a reference) and, when the
    instance has the field, its `sprint`. The envelope's `projects` reflects what the
    QUERY asked for; `sourceProjects` is the source's own scope. Responses default to
    `verbosity: "compact"` — the `data` envelope only. Pass `verbosity: "full"` to
    also get the same content rendered as markdown (~2× the tokens; you rarely need
    it).
  - `mcp__afl__jira_descobrir` `{ agentId, kind, project_key?, board_id?, issue_key?, query?, verbosity? }`
    — real Jira metadata: `projects | issue_types | transitions | priorities | boards |
    sprints | components | versions | users | fields`. Call it BEFORE a Jira write to
    get the real transition name, issue type, sprint/board id, assignee accountId or
    `customfield_NNNNN` — never guess them, and don't burn a `chat_with_agent` for it.
    On `kind: "sprints"`, `boardId` is the board that OWNS the sprint (the same value
    `jira_search` reports on an issue) and `foundOnBoardId` is the board it was listed
    through — a sprint shows up on every board whose filter reaches it, so those two
    legitimately differ. When more than one sprint is active, `data.warning` says so:
    **ask the user which one**, don't pick.
  - `mcp__afl__hubspot_search` `{ agentId, objectType, query }` — objectType is
    `contacts | companies | deals | tickets`.
  - **Per-provider reads, named after the native tool** (`tools:read`): `jira_ler_issue`,
    `jira_ler_comentarios`, `jira_exportar_anexo`, `hubspot_crm_activities`,
    `hubspot_crm_files`, `notion_pages_export`, `google_gmail_read`,
    `google_calendar_read`, `google_drive_read`, `google_sheets_read`,
    `microsoft_mail_read`, `microsoft_calendar_read`, `microsoft_onedrive_read`,
    `microsoft_sharepoint_scan`, `microsoft_sharepoint_document`, `github_search`.
    `jira_ler_issue` and `jira_ler_comentarios` take the same `verbosity` as
    `jira_search` (compact by default), and `jira_ler_issue` returns the issue's
    numeric `id` alongside `key`. `microsoft_sharepoint_scan` resolves `site_id`/
    `drive_id` from the agent's SharePoint source — **don't go digging for them**;
    pass `data_source_id` only when the agent has more than one SharePoint source
    (the tool says so, naming the candidates). Use these to **read the state before writing** instead of burning a
    `chat_with_agent`. The export ones (plus `google_drive_read`/`microsoft_onedrive_read`
    with `action: "export"`) return a **`file_key`** — feed it straight into
    `gerenciar_documentos` (knowledge base), `jira_anexar_arquivo` or `hubspot_crm_attach`
    without downloading anything. Drive/OneDrive reads are scoped to the folders
    configured in the agent's source, so a `list`/`search` won't walk the whole drive.
  - `mcp__afl__notion_query` `{ agentId, databaseId?, query? }` — pass `databaseId`
    (the Notion database id) to query a specific DB; otherwise `query` does a
    workspace search. `notion_database_schema` `{ agentId, database_id | database_name }`
    gives you the **column names and types** — call it before a
    `notion_database_*_entry` write instead of guessing property names (they are
    LITERAL for Notion: accents and case are part of the name).
  - `mcp__afl__whatsapp_messages_analyze` `{ agentId, analysis_type?, phone?, days?,
    max_results?, include_content?, datasource_name? }` — WhatsApp history, **no model
    in the middle** (SQL over the messages the Evolution webhook persists in real
    time). `analysis_type` is `contact_history` (needs `phone`) | `summary` |
    `top_contacts` | `recent_messages` | `daily_stats`; `days` defaults to 7 — raise it
    to reach older conversations. Covers contacts AND groups. **There is no sync
    step**: an empty result means there are no messages for that window/conversation in
    AFL (anything predating the number's connection does not exist in the database) —
    never that a sync is missing. See the WhatsApp block under writes for the source
    gate, which applies here too.
  - `mcp__afl__query_database` `{ agentId, dataSourceId?, query }` — natural-language
    or SELECT-style question over a connected DB. `dataSourceId` is optional and
    auto-resolved when the agent has a single database source.
  - `mcp__afl__list_mcp_tools` `{ data_source_id, organization_id? }` — the catalog of
    the tools **ENABLED** on an `mcp_server` source: `{ id, name, description,
    inputSchema, required, **`write`** }` per tool, plus `dataSourceId`, `name`, `scope`
    (`personal`|`organization`), `organizationId`, `allowAgentWrite`,
    **`mcpConnectionLinked`**, `total`, `writeTools` and `hint`.
    **`mcpConnectionLinked: false` means the source executes NOTHING** — the catalog is
    recorded in config but the link to the MCP server (the `mcpConnectionId` column) is
    missing, so every call fails with "Conexão MCP não configurada". Repair it with
    `update_data_source { data_source_id, integration_uuid }` and check
    `mcpConnectionLinked` in the answer. Check this **before** wiring the source to an
    agent: a source in that state passes every read-side inspection and only breaks on the
    first execution, which may be inside a squad, days later.
    **`write: true`** marks the tools classified as writes — they are refused on a
    read-only source and never reach the agent's prompt (see `allow_write` below).
    Org-aware: looks in the **organization** first (the token's — see *Token org binding*
    below — requiring active membership) and then in the token owner's **personal** scope — that
    order because the personal lookup filters by owner only and does NOT exclude org
    sources, so an org source you created comes back through it; calling it "personal"
    would run the read without org context (org credential unresolved, no `organization_id`
    in tracking). The org lookup is strict, so no personal source can come back from it.
    The gate is **`tools:read` — the same scope that executes**, not
    `datasources:read`: a read-only collector should not have to widen its token just to
    discover a tool's name. **The catalog is a SNAPSHOT** taken when the source was saved:
    a tool created on the MCP server afterwards does not show up here and is not callable
    until the source is edited and saved again.
  - `mcp__afl__mcp_call_tool` `{ data_source_id, tool_name, params?, organization_id? }` —
    calls a tool of an `mcp_server` source **directly** and returns the raw result,
    **without going through a model** (no LLM tokens spent). For MCP sources this is the
    equivalent of what `google_calendar_read`/`jira_search` are for the native
    integrations: use it to establish **FACT** and to put a cheap gate in front of a
    collector, keeping `chat_with_agent` for **judgement**. Same org resolution as
    `list_mcp_tools` (org first with active membership, then personal) — it works on a
    personal source of the token's owner and on an organization source. Result comes in
    the hub's standard read envelope (`data` + `message`); errors from the MCP server
    arrive verbatim. Four refusals, all of them **before** executing anything:
    - **Only ENABLED tools are callable.** A name outside the catalog is REFUSED naming
      the enabled ones, and **nothing runs** — never substituted by "the closest match".
      That is the heart of the guarantee: the runtime selector only fails in an actionable
      way when the name has a separator; a single-word name outside the catalog would fall
      into keyword scoring and be served by the sibling tool with the highest score — HTTP
      200 with data from another domain, no error at all.
    - **A source with no catalog recorded** (`config.selectedToolsMetadata` empty) is
      refused explaining how to record it (edit and save the source, or recreate it with
      `create_mcp_connection` + `create_data_source`). Exception: a legacy source carrying
      only `mcp_tool_name` stays callable for that single tool.
    - **The deletion family** (`delete/del/remove/archive/drop/trash/destroy/purge/erase`)
      is refused here: the MCP connector's delete permission is granted **PER AGENT** and
      there is no carrier agent in this call. To delete, use `chat_with_agent` with an
      agent that has "Permitir exclusão" enabled for that MCP connector.
    - **A WRITE tool on a read-only source** (`allowAgentWrite: false`) is refused. Which
      tools count as writes is in the `write` field of `list_mcp_tools`.
    - **A source with no link to the MCP server** (`mcpConnectionLinked: false`) is
      refused saying the whole source is inert — not just this tool — and how to repair it.
    - **A dependency being unavailable NEVER becomes "source not found"** — the answer
      says it **could not verify** and warns you not to recreate the source (same rule as
      "an empty list means empty; a missing block means *not checked*" under "Manage data
      sources").

    Flow worth remembering: `list_data_sources { source_type: "mcp_server" }` →
    `list_mcp_tools { data_source_id }` → `mcp_call_tool { data_source_id, tool_name,
    params }`.

- **Discovery — what EXISTS, answered deterministically** (all `datasources:read` except
  the skills one; none of them writes anything):
  - `mcp__afl__list_integrations` — which integrations are connected, whose account each
    one is, and the `integrationUuid` that `create_data_source` wants.
  - `mcp__afl__list_source_types` — the closed catalog of `source_type`, with aliases,
    which types need an `integration_uuid` and the config each one requires.
  - `mcp__afl__list_genie_spaces` — the Databricks Genie spaces the connected workspace
    exposes, with the space `id` the `native-databricks-genie-*` skills ask for.
  - `mcp__afl__list_organization_groups` — the org's groups (name → id), the input to
    `group_ids` in `create_agent`/`update_agent` **and** in `create_squad`/`update_squad`.
  - `mcp__afl__get_organization_topology` (scope `agents:read`) — the org **graph**: who
    exists, in which group, and what is wired to what. This is the one that answers "which
    other agents live in this org" and "where does this resource actually live" — and, in
    its `costCenters` block, the one that resolves the `cost_center <uuid>` a budget refusal
    names. See "Reading the org's shape" below.
  - `mcp__afl__list_skills { visibility: "platform", search }` — whether a native
    capability for X exists at all.

  These belong to the FACT side of the line. The native skill
  **`native-system-integrations-list` answers a similar question, but it answers it through
  an agent** — the reply is LLM prose, and by this hub's own rule (rule 9) prose is not
  evidence. Use it to help a user reason about their setup; **never** to establish that an
  integration is or isn't connected. For that, call the tool.

The read tools **auto-resolve** the provider's source from the agent's connections.
If an agent has more than one source of a provider, the tool returns an error asking
you to specify the source — surface that to the user.

### Reading the turn record — the verifiable part of `chat_with_agent`

**The record you can actually read is the SECOND text block of the result.**
`chat_with_agent` returns two blocks: the agent's reply, then a platform-generated
record titled `── AFL · registro verificável do turno ──`:

```
── AFL · registro verificável do turno (gerado pela plataforma, não pelo agente) ──
tools: 3 (ok: 2, blocked: 1) · 1 com retorno CORTADO (não chegou inteiro ao modelo)
  1. buscar_base_de_conhecimento (ok, 412ms)
  2. jira_search (ok, 1.2s, CORTADO: 65422→35246 chars entregues ao modelo)
  3. google_gmail_send (blocked, 12ms): sem permissão de escrita nesta fonte
llm: claude-sonnet-5 · in 12345 · out 678 · US$ 0.0421
```

`tools: NENHUMA` is the single most useful line in this skill: it means the answer
came out of the prompt alone — no search, no read, no write — so any inventory,
count, id or live state in it was **never checked against a source**. That is
exactly the failure that made an agent list "8 agents" with three invented names
while the correct list of eleven sat, indexed and retrievable, in its own knowledge
base: telling the agent to search does not make it search.

The same record is also in `_meta` (below), but **do not rely on `_meta`** — MCP
clients generally do not surface protocol metadata to the model, which is why the
visible block exists. Read the block; use `_meta` only if your client exposes it.

`_meta` is what the platform measured:

```jsonc
_meta: {
  conversationId, messageId,
  llmModel,                       // kept at the top for backwards compatibility
  llm: { model, inputTokens, outputTokens, totalTokens, costUsd },
  toolCalls: [
    { tool: "google_calendar_read", iteration: 1, status: "ok", durationMs: 812,
      params: { /* truncated preview, secrets masked */ } },
    { tool: "jira_criar_issue", iteration: 2, status: "error", durationMs: 240,
      error: "nenhuma fonte Jira gravável conectada a este agente" },
    { tool: "datasource_databricks_genie", iteration: 3, status: "ok", durationMs: 38410,
      truncated: true, originalChars: 65422, deliveredChars: 35246 }
  ]
}
```

- **`status`** is `ok` | `error` | `pending_confirmation` | `blocked`.
  `pending_confirmation` = a destructive action stopped waiting for
  `confirm_action` (it did **not** execute); `blocked` = a gate refused it before
  execution (write on a source without `allow_agent_write`, a project outside the
  source's scope, a missing scope). `error` carries the executor's **real** message,
  not the model's paraphrase of it.
- **Always reconcile the narrative against `toolCalls`** (rule 9). A tool missing
  from the list did not run, whatever the text says. `params` is a redacted preview —
  enough to identify *which* call was made, not to replay it.
- **`truncated: true`** means the tool succeeded and part of its answer **never
  reached the model** — `originalChars` → `deliveredChars` say how much. `status: ok`
  and a long duration look identical whether the whole answer arrived or half of it
  did, so this is the field that keeps you from crediting the agent with a *choice*
  that was really a *limit*: a step that returned schema instead of baseline did not
  prefer schema, it spent the cut on schema. `truncation[]` appears only when a
  result was cut more than once (`stage: 'tool' | 'turn' | 'provider'`, innermost
  first), so an outer cut never erases the inner one.
  **Absence of the marker means "not cut"** — with three named exceptions where the
  cut is real but not measurable in characters: cuts counted in *rows or items*
  (a Genie table, a capped listing) which say so in their own prose; document
  extraction in consumers that do not yet report the original size; and a cut made
  by a **remote MCP server** before it answered us, which is not ours to measure.
  In that last case you still see the platform's own turn-level cut, not the
  remote one.
- **`costUsd` may be `null`** — that means **not priceable** (a model with no price
  table, or a turn with no counted tokens), never "it was free". Use `llm` for spend
  caps instead of counting calls: a fixed estimate per call is a volume limiter
  wearing a spend limiter's clothes, where reading 3 emails and reading 200 cost the
  same.

**Deterministic tools carry `_meta.toolCalls` too.** Every hub tool returns the same
record — with a single entry, the call itself:

```jsonc
// google_gmail_send, google_calendar_create, mcp_call_tool, list_agents, …
_meta: { toolCalls: [{ tool: "google_gmail_send", status: "ok", durationMs: 934 }] }
```

- Same `status` vocabulary. A destructive write that returned a `confirmationId`
  reads `pending_confirmation` — **nothing ran**; a missing scope or an inactive
  membership reads `blocked`, which is **not** an integration failure.
- **`llm` is absent** — the key does not exist. Not `null`, not zeroed: there was no
  model. A zeroed `llm` would assert a zero-cost model call, which is false.
- Use it to audit writes uniformly: before this, the only proof an effect happened
  was `isError: false` plus each tool's own JSON (`data.threadId`, `data.eventId`) —
  a per-tool contract, with no `durationMs` and no shared status.
- Arguments that fail schema validation are rejected by the protocol **before** the
  handler, and that response has no `_meta` — there the problem is the call, not the
  execution.

### Write & action tools (Fase 3)

The hub is **no longer read-only**. Beyond the reads above there are now write and
orchestration tools, each gated by its own scope (`missing scope ...` = the session
lacks it — surface verbatim):

- **Write (scope `tools:write`)** — one MCP tool per native write executor, dozens of them
  (e.g. create/update issues, send messages, mutate provider records). Each takes an
  `agentId` plus the native tool's params. **Destructive actions return a
  `confirmationId` instead of executing** — call `mcp__afl__confirm_action`
  `{ confirmationId }` to actually run it. It takes no `agentId`: the pending
  confirmation carries the agent AND its organization from the call that emitted it,
  and the permission check on confirm is the same one that ran on emission. (It used
  to carry only the agent, so confirming a delete from an **org** agent was refused
  with "no writable Jira source connected to this agent" — about an agent that had
  just written to that very card. If you ever see a refusal on confirm that
  contradicts a write that just succeeded, that's the shape of it.) Per-source
  `allow_agent_write` still gates whether a source accepts writes. Don't enumerate
  all of them from memory; discover them with `listTools` (or `/mcp`) and consult
  the handbook.
  - **`jira_mudar_status` walks the workflow.** Pass the DESTINATION status
    (`"CLOSED"`), not the next hop: when the workflow requires intermediate states
    it advances on its own while the next step is unambiguous, and returns the
    `path` it took. Where the workflow branches it stops and lists the options
    rather than picking your team's process for you. A card already in the target
    is a declared no-op (`alreadyInTargetStatus`), never an error. So don't loop
    `jira_descobrir { kind: "transitions" }` + one hop at a time — that was 13
    calls to close 3 cards.
  - **`google_gmail_send` replies INSIDE the thread.** Pass `reply_to_message_id`
    = the `id` `google_gmail_read` returned; it derives the thread, the RFC 5322
    `In-Reply-To`/`References` headers, the thread's subject and the recipient, so
    the reply lands in the original conversation in Gmail *and* in Outlook/
    Thunderbird. `to` and `subject` are optional **only** in that case — the
    `inputSchema` marks just `body` (plus `agentId`) as required because the
    requirement is CONDITIONAL, not because a plain send may omit them: a send
    without `reply_to_message_id` that drops `to`/`subject` is refused by the tool
    with a message naming exactly what is missing, and nothing goes out. If the
    original message can't be read, the reply is refused too, rather than leaving
    as a loose "Re:" email that splits the conversation in two. Manual fallbacks
    when you only have part of it: `thread_id` (threads in Gmail only, and needs
    the same subject as the thread) and `in_reply_to`/`references` (thread
    everywhere else).
  - **`jira_criar_issue`: `epic` and `parent_key` both take an issue key.** The
    epic is applied as the issue's parent, and `parentApplied` in the result tells
    you whether it took — no second read needed.
  - **WhatsApp: `send_whatsapp_message`, `whatsapp_audio_send`, `whatsapp_call_reject`**
    (`tools:write`), plus the read `whatsapp_messages_analyze` (`tools:read`).
    `send_whatsapp_message` `{ agentId, phone, message, contact_name?,
    datasource_name? }` — **always prefer `phone`** (country code included,
    `5511999999999`): sending by number does not depend on the address book.
    `whatsapp_audio_send` turns `audio_text` (≤2000 chars) into a voice note by TTS.
    `whatsapp_call_reject` `{ agentId, enabled, message?, reject_video_only? }`
    CONFIGURES automatic call rejection for the whole number — it does not hang up a
    call in progress.

    **Three paths reach WhatsApp; pick deliberately.** The direct tool is a
    **deterministic FACT** (you supply the number and the text, no model decides
    anything). `chat_with_agent` is **JUDGEMENT** — the agent decides *what* to say.
    A squad's `send_whatsapp` action is **ORCHESTRATION** — the squad decides *when*.
    All three land on the same executor and the same gate, so the tool is not a way
    around a permission; it is a way around paying an LLM for a deterministic act.

    **The NUMBER is a data source, and it decides everything.** A connected number is
    a `whatsapp_data` source that must be **linked to the agent** — same gate as
    Jira/HubSpot. What follows from that:
    - **No carrier agent, no WhatsApp.** `execute_tool` (agent-less, org scope)
      REFUSES all four: without an agent there is no source, and without a source
      there is no outbound number. This is deliberate — it is what stops an org agent
      from sending through the personal number of whoever paired it.
    - **Two sources, no automatic pick.** The same number can carry several sources
      with different **scopes** ("Support" = 3 groups, "Sales" = another list). With
      more than one linked, the tool REFUSES (`WHATSAPP_SOURCE_AMBIGUOUS`) and names
      the candidates *with each one's scope* — that scope is how you choose. Pass
      `datasource_name` (exact name, substring, or the source id). It never picks one
      for you: sending through the wrong number is irreversible.
    - **Empty scope means the WHOLE number**, not "no conversations".

    **Writes need source ∧ link**: the source's `allow_agent_write` AND the
    agent-link's `allow_write`, re-checked on every call. Every refusal names the
    source and the fix, and none of them comes back as success — surface them
    verbatim and **never claim the message went out**:
    `WHATSAPP_SOURCE_NOT_CONNECTED` (link the source to this agent — the integration
    may well exist; the LINK is what is missing) · `WHATSAPP_SOURCE_AMBIGUOUS` (pass
    `datasource_name`) · `WHATSAPP_SOURCE_READ_ONLY` (the owner enables "allow write"
    on the source **and** on its link to this agent) · `WHATSAPP_TARGET_OUT_OF_SCOPE`
    (the recipient is outside this source's conversation list — **not** a connection
    or permission failure; use a source that covers them) ·
    `WHATSAPP_SETTING_OUT_OF_SCOPE` (`whatsapp_call_reject` from a restricted-scope
    source: the effect is number-wide, so only a source reaching the whole number may
    set it — true even with write enabled) · `APPROVAL_REQUIRED` (**double opt-in**:
    auto-approve WhatsApp in the user's profile AND on the agent; nothing was queued,
    so do not report the message as "pending") · `TIMEOUT` (the channels-service did
    not answer within 15s/30s — the call fails cleanly instead of hanging; retry).

    **Two WhatsApp tools deliberately do not exist on the hub.**
    `whatsapp_messages_sync` — there is nothing to sync, history is written in real
    time; use `whatsapp_messages_analyze`. And `save_contact` — AFL keeps no address
    book of its own (the contact list is a synced copy of the phone's), so to reach
    someone new call `send_whatsapp_message` with `phone`. If a user asks for either,
    say what replaces it; do not report a contact as saved.
  - **Web pages: `criar_pagina_web` → `listar_paginas_web` → `consultar_pagina_web`
    → `editar_pagina_web` → `consultar_pagina_web` again.**
    Creating returns the page's `url`; editing takes that same `url` and publishes
    a NEW version. When the user refers to a page that already exists without
    giving you the link ("edit that report page"), call `mcp__afl__listar_paginas_web`
    `{ agentId, busca?, limit? }` (`tools:read`) — it lists the pages generated for
    this user, newest first, each with the `url` ready to pass to the editor.
    **Do not recreate a page to change it:** recreating loses everything the
    original brief didn't repeat.
  - **`mcp__afl__consultar_pagina_web`** `{ agentId, pagina, formato?, busca?,
    limite_chars? }` (`tools:read`) **reads the page's content** — the piece that was
    missing from that chain. Use it **twice**:
    - **After `editar_pagina_web`, before saying the change was made.** The editor's
      `instrucoes` mode finishes in the BACKGROUND and can fail after having answered
      "dispatched", so "I edited it" is an unverified claim until you look. If the
      section you asked for is not there, it was **not** applied — say so and redo it.
      Same rule as `ler_app_web` for AFL Apps.
    - **Before `editar_pagina_web` with `substituicoes`**, because `de` must match the
      page's text LITERALLY. Reading gives you the exact string instead of a guess —
      guessing is the usual cause of "no substitution matched".
    Three formats, cheapest first: `estrutura` (title + section index — enough for "did
    section X land?"), `texto` (default, visible text without tags) and `html` (raw, to
    copy an exact snippet). `busca` returns only the passages citing a term, accent- and
    case-insensitive, with the surrounding context — and the snippets come back with the
    ORIGINAL accents and casing, which is exactly what a `de` needs. A page is tens of
    KB, so content is capped and any cut is announced (`truncado: true`): **cut content
    is not missing content** — refine with `busca` before concluding a section is absent.
- **Put a document INTO the knowledge base** → `mcp__afl__create_knowledge_document`
  (`tools:write`) `{ agent_id, title, content | file_key | file_url, filename?,
  source_url?, is_critical? }` — the symmetric write to `search_knowledge_base`, and the
  one to reach for when migrating content into AFL. `agent_id` and `title` are required and
  exactly one body (`content` / `file_key` / `file_url`) must be present — a call with no
  body is refused before it goes anywhere. It is a thin facade over
  `gerenciar_documentos` `op: "adicionar"` (same executor, same permissions, same
  asynchronous extraction/vectorization), so use the native one for `op: "listar"` /
  `"remover"`. **Indexing is asynchronous** — a search fired immediately after may still
  return nothing; confirm with `op: "listar"` before concluding anything. And
  `is_critical: true` documents are **injected into the prompt, not indexed**, so
  `search_knowledge_base` will never return them: verify those by asking the agent to
  quote a literal line (and note that only the 3 most recent critical docs reach the
  prompt).
- **Feed the knowledge base (full surface)** → `mcp__afl__gerenciar_documentos` (`tools:write`)
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
- **Generate a file** → `mcp__afl__criar_documento` (`tools:write`)
  `{ agentId, formato: "xlsx"|"docx"|"pptx"|"pdf"|"md"|"html", titulo, conteudo }`.
  Synchronous: when it returns, the file already exists in storage. Read **`data.url`**
  (download link) and **`data.s3Key`** — the key works as `file_key` and the URL as
  `file_url` in the tools that attach/upload (`jira_anexar_arquivo`,
  `hubspot_crm_attach`, `gerenciar_documentos`, `microsoft_onedrive_write`,
  `google_drive_create`). That closes *generate → attach* without going through
  `chat_with_agent`. Never build a URL out of the chat marker text.
  `conteudo` is a different shape per format, and passing the wrong one silently
  produces a near-empty file: `secoes[]` for pdf/docx/md/html (`{ titulo, texto,
  tabela?, grafico?, imagem_url? }`, `texto` in Markdown — never raw HTML tags),
  `planilhas[]` for xlsx (`{ nome, colunas, linhas }`, plus an optional `estilos`
  for colors/sizes — visual formatting IS supported, don't tell the user to format
  by hand), `slides[]` for pptx.
- **PPTX specifics** (`criar_documento` `formato: "pptx"`) — a slide is
  `{ titulo, conteudo, notas?, bullets?, colunas?: [{ titulo, conteudo }], tabela?,
  grafico?, layout? }`. Three traps worth knowing:
  - The slide's `conteudo` **coexists** with `bullets`, `tabela` and `grafico` (the
    text renders above the element) — you do not have to choose one.
  - `layout` accepts `capa | secao | conteudo | duas_colunas | imagem | imagem_texto
    | imagem_fundo | citacao | tabela | grafico | encerramento`. Anything outside
    that list is **ignored without error** and the layout is inferred from the
    fields you filled — so a typo reads as "the layout parameter did nothing".
  - `incluir_capa` (default `true`) generates a cover from `titulo`. Pass `false`
    when your **first slide already is the cover**, otherwise the deck ships with
    two covers in a row and every slide number is off by one.
  Use `design: "manual"` when you built rich slides yourself — it renders exactly
  what you sent and skips the AI layout pass (`"auto"`, the default, is slower).
- **Brand the deck instead of shipping a generic one** — `marca: { base?,
  cor_primaria?, cor_secundaria?, fonte?, fonte_titulo?, logo_url? }` applies YOUR
  identity over the theme: titles, rules, table header and the first chart color
  take `cor_primaria`, the cover takes `cor_secundaria`, the fonts reach the
  package's own `theme1.xml` (not just the text boxes) and `logo_url` goes on the
  cover. Colors are hex; an invalid one is ignored rather than failing the call.
  Without `marca` the deck carries AFL's identity. Also worth knowing: slides now
  ship at the **modern 16:9 size** (12192000×6858000 EMU), so pasting them into a
  current deck no longer rescales the content.
- **The download link is a BEARER capability.** `data.url` (`/api/documents/d/<token>`)
  is HMAC-signed, opens **without authentication** and **does not expire** — for
  `criar_documento`, `renderizar_pdf` and the `editar_*` tools alike. `isPublic: false`
  in the response describes the storage object's ACL, not the link. Whoever receives
  the URL downloads the file: treat it as a secret, don't post it in an open channel
  and don't reason about it as an internal URL.
- **Render a page/HTML as a faithful PDF** → `mcp__afl__renderizar_pdf` (`tools:write`)
  `{ agentId, html? | url?, titulo?, formato_pagina?, orientacao?, margens?, esperar_seletor? }`.
  Headless-browser render: preserves fonts, colors, positioning and page breaks — the
  PDF looks like the screen. This is NOT `criar_documento`, which builds a document from
  structured content using the AFL template and ignores HTML/CSS. The rule that decides it
  in one line: **an artifact with an identity of its own → `renderizar_pdf`; an artifact
  that should wear AFL's identity → `criar_documento`.** So "make me a report" →
  `criar_documento`; "save THIS page/HTML as PDF" → `renderizar_pdf`; and a report whose
  spec *is* fixed section order, brand hex colors, severity badges and conditional row
  colors → `renderizar_pdf`, because `criar_documento` would silently discard exactly the
  part that was specified. Same output contract
  (`data.url` + `data.s3Key`). Private/internal URLs are blocked; login-gated pages accept
  `headers`, but only for domains the renderer allows.
- **Edit a file you generated** → `mcp__afl__editar_documento` (routes by extension) or
  the direct ones `editar_planilha` / `editar_documento_word` / `editar_apresentacao`.
  Pass the **exact URL returned at creation** in `documento`; scope is the token's user,
  not the conversation. Each edit returns a **new** `data.url` — always use the latest.
  For a big spreadsheet, create it with the header + first batch and append the rest
  with `editar_planilha` `{ acoes: [{ intervalo: "A202", valores: [[…]] }] }` — a single
  call with thousands of rows truncates. PDF/MD/HTML aren't editable in place (regenerate).
- **`mcp__afl__execute_tool`** (scope-gated) — agent-less **org** tool call: run a
  named tool directly in the token's organization context without picking an agent.
  Use only when you have no suitable agent carrier and know the exact `tool_name`.
- **`mcp__afl__execute_in_background`** (`tools:write`) → returns a `task_id`; fetch it
  later with **`mcp__afl__get_task_result`** `{ task_id }` (`tools:read`). Use for
  long/multi-step work so you don't block.
- **Squads** — `mcp__afl__run_squad` (scope `squads:run`) fires an org squad
  asynchronously → returns a `run_id`; poll **`mcp__afl__get_squad_run`**
  `{ squad_id, run_id }` (scope `squads:read`) — the poll returns a **status projection** by
  default (run status + per step: status, timings, duration, error, and
  `hasContent`/`contentChars`), because polling used to re-download every finished
  step's full output on every call. Ask for content only when you need it:
  `fields: "full"` (everything, once at the end) or `step_key`/`step_id` (one step).
  **Content is not evidence of execution.** Each step also carries **`toolCallsCount`**
  (and `toolCalls` with tool name, status and duration when there were any).
  `toolCallsCount: 0` with a big `contentChars` means the agent only *wrote text* —
  a plan or a promise ("awaiting the knowledge-base lookup") — without consulting
  anything, and by every other field it looks exactly like a real dossier
  (`completed`, `hasContent: true`). **The counter can also lie in your favour**: one
  step made **three** calls, **all `ok`** in 37–45 ms, collected nothing, and reported
  "the analyses are processing, I will return with the consolidated data" — a background
  that does not exist. `ok` is a statement about the call, not about the data: read the
  tool *results*, and treat a suspiciously fast `ok` with an empty payload as a failure. Same rule as `_meta.toolCalls` in
  `chat_with_agent`: cross the prose with the record. A step's `toolCalls` carry
  **`truncated`/`originalChars`/`deliveredChars`** with the same meaning as above —
  read them before concluding anything about *why* a step returned what it returned,
  because a cut result and a complete one are indistinguishable by `status` and
  duration alone. `baseContextTokens` shows the
  step's context floor — the input tokens it pays before any tool runs, which is
  what tells you whether a step is over-equipped with skills. A step touched by a
  review loop carries **`loopIteration`** — the number of loops that gate has
  already closed. It has to be read: there is **one row per step per run**, and a
  loop *resets* that row instead of adding one, so without this number a step
  reopened for the third time is indistinguishable from its first execution (same
  `attempt`, same `status`). The run envelope's
  **`totalTokens`/`totalCost`** (USD) are the aggregate consumption, and they are
  updated **during** the run (including while it sits in `waiting_approval`), not
  only at the end.
  A step still in flight also carries its **deadline and proof of life**, so you can
  tell working from stuck without guessing: `timeoutSeconds`/`maxAttempts` (the
  contract that run froze), `elapsedSeconds`, `heartbeatAt`/`heartbeatAgeSeconds`,
  and `retrying`+`retryInSeconds`. Read it as: elapsed climbing with a **fresh
  heartbeat** = working; **stale heartbeat** = stuck; `retrying:true` = an attempt
  already failed (see `error`) and the next one is scheduled. A `running` step with
  `error` set is mid-retry, not healthy. `overdue`+`overdueSeconds` exist too, but
  are **anomalous and short-lived** — the deadline passed and nobody closed the step
  (executor process died), which the watchdog undoes in ~60–120s. **Not seeing
  `overdue` is the normal case, and is not evidence of health** — judge by heartbeat.
  The run envelope carries **`definitionDrift`** when the run is executing a frozen
  definition OLDER than the squad's current one. That is the answer to "I raised
  `timeoutSeconds` and the step still blew the old limit": a run keeps the snapshot
  it started with, and a manual step retry keeps it too — only a **new run** picks
  up the fix.
  Lost the `run_id` (new session, `run_squad`'s response is gone)? **`mcp__afl__list_squad_runs`**
  `{ squad_id }` (`squads:read`) lists that squad's runs, **most recent first** — same compact
  summary as each item, no step output. `page`/`limit` paginate (`limit` 1–100). Filter with
  `status` (one value or an array — `pending`, `running`, `waiting_approval`, `completed`,
  `partial`, `failed`, `cancelled`) and/or `start_date`/`end_date` (`YYYY-MM-DD`, over the run's
  creation date; `end_date` is **inclusive** — the whole day counts).
  `mcp__afl__list_squads` (`squads:read`) lists
  the org squads you can trigger — pass `include_all: true` to also see **drafts**,
  which is how you find the id of a squad you just created. **`mcp__afl__create_squad`**
  (scope `squads:write`) creates a squad (DAG of steps): pass `name`, `steps[]` and
  `edges[]` — build steps from `list_agents` (agent-type step `config: { agentId }`).
  A step's `type` is one of **five**, and each one needs a **different `config`** —
  an incomplete `config` fails the whole DAG, so get it right up front:
  `agent` → `{ agentId, instructions?, readOnly?, failureMarker? }` ·
  `automation` → `{ automationId }` ·
  **`approval`** (human gate, pauses the run) → `{ approverUserIds: [...] }` **or**
  `{ approverGroupRole: "admin" | "member" }` — **one of them is mandatory**
  (optional: `message`, `expiresInHours`, default 168) ·
  `action` → `{ actionType, messageContent?, actionConfig }` with `actionType` in
  `send_inbox_notification` · `webhook` (needs `actionConfig.webhookUrl`) ·
  `send_whatsapp` (needs `actionConfig.whatsappRecipient`) · `execute_integration`
  (needs `actionConfig.integrationId` + `integrationActionName`) · `generate_report`
  (needs top-level `config.reportAgentId` + `actionConfig.reportType`) ·
  `squad` → `{ squadId, waitForCompletion? }`, chaining **another** org squad.
  **`approval` is a step `type`, never an `actionType`** — that confusion (plus an
  `approval` step with no approver) used to come back as a bodyless
  `HTTP 400: "Definição do squad inválida"`, which reads like "approval isn't
  supported". It is; it just needs an approver. The hub now rejects an incomplete
  `config` at the boundary, naming the step and the missing field.
  **`readOnly: true` on an agent step is a real gate, not a prompt line.** Until
  2026-08-07 an agent step's `config` took exactly two fields — `agentId` and
  `instructions` — so "this step must not write" could only be *prose*. Prose
  loses: in one production run the step had `Somente leitura.` as the last line of
  its own instructions, got *"do not write anything in Jira in any step"* in the
  trigger message, and **called `jira_comentar` anyway**. What stopped the write
  was an unrelated bug. With `readOnly`, the tool is removed from the list offered
  to the model **and** refused at the execution funnel — including the write
  *operations* of source tools (`datasource_jira op='write'`, `datasource_mcp`
  calling a write tool of the server) and the delegating ones
  (`executar_em_background`, `executar_squad`, `mencionar_agente`), which would run
  in a turn where the policy is not re-evaluated. The read allowlist is the
  platform's and **denies what it cannot classify** — which includes tools that reach the
  agent through a **skill**, even genuinely read-only ones. Setting `readOnly` on every
  non-writing step is right, and it can still be what switches off the last read path the
  step had: when a step depends on a tool the allowlist does not know, decide that
  deliberately instead of discovering it as `Recusado pela política do step` mid-run.
  **There is no per-step tool selection, deliberately.** An `allowedTools` list
  existed briefly and was removed: choosing *which* tools an agent uses is the
  **agent's** configuration, and a per-step list would be a second source of truth
  about the repertoire, drifting silently from the agent every time a tool is
  added. What a step declares is **privilege**, which is the one dimension the
  agent cannot express — `allow_write` on the agent↔source link is per **agent**,
  so "step 2 reads, step 3 writes, same carrier" only becomes sayable here.
  `get_squad` echoes the recognised policy as **`toolPolicy`, outside `config`**:
  present = the step denies what falls outside it; **absent = the step has the
  carrier agent's full privilege**, no matter what `instructions` claims.

  **`modelResolution` answers "why did this step run on that model?" — before it
  runs.** A run where all three steps landed on a small model, while a direct
  `chat_with_agent` to the *same* agent minutes earlier used a bigger one, looked
  like squads pinning a model. They do not: both paths ask the matrix for the same
  `functionality`. The swap happens **after** the matrix, in the LLM preflight,
  which re-resolves with the turn's **real** token requirement — and a step ships
  the squad context, the trigger, the **entire output of every parent step** and
  every tool schema, while a chat message ships a question. *The path does not
  change the model; the path changes the size of the turn, and the size decides.*
  So `get_squad` now returns, per agent step, the `functionality`, the matrix's
  `matrixModel`/`matrixFallbackModel`, a `precedence[]` and
  `mayAutoSwitchOnContext`. **There is no `config.model`, and there will not be**:
  pinning a model on a step is the same LLM-matrix bypass that got `llm_model`
  removed from agents. If a step is landing on too small a model, the lever is the
  size of the dossier it inherits (`includeParentOutputs`), not a pin.

  **A rejection can send the work back instead of killing the run.** Two pieces,
  and neither works alone: an edge's **`onOutcome`** (`"approved"` | `"rejected"`;
  omit it for the normal unconditional edge) says **where** a rejection goes back
  to, and the gate's **`config.maxLoops`** (`0`–`5`, `0` = no loop) says **how
  many** times. Until this existed, rejecting failed the step, cascaded over its
  descendants and stranded the reviewer's note — and a back edge drawn by hand was
  *accepted*, leaving the run `running` forever with no step in flight. These
  rules **reject the squad at create/update time**, so you don't find them by
  error: `onOutcome` is only valid on an edge leaving an `approval` step; a
  `rejected` edge must target an **ancestor** of the gate (it's a loop, not a
  lateral jump); `rejected` requires `maxLoops >= 1`, and `maxLoops >= 1` requires
  at least one `approved` edge (otherwise approving leads nowhere); `maxLoops`
  outside 0–5 is refused; and two loops must be **strictly nested or fully
  disjoint** — partially overlapping ones are refused. On rejection the gate and
  the stretch between target and gate return to `pending` in a single transaction
  and the **reviewer's note is injected into the reopened step's prompt**
  (a "REVISÃO REPROVADA — refaça o trabalho (ciclo N de M)" block), so you do
  **not** stitch the feedback into the prompt yourself. Once the cycles are spent,
  a rejection behaves as it always did (fail + cascade), with an error saying it
  was rejected after N/N review cycles.
  Step and edge `id`s are **optional** — the hub generates them (and replaces
  non-uuid ones like `"s1"`, which the backend rejects); `fromStepId`/`toStepId` also
  accept a `stepKey`. Read the returned `steps[].id` if you plan to `update_squad`.
  Step limits are enforced at the boundary with a message that names the field:
  `timeoutSeconds` 30–1800 (default 170), `maxRetries` 0–3, `retryDelaySeconds` 5–300.
  An `agent` step may now hand long work to a **background task** and wait for it, which
  is why the ceiling is 30 min — but the turn the agent answers **inline** is capped at
  **245s** (270s of HTTP minus the margin the model needs to write the final answer). So:
  work that fits inline must fit in 245s; work that doesn't goes to background, and only
  then does a `timeoutSeconds` above 245 buy you anything. Above it the tools
  create/update **warn** instead of refusing (refusing would kill the background path and
  break every squad already carrying 1800), and `get_squad` attaches the resolved
  **`turnBudget`** to each agent step so the two numbers never have to be reconciled by
  hand.
  **A step that declares its own failure in prose still closes `completed`.** Six agent
  steps in one production run answered `FALHA: …`, exactly as their prompts told them to,
  and the DAG walked node by node to the human gate, which then sat three days waiting for
  approval of an empty set (`"error": null` on the envelope throughout). Set
  **`config.failureMarker: "FALHA"`** on the agent step to give that declaration authority:
  an output starting with the marker fails the step, enters the normal retry/cascade path
  and cancels the descendants. It is deliberately **opt-in and never a heuristic** — an
  honest report *describes* the failures it found, so hunting for error words would fail
  the good result. Without the field the run projection still **signals** it
  (`selfDeclaredFailure: true` on the step), and signalling is not deciding.
  **A trigger contract, so a run is refused before it spends a step.** `create_squad` and
  `update_squad` take **`trigger_schema`**: `{ fields: [{ key, label, type: "string" |
  "date" | "number" | "enum", required, options?, description? }] }`. `run_squad` then
  takes the values in **`inputs`**, a missing `required` field refuses the trigger up
  front, and the values are appended to the trigger message as a stable
  `[ENTRADA DO DISPARO]` block the steps can quote. Without it the trigger is free text:
  four root steps once fired in parallel on a reference date nothing guaranteed, and the
  first one answered "FALHA: the reference date was missing from the trigger". An
  `approval` step is not a substitute — it collects a **decision**, not a typed **value**.
  A squad is born as a **draft** (`is_active=false`) for review in the builder unless
  you pass `is_active: true`.
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
  **Scoping a squad to groups — `group_ids`, and the field the hub could read but not
  write.** `get_squad` always returned `groupIds` (plus a compat `groupId`), and no tool
  ever wrote it: you could create the group, create the agent, put the agent in the group —
  and then stop at the squad and open the UI. Both `create_squad` and `update_squad` now
  take **`group_ids`** (org **admin/owner**), resolved from `list_organization_groups` or
  freshly created with `create_organization_group`.
  It is **REPLACE, not append**, exactly like an agent's `group_ids`: in `create_squad`
  omitting it leaves the squad **org-wide** (everyone in the org); in `update_squad`
  omitting it **keeps** the current scope, the list you send becomes the whole scope, and
  `[]` **unlinks** it back to org-wide. Read the current scope from `get_squad` first when
  you mean to *add* a group.
  On the read side `groupIds` is the **canonical** field — the N:N scope — and `groupId` is
  1:1 compat that always equals `groupIds[0] ?? null`, so the two can never disagree; an
  empty `groupIds` with a null `groupId` means org-wide, not "unknown".
  This is not cosmetic metadata: a squad's group scope decides **who sees it**
  (`list_squads`, `get_squad`), **who may trigger it**, and who is resolved as approver for
  an `approval` step with `approverGroupRole`. That is why it needs org admin/owner — the
  refusal comes back as `blocked` **before** the write is attempted, and a group belonging
  to **another organization** is rejected outright. `create_squad` echoes the scope it
  stored in `groupIds`; `update_squad` echoes it only when your patch touched it, so
  "I left the scope alone" and "I made it org-wide" never look the same.
  **Scheduling (cron) is configurable from the hub** — no need to open the AFL builder.
  Frequencies include **`monthly`** with a day of the month (a month without that day
  fires on its last day) — monthly rituals used to be inexpressible and ended up on
  manual triggering.
  Both `create_squad` and `update_squad` accept `schedule_enabled`, `schedule_frequency`
  (`realtime` = every 10min · `every_15_minutes` · `hourly` = minute 0 · `every_6_hours` ·
  `daily` = 09:00 · `weekly` = Monday 09:00 · `monthly` · `custom`), plus
  `custom_schedule_days`
  (`0`=Sunday … `6`=Saturday) and `custom_schedule_time` (`HH:MM`, server timezone) —
  the last two are **required** with `custom` and ignored otherwise, and
  `schedule_day_of_month` (`1`–`31`, **required** with `monthly` together with
  `custom_schedule_time`).
  **Quarterly / four-monthly cadences are `monthly` + `schedule_months`** — an array of
  months `1`–`12` (`[3,6,9,12]` = quarterly committee, `[12,4,8]` = four-monthly strategic
  planning cycle, `[1,7]` = biannual). Omitting it means every month, so existing monthly
  squads are unaffected; it is **rejected** with any frequency other than `monthly`
  (a field accepted and ignored would silently run the ritual 12× a year). Passing
  `schedule_frequency` without `schedule_enabled` turns the schedule **on** (the response
  says so in `warnings`). A scheduled squad only fires while `is_active: true`, so a
  scheduled draft warns you. The tools reject combinations that would never fire
  (enabled without a frequency, `custom` without days/time); on `update_squad` the check
  is made against the **merged** state, so `{ squad_id, custom_schedule_time: "18:45" }`
  works on a squad that is already `custom`. Scheduled runs impersonate the squad's
  creator. `list_squads` reports each squad's schedule state (`scheduleEnabled`,
  `scheduleFrequency`, `customScheduleDays`, `customScheduleTime`, `scheduleDayOfMonth`,
  `scheduleMonths`).
- **Automations** — `mcp__afl__run_automation` (scope `automations:run`) fires an
  automation (fire-and-forget) → `{ queued, correlationId }`; read history with
  **`mcp__afl__get_automation_result`** (scope `automations:read`).
  `mcp__afl__list_automations` (`automations:read`) lists the visible automations.

### Manage agents and skills

CRUD of the user's own agents and skills — separate from `chat_with_agent` (which
*uses* an agent). These reuse AFL's CQRS commands directly (no LLM in the middle).

- **Agents (org-aware)** — the CRUD tools route automatically between **personal**
  agents and **organization** agents (which live in the b2b service). `mcp__afl__get_agent`
  `{ agent_id }` (scope `agents:read`) returns the full config for either — for an org
  agent it also carries `groups` (the groups it is linked to).
  **`mcp__afl__create_agent`** (scope `agents:write`): without `organization_id` it creates
  a **personal** agent (only `name` ≥3 chars required); with `organization_id` it creates an
  **org** agent (requires `name` + `prompt`, and the user must be **admin/owner** of the
  org). Other optionals: `description`, `prompt` (system instructions), **`agent_type`**
  (the subtype — see below), `level` (personal
  only), `temperature` (0–2), `category`/`target_audience` (personal only),
  `avatar_icon`, `avatar_color`, and `group_ids` (org only — see below).
  **The model is not one of them** — see the note below.
  **`agent_type` is the one model lever a tool has.** It is the same value `get_agent`
  returns as `agentType`, and it is the matrix's subtype row: precedence is *agent's own
  model → **subtype** → functionality → default*. Omitted, an agent is born `assistant`,
  usually the cheapest mapping — so a whole squad built through the hub runs on the small
  model unless you say otherwise, which is invisible from every listing. It is an enum
  (`assistant`, `analyst`, `researcher`, `advisor`, `teacher`, `coach`, `creative`,
  `developer`, `support`, `custom`; `alter`/`role` are personal-only) because a value
  outside the list does not fail — it just matches no matrix row and falls back to the
  generic model, in silence. Two consequences worth planning around:
  - **Setting a subtype does not promise a better model.** It selects the matrix *row* an
    admin configured; a subtype with no row simply falls through to the `chat` functionality.
    Confirm the outcome in `get_squad` → `modelResolution` (or the turn record) instead of
    assuming the write bought you anything.
  - **On an ORG agent the subtype can only be set at creation.** `update_agent` refuses
    `agent_type` for an organization agent — explicitly, changing nothing — because the
    internal route it uses does not carry the field; accepting it would report "updated" for
    a write that never happened. So order the migration accordingly: decide the subtype
    **before** `create_agent`, or change it in the organization's UI.
  **`mcp__afl__update_agent`** `{ agent_id, ... }`
  (`agents:write`) patches fields (omitted = preserved; on an org agent `category`/`is_active`
  are ignored and `group_ids` **redefines** the groups).

  > **`max_tokens` and `llm_model` no longer exist on this surface** (removed 2026-08-07),
  > and for the same underlying reason: both looked like controls and decided nothing.
  > `max_tokens` presented itself as *"the agent's response token cap"*, range-checked
  > 100–8000 — and was **never read by the execution loop**, which always took the
  > per-call ceiling from the *model*. An agent configured at 2000 produced 5,712 output
  > tokens in one turn, so anyone sizing cost by it was sizing fiction. (Related reading
  > trap that still applies: the `out` in the verifiable turn block is the **sum of the
  > turn**, which makes several calls when tools run — a per-call cap could never account
  > for it.) `llm_model` went because **model choice belongs to the LLM matrix**, resolved
  > per functionality/plan/subtype and administered by admins; a model pinned on the agent
  > bypasses that matrix, and the visible symptom was the same agent running on different
  > models depending on the invocation path, with nobody able to say why.
  **To change PART of the agent's `prompt`, use `prompt_edits` / `prompt_append` /
  `prompt_prepend`** — same anchored-edit contract as `update_skill` (literal match,
  must be unique, aborts the whole call on a miss, cannot be combined with the full
  `prompt`). The current value is read through the same route that writes it, so an
  org agent is read from the org side and a personal one locally.
  **`mcp__afl__delete_agent`** `{ agent_id }` (`agents:write`) soft-deletes.
  An org agent shows `type: "organizational"` + `organizationId` in `list_agents`; writing
  to it needs org admin/owner (enforced server-side) — being the agent's *creator* is not
  enough, so a refusal naming "admin/owner" means the role is missing, not the id. Don't
  route around it via `chat_with_agent` + the native `gerenciar_agentes` tool: that path
  enforces the same rule now.
- **Reading the org's shape — `mcp__afl__get_organization_topology`**
  `{ organization_id?, detail?, node_types?, max_nodes? }` (scope `agents:read`).
  Returns the organization **graph** — organization, groups, members, agents, data sources
  and integrations as nodes, plus the edges between them — **already cut down to what you
  can see**. Use it before acting: to find an org agent's **sibling agents**, to see which
  group something belongs to, and to locate **where a resource lives** (which source, which
  integration, which group) instead of guessing or spending a `chat_with_agent` on it.

  - **The result is scoped to YOU, server-side.** There is no user parameter and there will
    not be one: the cut is decided from the bearer token's user. `scope` (`"all"` |
    `"groups"`) and `visibleGroupIds` in the response tell you **which** cut you are looking
    at. A member who only sees group A never receives group B's nodes. Reading it requires
    an **active** membership; a refusal comes back as `blocked`, and a failure to *evaluate*
    the membership also refuses (unavailable never means authorized).
  - **The default is compact, on purpose.** `detail: "compact"` (default) gives `counts`
    (per-type totals for the **whole graph in your scope**), `visibleGroupIds`, and nodes as
    `{ id, type, label, sub? }`. It **omits every edge** and each node's `meta`. Ask for
    `detail: "full"` when you need the relations — edge `kind` is `org | pertence | acessa |
    depende | habilita | compoe | sequencia | executa | publica`.
  - **`node_types` narrows the presentation, not the scope** (`counts` still reflects
    everything you can see), and `max_nodes` (default 250, max 5000) truncates **with a
    `truncated` block** naming what was dropped per type — repeat with `node_types` for the
    part you still need.
  - **A partial graph is never presented as complete.** `partial: true` plus `unavailable[]`
    name what was not collected: today `squads`, `steps`, `skills` and `apps` come back as
    `not_implemented_v1`, and cross-service reads that failed appear as `read_failed`.
    As everywhere in this hub, **a missing node means this listing did not bring it — never
    that it does not exist.**
  - **`costCenters` — this is where a budget refusal becomes actionable.** When an LLM call
    is refused with `LLM access blocked for cost_center <uuid>` (or `LLM quota exceeded for
    cost_center <uuid>`), that uuid is **neither the organization nor the user** — it is a
    cost center, and this block is what resolves it. Each entry in `costCenters.visible[]`
    carries `id`, `name`, `limits` (**`monthlyCostLimitBrl`**, `monthlyTokenLimit`,
    `dailyRequestLimit`), `usage` (`currentMonthCostUsd`, **`currentMonthCostBrl`**,
    `currentMonthTokens`, `currentDayRequests`), `usagePercentOfCostLimit`, `appliesToYou`,
    `assignedTo` (groups and users) and, decisively, **`blocking`**, **`exhausted[]`** plus
    **`undecidable[]`** — which dimension ran out (`monthly_cost` | `monthly_tokens` |
    `daily_requests`). Raising a *different* ceiling (the account's, the plan's) does not
    unblock this one; say which dimension to raise.
    - **Two currencies, both declared (migration 0267).** The ceiling is quoted in
      **BRL** — that is the currency the customer contracts it in, and the `Brl` suffix in
      the name is the contract. Spend is metered in **USD**, because that is what Bedrock
      and OpenAI bill in, and converting the measured fact would rewrite it with a rate. So
      `usage` ships both: the metered figure (`currentMonthCostUsd`) and the same spend in
      the ceiling's currency (`currentMonthCostBrl`), with the rate that linked them in
      **`usdBrlRate`**. Always compare `currentMonthCostBrl` against
      `monthlyCostLimitBrl` — comparing the dollar figure to the real one reads roughly 5×
      more optimistic than the truth.
    - **`undecidable[]` is a third state, not a synonym for "all clear".** With no exchange
      rate, `currentMonthCostBrl` and `usagePercentOfCostLimit` come back `null`,
      `monthly_cost` lands in `undecidable[]` — and **`blocking` stays `true`**, because
      that is what the gate does: a declared ceiling that cannot be evaluated DENIES. In
      that state claim neither headroom nor exhaustion; say the budget could not be
      evaluated.
    - `configuredMonthlyCostLimitBrl` is the ceiling an admin typed on the cost center;
      `limits.monthlyCostLimitBrl` is the one actually in force (the `llm_usage_limits`
      row). When they diverge the difference shows up in `warnings` — the second one wins.
    - **The cut is declared, like the graph's.** `costCenters.scope` is `"all"` for org
      **admin/owner** (every cost center of the organization) and `"yours"` for a plain
      member, who only sees the centers that **apply to them** — a member does not enumerate
      other people's budgets. `administrators[]` ships in both cases: that is who to ask.
    - **The hub does not administer budget** (security decision — that stays outside the
      agent channel). `costCenters.manageAt` is the AFL interface path where a ceiling is
      changed; there is no tool for it and there will not be one.
    - `enforced: false` means "no limits row" — the center **blocks nothing**, it is not a
      zero ceiling. A configured ceiling that diverges from the enforced one shows up in
      `warnings`. If the budget read itself fails, `costCenters.unavailable` says so and the
      graph still comes back whole — never "it does not exist".
- **Put an org agent in a group — and create the group if it isn't there.**
  `mcp__afl__list_organization_groups` `{ organization_id? }` (scope `agents:read`) lists
  the org's groups (`{ id, name, description, groupType, hierarchyLevel }`); omit the id to
  use the token's org. Resolve the group **by name here** and pass its `id` in `group_ids` —
  never guess a group uuid. When the group does not exist yet,
  **`mcp__afl__create_organization_group`** `{ organization_id?, name, description?,
  group_type?, parent_group_id?, color?, icon?, responsibilities?, metadata? }` (scope
  `agents:write`, org **admin/owner**) creates it and returns the `id` you pass straight to
  `group_ids` — in `create_agent`/`update_agent` (the agent joins the group) **and** in
  `create_squad`/`update_squad` (the squad's group scope); building a whole org over MCP
  used to stop exactly here.
  **`mcp__afl__update_organization_group`** `{ group_id, organization_id?, … }` patches it
  (only the fields you send; an empty patch is refused instead of returning the group
  untouched and looking like it applied). `group_type` is `time | area | contexto | papel |
  organizacao`. Two things that decide whether your first call works:
  - **`hierarchy_level` is NOT a parameter.** The level is **derived** from
    `parent_group_id` — no parent = root, level 1; with a parent = the parent's level + 1,
    and the parent must belong to the **same organization**. Accepting the field would be
    offering a way to persist an incoherent tree.
  - **`name` is UNIQUE within the organization.** Repeating an existing name comes back as
    `Nome já está em uso` — that is not a create failure to retry, it is a **rename**:
    resolve the existing group with `list_organization_groups` and use
    `update_organization_group`.
  `group_ids` is **REPLACE, not append**: the list is the desired final state,
  groups left out are unlinked and `[]` unlinks all, which makes resending the same list
  idempotent. Omitting it in `update_agent` leaves the groups untouched — read the current
  ones from `get_agent` first when you mean to add to them. In `create_agent` the linking
  happens *after* the agent exists, so a failure there returns the agent plus a `warning`:
  fix it with `update_agent`, don't recreate. `group_ids` on a personal agent is an error.
- **Skills** — `mcp__afl__list_skills`
  `{ visibility?, type?, category?, search?, organization_id?, page?, limit?, fields? }`
  and `mcp__afl__get_skill` `{ skill_id }` (scope `skills:read`).
  **Start with `visibility`, not with the whole catalog.** It takes `personal |
  organizational | platform`, and it is the first filter because an unfiltered listing is
  dominated by the platform skills, which are the same for everyone and usually not what
  you are looking for: on 2026-08-04, one `limit: 200` call with no filter returned **170 skills
  (152 platform, 18 of the org) — 94,069 characters over 1,898 lines**, and blew the MCP
  client's response limit. 89% of that was the part nobody asked for.
  `visibility: "organizational"` (+ `organization_id`) answers "what has THIS org built";
  `"platform"` (+ `search`) answers "is there a native capability for X"; `"personal"` is
  yours. Omitted = everything.
  The listing is **paginated and compact by default** (`limit` 50, cap 200): it returns what you need to
  CHOOSE a skill, not its full config — `fields: "full"` or `get_skill` for that. Same for
  `mcp__afl__list_data_sources` (`scope`, `page`, `limit`, `fields`, `source_type`), whose
  items omit absent fields entirely instead of asserting `null`.
  **`mcp__afl__create_skill`** (scope `skills:write`) — required `slug`
  (`^[a-z0-9][a-z0-9-]*$`), `name`, `description`, `type` (`prompt|tool|composite`);
  optional `organization_id`, `category`, `prompt_injection`, `tool_definitions[]`,
  `execution_config`, `parameters_schema`, `default_parameters`. Visibility is **derived
  from the target org**: `organization_id` (or the org of an org-scoped token) → skill of
  that **organization** (caller must be a member); no org → **personal**. The response
  echoes `visibility` + `organizationId`. **`mcp__afl__update_skill`**
  `{ skill_id, ... }` (`skills:write`) patches (slug/type/visibility are immutable);
  **`mcp__afl__delete_skill`** `{ skill_id }` (`skills:write`).
  - **To change PART of a `prompt_injection`, never resend the whole text.** Use
    `prompt_injection_edits: [{ find, replace, replace_all? }]` — anchored
    substitutions applied server-side to the CURRENT value. `find` is matched
    **literally** (whitespace, line breaks and accents count) and must be unique;
    zero matches, or more than one without `replace_all`, aborts the whole call and
    nothing is written. `prompt_injection_append` / `_prepend` add a section without
    an anchor. `prompt_injection` (the full field) still exists but means "rewrite
    everything", and cannot be combined with the partial forms.
    Why it matters: these prompts run to tens of KB and an org skill's prompt is
    shared by everyone in the org. Resending it makes you re-emit from memory a text
    you were not asked to rewrite — any drift is a silent edit — and turns a
    one-line change into a full rewrite, which is what it looks like to anything
    watching the call. A failing anchor is also the cheap detector that the text
    changed since you read it: the edit fails instead of clobbering someone else's
    change. The result carries `promptInjectionEdit` (edits applied, size before/
    after) so you can confirm the effect without re-reading the prompt.
- **Enable a skill on an agent (opt-in)** — `mcp__afl__list_agent_skills` `{ agent_id }`
  (`agents:read`) lists the skills enabled on an agent, each with an `agent_skill_id`
  (requires access to the agent: yours, or one of your organizations').
  **`mcp__afl__enable_agent_skill`** `{ agent_id, skill_id, config? }` (`agents:write`)
  turns a skill on for that agent; **`mcp__afl__disable_agent_skill`**
  `{ agent_id, agent_skill_id }` (`agents:write`) removes it — use the `agent_skill_id`
  from `list_agent_skills`, not the `skill_id`.
- **Reading `usageCount` correctly** — `list_agent_skills` returns `usageMeasurement`
  alongside `usageCount`/`lastUsedAt`, and you must read them together:
  `"executions"` means the number is real, so **`0` means never executed**;
  `"not_instrumented"` means the skill only injects prompt text, so `usageCount` and
  `lastUsedAt` come back **`null`** — never `0`. A prompt skill is injected on every
  turn whether the model uses it or not, so counting injections would be a turn count
  wearing a usage costume: dead and live skills would score identically. `null` is the
  honest answer there; `0` would not be.
- **What a `native-*` platform skill actually adds:** the native tools — dozens of them; run
  `list_skills { visibility: "platform" }` for the current picture instead of trusting a number
  written here — are the agent's
  **default capability** — an agent with zero skills enabled already generates PDFs, writes
  Notion pages, searches Jira. What gates a tool is the **integration/data source connected
  to the agent** (and `allow_agent_write` for writes), not a skill. A `native-*` skill is
  **prompt guidance** about a tool the agent already has (when to use it, parameter shape,
  pitfalls). So enabling one grants **instruction, not capability** — enabling fifteen just
  inflates the prompt. To grant capability, connect the integration/source and allow writes.

### Manage data sources

Data sources (Jira/Notion/Google/DB/API…) are a **separate** subsystem from skills —
they live in `user_data_sources` and link to agents via `agent_data_connections`.
**Prerequisite:** for a business integration (Jira, HubSpot, Microsoft, Notion) the
integration must already be **connected via OAuth** (in the AFL UI) — but its
`integration_uuid` no longer has to be copied from a screen: **`mcp__afl__list_integrations`**
returns it, and the `integrationUuid` it gives you is *literally* the value
`create_data_source` expects in `integration_uuid`. Find existing sources/ids with
`list_data_sources`; find integration uuids with `list_integrations`; find the accepted
`source_type`s with `list_source_types`.

- **`mcp__afl__list_integrations`** (`datasources:read`)
  `{ scope?, organization_id?, integration_type?, include_disconnected?, page?, limit? }`
  lists the connected integrations (personal and/or the org's) with the identity of the
  account behind each: `connected`, `connectionStatus`, `grantedScopes` (which is what
  separates "expired token" from "that account never authorized this service") and
  `needsReconnect`. Org scope needs membership (member is enough).
  **Why the tool exists:** `list_data_sources` only ever exposes the uuid of an integration
  some source is **already** using, so an integration that was connected and never used was
  undiscoverable through the hub — the chain from "connect in the UI" to "create the source"
  broke exactly there. **Only connected ones by default** (`include_disconnected: true`
  brings the rest, as `connected: false`), because the integration row **survives the
  removal of the account**: without the filter, a removed account still reads as connected.
  If a scope can't be consulted, the answer is `partial: true` + `unavailable: [{ scope,
  reason }]` and that scope is **omitted** — absence means "could not verify", never "there
  is no integration".
  **Near-homonyms are flagged.** When two integrations of the **same type** have names
  differing only by case, accent, whitespace or punctuation — or not at all — the answer
  gains `ambiguousNames: [{ type, normalizedName, identicalNames, integrations: [...] }]`
  and a warning in `hint`. A real org has `Z2PAY` (connected) sitting next to `Z2Pay`
  (`connectionStatus: "error"`): customer data, not a bug — but whoever picks **by name**,
  and an LLM picks by name, is guessing. **Always pick by `integrationUuid`.** With no
  collision the field is absent and the response is unchanged.
- `mcp__afl__list_data_sources` (`datasources:read`) → `{ personal, organization }`.
  `scope: "all"` (the default) no longer blows up in a runtime error, and
  `scope:"all"` + `fields:"full"` is a valid combination.
  `mcp__afl__get_data_source` `{ data_source_id, organization_id? }` (`datasources:read`)
  reads one — looks in the personal scope first, then in the organization (membership
  required); the result carries `scope: "personal" | "organization"`.
- **An empty list means empty; a missing block means "not checked".** When a backing
  service is down, these reads no longer answer "there is nothing". `scope:"organization"`
  with the dependency down is an **error**; `scope:"all"` returns what it could plus
  `partial: true` and `unavailable: [{ scope, service, reason }]`, and **omits** the
  `organization` block and its counters rather than claiming `total: 0`. `list_agents`
  marks the same situation with `dataSourcesUnavailable: true` per agent (plus `partial`
  in the envelope), and `get_data_source` says it **could not verify** instead of
  "data source not found". This distinction exists because the old behaviour cost real
  work: during a 4-minute outage the hub reported healthy sources as absent, and the
  correct-looking conclusion — "the agent lost access, recreate the sources" — produces
  duplicate configuration. **Never recreate a source off a degraded read.**
- **Both reads now carry `integrationAccount`** — the identity of the account behind
  the source's `integrationUuid`:
  `{ email, label, type, connected, owner: "personal"|"organization", grantedScopes? }`
  (keys with no value are **omitted**, as everywhere else in the hub's listings; the
  whole object is absent when the source has no `integrationUuid` — or when the lookup
  itself failed, which is fail-soft — so absence never proves the account isn't there).
  Two things it answers:
  **who reads whose mailbox** — in an org with two Google accounts connected, nothing
  said which one a source used — and **"expired token" vs "that account never
  authorized this service"**: the first is `connected: false`, the second is
  `connected: true` with the scope missing from `grantedScopes`, and both surface as
  the same read error. **Note it well: a source's NAME is free text and is not
  evidence of which account it uses** — until now it was the only clue, and a source
  called "Agenda do Financeiro" could perfectly well be reading someone else's.
- **`mcp__afl__create_data_source`** (`datasources:write`):
  `{ source_type, name, organization_id?, integration_uuid?, description?, config? }`.
  With `organization_id` (or an org-scoped token) the source belongs to the
  **organization** — caller must be org **admin/owner**, and it is created org-wide (no
  group scoping; use the org UI when the source must be scoped to a group). Without an
  org it is **personal**. **`source_type` is a closed, canonical list** — the tool's own
  description now spells it out instead of trailing off in "…", so there is nothing left to
  discover by trial: `google_sheet`, `google_forms`, `google_doc`, `google_drive_file`,
  `google_drive_folder`, `google_services_data`, `google_sheet_public`, `google_doc_public`,
  `notion_page`, `notion_database`, `slack_channel`, `slack_data`, `databricks_table`,
  `databricks_genie`, `github_repo`, `api_endpoint`, `webhook`, `database_table`,
  `web_scraping`, `mcp_server`, `hubspot_data`, `linkedin_data`, `microsoft_365_data`,
  `tuya_devices`, `whatsapp_data`, `jira_data`, `education_platform`. Aliases (`jira`,
  `notion`, `databricks`, `postgres`, `sheets`, `mcp`, …) are normalized, and an
  unrecognized type is **refused listing the valid ones** rather than silently accepted.
  **`mcp__afl__list_source_types`** (`datasources:read`, no args, writes nothing) returns
  that catalog machine-readable: per type the group, whether it `requiresIntegrationUuid`,
  the `requiredConfig` without which creation is refused, and the type's `note`, plus the
  full alias map. It is the deterministic answer to "does AFL support provider X as a
  source?" — before it, the only way to find out was to attempt a write and read the error.
  Provider specifics go in `config` (e.g. Jira →
  `{ jiraProjectKeys, jiraJqlFilter }`; Notion → `{ notionDatabaseId }`; API →
  `{ apiEndpoint, apiMethod }`). A source is created **read-only** unless you pass
  `allow_agent_write: true` — plan for that: a fresh source + connect is not enough for
  the agent to write anywhere.
  - **`mcp_server` requires `integration_uuid`, and the answer tells you whether the link
    took.** The runtime resolves the MCP server by a dedicated column, so a source created
    without the integration would list, hold a full tool catalog and connect to an agent
    while executing **nothing**. Creating one without `integration_uuid` is now **refused**
    (nothing is created), and a successful create carries `mcpConnectionLinked: true`.
    If it ever comes back `false`, the answer says so with a `warning`: do **not** wire that
    source to an agent — repair it with `update_data_source { data_source_id,
    integration_uuid }`, or build it in the UI.
  - **Databricks/Genie is the exception that is NOT a data source.** `databricks_genie`
    (and `databricks_table`) *are* valid `sourceType`s, so the call is **ACCEPTED** — and
    it **enables nothing**: asking Genie a question is a **platform SKILL** capability
    (`native-databricks-genie-*`) over the organization's Databricks integration, not a
    source capability. The trap is that the accepted-and-inert source looks exactly like a
    working one. The chain that actually works:
    `list_integrations { integration_type: "databricks" }` (get the workspace) →
    **`mcp__afl__list_genie_spaces`** `{ organization_id?, integration_uuid? }`
    (`datasources:read`) → `list_skills { search: "genie" }` → `enable_agent_skill` on the
    carrier agent. The space `id` `list_genie_spaces` returns (32 hex chars) is what the
    skill asks for — it is **not** an `integration_uuid` and is not used in
    `create_data_source`.
  - **`list_genie_spaces` separates four facts that used to arrive as the same silence.**
    An **empty list** means Databricks answered and there are no spaces the service
    principal can see (the integration exists and is valid — a workspace permission
    matter). **404 without `integration_uuid`** means there is no Databricks integration in
    this scope at all — a different statement from "no spaces"; **404 with
    `integration_uuid`** means the integration **exists** in that scope (the uuid was
    already resolved) but is **not ACTIVE** — reactivate it; that is neither "no spaces"
    nor "no integration". **400** means the stored credential expired: **reconnect the
    workspace, do not recreate the integration**, it is there. **5xx / no status** means
    nothing was verified — neither the integration nor the spaces — so it justifies a retry
    and never a conclusion. (403 comes back as `blocked`: you are not an active member of
    that org.)
  - **`integration_uuid` decides the SCOPE — it is not just a tie-break.** Passing it
    defines **where** the tool looks: a **personal** integration uuid is answered in the
    **personal** scope even when the token is scoped to an organization, and the answer
    carries a `scopeNote` naming **both** scopes. This closes a real defect: the uuid came
    straight out of `list_integrations` (`personal: [{ integrationUuid: "56a70b96-…" }]`),
    was ignored, and the tool replied *"there is no Databricks integration configured in
    organization …"* — a confident answer about a different target, from which one
    concludes Databricks does not exist. A uuid **this token cannot read** (someone else's,
    or another org's) is **denied** as `blocked`, with the **same** message as a
    non-existent uuid — telling them apart would confirm that someone else's resource
    exists. That denial never says "there is no integration". An organization uuid requires
    **active membership**, and a failure to *evaluate* that authorization denies as a
    precaution — it never means "does not exist".
  - **Google sources must declare the service** (rule 10). On `google_drive_file`,
    `google_drive_folder` and `google_services_data`, `config.googleSourceType` is
    **required** — `gmail | drive | calendar | meet | sheets | docs | slides | forms`
    — and its absence is a refusal, not a warning. Minimal example:
    ```
    create_data_source({ source_type: "google_drive_file", name: "Agenda comercial",
                         integration_uuid: "...", config: { googleSourceType: "calendar" } })
    ```
    The value lives in two places (the `google_workspace_subtype` column, which the
    runtime reads first, and `config.googleSourceType`); the hub now **derives one
    from the other and writes both**, so sending either is enough. Types that already
    name the service (`google_sheet`, `google_doc`, `google_forms`) need nothing.
- **`mcp__afl__update_data_source`** (`datasources:write`):
  `{ data_source_id, organization_id?, allow_agent_write?, write_permission_note?, name?,
  description?, config?, integration_uuid?, sync_frequency?, is_active? }` — edits a source
  and, above all, **toggles whether the agent may WRITE to it** (`allow_agent_write`, the
  "Permitir escrita" switch in the AFL UI). Without it the provider's write tools (create a
  Jira issue, send mail, edit a sheet/page) are **not exposed to the agent**, even with the
  source connected — so a fresh source + connect is not enough for writes. Same scope
  resolution as `get_data_source` (personal first, then the org — org needs admin/owner);
  only the fields you send change, and `config` is **merged**. Takes effect immediately (the
  connected agents' toolset cache is invalidated).
  - **Same Google rule here, and this is the fix for a source already broken.**
    Sending `config: { googleSourceType: "calendar" }` now updates the **column the
    runtime reads** as well, not just the config — before, the config changed, the
    column stayed empty and the source stayed broken, so the obvious repair did not
    repair. The field is **not** required on update (the value may already be stored);
    an unrecognized value is refused, naming the accepted ones.
  - **An `mcp_server` source gets its server link repaired here too.** The runtime
    resolves the MCP server by the `mcpConnectionId` **column**, and neither
    `integration_uuid` nor `config` used to write it — an inert source had no repair by
    API at all. Now any update on an MCP source re-derives the column from its integration
    (or from `integration_uuid` when you send one), and the answer carries
    `mcpConnectionLinked` so you can confirm it took.
- **`mcp__afl__create_mcp_connection`** (`datasources:write`)
  `{ name, url, headers?, auth_type?, auth_value?, dedupe_by_url? }` — registers an external
  MCP server as a connection and discovers its tools (handshake + `tools/list`), returning the
  `integrationUuid` plus the tool catalog with the ids a data source needs. This is what lets you
  build an `mcp_server` source without the UI: feed the uuid into `create_data_source` with
  `config: { selectedToolIds, selectedToolsMetadata }`. Idempotent by URL; the same URL with a
  DIFFERENT credential is refused (it never overwrites another account's credential). Personal
  scope only — an org MCP connection is still a UI step. The URL must be publicly reachable
  (private IPs, internal hosts and cloud metadata endpoints are blocked).
  - **Once the source exists, its catalog is a first-class read and its tools run without
    an agent and without an LLM**: `mcp__afl__list_mcp_tools` → `mcp__afl__mcp_call_tool`
    (both `tools:read`, described under "Which tool to use"). Before them the catalog only
    existed in this tool's response (once, at creation) or inside
    `config.selectedToolsMetadata` of `get_data_source` — internal configuration doing the
    job of API documentation, under a scope a read-only collector has no other reason to
    hold.
- **`mcp__afl__connect_agent_data_source`**
  `{ agent_id, data_source_id, sync_frequency?, allow_write? }` and
  **`mcp__afl__disconnect_agent_data_source`** `{ agent_id, data_source_id }`
  (`datasources:write`) attach/detach a source — both **org-aware** (org agent → b2b path,
  caller must be org admin/owner; else the personal connection). `allow_write` grades the
  new link's write permission at connect time; omitting it inherits the source (and, on a
  re-connect, keeps whatever the link already had).
- **`mcp__afl__update_agent_data_source_connection`**
  `{ agent_id, data_source_id, allow_write? }` (`datasources:write`) changes the write
  permission of a link that **already exists**, without disconnecting and reconnecting —
  a reconnect wipes the link's configuration, and that detour was the whole reason this
  tool exists. It identifies the link by the **pair** `agent_id` + `data_source_id`, the
  two ids `list_agents` and `list_data_sources` already hand you. That pair is the natural
  key on the org side; on the personal side the key is the connection's own `id`, which
  **no hub read publishes**, so the tool resolves the pair to it internally (the same way
  `disconnect_agent_data_source` has always done). Org agent → b2b path, caller must be an
  org admin/owner; a non-admin is refused **before** anything is written.
  - **Writing is an INTERSECTION: source ∧ link.** The source grants
    (`allow_agent_write`, via `update_data_source`); the link **restricts**. The two
    defaults are deliberately opposite — a source with no opt-in does **not** write; a
    link with no opt-in **inherits** the source. So `allow_write: true` on a link whose
    source is read-only **still does not write**: it is an opt-in, never a grant. Fix the
    source, not the link. Conversely `allow_write: false` denies even when the source is
    writable.

    | link `allow_write` | source `allow_agent_write: false` | source `allow_agent_write: true` |
    |---|---|---|
    | `true` | **no write** (opt-in is not a grant) | writes |
    | `false` | no write | **no write** (the link denies) |
    | omitted / `null` (inherit) | no write | writes |

  - **What "no write" does, per source type.** On fixed-contract providers (Jira, Gmail,
    Notion, Microsoft, HubSpot…) the write tools are not exposed to the agent and the
    execution is blocked. On `mcp_server` sources the same now holds, with one caveat
    worth knowing before you design a read-only collector: MCP tools have no contract, so
    the read/write split uses the server's `annotations.readOnlyHint` when declared and,
    without it, the verb in the name (`create`/`update`/`move`/`comment`/`delete`…). Tools
    classified as writes are **omitted from the agent's prompt and refused at execution**.
    Check the split tool by tool in the `write` field of `list_mcp_tools`, and have the
    server declare `readOnlyHint` when the name misleads. Until 08/2026 this toggle did
    **nothing** on MCP sources: a collector connected with `allow_write: false` still
    listed — and could call — the server's write tools.

  - The response echoes what was **stored** (`allowWrite`) and, whenever the source could
    be read, the already-computed `effectiveAllowWrite` — that difference is what keeps
    you from repairing the wrong side ("I stored `true` and it still won't write" means
    the SOURCE is read-only). It takes effect **immediately**: the agent's toolset cache
    is invalidated on both paths.
  - **This is what makes least privilege free.** Before, one agent reading and another
    reading+writing the same Jira board, mailbox or sheet meant **duplicating the data
    source** — two rows, two syncs, two things to keep aligned. Now it is one source and
    two links: connect both, then `update_agent_data_source_connection` with
    `allow_write: false` on the one that should only read.
- **"This agent has no access to the source" now separates a MISSING CONNECTION from the
  WRONG SCOPE.** When an **organization** agent reads a source that is in fact the
  caller's **personal** one, the refusal says the problem is the source's **scope** —
  naming the source, its id and its type — and that **connecting the source to the agent
  does not fix it** (recreate the source in the organization, with an org integration, or
  read through a personal agent). The old message pointed at the connection, which sends
  you to the wrong repair.

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

## Bulk/tabular extractions — never let the agent enumerate

When the user wants a **listing, a count, or a spreadsheet** built from a connected
source ("planilha com remetente/data/assunto de todos os e-mails de maio", "quantos
cards na sprint ativa", "exporta X"), do NOT let the agent walk the items. Inline
enumeration passes every row through the model's context, which **truncates and
fabricates** — and the reply sounds just as confident when it does.

Tell it, in the `chat_with_agent` message, to use the deterministic tabular engine:

> Use a ferramenta `analisar_planilha` sobre a fonte <nome da fonte> com
> `source_filter: "<query nativa>"` e `formato_saida: "documento"`. NÃO liste os
> itens um a um.

`source_filter` is the source's **native** query, applied server-side before the
analysis. It is not optional flavor — it is what makes a scoped question honest:

- **Gmail** — native Gmail query: `after:2026/05/01 before:2026/07/01`,
  `from:x@y.com`, `label:INBOX`. Without it the whole mailbox is materialized, up to
  a 5000-message ceiling (most recent first), so a long period silently truncates.
- **Jira** — JQL, and **mandatory for anything dynamic**. The engine receives a table
  and cannot resolve "sprint ativa" (that is resolved by Jira, not by the dump).
  Asking "quantos cards na sprint ativa?" *without* `source_filter` counts the
  ENTIRE board. Use `sprint in openSprints()`, `assignee = currentUser()`, and so on.
  The source's configured project filter is applied automatically on top.

`formato_saida: "documento"` produces an `.xlsx` with the complete result — use it
whenever the answer must carry every record, not a sample.

**Honesty rule that comes with this:** only state that a result is "da sprint ativa"
(or of any other slice) if you actually passed the corresponding `source_filter`.
Otherwise say plainly that it covers the whole source.

**Gmail gotcha —** `after:`/`before:` filter by **internalDate**. Imported or
migrated mailboxes stamp the *import* date on old mail, so date distributions can
spike on the migration day. Flag that to the user instead of reading it as corrupted
data.

**Always validate the artifact** before handing it over — row count, distribution per
period, no placeholder domains like `exemplo.com`. If the agent compiled inline
instead of using the engine, the file can be truncated or invented even when the
prose around it is confident.

## Getting the most out of it

- **Ground, then synthesize:** `list_agents` → `search_knowledge_base` (or a provider
  read) to pull facts → feed them into your answer, or hand the task to
  `chat_with_agent` for the agent to reason over its own tools.
- **Prefer the specific read tool** when the user wants exact records/rows; prefer
  `chat_with_agent` when they want an interpreted answer ("resume", "analise", "o que
  mudou").
- **Iterate queries:** if `search_knowledge_base` returns `[]`, retry with more
  specific/literal terms before concluding there's no data — and remember that `[]` is a
  fact about the **index**, not about the corpus: it looks identical whether the document
  is missing, still processing, or `is_critical` (injected, never indexed). Check with
  `gerenciar_documentos op: "listar"` before telling anyone the agent does not know it.
- **Multi-turn:** keep the `conversationId` to preserve context across
  `chat_with_agent` calls.
- **Report tool errors verbatim** to the user (don't silently swallow) — they usually
  say exactly what to fix.

## Auth, scopes, ownership

- **Token org binding is a BOUNDARY, not a default.** A key issued for an
  organization (`/api-tokens` → *Organização*) reaches **only that organization**:
  - `list_organizations` returns that one org — never the other orgs the issuer
    belongs to;
  - `list_agents` returns only that org's agents — **personal agents are not
    listed**, and are refused if you pass their `agentId` anyway (hiding is not
    denying, so both are enforced);
  - passing `organization_id` of a different org is **refused** (`blocked`), even
    when the person who issued the key is a member of it — the limit is the
    KEY's, not the person's.

  Need two organizations? Issue two keys. That is also what lets you revoke one
  without taking the other down. A key with **no** org (personal) is unchanged:
  `organization_id` still selects the org, and active membership still decides.

  Before 2026-08, the binding only supplied a default and an explicit
  `organization_id` overrode it — so an org-scoped key carried its issuer's full
  cross-org authority. If a call that used to work now returns "esta chave está
  vinculada à organização …", that is this change, and the fix is a key for the
  right org.
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
  need `tools:read`/`agents:chat`; writes need `tools:write`; agent CRUD and
  `list_organization_groups`/`get_organization_topology` need
  `agents:read`/`agents:write`; skill CRUD needs `skills:read`/`skills:write`; data-source
  CRUD needs `datasources:read`/`datasources:write`; squads need their own (`squads:read`
  to list/read, `squads:run` to fire, `squads:write` to create/edit); automations likewise.
  `list_agents` / `list_organizations` need no scope. If a call returns `missing scope <x>`,
  re-authorize (or mint an API key) with that scope selected — **existing tokens must
  re-consent to gain the new `agents:*`/`skills:*`/`datasources:*` scopes**.
- You can only use agents you own or that belong to your session's organization. One
  session = one org context (+ agents you created).
- **Issuing an API key is deliberately UI-only — stop looking for the tool.** There is
  no hub tool that creates, lists or revokes a service key, and there will not be: the
  channel the agents reach must not mint the credential that grants access to it,
  otherwise an agent (or text it read) could forge its own persistent key with any
  scopes it liked, and revoking the token in use would fix nothing. The path is human:
  **`/api-tokens`** in the AFL UI (`https://app.agentsforlife.org/api-tokens`) — create
  the key, pick the **scopes** from the list above, optionally bind it to an
  organization. The `afl_live_...` value is shown **once**. Its HTTP equivalent, under
  a user JWT, is `POST /api/auth/api-tokens` (the same route the screen calls); see the
  handbook §2. When a user asks you to "create an AFL API key", point them there — do
  not go hunting for a tool.

## Reading an app — contract *and* page — `get_app_web`

**`mcp__afl__get_app_web`** `{ app_id, incluir_pagina? }` (`agents:read`) returns one
app of yours **with the full, effective manifest** — the same object the runtime reads
to decide whether a page action may execute. `listar_apps` deliberately does not carry
it (it is a listing); this is where it lives.

Two reasons to reach for it:

- **Diagnosis.** An app whose contract was accepted at creation can still be refused
  at execution. The manifest is the only place that says why: per action, `tool` is the
  native executor that actually runs, `agentId` is the carrier agent whose credential
  authorizes the source, `bind` are the **frozen** params the page cannot change,
  `params` is what it may send (with the type/pattern the server enforces), and `mode`
  (`read`/`write`) picks the door. Compare the refused action against its entry here.
- **Audit.** It is the only way, through the hub, to verify what a **published** app
  authorizes — which tools, under whose credential, with which params frozen, for which
  audience. Treat it as a governance answer, not a debugging convenience.

Alongside the manifest it returns the state that explains behaviour: `status`,
`generationStatus`, the **effective** `visibility` (the column the runtime applies),
`isActive`, `currentVersion`, `publishedAt`/`expiresAt`, spend caps and
`allowedDomains`, plus `usage`. When the manifest's own `visibility` differs from the
effective one — `editar_app_web` edits the manifest without publishing — the response
says so in `visibilityDivergence`.

**`generationSpec` is the frozen brief, not the page — never answer form questions from
it.** It is the `descricao` you passed to `criar_app_web`, written once and **rewritten
by no edit**. It describes the app as it was *asked for*. It used to be called
`description`, and the name made it the most specific, most actionable text in the whole
response: in production it announced *"a form with 6 required fields"*, field by field
with type and limit, over a page that had **five** — two had been merged into a single
500-character one, and what the brief called "a six-option selector" was free text.
Fifteen submissions were drafted against the brief; **eleven blew the real limit** and
had to be rewritten. A human opening the URL is what caught it.

**`incluir_pagina` is the portrait.** It projects the HTML that is serving *right now* —
nothing is cached, nothing is synchronised, so nothing can drift. Four formats: `campos`
(the default and the point of it: `rotulo`, `tipo`, `obrigatorio`, `limiteCaracteres`,
`opcoes` per control, plus `acoesChamadas`), `estrutura` (section index only), `texto`,
and `html` (the raw document, expensive). Use it **after every page edit** and **before
sending anyone to fill the form**. It is also the only way, through the hub, to see
*what* an edit changed: `consultar_pagina_web` rejects an app URL (it resolves S3 keys
from `criar_pagina_web`) and `listar_paginas_web` cannot see apps — leaving `updatedAt`,
which tells you *that* something changed, never *what*.

**Read `pagina.cobertura` before you read anything else — the two directions are not
equally safe.** The projection is static analysis over HTML, so **presence is a fact and
absence is not proof**, and it now says so **per dimension** (`campos`, `rotulos`,
`acoesChamadas`, `secoes`), each with `estado` + `motivo`:

- `completa` — proven sufficient (nothing in the document can change it later). Rare.
- `indeterminada` — the normal state: what is listed happened; what is missing is unknown.
- `parcial` — proven wrong (a label still holding `${...}`, a control mounted by script, an
  action dispatched through a variable). Do not use that dimension.

So, crossing `acoesChamadas` with `declaredActions`: **called-but-undeclared** rests on
presence and is always safe to report — it is a `not_allowed` in the user's face.
**Declared-but-never-called** rests on absence, and only means "capability granted for
nothing" when `cobertura.acoesChamadas.estado === "completa"`. Anywhere else it means the
projection could not see the call — open the page URL before telling anyone their buttons
are shells. That mistake was made for real: a page whose button demonstrably worked was
reported as dead code because the old projection listed 1 of 4 actions and claimed to be
complete.

`pagina.parcial` survives as a compatibility aggregate and is **true only when some
dimension is provably wrong** — `false` is *not* a certificate of completeness.

**Visibility is not settable by tool — and now it says so.** `criar_app_web`
takes `publico_alvo` and `editar_app_web` takes it too, but neither *publishes*:
the parameter only hardens validation against the intended audience. The
effective audience is the **column**, written solely by the owner publishing on
the manage screen — a human act with no tool and no MCP equivalent. This used to
be silent, and the silence was the bug: one call asked for two things, got an
excellent warning about a clamped `maxLength` and **nothing at all** about
`visibility: owner → org`, while `updatedAt` advanced and the manifest stayed
`owner`. Half the truth reads worse than none. Any audience request — through
`publico_alvo` or the aliases people actually try (`visibility`,
`manifest.visibility`, `visibilidade`, `audiencia`) — now comes back as a warning
in `data.warnings` and in the message: what you asked, what stayed in effect, and
that the path is the owner publishing. **Recognising an alias does not honour
it** — nothing persisted changed.

**`pendingPageEdit` means a rewrite is in flight — do not re-dispatch.** `editar_app_web`
answers `ok` in a few hundred milliseconds and the new page lands minutes later. In
between, `currentVersion`, `updatedAt` and `generationStatus` say exactly what they would
say if the edit had been lost — and a carrier agent, in production, read the app right
after the `ok`, saw everything unchanged and announced that *"the previous edit did not
go through"*, about to redo it on top of one still in flight. When the field is there
(`since`, `taskId`, `versionAtDispatch`), wait and read again.

**Three numbers called "version", measuring three different things.**
`currentVersion` is the **page** version; `manifestHash` fingerprints the **manifest**;
`manifestFormatVersion` (= `manifest.version`, always `1` today) is the **manifest
format** version. The platform's own agent compared `manifest.version: 1` against the
app's version 2 and concluded the contract had not changed — when the opposite was true:
on a **page** edit the manifest *must* stay identical, and an unchanged `manifestHash` is
the desired outcome, not a symptom. One caveat on the hash: it covers the **whole**
manifest, theme included, so it also moves on a look-only edit. The question it answers is
"is what I sent what is stored?", not "did the contract change?" — for the latter read the
effective manifest (`actions`, `data`); for the look, `themeChange`/`themeTokensChanged`.

Notes that save a wrong conclusion:

- **An empty `actions` is not always "app with no capability".** While `status` is
  `gerando` the contract has not been written yet (it lands at the end of generation) —
  `manifestState: "vazio"` plus the `hint` tell you which case you are in. On a `ready`
  app, empty really does mean the page may call nothing, and that is the cause of the
  runtime refusal.
- **The fix is `editar_app_web`, not a new app.** Pass `capacidades` with the **whole**
  action list (existing ones *plus* the new); it replaces, it does not append. Recreating
  loses the link, the history and every adjustment. Only `status: "falhou"` justifies
  `criar_app_web` again.
- **`despublicar: true` leaves the app `revogado`, and `revogado` is still editable.**
  Changing the contract of a live app is refused until you pass it; what comes out is
  not called "draft" — `listar_apps` shows `revogado` — but `editar_app_web` keeps
  working on it, for as many correction rounds as you need. Only republishing is a human
  act, on the manage screen; there is no MCP equivalent. Never recreate the app.
  **The theme is not contract and does not need this** — see "Changing an app's look"
  below.
- **Owner-only.** Someone else's app answers `não encontrado` — 404, never 403, and the
  same answer as an id that never existed. Resolve ids with `listar_apps`.
- **`usage` separates attempt from outcome — read all of it before concluding.**
  `invocations` counts everything that arrived; `deniedInvocations` was blocked by the
  guard (never ran, never cost); `failedInvocations` **ran and failed**;
  `succeededInvocations` delivered. Volume alone cannot tell a healthy app from a broken
  one — 12 invocations with 12 failures used to read exactly like 12 deliveries. And
  `lastInvocationStatus` (`ok`/`denied`/`error`) is the one that answers "is it broken
  *now*?": the window average cannot, since 10 consecutive failures at the end score the
  same as 10 spread out. `listar_apps` carries the two that change how a row reads
  (`failedInvocations`, `lastInvocationStatus`); the full breakdown is here.

## Linking one app to another — `navegacao`

`criar_app_web` and `editar_app_web` take **`navegacao`**: a list of
`{ nome, app, descricao? }`, where **`app` is the NAME of another app of the same owner**.
There is no URL parameter, and there will not be one — the manifest is `strict()` and the
resolver freezes the name into an id at authoring time, exactly like `fixos` freezes a tool
action's target. The page then calls `afl.nav.open('<nome>')`; the id never reaches the
page, and a page that tries to pass a URL where a declared name belongs is refused.

**Do not ask for a link by writing an anchor.** A generated page runs inside a sandboxed
iframe, and an `<a target="_blank">` there is blocked **silently** by the browser — no tab,
no navigation, no error. That was a real 🔴: an entry-point app whose three buttons did
nothing, with no symptom to report. Declaring the destination is the only path that works;
for an address outside AFL, render copyable text, never an anchor.

Destinations are checked **twice**: at publish (a destination that does not exist, is not
published, or is narrower in audience than the app being published refuses the publish) and
**per viewer** at click time, with the same rule the destination's own bundle applies — so
opening another app never bypasses that app's authorisation. Every refusal is visible in
the shell chrome; nothing about this path is allowed to be silent again.

## Changing an app's look — `tema`

`criar_app_web` and `editar_app_web` both take **`tema`** `{ modo, cor_primaria?, fonte? }`.
It is already in the `inputSchema` your client sees — the hub derives that schema from its
own tool map — so what was missing here was never access to the parameter, it was the
guidance below. Guided personalisation only: two tokens, never free CSS.

**`modo` is mandatory inside `tema`, and it decides everything else.**
`personalizado` applies `cor_primaria`/`fonte` and requires at least one of them (neither
= the call is **refused**). `padrao` means the AFL design system and **ignores** both —
`modo: "padrao"` with a colour attached is a contradiction, resolved in favour of the
explicit mode: the token is dropped and a `tema_tokens_ignorados` warning names it. **Do
not report a dropped colour as applied.** Omitting `modo` while sending a token makes the
backend infer `personalizado` and warn `tema_modo_inferido` — a safety net for a
malformed call, not the contract. Declare `modo`.

**On an edit, `personalizado` MERGES token by token.** What you send lands on top of what
was in effect; what you do not send is **preserved** — sending only `fonte` does not wipe
the `cor_primaria` the app already had. `padrao` is the opposite, and the only way to
clear: it replaces, removing the custom tokens. Theme is the one section of the manifest
that merges; `capacidades` and `dados` replace, deliberately.

**Read `themeChange`, not your own intent.** The response carries `themeChange` — `none`
(no `tema` sent), `unchanged` (what you sent is exactly what was already in effect),
`reset` (back to the design system), `merged` (new tokens over existing ones), `set`
(custom where there was none) — plus `themeTokensChanged[]` naming the tokens that
actually moved. Report from those two. "Theme: replaced" used to be printed whenever
`tema` appeared in the call, and it was false in three of the four cases.

**A theme-only edit applies straight to a LIVE app.** Colour and font authorize nothing,
so they are not contract and do not meet the unpublish gate: `editar_app_web
{ app_id, tema }` writes on the spot, does not unpublish, does not take the link down —
**never pass `despublicar: true` to change a colour**. What stays locked while the app is
live is a **contract** change (`capacidades`/`dados`), including when it arrives in the
**same call** as the theme: then the whole call is refused and **nothing is written, not
even the theme**. Resend the theme alone if only the look matters now. A theme-only edit
is also synchronous — no `task_id`, no page rewrite, no bump to `currentVersion`.

**`fonte` is a family name, not CSS — this is the one that fails.** One family
(`Georgia`) or a comma-separated stack (`Helvetica Neue, Arial, sans-serif`): letters,
digits, space, `-` and `_`, each segment starting with a letter, 80 chars max. **No
quotes** — the reflex spelling `"DM Sans", sans-serif` is **rejected**; `DM Sans,
sans-serif` passes (a name with a space needs no quotes in CSS). No parentheses either,
so no `var(--x)` and no `Roboto (Google Fonts)`; no `;`, no `{}`, no accents. Anything
outside that comes back as a `schema` issue on `theme.tokens.fontFamily`. The family must
also be web-safe or already on the machine: the app runs in a sandboxed iframe with no
access to an external font CDN. `cor_primaria` is a 6-digit hex (`#0EA5E9`).

**The reach is narrow, and this is the false report waiting to happen.** The theme moves
the design-system CSS variables — `--afl-color-primary` (plus the CTA background/label
pair derived from it, so the brand colour lands on the button's *fill*, with the label
picked for contrast) and `--afl-font-family` — which only the ready-made components
(`afl.el.table/list/form/state`) and whatever the page reads through `var(--afl-*)`
consume. A colour the generated HTML **hardcoded** does not move. So a call can come back
`themeChange: "merged"`, perfectly correct, and the user opens the link and sees no
difference. State what changed in terms of the tokens; when the visible part is
hardcoded, the fix is `instrucao` — rewriting the page of a published app is allowed and
takes nothing off the air.

## Publishing an app publicly — what the gate rejects

A **public** app is executed by anonymous visitors using the owner's credentials, so
the manifest is held to a stricter contract. The refusals name the field; these are
the ones worth knowing before you author one.

**String params need a real pattern.** Anchoring alone is not enough — `^.*$` and
`^[\s\S]{1,4000}$` are anchors around "anything". The pattern must also *restrict*:
a closed character class with a **minimum of 1**. `^[A-Za-zÀ-ÿ0-9\s.,;:!?()\-]{1,500}$`
passes; the same pattern with `{0,500}` is **rejected**, because a zero minimum makes
the restrictive part optional again. If the field should be optional, mark it
`optional: true` — do not express that with `{0,`.

**Free text is allowed where it is content, not where it selects.** A `tool` action
may declare `unsafePublicFreeText: ["<param>"]` for a param that becomes a *body* —
an issue summary/description, a comment, a page title/content. It stays forbidden on
params that decide **what the executor reaches** (`jql`, `query`, ids, project, folder):
whoever controls those controls the owner's Jira. Cap: 2000 chars, sanitized server-side.

**Writes must have their destination frozen.** A write action in an app is allowed
only from a curated list, and each tool declares what must sit in `bind` rather than
be exposed to the visitor — for `jira_criar_issue` that is `project_key`, `issue_type`
and `labels`. Mandatory `labels` is what keeps the whole thing auditable: every card
an anonymous visitor creates carries a filterable provenance mark. Exposing a selector
blocks publication, naming the param.

**`sourceId` in the manifest does not restrict anything today** — what freezes the
destination is `bind`. Do not rely on it.

**A declared carrier `agente` is honoured even where it is optional.** On a read action
in an org app it used to be dropped silently — `success: true`, `warnings: []`, and no
`agentId` in the effective manifest. It now routes the action through that agent's
connections instead of the org credential, which is what disambiguates a provider the
org has more than one source for. An `agente` that matches no agent of the owner is
refused, not ignored; on a tool that cannot run via a carrier the refusal says to drop
the field.

**Reads without a carrier agent resolve the source by organization.** An action whose
tool runs on the org credential (`jira_buscar_issues`, `jira_processar`, `datasource_*`
read) has no agent to narrow the choice, so it resolves against the org's active
sources of that type. One source resolves. **Two or more are refused** — the error
names the candidates — because picking one would mean guessing on behalf of an
anonymous visitor running under the owner's credential. Freeze the choice in `bind`:
`{ "name": "<source name>" }`, which is a declared param of those tools. A `name` that
matches nothing is also refused rather than falling back to the only source there is.

`hubspot_crm_search` behaves the same way with one difference: what it disambiguates is
the **portal**, not the source row — several sources pointing at the same HubSpot account
are not ambiguous. And it has no source param to freeze, so the fix there is to declare
the carrier `agente` on the capability (its connection picks the portal), or to keep a
single account active.

## Generators you drive through the agent, not directly

Looking for `gerar_imagem` or `editar_imagem`? They are **deliberately not hub tools**.
Ask an agent for them with `chat_with_agent` — the agent has them natively, and the
result lands in the AFL product.

The criterion is not "it generates something". It is **whether what comes back is usable
outside the chat**:

- `criar_documento` and the document editors **are** hub tools — they return a real file
  in storage with a download `url`, and the hub consumes files by URL everywhere.
- the image generators **are not** — they return markers that only the AFL chat knows how
  to materialize. Handed to an MCP client, the reply is not something you can use on your
  own; if you want that kind of content client-side, use your own model.

**AFL Apps used to be on the "not" side, and are not any more** (2026-08-07).
`criar_app_web` and `editar_app_web` are first-class hub tools now (`tools:write`),
because the old criterion did not survive contact: `editar_app_web` is a *deterministic*
operation, and routing it through `chat_with_agent` meant paying for an LLM — plus prose
to verify — to run it. Worse, two of the hub's own hints told you to use
`editar_app_web`, a tool the client did not have; a hint pointing at an unavailable tool
is worse than no hint, because it burns an attempt before failing. They are async here
(`task_id` → `get_task_result`), and none of the consent gates moved: creation lands as a
**draft** (publishing stays a human act on the manage screen), and editing the contract of
a **live** app still requires `despublicar: true`.

Reading is exposed for the same reason it always was — these generators answer before
finishing. `listar_apps` and `get_app_web` (`agents:read`) and `listar_paginas_web` are
all reachable: without the read you would be stuck between "don't call it again" and "I
can't tell whether it exists", and the safe-looking move — recreating — produces a second
app with a second link.

So: **dispatch, then verify with the reader.**

## Known limitations (current)

- **Writes are opt-in per source** (rule 6 above) — and destructive actions always
  route through `confirm_action`, which is a separate gate: enabling the source does
  not skip the confirmation.
- **A source NAME is not a unique id** — write tools resolve the source by
  `datasource_name`, and nothing prevents two sources with the same name. When two or more
  connected sources match the name EXACTLY, resolution fails listing the candidates with their
  ids (it used to pick one arbitrarily, so a write could land on the wrong source). Pass the
  id to disambiguate; a partial match still resolves to the first hit.
- **Deleting an organization group is UI-only.** The hub creates and updates groups
  (`create_organization_group` / `update_organization_group`) but does not delete one, and
  that is deliberate: the removal is a **cascade** — the group *and* every membership
  association with it — not a soft delete, and its real blast radius is a change in
  **authorization** that the hub cannot show you before the fact. Do it on the AFL org
  screen, with the members in front of you.
- **`list_integrations` reports STORED state, not a live probe.** `connected`,
  `connectionStatus` and `grantedScopes` are what the platform recorded at connect/refresh
  time; the row also survives the account being removed on the provider's side. So a
  `connected: true` is "nothing has told us otherwise", not "we just checked". When it
  matters, the proof is a read through the integration.
- **An `mcp_server` source's tool catalog is a snapshot of its configuration** —
  `list_mcp_tools`/`mcp_call_tool` only see the tools recorded when the source was last
  saved, so a tool added on the MCP server afterwards needs the source edited and saved
  again. And **deletion never goes through `mcp_call_tool`**: that permission is per
  agent, so it goes through `chat_with_agent`.
- **SharePoint covers the document LIBRARY, not the site** — navigation/read, upload,
  `create_file`, `create_folder`, `copy`, delete-to-recycle-bin. There is no site, modern page
  (`.aspx`), navigation or web part creation.
- **The open-web tools are NOT on the hub surface** — `web_search`, `scrape_webpage`
  and `crawl_website` are agent-side tools; they reach you only indirectly, through
  `chat_with_agent`. Two things about what they return change how you must read the
  agent's answer. A **scrape declares every cut it made**: `[Recorte: X de Y …]` is a
  slice the caller asked for and *can* be continued (there is a `next_offset`), while
  `[Conteúdo TRUNCADO no limite de leitura da página]` means the page is bigger than
  the read cap and there is **no** continuation — neither one is "the whole page". A
  **crawl declares whether the sweep finished** (`complete`, `stoppedReason`:
  `completed | time_budget | page_cap | aborted`): an interrupted sweep covers only the
  pages actually visited and **never justifies concluding the site lacks the
  information** — say what was seen, then widen `max_pages`/`max_depth` or sharpen the
  instructions.
- **Squad/automation tools need an org-bound token** (`squads:read` returns nothing
  useful for a purely personal token).
- No path versioning yet; the contract may evolve.

## Reference

Full handbook: `docs/afl-hub-mcp-handbook.md` in the labzz-afl repo (endpoint,
token creation, service-agent model, per-tool args, troubleshooting).
