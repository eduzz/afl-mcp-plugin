---
name: afl-design
description: >-
  Design an AFL structure before building it: decide what should be an agent, a
  skill, a data source, a knowledge document, a squad or a web app; scope least
  privilege; pick the carrier agent; and verify what you built. Use when the user
  wants to set up, structure, plan or review an AFL setup — "montar/estruturar um
  agente", "criar uma skill", "organizar meus agentes", "como monto isso no AFL",
  onboarding an area or a process onto AFL, or when a write fails for permission
  and the fix is structural. Complements `afl-hub`, which covers HOW to call the
  tools; this one covers WHAT to create and in WHICH order.
---

# Designing an AFL structure

Everything in AFL runs in the context of an **agent** you choose — the agent carries
the organization, the connected sources and the credentials. So the design question
comes before the tool question. This skill answers the design question. For tool
signatures, return shapes and parameter traps, read **`afl-hub`** — do not restate
them here.

## The one question that routes everything

> **Is this capability, instruction, data, orchestration, surface, or a service role?**

| The answer is… | Build a… | The symmetric mistake |
|---|---|---|
| **Capability** — reach a system | **data source**, connected, `allow_agent_write` if it writes | creating a skill and expecting access |
| **Instruction** — how to decide, what never to do | **prompt skill** | spawning an agent just to carry a text |
| **Data** — full text, minutes, a contract | **knowledge document** | pasting 30 KB into a skill prompt |
| **Orchestration** — steps with dependencies | **squad** (a DAG) | re-chaining it by hand every run |
| **Surface** — someone outside must fill in or read | **web app**, audience `organizacao` | asking the person to dictate while you type |
| **Service role** — who carries the credential | **agent** | one agent per topic instead of per credential |

Get this wrong and the structure still *looks* right. That is the whole problem.

**Start from the ritual, not from the component.** A structure grown from the
platform's feature list becomes a collection; one grown from the recurring ritual
(register, qualify, audit, report) becomes a pipeline. List the rituals first, then
ask the routing question once per ritual.

## Build order — and what breaks if you skip a step

```
1. OAuth integration (AFL UI)      no integration_uuid without it
2. data source                     born READ-ONLY
3. allow_agent_write (if writing)  otherwise the write tools are not even exposed
4. scope the source                least privilege: project keys / JQL / folder
5. agent                           the carrier; an org agent needs prompt + admin/owner
6. connect source ↔ agent          explicit opt-in
7. skill                           instruction; enabled per agent, opt-in
8. knowledge document              full text; asynchronous — confirm processing
9. squad                           DAG; born a draft; schedule optional
10. web app                        org audience; destination frozen in `bind`; publishing is human
11. verify                         list it back and read the state, not the return value
```

The three expensive skips: **4** (a broad source hands power to every agent that
consumes it), **3** (the agent "can't write" and nobody knows why), and treating
**11** as optional.

## Agents — carriers, not personas

- **An agent is defined by what it reaches**, not by its name or its persona. Name and
  description are labels; `dataSources` is the fact.
- **Pick the carrier from `dataSources`, never from the name.** `list_agents` returns
  each source with `sourceType`, `allowAgentWrite` and, for Jira, `jiraProjects`. That
  is what answers "which agent can write in project X". Guessing from the name costs a
  failed write — and the failure can surface as a permission complaint about the wrong
  thing.
- **One source per provider per agent.** Two Jira sources (or two SharePoint sources)
  on the same agent make resolution ambiguous, and the read tools then demand you name
  the source. Design it away instead of disambiguating at every call.
- **A service agent does one job.** Split by credential and responsibility (collector /
  applier of a rule / publisher), not by subject. It keeps least privilege real and
  makes the access review short.
- **An empty `description` is debt.** `capabilitySummary` is served from cache and comes
  back `null` until it is computed; when the description is empty too, nothing but trial
  and error is left for whoever picks the agent later.
- **A visible naming convention beats a taxonomy.** A prefix (`GOV — …` for agents,
  `gov-ia-*` for skills) makes a family legible in a flat list. Listings are flat.

## Data sources — where least privilege lives

- **Scope the source, not the agent.** A Jira source restricted to one project key
  bounds every agent that consumes it. The project scope binds **all** writes —
  comment, transition, field update, create — not just issue creation.
- **Writes are opt-in per source.** A fresh source plus a connection is *not* enough to
  write; `allow_agent_write` is a separate switch and takes effect immediately.
- **A source name is not an id.** Nothing stops two sources sharing a name, and an
  exact collision makes resolution fail (by design — it used to pick one). Name sources
  for what they *reach*: `Jira LAB — Governança de IA` beats `Jira`.
- **Write down why the write is allowed.** `write_permission_note` exists for the audit
  you will run later.

## Skills — instruction, not capability

- **A skill grants instruction; a source grants capability.** The native tools are the
  agent's default capability — an agent with zero skills already writes Notion pages
  and searches Jira. Enabling fifteen skills does not widen reach; it inflates every
  turn.
- **One skill per decision, not per document.** "How to classify a track", "how to
  apply the matrix", "where each artifact gets published". One skill per source file
  produces overlapping skills and an agent that cannot tell which one governs.
- **Never hard-code a count.** Component counts change without the text changing: a
  skill claiming "6 agents, 7 skills, 2 squads" was read against a platform holding
  8/10/4. Put the **method to check** in the prompt, and make measured facts carry
  their date in the sentence — "as of 29/07 there were 8" survives; "there are 8" rots
  silently.
- **Say what must never be inferred.** The most valuable block in a mature skill is the
  list of facts that are plausible-but-wrong when reconstructed from memory. Write them
  as "never rewrite these from your head".
- **Edit with an anchor; a full rewrite is a different operation.** `prompt_injection_edits`
  applies literal, unique substitutions to the current value and **fails** when the
  anchor does not match. That failure is the cheap detector that someone else changed
  the text since you read it. Resending the whole prompt means re-emitting from memory
  a text you were not asked to rewrite — in an org skill, shared by everyone.
- **Skills are opt-in per agent.** Creating one changes nothing until it is enabled.
- **Reading usage:** a prompt skill reports `usageMeasurement: "not_instrumented"`,
  so `usageCount` comes back `null`, never `0`. It is injected every turn whether the
  model leans on it or not — a count there would be a turn count in a usage costume.

## Knowledge documents — when full text beats a skill

- **Skill = rule and method** (short, injected every turn). **Document = the text
  itself** (retrieved on demand). A 30 KB document inside a prompt is paid on every
  turn, forever.
- Ingestion is **asynchronous**: it returns `pending` and is only searchable once
  processed. Confirm before telling anyone the agent "knows" it.
- **Search it with literal terms** — section names, numbers, acronyms. Generic queries
  fall below the similarity threshold and come back empty, which reads exactly like
  "there is no data".
- **Every copy of a text drifts.** If the same content lives in a repo file, in context
  documents and on a published page, say so *inside* the skill that covers the subject
  and state that they move together. Otherwise semantic search will happily return two
  different numbers for the same question.

## Squads — a DAG with a declared time budget

- **Born a draft.** Nothing runs until it is activated; a scheduled draft never fires.
- **Two ceilings, not one.** The synchronous leg cuts at **270s**; the step ceiling
  reaches **1800s** and only buys you anything when the work goes to background.
  Design the steps around that before writing them.
- **A run freezes the definition it started with.** Raising a timeout does not rescue a
  run already in flight, and neither does retrying a step — only a new run picks it up.
- **Judge a live step by its heartbeat**, not by the absence of an anomaly flag.
- **Schedule from the hub**, including monthly rituals — a cadence you cannot express
  ends up on someone's calendar reminder, which is where rituals go to die.

## Web apps — the surface, with the destination frozen

- **Org audience, never public**, unless a specific review says otherwise (the manifest
  records it as `visibility: org`).
- **The manifest is the authorization contract.** Whatever decides *where* a write
  lands — project, issue type, labels — belongs in `bind`, out of reach of whoever
  fills the form. Free text is fine where it is *content*, never where it *selects*.
- **A mandatory provenance label** is what makes every record created through a page
  traceable back to it. Cheap, and the only thing that keeps the audit answerable.
- **Unpublishing leaves the app revoked, and revoked is still editable.** Edit it;
  never recreate. Recreating loses the link, the history and every adjustment made
  since.

## Verify what you built — the last step, not an optional one

**Read the system, not the report about the system.** A card, an agent's summary and
your own previous message describe what was true when they were written.

- List it back through the hub: agents, skills, data sources, squads and apps are all
  listable there, and an app's effective contract is readable. (Inside an agent's own
  chat the picture is narrower; do not carry a conclusion from one context into the
  other.)
- After a write, read the state — not just the `✅`. A success can still have dropped a
  field it declares in the response.
- When a card claims something was published, list the destination folder. That single
  habit caught a card that was about to be closed on a false claim.

## Three questions and the answer this skill owes

| The user says | The design answer |
|---|---|
| "My team fills a form and it becomes a Jira card." | A **web app** with an org audience and project/type/label frozen in `bind` — not a new skill, not a new agent. |
| "I created the source and connected it, but the agent won't write." | Step **3**: `allow_agent_write` is a separate switch. Then step **4**: check the source's scope covers the target. |
| "How many GOV agents are there?" | **List them.** Never answer from a prompt, a summary or this conversation. |

## Anti-patterns worth naming out loud

| Anti-pattern | What it looks like later |
|---|---|
| A skill created to grant access | The agent still cannot reach anything, and the prompt got heavier |
| One agent per subject | Ten credentials to review, none with a clear owner |
| A source scoped to the whole instance | Write access nobody intended, granted to every agent that consumes it |
| A count written into a prompt | Confident, precise, wrong — with the shape of a fact |
| The same text in three places, unnamed | Semantic search returns two different answers to one question |
| Rewriting a whole prompt to change one line | A silent edit of everything you re-emitted from memory |
| Recreating an app instead of editing it | Link, history and versions gone |
| Publishing straight to public | An anonymous visitor executing on the owner's credential |
