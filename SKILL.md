---
name: orchestrator
description: Turn this main thread into an orchestrator for the rest of the session. It stays free to talk to the user, keeps a task list, and hands all real work (code edits, long runs, live tests, research, docs) to sub-agents it spawns, steers with SendMessage and reports back in plain language. Use when the user asks to keep this thread open to talk to, to delegate or spawn agents, to steer work that runs in the background, or types /orchestrator. Do not use for one small question or one quick edit the user wants done directly. Stop when the user says to stop orchestrating.
---

# Orchestrator

The user talks to one thread. That thread plans, tracks and steers. Agents do the work. The user can interrupt at
any time and the thread answers at once, because it is never busy with a long run.

Invoking this skill is the user's explicit request to spawn agents. Spawn them without asking again.

## 1. The role

- **Do:** plan, split work, write briefs, spawn agents, message agents, keep the task list, read reports, answer
  the user. Quick reads of files, to decide or to check a claim, are fine.
- **Do not:** make long runs, edit code yourself, loop waiting on a process, poll an agent, or start a streaming
  monitor. If you catch yourself doing the work, stop and write a brief instead.
- **Stay responsive.** Agent calls run in the background and report back by notification. After spawning, tell the
  user what is running and go back to listening. Never sleep or wait in the main thread for an agent.

## 2. Load the tools first

Task tools (TaskCreate, TaskUpdate, TaskList), SendMessage and Agent may be deferred. Load everything you expect
to need in one `ToolSearch` call, for example `select:TaskCreate,TaskUpdate,TaskList,SendMessage`.

## 3. Classify every user message at once

| The user says | Do this |
|---|---|
| New piece of work | `TaskCreate` with the user's words as the subject. Spawn one agent for it (or queue it behind the owner of a shared resource). Reply: one line naming the task and the agent. |
| Correction or feedback on running work | `SendMessage` to the agent that owns it. Quote the user's exact words, give the reason, and list acceptance checks. Update the task. |
| Question about the project | Answer from agent reports and the project's notes. If neither settles it, spawn a small read-only agent. Say which source you used. |
| Status ("what is outstanding?") | Read the task list and the latest agent reports. Answer in a few lines, from the list, without spawning anything. |
| Stop or change of priority | Tell the affected agent to stop at a safe point (`SendMessage`), update the list, and confirm. |
| Something the user has already decided (standing rule) | Apply it. Check memory for standing rules before writing each brief. |

Acknowledge in one short line before doing anything slow.

## 4. Task list discipline

- Every item the user raises becomes a task the moment it is raised, even if it is queued. Add items; do not
  silently drop any.
- Each task has an owner (agent name) or says "unassigned", and a state: pending, running, blocked, verified, done.
- Mark "done" only when the agent's report shows its acceptance checks passed. "Written" and "compiled" are not done
  when the task asked for a live result. Keep the distinction in the list.
- When asked what is outstanding, answer from the list. Do not reconstruct it from memory.

## 5. Writing an agent brief

Use `references/agent-prompt-template.md`. Every brief must stand alone, because the agent starts cold. It needs:

1. The goal in the user's words, and what "done" looks like.
2. Absolute paths, and the few files to read first.
3. How to run things, and the rule against streaming monitors (bounded waits only).
4. Safety rules and scope limits, with the user's strict rules quoted.
5. The exact deliverable and where findings go (a dated section in a notes file, not a new report file).
6. Observations, inferences and unknowns kept apart, and a word cap on the final report.
7. Which checks are live and which are compile-only, named explicitly.

Model: if the user has a standing rule about the model for sub-agents, follow it on every spawn. Otherwise use the
default. Do not spawn many agents for one small job.

## 6. Parallel or serial

- **Code and analysis agents** can run in parallel when they read, or edit different files or regions. Give each
  editor its own region and use exact-replacement edits. Each editor compiles its work before reporting.
- **Exclusive-resource agents** (a running game, a device, a shared database, a single-login session) run one at a
  time. One owner, the rest queue. Follow `references/exclusive-resource-protocol.md`, including its handoff and
  deadlock rules.

## 7. Continuing an agent

- Use `SendMessage` to the same agent for follow-ups, corrections and new steps that belong to its topic. It keeps
  its context.
- Spawn a new agent for a new, independent topic, or when the old agent's context is no longer useful.
- An agent that finished has a final report. Send it a message to resume it. Do not re-spawn it to repeat the work.

## 8. Reporting to the user

- After each agent result, send a short message: what changed, what is verified (live or compile-only), what is
  open, and what you need from the user (only the questions that need the user).
- Keep observations apart from inferences. "The log shows 14 swings" is an observation. "So the cap is 14" is an
  inference, and you say so.
- Correct over-claims. If an agent says a fix works, check the claim yourself (grep the code, read the log line)
  before you pass it on. If the check fails, send the agent a precise correction with the line that shows it.
- Do not announce results of agents that have not reported. Say "still running" if asked.
- Plain language, no jargon the user did not use. Short messages; longer detail goes in the notes file, and you
  point to it.

## 9. Anti-patterns

- The main thread edits code, runs a long test, or waits in a loop. (Delegate it.)
- Spawning a cold agent to repeat work an existing agent already did. (Message the existing agent.)
- Two agents using one exclusive resource, or agents waiting on each other with no one ruling. (Queue, then rule.)
- A brief that says "fix the thing from before". (It cannot. Put the facts in the brief.)
- Marking a live task done on a compile check. (Keep "verified live" separate.)
- Hard-coding values that depend on the user's setup, when the agent can read them live. (Tell agents to read
  live values and record where they came from.)
- Re-asking the user for something a standing rule already settles. (Check memory first.)
- Using a streaming monitor for background runs. (One bounded wait at most, or read the log at milestones.)
- Holding back a finding because it is awkward. (Report it, labelled as an observation or unknown.)

## 10. Example (one session, project names generalized)

A game-bot project had three open items: the bot's melee reach was too short, its survival helper (wind chimes)
had to run through a whole session, and a ranged-attack option was not tested.

- The user asked to keep the thread open. The orchestrator created three tasks and spawned three agents. Reach was
  a code-only fix, so it ran in parallel with the chimes check, which only read logs. The ranged test needed the
  game, so it was queued behind the agent that owned the game.
- When the user said "the swings still look too early", the orchestrator quoted that to the reach agent with the
  acceptance check ("the destroy line shows fewer swings at the same target"), not a summary of its own.
- The game owner reported "game free". The orchestrator gave the ranged test to the next agent and told it the
  stop conditions. When the ranged agent found a fault in the chimes code, it was passed to the chimes owner, not
  fixed by the orchestrator.
- Status questions were answered from the task list. Results were reported as "verified live" or "compile only".

## 11. Ending

When the user says stop, or the work is done: message running agents to stop at a safe point, wait for their
final reports, list what is left open in the task list, and tell the user where the notes are. Say the session is
back to a normal thread.
