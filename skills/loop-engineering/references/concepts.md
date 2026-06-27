# Loop Engineering: Concepts

## Contents
- Where the term comes from
- The progression: prompt -> context -> harness -> loop
- The agent harness (the thing the loop runs)
- The five building blocks of a loop, plus the sixth
- The part that doesn't automate away: four hard questions
- The actual shift in what a person's job becomes
- Failure modes to teach people to watch for
- Is it real, or hype?

## Where the term comes from

In mid-2026, a cluster of people building AI coding agents converged on the same observation, phrased a few different ways:

- Peter Steinberger (built the agent gateway OpenClaw): "You shouldn't be prompting coding agents anymore. You should be designing loops that prompt your agents."
- Boris Cherny (leads Claude Code at Anthropic): "I don't prompt Claude anymore. I have loops running that prompt Claude and figuring out what to do. My job is to write loops."
- Akshay Pachaar summarized the catch: "Loop engineering takes you off prompting. It takes you off curating context. It takes you off babysitting a single run. It does not take you off debugging. It just moves the debugging somewhere worse, into runs you were never watching."
- Addy Osmani gave it the cleanest one-line definition: "Loop engineering is replacing yourself as the person who prompts the agent. You design the system that does it instead."

It caught on because the underlying pieces had genuinely shipped as product features (scheduled tasks, run-until-done modes, hooks, skills, subagents) rather than staying a bespoke pile of bash scripts only one engineer understood. That's the real reason it's worth teaching: it's not a metaphor, it's a checklist of concrete things a builder can go turn on.

## The progression: prompt -> context -> harness -> loop

It helps to place loop engineering on a timeline of "what got automated, and what got handed back":

1. **Prompt engineering** — write a good single message. The skill is wording a request well.
2. **Context engineering** — curate what the model sees: which files, which history, which instructions. The skill is managing a scarce attention budget.
3. **Agent harness engineering** — build the environment a single agent run lives inside: tools, memory, error handling, guardrails, state persistence. The skill is designing the scaffolding around one run.
4. **Loop engineering** — build the system that decides *when* a harness runs, *what* it works on, and *whether its output is trusted*, repeatedly, without anyone prompting it by hand. The skill is designing a system that runs many harness invocations and supervises itself.

Each layer doesn't replace the one below it — it sits on top. A loop is built out of harnesses; a harness still needs good context; good context still benefits from a well-worded prompt at the core. Teaching someone loop engineering without the layers underneath it produces brittle automations that look impressive in a demo and fall over in week two.

## The agent harness (the thing the loop runs)

Before talking about loops, it's worth being precise about the harness, since the loop is just "many harness runs, orchestrated." A harness is the complete software shell wrapping a model for a single run: the orchestration loop inside one turn-sequence (the classic *assemble prompt -> call model -> parse output -> execute tool calls -> feed results back -> repeat* cycle, sometimes called ReAct or Thought-Action-Observation), plus the tools it can call, the memory it can read, the context management that keeps it from drowning in its own history, error handling, and guardrails.

A useful framing for the user: **a lot of an agent's real-world performance comes from the harness around the model, not from the model itself.** Two people running the same model can get wildly different results because one of them built a harness that gives it clean context, well-scoped tools, and a real way to fail safely, and the other didn't.

## The five building blocks of a loop, plus the sixth

A loop is the harness, run on a timer, with helpers. Concretely, that decomposes into:

1. **Automations** — the heartbeat. Something fires on a schedule or an event (a cron job, a webhook, an issue being created) and kicks off a run that finds and triages work on its own, without someone typing a prompt that morning.
2. **Isolation (worktrees)** — the moment more than one agent touches the same project, file collisions become the failure mode, exactly like two engineers committing to the same lines without talking. Isolation (a git worktree, a separate branch, a sandboxed run) means concurrent agents can't step on each other.
3. **Skills / playbooks** — written-down project knowledge: conventions, "we don't do it that way because of an incident," build steps, definitions of done. Without this, a loop re-derives the whole project from zero every cycle and fills any gap with a confident guess. With it, runs compound instead of starting cold.
4. **Connectors / tools** — the loop's actual reach into real systems via APIs/MCP/integrations: an issue tracker, a repo, a chat channel, a database. A loop that can only produce text is not a loop, it's a more annoying chatbot. The connectors are what let it open the PR, update the ticket, and post the result, rather than just describing what it would do.
5. **Sub-agents** — splitting the one who writes from the one who checks. A model grading its own work is too lenient on itself; a second agent with different instructions (sometimes a different, stronger model) catches what the first one talked itself into. The common split is explore / implement / verify.
6. **Memory / state** — the thing that survives between runs: a markdown file, a project-management board, a ticket. The model forgets everything when a run ends; the loop only works if something outside the model remembers what's done and what's next.

## The part that doesn't automate away: four hard questions

This is the part worth emphasizing most, because it's where almost all real-world loop failures come from, and it's the part marketing copy about "autonomous agents" tends to skip.

**1. Knowing when to stop.** A loop needs a *verifiable* stopping condition — not "looks done," but something a checker can actually test: "all tests in this suite pass and lint is clean," "the ticket's acceptance criteria are each demonstrably true," "no more matching records exist." Without this, a loop either stops too early (silently incomplete) or never stops (burning time and tokens, often badly). The thing the loop's checker checks against is usually a written spec or acceptance criteria — if that doesn't exist, write it before building the loop, not after.

**2. Keeping the context clean.** A run that goes for many turns accumulates noise: stale tool output, dead ends, contradictory earlier conclusions. Left unmanaged, this is "context rot" — the agent's effective intelligence degrades not because the model got worse but because what it's looking at got worse. Good loops compact, summarize, or discard aggressively, and keep the skill/playbook content tight and current rather than letting it sprawl.

**3. Tools the agent can actually use.** A capability the agent can't reliably call (badly documented, too broad, missing error handling, returns ambiguous results) is worse than no capability, because the agent will try it, half-succeed, and confuse its own state. Good tools are scoped narrowly enough to reason about, return information the agent can act on, and fail in ways that are obvious rather than silent.

**4. Something that can say no.** This is the verifier. It can be a sub-agent, a separate grading pass, a human approval gate, a hard policy check — but it has to be something the builder genuinely trusts, because trusting the checker is the *entire* reason it's safe to stop watching the loop run. A checker nobody trusts just moves the manual review from "every output" to "every output, but later and angrier."

## The actual shift in what a person's job becomes

Across all four layers (prompt -> context -> harness -> loop), the manual labor keeps shrinking and the judgment keeps concentrating into one thing that doesn't automate: **deciding what "done" actually means, precisely enough that a machine can check it.** That's a spec — not a vague ticket, a requirement with testable acceptance criteria. Loop engineering doesn't remove this work. It makes it the *whole* job, because everything downstream depends on it.

## Failure modes to teach people to watch for

- **Comprehension debt.** The faster a loop ships work nobody read carefully, the bigger the gap between what exists in the system and what any human actually understands about it. This compounds — a loop that builds on a loop's previous unreviewed output drifts fast.
- **Cognitive surrender.** When a loop runs itself, it's tempting to stop having an opinion about its output and just accept whatever comes back. The same loop design can be the cure (used by someone who stays engaged and reviews critically) or the accelerant (used by someone who's checked out) — the loop itself can't tell the difference.
- **Unattended failure.** A loop running unattended is, definitionally, also a loop *failing* unattended. The output volume from a loop running overnight can exceed what a person can responsibly review by morning — "ten PRs, half of them subtly wrong" is the canonical bad outcome.
- **Token cost variance.** Run-until-done patterns and multi-agent verification both multiply token spend versus a single prompt-response turn, and usage can spike unpredictably depending on how chatty the loop's internal back-and-forth gets. Don't let someone wire up an unattended loop against a budget they haven't sanity-checked.
- **Orchestration tax.** Removing file collisions (via isolation) doesn't remove the review bottleneck — a person's review bandwidth is usually the actual ceiling on how many parallel loops are worth running, not the tooling.

## Is it real, or hype?

Both, honestly, and it's worth saying so rather than oversimplifying. The mechanics are real and shipped: scheduled, self-checking agent runs are a genuine capability jump, available today across most major agent platforms. The hype is in the implication that *designing the loop* is the hard part, or the whole job. It isn't — wiring up a schedule and a sub-agent is comparatively easy. Writing a spec precise enough for a machine to verify against, and building a checker someone actually trusts, is the part that was always hard and still is. Tell users this directly: a loop with no real definition of done is just a faster way to generate work they still have to inspect line by line.
