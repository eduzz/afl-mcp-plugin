---
name: afl-recipe
description: >-
  Build a deterministic squad step (`type: 'recipe'`) through the AFL hub: fixed reads from
  Databricks, Jira and Notion with no LLM in the middle. Use when a squad step only repeats the
  same tool calls and checks totals, when the user wants to cut the cost of a squad backed by SQL,
  Jira or Notion, or asks to "cristalizar", "tirar o LLM de", "transformar em receita" or "passo
  determinístico". Covers when a recipe fits, the contract with real examples, safe interpolation
  into SQL/JQL, and turning a good run into a recipe. Complements `afl-hub` (how to call the tools)
  and `afl-design` (what to build).
---

# Deterministic squad steps (`type: 'recipe'`)

**The design doctrine is not in this file.** It lives in the platform and comes back from
`planejar_estrutura` with `tipo: squad`. This file is the signature of the step and how to build it
through the hub.

Behind the flag `SQUAD_RECIPE_STEPS`. If `create_squad` answers "passo de receita está desligado",
the flag is off: use an agent step and tell the user.

## The test

**FACT repeated with the same tool sequence goes to a recipe. JUDGMENT, synthesis and prose stay
in an agent.**

| Recipe | Agent step |
|---|---|
| Fixed SQL with parameters, totals reconciled | Interpreting those totals |
| Reading a known Jira/Notion query | Choosing what to look at next from the content |
| Stopping with a fixed sentence for known cases | Writing the recommendation |

Why it saves: every LLM step pays its own context floor just to exist, about 39.6 thousand tokens
per step measured on a real squad. A recipe spends zero tokens. Warehouse consumption is billed
elsewhere and is not LLM cost. V1 is **read-only**: any write stays with an agent step.

The usual shape is: recipe step(s) produce facts → an agent step reads `content` from its parents,
judges and writes.

## Contract

`config` of a step with `type: 'recipe'`:

| Field | Meaning |
|---|---|
| `agentId` | Carrier agent. Sources, credential and organization come from it. Same org as the squad, and reachable by whoever triggers it. |
| `inputs[]` | `{ name, from, type, optional? }`. `from` is `trigger.<field>` or `steps.<stepKey>.vars.<name>` (the step must be an **ancestor**). |
| `calls[]` | Ordered `{ key, tool, params, when?, parallelGroup?, allowEmpty?, timeoutSeconds?, maxRows?, stop?, set? }`. |
| `set[]` | After all calls: `{ name, from, type, when? }`. |
| `stop[]` | After all calls: `{ when, failure, resumo? }`. Ends the step as a declared failure. |
| `checks[]` | After `stop`: `{ left, op, right, tolerance?, message }` over numbers. |
| `output` | `{ template?, tables?: [{ call, title?, columns? }], resumoApp? }`. At least a template or a table. |

Allowed tools (closed list, each with a closed parameter list):

| Tool | Params |
|---|---|
| `datasource_databricks` | `name` or `id`, `query` (one SELECT), `max_rows` |
| `jira_buscar_issues` | `name`, `jql`, `aggregate`, `include_description`, `max_rows`, `offset` |
| `jira_ler_issue` | `issue_key`, `ler_anexos` |
| `jira_ler_comentarios` | `issue_key`, `max_rows`, `offset` |
| `notion_database_query` | `fonte`, `database_id`, `database_name`, `filter`, `sorts`, `max_results` |
| `notion_pages_read` | `fonte`, `page_id`, `page_name` |

Limits: 40 calls, 5 in a parallel group, `maxRows` up to 5000 (default 1000), per-call
`timeoutSeconds` up to 600 (default 120), step `timeoutSeconds` up to 1800, `content` up to
120,000 characters.

### Input and variable types

`integer`, `date` (YYYY-MM-DD), `month` (YYYY-MM), `email`, `enum` (with `values`), `pattern` (regex
anchored `^…$`, with `pattern`), and `union` for "integer OR e-mail":
`{ "type": "union", "of": [{ "type": "integer" }, { "type": "email" }] }`.

A value enters text only after passing its type. `set` entries carry a type too: a value read from
a result goes back through the same gate before reaching the next query.

### Slots

| Slot | Where |
|---|---|
| `{{input.x}}`, `{{vars.x}}` | Params, `when`, texts |
| `{{sql_literal(input.x)}}` | Inside `query` |
| `{{jql_literal(input.x)}}` | Inside `jql` |
| `{{calls.k.rowCount}}`, `{{calls.k.rows[0].col}}` | Texts, `when`, `set`, `checks`. Never directly in params. |

**Inside SQL or JQL, a value that can carry free text (`email`, `pattern`, a `union` with either)
must use the literal function; the plain slot is rejected on save.** `integer`, `date`, `month` and
word-only `enum` may enter raw. A call's result never enters a query directly: extract it with `set`
and a type. Slots that are not declared before use reject the save.

### `when`

`{ left, op, right | rightRef }` with `op` in `eq`, `ne`, `gt`, `lt`, `matches` (anchored regex) and
`empty`. No nesting and no boolean combinations. If you need more, you need another call or an
agent step.

### Order after each call

`stop`, then the empty check, then `set`. They run even when the call was skipped by `when`, and a
condition about a skipped call is false, so use `calls.k.skipped` to branch.

A `stop` handles the expected zero-row case. If rows may legitimately be empty and no `stop` covers
it, set `allowEmpty: true`. Otherwise zero rows fail the step.

## Examples

Databricks: resolve the producer by id or e-mail, then query.

```json
{
  "agentId": "<carrier uuid>",
  "inputs": [
    { "name": "produtor", "from": "trigger.produtor", "type": "union", "of": [{ "type": "integer" }, { "type": "email" }] },
    { "name": "mes", "from": "trigger.mes_referencia", "type": "month" }
  ],
  "calls": [
    {
      "key": "resolve",
      "tool": "datasource_databricks",
      "when": { "left": "input.produtor", "op": "matches", "right": "^[^@]+@[^@]+$" },
      "params": { "name": "Tecnologia", "query": "SELECT cli_cod AS id FROM silver.clientes WHERE cli_email = {{sql_literal(input.produtor)}}" },
      "stop": [
        { "when": { "left": "calls.resolve.rowCount", "op": "eq", "right": 0 }, "failure": "Nenhum produtor com esse e-mail.", "resumo": "E-mail não encontrado" },
        { "when": { "left": "calls.resolve.rowCount", "op": "gt", "right": 1 }, "failure": "Mais de um produtor com esse e-mail.", "resumo": "E-mail ambíguo" }
      ],
      "set": [
        { "name": "ID", "from": "calls.resolve.rows[0].id", "type": "integer", "when": { "left": "calls.resolve.skipped", "op": "eq", "right": false } },
        { "name": "ID", "from": "input.produtor", "type": "integer", "when": { "left": "calls.resolve.skipped", "op": "eq", "right": true } }
      ]
    },
    {
      "key": "vendas",
      "tool": "datasource_databricks",
      "params": { "name": "Tecnologia", "query": "SELECT COUNT(*) AS n, SUM(valor) AS bruto FROM silver.vendas WHERE produtor = {{vars.ID}} AND mes = '{{input.mes}}'" }
    }
  ],
  "checks": [{ "left": "calls.vendas.rows[0].n", "op": "gt", "right": 0, "message": "O produtor {{vars.ID}} não teve vendas em {{input.mes}}." }],
  "output": {
    "template": "Produtor {{vars.ID}}: {{calls.vendas.rows[0].n}} vendas em {{input.mes}}.",
    "tables": [{ "call": "vendas" }],
    "resumoApp": "Produtor {{vars.ID}} identificado"
  }
}
```

Jira: open bugs of one project, counted by status.

```json
{
  "agentId": "<carrier uuid>",
  "inputs": [{ "name": "projeto", "from": "trigger.projeto", "type": "pattern", "pattern": "^[A-Z][A-Z0-9]{1,9}$" }],
  "calls": [{
    "key": "bugs",
    "tool": "jira_buscar_issues",
    "allowEmpty": true,
    "params": { "name": "Jira Suporte", "jql": "project = {{jql_literal(input.projeto)}} AND type = Bug AND statusCategory != Done", "max_rows": 50 }
  }],
  "output": { "template": "Bugs abertos em {{input.projeto}}: {{calls.bugs.rowCount}}.", "tables": [{ "call": "bugs", "columns": ["key", "summary", "status"] }] }
}
```

Notion: entries of a database filtered by a typed value, kept structured.

```json
{
  "agentId": "<carrier uuid>",
  "inputs": [{ "name": "status", "from": "trigger.status", "type": "enum", "values": ["Aberto", "Fechado"] }],
  "calls": [{
    "key": "itens",
    "tool": "notion_database_query",
    "allowEmpty": true,
    "params": { "database_id": "<id>", "filter": { "property": "Status", "select": { "equals": "{{input.status}}" } }, "max_results": 500 }
  }],
  "output": { "template": "{{calls.itens.rowCount}} itens com status {{input.status}}." }
}
```

## Turning a good run into a recipe

1. Pick a run you trust and read it: `mcp__afl__get_squad_run` with `step_key` and full detail.
   `toolCalls` lists tool, status and duration in the order they ran; `params` is a preview.
2. If two runs of the same step call a different sequence of tools, it is not a fact pipeline.
   Leave it as an agent step.
3. Take the literal queries from the skill the step follows, not from the model's paraphrase.
4. Every value that varied between runs becomes an `input` or a `set`, with a type. Everything else
   is fixed text.
5. Keep the same `agentId`: the carrier's sources and credentials are what the step already used.
6. Replace the step with `update_squad_step`, run it, and compare `content` and totals against the
   good run before retiring the old one.
7. Downstream agent steps keep reading the parent's `content`; recipes can also read each other's
   `vars`.

## Validating

Everything is checked on save. A rejected definition lists each problem, so read the list and fix
it all at once. After `create_squad` or `update_squad_step`, confirm with `get_squad { step_key }`,
then `run_squad` and `get_squad_run`. `toolCalls` shows one entry per executed call, token cost is
zero, and a declared failure comes back as `FALHA:` with the `RESUMO_APP` line.

## Common errors

| Message | Fix |
|---|---|
| `tem tipo que pode carregar texto livre` | Use `{{sql_literal(...)}}` or `{{jql_literal(...)}}`. |
| `usa resultado de chamada em parâmetro` | Extract with `set` and a type, then use `{{vars.x}}`. |
| `não é ancestral deste` | The source step needs a path of edges to this one. |
| `regex precisa ser ancorada` | Start with `^` and end with `$`. |
| `grupo quantificado` / `quantificadores demais` | Regexes run synchronously: no quantified group holding `|` or another quantifier, at most 8 quantifiers (3 ranges), and text up to 256 characters. |
| `não é permitida em receita` | It is a write or outside the list. Use an agent step. |
| `tool_not_in_agent` / `source_not_in_agent` at run time | Connect the source to the carrier agent. |
| `trouxe resultado TRUNCADO` | Aggregate in the query or raise `maxRows`. A recipe does not conclude on partial data. |
| `TRUNCADO` on `jira_buscar_issues` | Listing pages at 50 and a recipe has no loop: narrow the JQL, or use `aggregate` for counts. |
| `não previu esse caso` | Add a `stop` that names the case or `allowEmpty: true`. |
| `sem_humano` or login refusals | The source uses each person's login and this run has no person (scheduled or agent-triggered). Trigger it manually or use a service-account source. |
