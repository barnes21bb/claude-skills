# Context Engineering Techniques

## Contents
- Why context rot happens
- The core techniques
- A diagnostic table: symptom → likely cause → fix
- Worked examples
- Relationship to loop engineering

## Why context rot happens

A context window holds everything the model conditions its response on: system prompt, conversation history, file contents, tool output, retrieved chunks. Two distinct problems show up as that fills:

- **Dilution.** The signal-to-noise ratio drops. If 90% of the window is stale tool output or resolved side-discussion, the 10% that actually matters has to compete with it, and the model's attention isn't infinite or perfectly selective.
- **Contradiction.** Long sessions accumulate corrections, dead ends, and superseded decisions. If the context contains both "use approach A" and the later "actually, use approach B instead," the model has to resolve that conflict every turn, and it doesn't always resolve it the way the person intended.

Neither problem requires hitting the token limit to bite — quality can degrade well before that, which is why "the window is big enough" is not the same question as "the context is well-managed."

## The core techniques

### Structure over prose

Information the model needs to scan or reference (a list of fields, a decision log, a set of rules) should be structured — tables, headers, short bullets — rather than buried in paragraphs. Structure makes it cheaper for the model to find the relevant piece without rereading everything, and cheaper for a human to spot what's stale.

### Progressive disclosure

Don't load everything by default. Organize knowledge so the high-level entry point (a SKILL.md, a README, an index) is small and always loaded, while the detailed content lives in separate files loaded only when that specific topic comes up. The entry point's job is routing, not explaining — a short table mapping "if the user needs X, read file Y" beats inlining all of Y's content into the entry point "just in case." This is also why a skill or knowledge base with many reference files is often *more* token-efficient than one giant file, as long as the routing layer is good.

### Separation of concerns across memory layers

Not all persistent information should live in the same place or update at the same cadence:

- **Ephemeral** — true only for this run or this conversation (today's specific task, a value just computed). Belongs in the active context, not saved anywhere.
- **Durable / slow-changing** — true across runs and changes rarely (project conventions, a team's standing preferences, how a system is architected). Belongs in a file loaded at the start of relevant sessions.
- **Evolving / event-driven** — changes as things happen and needs updating when they do (current ticket status, what's been tried already, an ongoing project's state). Belongs in a place that gets *written*, not just read — a state file, a board, a database row — so the next run sees the current truth instead of a stale snapshot baked into a prompt.

Mixing these up is a common root cause of confusion: writing evolving state into a system prompt means it's stale the moment anything changes; writing durable conventions into ephemeral conversation means they vanish at the end of the session and get re-explained every time.

### Tool output hygiene

Raw tool output (a full API response, an entire file, a long log) is often much larger than what's actually needed. Prefer tools/queries that return only the relevant slice, summarize or truncate verbose output before it re-enters context, and avoid re-fetching or re-pasting the same large output multiple times in one session. A good rule: if a human wouldn't read the whole thing, the model probably doesn't need the whole thing either — extract what's relevant and discard the rest.

### Retrieval vs. inclusion

For knowledge bases beyond a certain size (more documents than fit comfortably loaded at once, or content that changes too often to keep manually curated), retrieve the relevant slice per query rather than including everything up front. The trade-off: retrieval adds a step that can miss relevant content if the retrieval itself is poor (bad chunking, bad query), while full inclusion guarantees coverage but costs tokens and attention on every single turn even when most of it is irrelevant to that turn. As a rule of thumb: a handful of documents that are nearly always relevant → just include them; a large or fast-changing corpus where most documents are irrelevant to any given question → retrieve.

### Compaction and summarization

For long-running conversations or multi-step tasks, periodically compress what's accumulated: keep the decisions made, the current state, and anything still open; drop the exploratory back-and-forth that led there. This is most valuable right before a natural transition point (handing off to a fresh agent, starting a new phase of work, a context window approaching its limit) — compact deliberately at that point rather than letting the system truncate arbitrarily or letting the conversation degrade until something breaks.

### Context budget allocation

Treat the context window like a budget with line items, not an undifferentiated pool: roughly, how much goes to standing instructions, how much to reference material, how much to live conversation/tool output, how much kept in reserve for the current task. When something is added, something else should usually be evaluated for removal — a context that only ever grows, never prunes, is the most common path to context rot.

## A diagnostic table: symptom → likely cause → fix

| Symptom | Likely cause | Fix |
|---|---|---|
| Agent re-asks something already established earlier in the same session | The earlier answer is buried in volume, or was in a part of context that got truncated | Surface key decisions more prominently (structure over prose); compact periodically so decisions survive even as raw history gets trimmed |
| Agent contradicts an earlier decision or follows stale guidance | Context holds both the old and new instruction with no clear signal which wins | Remove or explicitly supersede outdated content rather than appending corrections on top of it |
| System prompt or CLAUDE.md-equivalent has grown very long and hard to maintain | Everything was added to the always-loaded file instead of split into on-demand references | Apply progressive disclosure: move detail into separate reference files, leave a routing table in the main file |
| Agent's answers get noticeably worse deeper into a long session, even though nothing is technically wrong | Context rot — dilution from accumulated tool output, dead ends, resolved side-discussion | Compact: summarize what's settled, drop what's resolved, start a fresh session or sub-task if the work has a natural seam |
| RAG/retrieval setup returns irrelevant chunks, or misses the chunk that actually answers the question | Poor chunking, overly broad or narrow queries, or a corpus where full inclusion would actually have been simpler | Re-evaluate chunk boundaries and query construction; if the corpus is actually small, consider just including it instead of retrieving |
| Agent seems to ignore part of a long document it was given | Relevant content diluted by surrounding irrelevant material, or buried past where attention is strongest | Extract just the relevant section before including it, rather than pasting the whole document and hoping the model finds the right part |
| Each new run/session starts cold, missing context from before | Evolving state was kept only in conversation (ephemeral) instead of written somewhere durable | Add a state file, board, or record the next run actually reads, per the separation-of-concerns layers above |
| Tool calls return huge raw output that crowds out everything else | No post-processing between the tool and the context window | Summarize, filter, or paginate tool output before it re-enters context; don't pass raw API responses straight through by default |

## Worked examples

**A CLAUDE.md-style file that's grown to several thousand lines.** Symptom: slow to load, hard to keep accurate, and most of any given session doesn't need most of it. Fix: keep a short always-loaded file with project essentials and a routing table; move detailed sections (deployment steps, a style guide, historical decisions) into separate files referenced by topic, loaded only when that topic comes up.

**An agent doing multi-day research that keeps losing the thread.** Symptom: by day three, it re-investigates things already settled on day one. Fix: write a running summary file (decisions made, open questions, sources already checked) updated at the end of each session — durable/evolving state, not left to live only in conversation history that may get truncated or compacted away.

**A support-ticket triage agent given the entire knowledge base on every call.** Symptom: slow, expensive, and prone to citing irrelevant articles. Fix: retrieve only the handful of knowledge-base articles relevant to the specific ticket's content, rather than including the whole base every time — a retrieval problem, not a reason to add more context.

## Relationship to loop engineering

[`loop-engineering`](../loop-engineering/SKILL.md) is the layer above this one: it's about scheduling agent runs, verifying their output, and scoping their tools and connectors *across* many runs over time. This skill is about what's actually inside any single run's context, including the runs inside a loop. A loop with a perfectly designed verifier and connector scope will still degrade if every run drags forward a bloated, contradictory context — and conversely, a beautifully managed context doesn't by itself solve "when does this stop" or "what can it actually touch." Use both together: this skill for what the agent sees, `loop-engineering` for how often it runs and who checks its work.
