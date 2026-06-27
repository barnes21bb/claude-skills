---
name: context-engineering
description: Teaches "context engineering" — deciding what actually goes into an AI agent's context window (system prompt, files, tool output, memory) and how to keep it useful as work gets longer or more automated. Use whenever the user asks about context windows, "context rot," what belongs in a prompt vs. a file vs. retrieved on demand, an agent that "forgets things" or repeats itself, a system prompt or CLAUDE.md that's grown too long, designing a knowledge base / memory system for an agent, RAG vs. just including the document, or compacting/summarizing long-running conversations. Trigger even without the phrase "context engineering" — "my agent's prompt is huge," "it keeps losing track of what we discussed," "should I paste the whole doc or just relevant parts," "design a memory system" all qualify.
---

# Context Engineering

## What this skill is for

Every agent run is shaped by what's actually in its context window at the moment it responds — the system prompt, the conversation so far, any files or tool output it's pulled in, anything written to memory. **Context engineering** is the practice of deciding, deliberately, what belongs in that window, what belongs outside it until needed, and how to keep it useful as a task gets longer, more automated, or more repetitive. It's the layer underneath everything else an agent does: a perfectly designed tool or workflow still fails if the model can't actually see the right information at the right moment, and a botched prompt still works for a while if the underlying context is clean.

This is the layer beneath [`loop-engineering`](../loop-engineering/SKILL.md) — loop engineering assumes a well-managed context and focuses on scheduling, verification, and tool access *across* runs; this skill is about what's inside any single run's context, whether that run is part of a loop, a one-off agent session, or a plain conversation.

Use this skill to:

1. **Diagnose context problems** — figure out why an agent is forgetting things, contradicting itself, repeating questions, or producing worse answers as a session goes on.
2. **Design context from scratch** — decide what goes in a system prompt vs. a referenced file vs. something retrieved on demand vs. left out entirely, for a new agent, skill, or workflow.
3. **Fix an existing context that's grown unwieldy** — a system prompt or CLAUDE.md-style file that's ballooned, a RAG setup pulling in too much or too little, a long conversation that's degrading.

Read `references/techniques.md` for the full set of techniques, the symptoms-to-fixes diagnostic table, and worked examples. Don't reproduce the whole file in conversation — pull the specific technique or row that matches what the user is dealing with.

## The core idea, in brief

A context window is not free, and it is not infinite, even when the platform's limit is large. Two costs apply well before the actual token ceiling is reached:

- **Cost in tokens/latency** — every token in context is paid for and slows the response, whether or not the model actually needs it for this turn.
- **Cost in attention** — models perform worse, not just slower, as context fills with irrelevant, stale, or contradictory material. This is sometimes called **context rot**: quality degrading as the window grows, even when the limit isn't hit. A long context full of mostly-irrelevant material is worse than a short, well-curated one — more context is not automatically better context.

The practical job is sorting information into the right place:

- **In the system prompt / always-loaded file** — only what's needed on *every* turn, because it's paid on every turn whether relevant or not.
- **In a referenced file, loaded on demand** — knowledge that's needed sometimes, organized so the agent (or a retrieval step) can find and pull in the relevant slice without loading all of it. This is progressive disclosure: structure so the model finds what it needs without reading everything.
- **Retrieved per-query rather than included wholesale** — large or frequently-changing knowledge bases, where "include everything" doesn't scale and isn't necessary for any single question.
- **Summarized or compacted** — long-running conversation history that's served its purpose; keep the decisions and current state, drop the back-and-forth that produced them.
- **Left out entirely** — the default for anything not clearly needed. The bias should run toward leaving things out and pulling them in when actually needed, not toward including everything "just in case."

## How to use this with a user

- If they're asking "what is this" or "why does my agent get worse over a long session" — explain context rot and the cost-of-attention point above, then move to diagnosis.
- If they describe a symptom (forgetting, repeating itself, contradicting earlier turns, a prompt that's grown huge, a RAG setup that returns junk) — go straight to the diagnostic table in `references/techniques.md` rather than re-deriving first principles each time.
- If they're designing something new — walk through the sort above (always-loaded / on-demand / retrieved / left out) for their actual content before recommending a specific mechanism (a system prompt section, a skill file, a vector store, a database).
- If the conversation is really about scheduling, verification, or tool/connector access *across multiple runs* rather than what's inside one run's context, that's [`loop-engineering`](../loop-engineering/SKILL.md) — point there instead of stretching this skill to cover it.
- Be concrete: ask to see the actual system prompt, file, or transcript rather than reasoning about it in the abstract. Context problems live in the specific structure and wording, not just the general approach.
