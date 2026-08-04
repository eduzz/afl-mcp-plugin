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

Every number below comes from one measured migration on **2026-08-04** — one framework,
one pipeline, one organization. They are orders of magnitude with a date on them, not
constants.

## Step 0 — three things change shape

Name them before mapping anything.

| In the source pipeline | In AFL | Why it matters |
|---|---|---|
| **A file is the handoff** — each agent writes `output/x.md`, the next reads it | **The step's return value is the handoff** | There is no shared disk between steps. A prompt saying "write to `output/x.md`" produces an agent describing a file nobody will find |
| **Sequence is the default** — the runner executes in order | **The DAG is the default** — no edge means parallel | The source order usually reflects the runner, not a real dependency |
| **Credentials are local** — `.env`, a logged-in browser profile, stdio MCP | **Credentials belong to the organization**, scoped per source | This is the whole point of the crossing. It is also where a local connector with no counterpart will hurt |

## The source's sequence is guilty until proven innocent

Read each agent's **declared dependencies**, not the order of the steps. In the measured
migration, five agents ran as five sequential steps and *none of them depended on
another* — the sequence existed because the runner was sequential.

With the edges removed, all five started within **39 ms** of each other: **42 s** of wall
clock against **86.5 s** of summed durations. Draw an edge only where one step consumes
another's output.

## Human checkpoints do not force you to split the squad

A step of type **`approval`** pauses the run inside the DAG. It takes
`approverUserIds: [...]` **or** `approverGroupRole: "admin" | "member"` — one of them is
mandatory — plus an optional `message` and `expiresInHours` (default 168, seven days).

Measured: a run sat in `waiting_approval` past **407 seconds** with `timeoutSeconds: 300`
on that same step. Approval is **not** governed by the step timeout; the two deadlines
govern different things. Without knowing this, the instinct is to break a 12-step
pipeline into four hand-stitched squads. Don't.

## What has no equivalent

| In the source | In AFL | Do this |
|---|---|---|
| Review loop (the reviewer sends work back to the writer, N times) | **Absent.** A DAG has no back edge, and `maxRetries` retries a *failure*, not a rejection | A `squad` step with `waitForCompletion: true` — the parent pauses in `waiting_subrun` and the child's output feeds the next steps — re-triggered by the following gate. Less automatic, and visible |
| A local connector with no counterpart (own messaging API, an authenticated browser) | Depends | Replace it with a native surface. A **web app** at `visibility: org` solves this more often than expected, and removes the token |

**Quarterly and four-monthly cadences now port directly.** They are not a frequency of their
own: `schedule_frequency: "monthly"` plus **`schedule_months`**, an array of the months it
runs — `[3,6,9,12]` quarterly, `[12,4,8]` four-monthly, `[1,7]` biannual — alongside the
`schedule_day_of_month` and `custom_schedule_time` that `monthly` already requires. Omitting
it means every month, and it is **rejected** with any other frequency. Port the cadence as it
is; the workaround this table used to prescribe — a monthly reminder whose message says
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

## Porting a prompt is rewriting, not copying

- **File paths go.** "Read `strategy.md`" becomes "search your knowledge base with literal
  terms". "Write to `output/x.md`" becomes "your final answer **is** the dossier".
- **Persona, principles, vocabulary and anti-patterns stay** — that is instruction, and
  instruction must be present every turn. It belongs in the **agent's prompt**.
- **Numbers leave the prompt.** OKRs, targets, counts go to a **knowledge document** —
  they change every cycle and rot silently inside a prompt.
- **Method goes to a skill.** Prioritization framework, verdict criteria, forbidden
  vocabulary: one skill per decision, shared by the agents that make it.
- **Add the stopping rule.** *"If a tool fails, answer `FAILURE: <exact error>` and stop
  immediately."* Without it the agent works around the error, and working around it inside
  an asynchronous step burns the time budget and delivers nothing.

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

Both cases are why the verification below is not optional:

- **For a document, read the reachability signal the platform reports, not the upload's
  success message.** `gerenciar_documentos` `op: "listar"` reports, per document, whether
  it is actually reaching the agent — indexed for the searchable path, injected for the
  critical one. Then confirm the one way that cannot lie: ask the agent to **quote a
  literal line** from the document back to you.
- **For a step, the signal is "produced content *having used a tool*".** Read the step's
  tool calls alongside its output — a dossier built without a single call to a source is a
  dossier written from memory. And the generic rule survives any improvement to the
  projection: **`completed` is not proof that work was done.** On a migration, read the
  entire content of the first run.

What kept those two defects from becoming invented data was an anti-patterns skill
instructing the agents to declare a missing source instead of filling the gap — containment
in the prompt, not in the platform. The agent with no data at all returned every numeric
cell empty and every assumption flagged "not validated". **Write that skill before the
first run, not after.**

## Context floor is an architecture decision

Measured: **~50k input tokens per step** before any work, with agents carrying 2–4 prompt
skills and one document. The step that actually read an external source reached **187k**.
The whole run: **400,112 tokens, US$ 0.34**.

In a 5-step squad the floor alone is ~250k tokens per run. That decides how many skills to
enable per agent and whether to slice into many small steps or few large ones. You cannot
discover it before running — so budget for one throwaway run.

## The protocol

```
1.  Inventory the source        agents, steps, checkpoints, sources, knowledge files
2.  Read the real dependencies  each agent's declared inputs — not the step order
3.  Map every source            data source? native skill? no equivalent?
4.  Prove each source's reach   search a term that exists only in the right destination
5.  Route each piece            afl-design's routing question
6.  Build in afl-design's order source → write → scope → agent → connect → skill → document → squad
7.  Rewrite the prompts         drop file paths and numbers; keep persona; add the stopping rule
8.  Draw the DAG                edges only for real dependencies; `approval` at checkpoints
9.  Run it once                 with scope declared and missing sources named out loud
10. READ THE WHOLE OUTPUT       and the tool calls behind it — `completed` is not proof
11. Record it                   what worked, what was missing, and what the run cost
```

Most skipped, in order: **2** (the sequence gets copied), **4** (the source name gets
trusted) and **10** (the status gets read instead of the content).

## What to leave behind

Not everything in the source deserves the crossing.

- **Dead knowledge files** — the ones no agent reads. Migration is the cheapest moment to
  find out.
- **Persona flourishes with no behavioral consequence.** Keep what changes an output; drop
  what only decorates the character.
- **Disciplines tied to the old connector** (see the Databricks case above).
- **Anything that only existed to work around the old runner** — file-based handoffs,
  manual ordering, retry loops the platform now owns.

## Anti-patterns of migration

| Anti-pattern | What it looks like later |
|---|---|
| Copying the step order | A pipeline that takes 3× longer than it needs to, forever |
| Splitting a squad per human checkpoint | Four squads hand-stitched where one `approval` step would do |
| Porting a prompt verbatim | An agent describing files nobody will ever find |
| Trusting a source's name | Two collectors silently reading the wrong workspace |
| Declaring success from the status projection | A dossier of promises, filed as a dossier |
| Migrating everything at once | No way to tell which piece broke |
