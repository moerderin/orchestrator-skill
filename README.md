# Orchestrator skill: design memo (2026-10-23)

## Recommendation

A user-level skill (this folder) is the right vehicle for the role, with two reference files. It loads on demand,
costs nothing in other sessions, and holds the procedure (routing table, brief checklist, resource protocol) that a
memory line cannot. Worker agent definitions can be added later if a worker needs fixed tools or a fixed model.

| Vehicle | Fit | Why |
|---|---|---|
| **Skill** (chosen) | Good | Loaded by `/orchestrator` or by description match. Its body stays in context for the session, so the role persists. Holds procedure and templates as reference files. |
| Custom subagent (`.claude/agents/*.md`) | Worker only | Defines a worker's tools and model, but a subagent cannot make the main thread an orchestrator. Useful later for pinning sub-agents to Haiku. |
| CLAUDE.md / AGENTS.md | Too heavy | Always on, in every session and every project. The role is wanted on request, not always. |
| Memory entry | Preference only | `orchestrator-role.md` records the wish; it does not carry the procedure or templates. Keep it as the pointer. |
| Output style | Wrong layer | Replaces the system prompt for the whole session and does not add tools or templates. |
| Hooks | Complement | Deterministic guards (for example blocking streaming monitors, checking a lock file). Not the role itself. |

## How it is invoked

- `/orchestrator`, or a message such as "keep this thread open for me and spawn agents to do the work". The
  description matches both.
- It ends when the user says to stop orchestrating.

## Limits

- The skill is a prompt. It cannot force the main thread to follow it; a long session may compact it, so the
  orchestrator should re-read it after compaction if behaviour drifts.
- Agent results arrive as messages when each agent stops. The main thread cannot poll them.
- The harness default discourages spawning agents unless asked. The skill counts the user's invocation as that
  request, so the wording must stay explicit about it.
- Agents start cold; the brief is the only context they get.
- Two agents on one exclusive resource (a game, a device) will deadlock unless one owner is named. The protocol
  reference covers this, but it is still judgement, not enforcement.
- Standing user rules (Haiku sub-agents, no monitors, game start allowed) live in memory. The skill tells the
  orchestrator to check memory before each brief; it does not copy them.

Enabled only here: it is not referenced from settings, CLAUDE.md or AGENTS.md.
