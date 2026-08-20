---
name: afl-migrate
description: >-
  Port an existing multi-agent pipeline into AFL — one that runs today in a local
  framework (OpenSquad, CrewAI, LangGraph), in a repo of prompts, or as a runbook.
  Covers what changes shape in the crossing, which disciplines from the source do
  not transfer, and the traps that only surface once you run it. Use when the user
  wants to migrate, port or bring into AFL something that already works elsewhere —
  "migrar minha squad", "portar esse pipeline", "isso hoje roda no meu terminal",
  "tenho isso em OpenSquad/CrewAI/LangGraph" — or shows up with a repository of
  agents and asks where to start. Complements `afl-design`, which covers building
  from scratch, and `afl-hub`, which covers calling the tools.
---

# Porting an existing pipeline into AFL

`afl-design` assumes you start from a ritual. This one assumes you start from something
that **already works** — and the danger is different: you will be tempted to copy the
*shape* instead of translating the *intent*.

Read **`afl-design`** for the routing question and the build order, and **`afl-hub`** for
signatures. Do not restate them here.

Numbers below come from two measured migrations — **2026-08-04** (five agents, one
framework) and **2026-08-20** (nine steps, four agents, audited run by run). They are
orders of magnitude with a date on them, not constants. The second one is the reason this
skill has a **completion gate** at the end: it produced a squad that *looked* finished —
prompts, scoped source, least privilege, human gate — and **every agent step answered
`FALHA`** while the run sat three days in a gate waiting for approval of an empty set.

## Step 0 — three things change shape

Name them before mapping anything.

| In the source pipeline | In AFL | Why it matters |
|---|---|---|
| **A file is the handoff** — each agent writes `output/x.md`, the next reads it | **The step's return value is the handoff**, delivered only to its **direct children** | There is no shared disk between steps. A prompt saying "write to `output/x.md`" produces an agent describing a file nobody will find — and every `inputFile` the source declares is an **edge you must draw**, or data that dies where it was computed |
| **Sequence is the default** — the runner executes in order | **The DAG is the default** — no edge means parallel | The source order usually reflects the runner, not a real dependency |
| **Credentials are local** — `.env`, a logged-in browser profile, stdio MCP | **Credentials belong to the organization**, scoped per source | This is the whole point of the crossing. It is also where a local connector with no counterpart will hurt |

## The source's sequence is guilty until proven innocent

Read each agent's **declared dependencies**, not the order of the steps. In the measured
migration, five agents ran as five sequential steps and *none of them depended on
another* — the sequence existed because the runner was sequential.

With the edges removed, all five started within **39 ms** of each other: **42 s** of wall
clock against **86.5 s** of summed durations. Draw an edge only where one step consumes
another's output.

## The edge IS the file system — convert every `inputFile`, one at a time

The mirror image of the rule above, and the one that gets skipped: the source had a shared
disk, so any step could re-read `kpis.md` whenever it liked. In AFL a step receives **the
whole output of its parent steps and nothing else** — parents, not ancestors. An
`approval` gate standing between producer and consumer is a **wall, not a window**: what it
forwards is the decision, not the dossier it reviewed. When the consumer needs the data,
draw the edge **from the producer**, in addition to the gate's edge.

Measured (2026-08-20): a report step declaring **six** `inputFile`s in the source was ported
with a single parent — the gate. The KPI table, the top 20 and a queue listing that the
business rule forbids truncating never left the step that computed them. Nothing failed:
every step closed `completed`. And the gate's message asked the approver to confirm "whether
the KPIs add up" without handing them a single KPI.

**Build the conversion table before you call the DAG done. One row per `inputFile` of every
step, and no row may be left without an answer:**

| Step in the source | `inputFile` | Edge that delivers it in AFL |
|---|---|---|
| `relatorio` | `kpis.md` | `kpis → relatorio` |
| `relatorio` | `anomalias.md` | `anomalias → relatorio` |
| `revisao` (gate) | `kpis.md` | `kpis → revisao` — the approver reviews what you give them |

An intermediate step is **not** a transport. If the answer to a row is "the anomaly step
will carry it forward", read that step's own prompt: in the measured case it said, in
writing, that it does **not** produce KPIs.

## Human checkpoints do not force you to split the squad

A step of type **`approval`** pauses the run inside the DAG. It takes
`approverUserIds: [...]` **or** `approverGroupRole: "admin" | "member"` — one of them is
mandatory — plus an optional `message`, `expiresInHours` (default 168, seven days) and
`maxLoops`, the review loop described below.

Measured: a run sat in `waiting_approval` past **407 seconds** with `timeoutSeconds: 300`
on that same step. Approval is **not** governed by the step timeout; the two deadlines
govern different things. Without knowing this, the instinct is to break a 12-step
pipeline into four hand-stitched squads. Don't.

**But a checkpoint that produces a PARAMETER is a different animal from one that produces a
DECISION.** `approval` collects a decision. When the source's checkpoint emitted an
artifact that later steps read — a reference date, a period, a chosen target — porting it as
"the value comes in the trigger message" ports nothing: the trigger is free text, and four
root steps fired in parallel on an input no structure guaranteed. The first one answered
*"FALHA: the reference date was missing from the trigger"*, and it was right.

**Rule: every execution parameter has a step that fixes it and emits it.** Two ways, in
order of preference:

1. Declare the contract on the squad — **`trigger_schema`** on `create_squad`/`update_squad`,
   a list of typed fields (`key`, `label`, `type` in `string | date | number | enum`,
   `required`, `options?`, `description?`). `run_squad` then takes the values in **`inputs`**,
   a missing `required` field **refuses the run before the first step is spent**, and the
   values reach the steps as a stable `[ENTRADA DO DISPARO]` block in the trigger. The
   contract is frozen into the run's snapshot along with steps and edges.
2. When the value has to be looked up or normalized rather than typed in, make it a real
   step: an `agent` step that reads and normalizes the value, followed by an `approval` so a
   human confirms it. It costs one LLM turn to do form validation — which is why option 1
   comes first — but it is a step that **emits** the parameter, and its children inherit it
   by edge like any other output.

What is never acceptable is the third option: leaving it implicit and hoping whoever fires
the run remembers.

## What has no equivalent — and the obligation to say so in writing

**A local connector with no counterpart** — an own messaging API, an authenticated browser
profile. Replace it with a native surface. A **web app** at `visibility: org` solves this
more often than expected, and removes the token. (This used to be a table. Two of its rows
have since become capabilities, below — which is the more useful lesson: check before you
design the workaround.)

**And the class that has no replacement at all: anything whose object was code running on a
machine.** AFL executes agents, not programs. A source pipeline that shaped its PDF with a
layout library (`KeepTogether`, table-style background rules, donut wedge parameters) and
QA'd it by rasterizing the page and looking at it has **lost the object**, not gained a
different tool for it. The same goes for the shared disk and for anything the old runner
did between steps.

**Declare every one of these in writing, in the migration's own record.** The 2026-08-20
migration left all of the above behind and declared none of it, so the loss was
indistinguishable from a decision. A migration that does not name what it dropped is not
reporting a result — it is hiding a diff.

**Review loops now port directly.** The reviewer sending work back to the writer, N times,
is two fields on the DAG you already draw: an edge **`onOutcome: 'rejected'`** leaving the
`approval` step and pointing back at a step that precedes it — *where* the rejection goes —
plus **`maxLoops`** (1–5) on the gate — *how many* round trips. Neither works alone, and the
platform requires at least one `onOutcome: 'approved'` edge for the happy path. On a
rejection the stretch between target and gate reopens and the **reviewer's note lands in the
reopened step's prompt by itself** — do not hand-stitch the feedback; read `loopIteration` on
the step to know which cycle you are looking at, since the loop overwrites the same row.
`maxRetries` remains the wrong tool: it retries a *failure*, and a rejection is not one. The
workaround this section used to prescribe — a `squad` step with `waitForCompletion: true`,
re-fired by the following gate — meant a second squad to maintain for a loop the DAG now
holds; and the obvious alternative, drawing the back edge by hand, used to be *accepted* and
left the run hanging forever.

**Quarterly and four-monthly cadences now port directly.** They are not a frequency of their
own: `schedule_frequency: "monthly"` plus **`schedule_months`**, an array of the months it
runs — `[3,6,9,12]` quarterly, `[12,4,8]` four-monthly, `[1,7]` biannual — alongside the
`schedule_day_of_month` and `custom_schedule_time` that `monthly` already requires. Omitting
it means every month, and it is **rejected** with any other frequency. Port the cadence as it
is; the workaround this section used to prescribe — a monthly reminder whose message says
whether this month counts — fires 12× a year for the 3 that matter.

## Check how AFL exposes each source — the discipline may invert

**For every source in the origin, read how AFL exposes it before porting the prompt that
operates it.** It may be a data source, a native skill with parameters, or nothing.

The case that proves it: two ported agents carried the rule "always consult the metadata
catalog before writing a query". In AFL, Databricks is **not a data source** — it is a
native skill (`native-databricks-genie-perguntar`) configured with a `genieSpaceId`, and
Genie takes natural language. There is no catalog to consult, and the discipline
*inverts*: the rule that matters becomes **do not rewrite the question** before sending it
(`rawQuestionPassthrough` defaults to on), because the intermediate rewrite is where the
period gets lost, the metric gets swapped and a filter gets invented.

Copying the old discipline would have produced two agents obeying a rule that does not
apply, against a connector that works another way.

## Prove each source reaches what you think it reaches

A source's **name is not evidence of its reach**. Search for a term that exists *only* in
the intended destination, before designing anything on top of it.

Real case: an organization's Notion integration reached an internal-academy workspace and
nothing else. The three pages the pipeline depended on lived in a different workspace,
unreachable. One `notion_query` caught it — and the same call revealed that a "browsable"
publishing destination had been pointing at the wrong place for days.

**And reach is not the same as a read path. A source can be perfectly scoped and
functionally inert.** In the 2026-08-20 migration the source was the best-built object in
the whole squad — seven named tables, catalogs and schemas bounded, `allowAgentWrite:
false`, a `writePermissionNote` explaining the decision. It listed correctly in
`list_data_sources`, read correctly in `get_data_source`, and appeared as an `acessa` edge
in the org topology. **No agent could read a single row from it.** There was no read tool
for that source type at all; the source orchestrator, asked how to read it, answered
`"tool": ""` with `status: ok`; and the tabular tool the agent reached for returned `ok` in
40 ms **with no data**, which the agent narrated as "query dispatched in background, results
to follow" — a promise of a future that does not exist.

The three lessons, and they only surfaced by running it:

- **Absence of a path can arrive dressed as success.** An empty tool name with `status: ok`,
  and an `ok` in 40 ms with no rows, are both "no", said in the grammar of "yes".
- **Tool-call counters can lie in your favour.** One step made **three** calls, **all `ok`**,
  and collected nothing. The usual heuristic — `toolCallsCount: 0` means the agent only
  wrote text — does not catch this. Read the tool *results*, not the count.
- **Least privilege can switch off the last remaining read.** A step correctly marked
  `readOnly` refused a natural-language *ask* tool that writes nothing anywhere, because the
  platform's read allowlist denies what it cannot classify — and it cannot classify a tool
  that arrives through a skill. Doing the right thing closed the only door left.

So: for **every** source, execute one real read through the carrier agent before designing a
step on top of it, and read the returned rows. "It was created, connected and scoped" is
four facts about configuration and zero facts about execution.

## Classify every file in the source's `data/` before you port one prompt

This is the decision the 2026-08-20 migration got wrong, and it caused more failures than
anything else on this page. The rule:

> **`data/` that every execution uses whole → a skill.**
> **`data/` that you consult a piece of at a time → the knowledge base.**
> **Never instruct a semantic search for a literal term.**

A fixed SQL query, a ten-point checklist, the section order of a report: that is **method** —
small, stable, used on every run. Method belongs in a **skill**, which is already in the
context when the turn starts. The knowledge base is for **corpus**: large, variable,
something you retrieve a slice of.

Get it backwards and you write, as that migration did in five separate steps, instructions
of the form *"search the knowledge base for the 7-day query (literal terms: `tpsplogs`,
`plr_normalized_status`, `DATEADD(day, -7`)"*. That is asking a vector search to behave like
`grep`, and it puts the most fragile call in the pipeline in the **first act of every step** —
which is exactly where the turn budget got spent. When the same pipeline was rebuilt with the
six queries as one skill, the checklist as another and the report structure as a third, the
result was **zero tool calls to obtain method**, and the SQL appeared verbatim in the query
parameters.

**Every file gets a declared destination. None may be left without one.** In that migration
five files existed and **one** arrived — uploaded by hand through a chat — while the other
four vanished with no error, no warning and no visible gap anywhere.

**The hub does write to the knowledge base — believing otherwise is the defect.** Use
**`create_knowledge_document`** `{ agent_id, title, content | file_key | file_url,
filename?, is_critical? }` — the symmetric write to `search_knowledge_base`, and a thin
facade over the native `gerenciar_documentos` `op: "adicionar"` (same executor, same
permissions; use the native one for `op: "listar"` / `"remover"`). Indexing is
**asynchronous**: the document is stored at once and becomes searchable a moment later.
Nobody in that migration had to leave the flow — the capability was there and was not found,
because the read is called `search_knowledge_base` and the write was called something that
did not look like its pair.

**Then prove it landed, and know which proof applies:**

- **Indexed documents (the default)** — run `search_knowledge_base` with a literal term that
  exists **only** in that document. **An empty result is a migration failure, not a result** —
  and it now tells you *which* failure: the response carries
  `knowledgeBase: { total, critical, awaitingIndexing, processing }`. `total: 0` means
  **nothing was ever written** (the four lost files would have shown here, immediately);
  `total > 0` with nothing returned means the document is in the base and your query missed
  it; `awaitingIndexing`/`processing` above zero mean **wait and search again** — indexing is
  asynchronous, and verifying too early is its own false negative.
- **`is_critical: true` documents are injected into the prompt and are NOT indexed**, so
  `search_knowledge_base` will not return them and an empty search there proves nothing.
  Verify those the only way that cannot lie: ask the carrier agent to **quote a literal line**
  back to you. (Also: at most three critical documents reach an agent's prompt — the most
  recent win — so a fourth is silently absent.)

## Porting a prompt is rewriting, not copying

- **File paths go — but they do not all become searches.** "Read `strategy.md`" becomes
  *nothing at all* when `strategy.md` was classified as method: the skill is already in the
  context, so the instruction is simply "apply the query/checklist/structure you carry". It
  becomes "search your knowledge base for `<literal section title>`" **only** when the file
  was classified as corpus. "Write to `output/x.md`" becomes "your final answer **is** the
  dossier".
- **Persona, principles, vocabulary and anti-patterns stay** — that is instruction, and
  instruction must be present every turn. It belongs in the **agent's prompt**.
- **Numbers leave the prompt.** OKRs, targets, counts go to a **knowledge document** —
  they change every cycle and rot silently inside a prompt.
- **Method goes to a skill.** Prioritization framework, verdict criteria, forbidden
  vocabulary: one skill per decision, shared by the agents that make it.
- **Name the right tool for each artifact the source produced.** The one-line rule:
  **an artifact with an identity of its own → `renderizar_pdf`; an artifact that should
  wear AFL's identity → `criar_documento`.** `criar_documento` builds from structured
  content using the AFL template and **ignores HTML/CSS**, so pointing it at an artifact
  whose entire specification is layout and palette throws away exactly what the source
  specified. That migration instructed *"generate the PDF with the document creation tool"*
  for a report defined by eleven sections in fixed order, seven brand hex colors, severity
  badges and red rows above a threshold. Nothing in the path flagged the swap.
- **Add the stopping rule — and then make the platform hear it.** *"If a tool fails, answer
  `FALHA: <exact error>` and stop immediately."* Without it the agent works around the error,
  and working around it inside an asynchronous step burns the time budget and delivers
  nothing. But the rule is **prose until a step declares it**: six steps obeyed it perfectly,
  all six closed `completed`, and the DAG walked to the human gate with nothing in it. Set
  **`failureMarker: "FALHA"`** in the agent step's `config` so the declaration actually fails
  the step and cancels its descendants. Without the field, the projection only *signals* it
  (`selfDeclaredFailure`) — signalling is not deciding.

## The first run is the acceptance test — read the content, not the status

Two defects surfaced in the first real run that no schema reading would have found. Both
had the same signature: **green gate, work not done.**

- A knowledge document marked **critical** came back inert. The tool reported
  `isCritical: true`, `processingStatus: completed` and "goes straight into the prompt".
  Asked directly, the agent answered that it did not have the content. The same text as a
  **non-critical** document worked — the agent quoted the exact line.
- A step returned the *plan* instead of the work and closed as `completed`: 6.5 s,
  `hasContent: true`, 1,552 characters of "awaiting knowledge base lookup", under a header
  falsely claiming the lookup had happened and naming a source that does not exist. By the
  status projection alone it was indistinguishable from a real dossier.

Both were fixed at the platform level after that migration. They stay here as the *shape* of
the failure, not as open defects — the next one will look like this and have another cause,
which is exactly why the verification below is not optional:

- **For a document, read the reachability signal the platform reports, not the upload's
  success message.** `gerenciar_documentos` `op: "listar"` reports, per document, whether
  it is actually reaching the agent — indexed for the searchable path, injected for the
  critical one. Then confirm the one way that cannot lie: ask the agent to **quote a
  literal line** from the document back to you.
- **For a step, the signal is "produced content *having used a tool that returned data*".**
  Read the step's tool calls alongside its output — a dossier built without a single call to
  a source is a dossier written from memory, and a call that came back `ok` in 40 ms with
  nothing in it is not better. Both `toolCallsCount: 0` and `toolCallsCount: 3, all ok` have
  hidden an empty step. And the generic rule survives any improvement to the projection:
  **`completed` is not proof that work was done.** On a migration, read the entire content of
  the first run.

What kept those two defects from becoming invented data was an anti-patterns skill
instructing the agents to declare a missing source instead of filling the gap — containment
in the prompt, not in the platform. The agent with no data at all returned every numeric
cell empty and every assumption flagged "not validated". **Write that skill before the
first run, not after.**

## Calibrate against the real budget, not against what the field accepts

Two per-step settings from the source do **not** land where you expect, and both were
copied straight across in 2026-08-20 with nothing complaining.

**The step's time.** `timeoutSeconds` accepts up to **1800**, and that is the step's
deadline — but the **inline turn** of an agent step is capped at **245 s** (the transport
below it ends at 270 s and the model needs the difference to write its final answer). Steps
designed for thirty minutes got two and a half, and the tools dispatched in the last act
came back with `durationMs: 1` and "the task deadline was reached before this tool
finished". Four of the six failures in that run were this, and nothing but the failure
itself said so. Two places now do the arithmetic for you instead of leaving you with the
number you wrote: `get_squad` attaches a **`turnBudget`** block to every agent step
(`configuredTimeoutSeconds`, `effectiveInlineTurnSeconds`, `inlineTurnCapSeconds` and a note
when they diverge), and `create_squad`/`update_squad` **warn** — they do not refuse, because
the background path legitimately uses the larger number. In the run projection the step
carries `effectiveTimeoutSeconds`, with `configuredTimeoutSeconds` beside it when what you
asked for was above the cap.

So: **size each step to finish inline in under ~245 s, or push the long work to a background
task** (`executar_em_background`, ~20 min, collected by polling — this is the only path on
which a `timeoutSeconds` above the inline cap buys anything). A source step that did
"fetch the query + run heavy SQL + aggregate four weeks + compute deviations" is not one AFL
step; it is three, or one background task.

**The step's model.** A `model_tier: powerful` (or any per-step model selector) has **no**
step-level equivalent: there is no `config.model` and there will not be one, because the
model comes from the LLM matrix. The real lever is the agent's **subtype**, which the matrix
consults right after the agent's own model. Every agent created through the hub is born
`assistant`, so a migration that never touches the subtype has silently downgraded every step
the source marked as needing the strong model — which is what happened: two `powerful` steps
ran on the small model, invisibly. Set it with **`agent_type`** on
`create_agent`/`update_agent` — the same value `get_agent` returns as `agentType`, an
enumerated subtype (`analyst`, `researcher`, `advisor`, `developer`, `teacher`, `coach`,
`creative`, `support`, `custom`, `assistant`; `alter`/`role` are personal-only). Two things
change how you sequence the work:

- **Decide the subtype BEFORE creating an org agent.** `update_agent` refuses `agent_type`
  on an organization agent (the internal route does not carry the field), so in an org
  migration the subtype is a **creation-time** decision or a trip to the UI.
- **Writing a subtype does not promise a better model.** It selects the row an admin
  configured in the LLM matrix; a subtype with no row falls through to the `chat`
  functionality and nothing changes. So map the source's tier to the subtype whose *job*
  matches the step, then **verify**: `get_squad` reports, per agent step, the
  `functionality`, the model the matrix resolves and the precedence that produced it.

## Context floor is an architecture decision

Measured: **~50k input tokens per step** before any work, with agents carrying 2–4 prompt
skills and one document. The step that actually read an external source reached **187k**.
The whole run: **400,112 tokens, US$ 0.34**.

In a 5-step squad the floor alone is ~250k tokens per run. That decides how many skills to
enable per agent and whether to slice into many small steps or few large ones. You cannot
discover it before running — so budget for one throwaway run.

## The protocol

```
1.  Inventory the source        agents, steps, checkpoints, sources, data/ files, inputFiles
2.  Read the real dependencies  each agent's declared inputs — not the step order
3.  Classify every data/ file   method → skill · corpus → knowledge base · none left over
4.  Map every source            data source? native skill? no equivalent?
5.  Prove each source EXECUTES  one real read through the carrier — scoped is not readable
6.  Route each piece            afl-design's routing question
7.  Build in afl-design's order source → write → scope → agent → connect → skill → document → squad
8.  Rewrite the prompts         method comes from the skill; keep persona; stopping rule
                               + `failureMarker`; right tool per artifact
9.  Draw the DAG                one edge per `inputFile` (conversion table), `approval` at
                               checkpoints, + `onOutcome`/`maxLoops` for rejection loops;
                               parameters via `trigger_schema` or a step that emits them
10. Size the steps             inline turn ~245 s; longer work goes to background
11. Run it once ponta a ponta  with scope declared and missing sources named out loud
12. READ THE WHOLE OUTPUT      and the tool calls behind it — `completed` is not proof
13. Close the gate below       eight items; unfinished means unmigrated
14. Record it                  what worked, what was missing, what was left behind, the cost
```

Most skipped, in order: **2** (the sequence gets copied), **3** (method ends up in the
knowledge base), **5** (the source name and its scope get trusted) and **12** (the status
gets read instead of the content).

## The completion gate — eight items, and the last one closes the rest

**While an item is open, the migration is not finished.** Every one of these is something a
real migration missed while looking complete from every angle the hub offers.

- [ ] Every file in the source's `data/` has a **declared destination**: skill (method) or
      knowledge base (corpus). None left without one.
- [ ] Whatever went to the knowledge base was **verified with `search_knowledge_base`**, and
      the `knowledgeBase` census in the response was read: `total: 0` means nothing was
      uploaded; `awaitingIndexing`/`processing` mean search again later; `critical` documents
      are verified by asking the agent to quote a line. **An empty search is a diagnosis to
      act on, never a result to accept.**
- [ ] For **every `inputFile`** of every source step there is a corresponding **edge** in the
      DAG. Conversion table attached.
- [ ] Every **execution parameter** (reference date, period, target) has a step that fixes and
      emits it — or a declared trigger contract. Never "it comes in the trigger".
- [ ] No instruction asks for **method by literal term in a semantic search**.
- [ ] The steps were calibrated against the **real turn budget** (~245 s inline; longer work
      in background), not against the `timeoutSeconds` the API accepts.
- [ ] What the source did and AFL has **no equivalent** for is **written down**: code
      execution, shared disk, `model_tier`, layout libraries, local connectors.
- [ ] The first **end-to-end run** was executed and checked **by its `toolCalls`** — not by
      the text the agents wrote.

> **A migration without a verified run is not a finished migration — it is a
> well-formatted hypothesis.**

## What already ports well — do not break it while fixing the rest

The 2026-08-20 audit is not a verdict on the person who migrated. These came across right,
and they are what this skill already teaches correctly:

- **Least privilege, correctly modelled.** Only the data agent had a source connected; the
  other three had none, because in AFL they receive by edge. `readOnly: true` on every step
  except the one that writes.
- **The source scoped to the bone** — named tables, bounded catalogs and schemas,
  `allowAgentWrite: false`, and a `writePermissionNote` recording the decision.
- **The human checkpoint became an `approval` inside the DAG**, with a two-cycle rejection
  loop — the correct translation of the source's rite.
- **Real parallelism.** The four collectors fired in the same second.
- **A distribution step deliberately cut**, with the agent left `readOnly` and the instruction
  stating that sending is a separate human act. Publishing is not communicating, and the
  decision was recorded.
- **Good agent prompts**: identity, principles, forbidden vocabulary, stopping rule. The
  problem was never what the agents knew — it was what never reached them.

## What to leave behind

Not everything in the source deserves the crossing.

- **Dead knowledge files** — the ones no agent reads. Migration is the cheapest moment to
  find out.
- **Persona flourishes with no behavioral consequence.** Keep what changes an output; drop
  what only decorates the character.
- **Disciplines tied to the old connector** (see the Databricks case above).
- **Anything that only existed to work around the old runner** — file-based handoffs,
  manual ordering, retry loops and review round-trips the platform now owns.

## Anti-patterns of migration

| Anti-pattern | What it looks like later |
|---|---|
| Copying the step order | A pipeline that takes 3× longer than it needs to, forever |
| Splitting a squad per human checkpoint | Four squads hand-stitched where one `approval` step would do |
| Porting a prompt verbatim | An agent describing files nobody will ever find |
| Trusting a source's name | Two collectors silently reading the wrong workspace |
| Trusting a source's *scope* as proof it can be read | An exemplary source, connected and inert, discovered only by running it |
| Putting method in the knowledge base | Five steps opening with a vector search for `DATEADD(day, -7` |
| Uploading `data/` and not searching for it afterwards | Four of five files missing, with no error and no visible gap |
| Copying `inputFile` as prose instead of drawing the edge | KPIs that die in the step that computed them, and a gate reviewing nothing |
| Leaving a parameter to "the trigger message" | Four parallel steps failing on an input nothing guaranteed |
| Calibrating steps by `timeoutSeconds` | Thirty minutes of design running in two and a half |
| Declaring success from the status projection | A dossier of promises, filed as a dossier |
| Declaring the migration done without a run | Three days parked in a gate, looking finished |
| Migrating everything at once | No way to tell which piece broke |
