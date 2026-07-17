---
name: afl-hub
description: >-
  Operate the Agents for Life (AFL) hub via its MCP server (`mcp__afl__*` tools):
  talk to AFL agents and read data connected to them (Jira, HubSpot, Notion,
  knowledge base, databases). Use whenever the user wants to ask/act through their
  AFL agents, search an AFL agent's knowledge base, look up Jira/HubSpot/Notion or
  database data that lives in AFL, or mentions "AFL", "Agents for Life", "hub",
  "meu agente"/"my agent", or an agent by name. Requires the `afl` MCP server
  configured (`claude mcp list` → afl ✔ Connected).
---

# AFL Hub — operating the MCP at maximum capability

The `afl` MCP server exposes AFL's capabilities as tools. **Everything runs in the
context of a real AGENT you choose** (the service-agent model): the chosen agent
carries the organization, connected data sources, and OAuth credentials used to
execute. There is no generic/global execution — always pick an agent first.

## Golden workflow (always)

1. **`mcp__afl__list_agents`** (no args) → returns `{ agents: [{ id, name, description,
   type, organizationId, organizationName }] }` — personal agents plus the org agents
   your RBAC reaches (`type: "organizational"`; `organizationId`/`organizationName`
   null for personal ones).
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
  **Long-running calls are supported** (the server sends transport keepalives and
  MCP progress notifications since 2026-07-17): a call that takes minutes will NOT
  drop. Client harnesses may move a >120s call to a background task and deliver the
  result via notification — that is normal; wait for it instead of retrying.

- **Deterministic structured data** → the per-provider read tools (return raw JSON,
  no LLM in the middle, cheaper/faster):
  - `mcp__afl__search_knowledge_base` `{ agentId, query }` — hybrid semantic+keyword
    search over the agent's indexed KB. **Use specific, literal terms** (names,
    numbers, section titles); generic queries fall below the similarity threshold and
    return `[]`.
  - `mcp__afl__jira_search` `{ agentId, query, project?, maxRows? }` — `query` is
    **JQL** (e.g. `ORDER BY created DESC`, `project = AV AND status = "In Progress"`).
    Never pass natural language as JQL.
  - `mcp__afl__hubspot_search` `{ agentId, objectType, query }` — objectType is
    `contacts | companies | deals | tickets`.
  - `mcp__afl__notion_query` `{ agentId, databaseId?, query? }` — pass `databaseId`
    (the Notion database id) to query a specific DB; otherwise `query` does a
    workspace search.
  - `mcp__afl__query_database` `{ agentId, dataSourceId?, query }` — natural-language
    or SELECT-style question over a connected DB. `dataSourceId` is optional and
    auto-resolved when the agent has a single database source.

The read tools **auto-resolve** the provider's source from the agent's connections.
If an agent has more than one source of a provider, the tool returns an error asking
you to specify the source — surface that to the user.

## Bulk/tabular extractions (emails, large listings, spreadsheet exports)

When the user wants a **bulk listing or a spreadsheet** built from a connected
source (e.g. "all emails with sender/date/subject for a period", "export X to a
spreadsheet"), do NOT let the agent enumerate items one by one — inline compilation
through the agent's LLM context **truncates, duplicates, and can fabricate rows**.
Instead, instruct it explicitly in the `chat_with_agent` message to use the
deterministic tabular engine:

> Use a ferramenta `analisar_planilha` sobre a fonte <nome da fonte> com
> `source_filter: "<query nativa>"` e `formato_saida: "documento"`. NÃO liste
> itens um a um.

- **Gmail sources are analyzable tabular sources** (since 2026-07-17): the engine
  materializes the whole mailbox server-side (sender, date, subject, labels…).
  `source_filter` takes a native Gmail query, e.g.
  `in:inbox after:2026/05/01 before:2026/07/18`.
- Gotcha: Gmail `after:`/`before:` filter by **internalDate**. Imported/migrated
  mailboxes stamp the import date on old mail, so date distributions can spike on
  the migration day — flag it to the user rather than assuming corruption.
- Other tabular sources (Sheets, Excel/M365, Notion databases, Jira, HubSpot, SQL)
  go through the same engine.

## Files returned by agents

Agents return generated files as `[AFL_FILE_URL:/api/documents/d/<token>|<filename>]`.
The URL is relative — prepend the environment base URL (prod:
`https://app.agentsforlife.org`) to download (works with `curl`, token-signed).
**Always validate a generated artifact before handing it to the user** (row count,
per-period distribution, no placeholder domains like `exemplo.com`): if the agent
compiled the data inline instead of using the deterministic engine, the file may be
truncated or fabricated even when the reply sounds confident.

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
  granted scopes (typically `agents:chat` + `tools:read`) gate the tools —
  `missing scope ...` means the session lacks that scope. (A static API key via
  `claude mcp add --header "Authorization: Bearer afl_live_..."` is the fallback
  when OAuth isn't yet available in the target environment.)
- You can use agents you own, agents of your session's organization, and (since
  2026-07-17) any org agent your RBAC reaches (super-admin, or direct/group/container
  permission) — `list_agents` includes those with `type: "organizational"` +
  `organizationId`/`organizationName`, and execution applies the same RBAC.

## Known limitations (current)

- **Read-only** — no write tools (create issue, send email, etc.) yet. An expansion
  of the read-tool set (Gmail/Drive/Sheets/Calendar, Outlook/OneDrive, Slack,
  GitHub, web search, Databricks, …) is mapped but not shipped yet.
- **Org Jira** may return "Integração Jira não configurada" (pending an integration
  fix). Personal Jira, knowledge base, HubSpot, Notion, and chat work.
- No path versioning yet; the contract may evolve.

## Reference

Full handbook: `docs/afl-hub-mcp-handbook.md` in the labzz-afl repo (endpoint,
token creation, service-agent model, per-tool args, troubleshooting).
