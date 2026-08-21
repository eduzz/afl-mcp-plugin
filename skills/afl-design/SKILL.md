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

**The doctrine is not in this file.** It lives in the platform (`architecture-doctrine.ts`, AFL
monorepo), served cut to the case inside one tool result. A second long copy drifts, so design
guidance you are tempted to write here belongs there, and this file stays a pointer.

## What to do

1. **`mcp__afl__planejar_estrutura`** — read-only, four actions, in this order:
   `mapear` (what exists in scope) → `diagnosticar` (the gaps) → `planejar`
   (`tipo`: `agente | skill | fonte | documento | squad | automacao | app | grupo`) →
   `verificar` (after building — the last step, not an optional one).
   **`planejar` takes two calls.** First without `passos`: it returns the doctrine for that
   `tipo` and the closed vocabulary of step actions, and **persists nothing**. Then again with
   `objetivo` **and** `passos` together — the plan is written whole, in one transaction, because
   a plan with no steps does not exist. Sending `plano_id` **resumes** a plan for reading (its
   checklist and what is still pending); it does not append steps — if the path changed, design
   a new plan.
2. **Follow the plan it returns.** It carries the doctrine for that type and names the execution
   tool per step. Never invent a step it did not name, nor answer the design question from memory.
3. **Execute with the tools from `afl-hub`** (signatures, return shapes, parameter traps).
   Porting something that already runs elsewhere? Read **`afl-migrate`** first.

## Four things that only apply to an MCP client

- **The hub has no chat session.** Every call is stateless and carries its own `agentId`.
  Pick the carrier agent from `list_agents` → `dataSources`, never from the name.
- **A destructive write does not execute on its own.** It returns `pending_confirmation`
  — nothing happened yet — and only `confirm_action` runs it.
- **Prose is not evidence.** Cross every claim against the block *"AFL · registro
  verificável do turno"*: a tool not listed there with status `ok` did not run.
- **No `planejar_estrutura` in `tools/list`?** The server is older than this skill: take
  the design question to the user's Alter via `chat_with_agent`, and say that is what you did.
