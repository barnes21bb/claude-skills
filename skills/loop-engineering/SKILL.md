---
name: loop-engineering
description: Teaches "loop engineering" — designing systems that prompt and supervise AI agents on a schedule instead of a human prompting turn by turn. Use whenever the user wants to (1) learn what an agent loop or agent harness is, (2) design or build a loop, automation, scheduled agent, or autonomous workflow on Microsoft 365/Copilot Studio, Jira, GitHub, Claude (Cowork, Claude Code, Console/Managed Agents), n8n, or Oracle Fusion AI Agent Studio, or (3) evaluate/audit an existing agent or loop, e.g. "review my agent," "is this safe to run unattended," "how does my agent know it's done." Trigger even without the phrase "loop engineering" — "autonomous agent," "scheduled agent," "stop babysitting my agent," "set up an agent to handle X automatically" all qualify.
---

# Loop Engineering

## What this skill is for

Most people start with an AI agent by prompting it turn by turn: you write a message, you read what comes back, you write the next one. **Loop engineering** is the practice of designing a system that does that prompting *for* you — on a schedule, against a goal, with a built-in way to check its own work — so you can step back from the conversation without the work stopping or quietly going wrong.

Use this skill to:

1. **Teach the concepts** — explain what a loop is, how it differs from a single agent run or an "agent harness," and why most of the real engineering work lives outside the model itself.
2. **Guide loop design** — walk a user through building a specific loop, including how to do it on the platform they actually have access to.
3. **Evaluate an existing agent or loop** — review what someone has already built (a description, a config, a screenshot, a repo) and give specific, prioritized improvements.

Read the reference file that matches what the user needs — don't load all of them by default.

| User needs... | Read |
|---|---|
| An explanation of the concepts, vocabulary, or "why does this matter" | `references/concepts.md` |
| Step-by-step help designing or building a loop, on any platform | `references/building-loops.md` |
| Platform-specific detail (M365, Jira, GitHub, Claude Cowork/Code/Console, n8n, Oracle Fusion AI Agent Studio) | `references/platforms.md` |
| A review of something they already built, or a checklist to self-audit | `references/evaluation-rubric.md` |
| Prepping for a security/InfoSec review of an agent or loop, or "is this safe from a governance standpoint" | `references/governance-and-security.md` |

These five files overlap on purpose — a teaching conversation will naturally drift into design, and a design conversation will naturally drift into evaluation. Pull from more than one file when the conversation calls for it.

## The core idea, in brief

A loop is not one clever prompt. It's roughly **five building blocks plus one**:

1. **Automations** — something that fires on a schedule or event and finds work, without a human kicking it off.
2. **Isolation** — parallel work happens in its own sandboxed space (a worktree, a branch, a separate run) so concurrent agents don't collide.
3. **Skills / playbooks** — the project- or task-specific knowledge written down once, so the agent doesn't re-derive (or guess) it cold every run.
4. **Connectors / tools** — the agent's actual reach into real systems (issue trackers, repos, databases, chat). An agent that can only talk is not a loop.
5. **Sub-agents** — splitting the one who *does* the work from the one who *checks* it, because a model grading its own homework is generous with itself.
6. **Memory / state** — a record that lives outside any single run (a file, a board, a ticket) and tracks what's done and what's next, since the model forgets everything between runs even if the world doesn't.

Stacking these up takes prompting off a person's plate — but it does **not** take debugging off their plate. It just moves the debugging somewhere worse: into runs nobody is watching. That's why the harder, *non-automatable* part of loop engineering is answering four questions, every time:

- **When does it stop?** (a verifiable definition of done, not a vibe)
- **What does it actually see?** (keeping context clean and relevant across many turns)
- **What can it actually touch?** (scoped, real tools — not just the ability to talk about a fix)
- **What can tell it no?** (a checker the builder trusts enough to walk away from)

A fifth question is worth asking before a loop goes anywhere near production: **what happens if it's compromised or wrong in the worst way?** Thinking through data sensitivity, reversibility, credential scope, and a kill switch up front — before InfoSec asks — is the subject of `references/governance-and-security.md`.

See `references/concepts.md` for the full explanation, the history of the term, and the failure modes (comprehension debt, cognitive surrender, runaway token costs) worth knowing before someone wires up something they can't safely stop watching.

## How to use this with a user

- If they're asking "what is this" — teach first, using `concepts.md`, and check whether they actually want to build something before diving into platform mechanics.
- If they're asking "how do I build X" — get the goal and the platform first (see the questions in `building-loops.md`), then design step by step using the matching platform section in `platforms.md`. Don't recommend a platform-agnostic loop when they've already told you which platform they're on.
- If they're asking "is this any good / what's wrong with this" — use `evaluation-rubric.md`. Ask to see the actual thing (prompt, config, workflow export, repo, screenshots) rather than evaluating a verbal description alone; loops fail in the details, not the concept.
- If they're prepping for a security/governance review, or asking "what will InfoSec ask me" — use `governance-and-security.md`. This is a builder self-prep tool, not an audit checklist written from the reviewer's side — help them arrive at their own review already having answered the obvious questions.
- Always be honest that loop engineering is genuinely unproven at scale and token costs can spike unpredictably — don't oversell autonomy as strictly better than a human-in-the-loop approach. The right call depends on how much the user trusts their checker and how expensive a wrong answer is.
