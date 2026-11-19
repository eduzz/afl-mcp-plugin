---
name: afl-architect
description: >-
  Drive AFL's configuration architect from Claude Code on the CLIENT engine
  (`start_architect_run` with `motor: "cliente"`): you do the discovery, the questions, the plan,
  the steps without args and the fixes through hub tools, and the platform calls no model of its
  own — it keeps the guards (per-phase perimeter, plan validation, approval, running the steps with
  args on its own, verification, undo, the run visible in the UI and in the Alter). Use when the
  user says "arquiteto no Claude Code", "configurar a conta com o arquiteto sem gastar modelo da
  plataforma", "monte isso no AFL pelo arquiteto", "rode o arquiteto aqui", or wants a multi-resource
  AFL setup (agents + sources + skills + squad + groups) built from this session with a plan they
  approve. Complements `afl-design` (what to build) and `afl-hub` (how each tool is called).
---

# The configuration architect, driven by Claude Code

**The doctrine is not in this file.** `architect_context` returns it in `orientacoes`, from the
platform (one source). Read it on every run and follow it; do not answer design questions from
memory and do not copy it here.

## When to use what

| Situation | Path |
|---|---|
| Tweak 1-2 existing resources (rename an agent, enable a skill, connect one source) | Hub tools directly (`afl-hub`). No architect. |
| A setup with 3+ resources, an org topology, a plan the person must approve and be able to undo — **you** (Claude Code) do the reasoning | **This skill**: `motor: "cliente"`. The model cost is the subscription of whoever runs Claude Code — personal use. |
| Same setup, but the platform's own architect reasons (the person is not on Claude Code, or it is an end customer) | `start_architect_run` with `motor: "plataforma"` (the default) and poll `get_architect_run` — see `afl-hub`. |
| Only deciding WHAT to build | `afl-design` (`planejar_estrutura`). |

## The loop

1. **Start.** `mcp__afl__start_architect_run { prompt, organization_id?, motor: "cliente" }` →
   `runId`, `url`. Give the person the `url`: the run is visible on the screen and in the Alter the
   whole time.
2. **Read the context.** `mcp__afl__architect_context { run_id, after_seq? }` →
   `{ motor, estado, fase, vez, motivoDaVez?, orientacoes, plano, perguntas, eventos,
   toolsDaFase, proximoPasso }`. Read `orientacoes` and follow them. `toolsDaFase` is the perimeter
   of this phase (`{nome, classe, obrigatorios}`): a tool not listed there is refused. Pass
   `after_seq` (the last `seq` you saw) to get only new events.
   **Schemas:** you do NOT know the hub tools' argument names. Before proposing a step with `args`
   or calling `architect_call_tool`, read the schema with
   `architect_describe_tools { run_id, tool_names: [...] }` (up to 15; returns `inputSchema`,
   `description`, `obrigatorios`, `classe`, `chamavelNaFase`) and use the EXACT argument names
   (e.g. `create_automation` takes `personal_agent_id`, not `agent_id`). A guessed name fails
   validation, the step fails and the plan has to be re-approved by the person.
3. **Discover (read only).** `architect_map_account` and `architect_diagnose` (both `{ run_id }`),
   plus the read tools in `toolsDaFase`. Discovery never writes, and writing **data** (sending an
   e-mail, creating a Jira issue, editing a Notion page) is never part of an architect run.
4. **Ask when information is missing.** Ask the PERSON here in Claude Code — never invent an
   answer. Record the questions with `architect_ask { run_id, perguntas[] }` (the run stops waiting
   for answers) and the person's answers with `answer_architect_questions` (shapes in `afl-hub`).
   The person may also answer on the screen; re-read `architect_context` before asking twice.
5. **Propose.** `architect_propose_plan { run_id, objetivo, motivoDaMudanca?, passos[] }`. Each
   step: `{ titulo, detalhe?, acaoTipo, toolName, alvo, args | semArgs, destrutivo, verificacao? }`.
   - Give **complete `args` whenever you can** (with the exact names from
     `architect_describe_tools`): after approval the platform runs those steps on its own, and the
     required arguments are checked when you propose. Use `semArgs` (a short string saying WHICH value is not knowable yet) only when the args
     depend on something you will learn during execution; you will run that step yourself. In
     `architect_context`, `semArgs` comes back as a boolean (`true` = tool step without args).
   - A resource created by an earlier step is referenced as `{"$passo": n, "campo": "id"}`.
   - **`destrutivo: true` on every step that removes, disconnects or replaces** — never hide it.
   - A validation refusal (dependency order, source scope, role in the org…) means **fix the plan
     and propose again**. Do not route around it with direct tool calls.
6. **Approval is the person's.** Show the plan (objective, each step, which are destructive,
   what runs without you) and call `approve_architect_plan { run_id, versao, destrutivos_aprovados? }`
   **only after an explicit "yes"**. Ask about each destructive step on its own. Never approve on
   your own initiative; the person may also approve on the AFL screen — then just carry on.
7. **Wait for your turn.** Poll `architect_context` (every ~20 s, with `after_seq`). While
   `vez: "plataforma"` the platform is running the steps with args: **do not write**. When
   `vez: "cliente"`, read `proximoPasso` and the plan: steps with `semArgs` and failed steps are
   yours.
8. **Your steps.** Read the step tool's schema first (`architect_describe_tools`). Every write goes
   through `architect_call_tool { run_id, tool_name, args }` — it matches the pending step and enters the
   undo record. Then `architect_mark_step { run_id, passo, estado: "executado"|"pulado"|"falhou",
   resultado_ref?, erro? }` with what the tool actually returned.
9. **Verify and conclude.** `architect_verify_plan { run_id }` (read), fix what it flags (again via
   `architect_call_tool`), then `architect_conclude { run_id, resumo }`. Report to the person the
   final state of each step, not your own summary alone.

## Rules

- **Schema before args.** Before proposing a step with `args` or calling `architect_call_tool`,
  read the schema with `architect_describe_tools` and use the EXACT argument names; prefer
  proposing the step with complete `args` (the platform runs it and validates the required ones at
  proposal time).
- **Validation refusal ≠ failure.** If `architect_call_tool` on a step you compose (no approved
  `args`) comes back refused only by argument validation, the step **stays pending** ("tentativa n
  de 3"): fix the arguments from the error and the schema and call again — do **not** propose a new
  plan. After the 3rd validation refusal the step fails. Any other error (5xx, timeout, 409) fails
  the step as before, and a step approved with full `args` still needs a new plan to change them.
- **FACT before claims.** Read the result of every tool before saying something exists, ran or
  worked. A step is `executado` only if the tool returned success with the resource.
- **Never call `create_agent`, `update_squad`, `create_data_source`, … directly during a run.** A
  direct call is outside the plan, outside the guards and outside the undo. Inside the run, writes
  are `architect_call_tool` only.
- **Do not mark a person's step** (an OAuth consent, a connection only they can make). Tell them
  what to do and where; the step closes when they do it.
- **Undoing** is on the run's page (`url`), as a new plan to approve.
- **Inactivity:** a client-engine run with no activity for 24 h ends as failed; the person can
  continue it from the screen.

## Server mode (headless worker)

On the server (homolog) a worker runs Claude Code headless and drives client-engine runs on its
own: when the platform hands over the turn (`vez: "cliente"`), it picks the run up. **There is no
person in this session** — the person answers and approves on the AFL screen, as on the platform
engine. In this mode:

- **Never approve a plan and never answer a question for the person.** Not even an "obvious" one.
- **Missing information → `architect_ask`, then stop.** The person answers on the screen; the
  platform hands the turn back to you after that.
- **Plan proposed (`architect_propose_plan` accepted) → stop.** Approval happens on the screen.
- **Only the `architect_*` tools are available** — no `approve_architect_plan`,
  `answer_architect_questions` or direct hub tools. Do not look for a way around it.
- **Do not poll and wait.** When you hit a point that needs the person (or `vez` is no longer
  `"cliente"`), end the turn.
- **End every turn with a one-line summary** of what you did (e.g. "Mapped the account and proposed
  a 5-step plan; waiting for approval on the screen.").

## Refusal codes

| Code | Meaning / what to do |
|---|---|
| `ARCHITECT_NOT_CLIENT_TURN` | It is the platform's turn (or the run is waiting for the person to answer/approve). Wait and read `architect_context` again. |
| `ARCHITECT_RUN_NOT_CLIENT_ENGINE` | The run is on the platform engine: poll `get_architect_run` instead. |
| `ARCHITECT_RUN_TERMINAL` | The run has ended. Report its final state; start a new run if needed. |
| `ARCHITECT_RUN_NOT_FOUND` | Not yours, or bound to another org/the personal scope. |
| `ARCHITECT_ORG_MEMBERSHIP_INACTIVE` | The person is not an active member of that org. Stop. |
| `ARCHITECT_ROLE_UNAVAILABLE` | The person's role in the org could not be checked right now (503). Retry later; do not work around it. |
| `ARCHITECT_CLIENT_ENGINE_UNAVAILABLE` | The client engine is not available on this server (503). Tell the person. |
| `ARCHITECT_CLIENT_TOOL_INSIDE_RUN` | These tools are for an external client, not for calls made inside a run. |
