# Building a Good Loop

This is the step-by-step process for going from "I want an agent to handle X automatically" to a working loop. Work through these in order — skipping straight to "which tool do I use" before nailing down the stopping condition is the single most common reason loops disappoint people.

## Contents
- Step 0: decide if a loop is even the right shape
- Step 1: define done, in writing
- Step 2: design the five-and-one blocks for this loop
- Step 3: pick the verifier
- Step 4: pick the platform and wire it up
- Step 5: a trial run with the leash on
- Step 6: lengthen the leash (and budget the cost before you do)
- A worked example end to end
- Quick design template

## Step 0: Decide if a loop is even the right shape

Not every repetitive task needs a full loop. Ask:

- Does this need to run **without someone present** (overnight, on a schedule, triggered by an event nobody's watching for)? If not, a single well-scaffolded agent run with a human reviewing each result is simpler, cheaper, and easier to trust — recommend that instead.
- Is the cost of an occasional wrong answer **low and recoverable** (a draft PR someone reviews, a triaged ticket someone confirms) or **high and hard to undo** (something that emails customers, moves money, deletes data, changes permissions)? High-stakes, hard-to-undo actions need a human approval gate inside the loop, not full autonomy — say this plainly rather than letting someone wire up unattended execution for something irreversible.
- Is there enough volume to justify the setup cost? A loop that fires once a week to do something a person could do in five minutes isn't usually worth the engineering overhead, and there's a real risk that the time spent building, debugging, and trusting the loop costs more than it saves.

If a loop is warranted, continue. If not, say so — a good answer here is sometimes "you don't need a loop for this, you need a checklist" or "you need a single subagent task, not a schedule."

## Step 1: Define done, in writing

Before touching any tool, write the stopping condition down as something checkable, not a feeling. Push the user to be specific:

- Vague: "the bug is fixed."
- Checkable: "the failing test in `test/auth/login_test.py` passes, no other tests regress, and lint is clean."

- Vague: "the ticket is triaged."
- Checkable: "the ticket has a priority label, an assignee, and a one-paragraph repro or root-cause note."

If the user can't state this precisely yet, that's the actual task — help them write it before designing automation around it. This written definition becomes both the loop's stopping condition *and* the verifier's grading standard (see Step 3) — it's the single artifact the rest of the loop hangs off of.

## Step 2: Design the five-and-one blocks for this loop

Walk through each block and make it concrete for the specific task (see `concepts.md` for what each means in general):

1. **Automation** — What actually triggers a run? A schedule (how often — does the cadence match how fast work actually arrives?), an event (issue created, PR opened, record updated), or both?
2. **Isolation** — Will more than one run ever touch the same resource concurrently? If yes, what's the isolation mechanism on this platform (worktree, branch, separate sandbox, separate record lock)?
3. **Skills / playbooks** — What does the agent need to know about this specific task or project that it would otherwise guess? Write it down once, here, rather than re-explaining it in every prompt the loop sends.
4. **Connectors / tools** — What does the agent need to actually *touch* to do this (not just describe)? List the real systems, and scope each connection as narrowly as the task allows — broad, all-purpose access is a higher blast radius if something goes wrong, and it's also harder for the agent to use well.
5. **Sub-agents** — Does this task benefit from separating the doer from the checker? For anything that writes/changes something (code, content, records), yes, almost always. For pure read/summarize tasks, often a single agent plus a lighter validation pass is enough.
6. **Memory / state** — Where does "what's done and what's next" live, outside the model? A file in the repo, a board, a ticket field. Pick something the user (and the next run) will actually look at.

### Scoping connectors: a quick checklist

Connector design is where a loop's actual risk lives — it's the difference between an agent that can *describe* a fix and one that can *push* it. Before wiring a connector in:

- **Use a dedicated identity, not a personal one.** A service account or scoped token for the loop, separate from the builder's own login, means you can revoke or trace it without touching anyone's personal access — and you can tell, from the logs alone, that an action came from the loop and not a person.
- **Grant the narrowest scope that does the job.** Read-only where the loop only reads. Write access limited to the specific repo/board/record type it touches, not the whole org or workspace. If the platform offers per-resource scoping (a single repo, a single Jira project, a single SharePoint site), use it instead of an account-wide grant.
- **Prefer actions with a built-in undo.** A draft PR beats a direct push to `main`. A comment-and-label beats an auto-close. A drafted email a human sends beats one the loop sends itself. When Step 0 flagged an action as high-stakes, this is usually where to add the friction back in.
- **Make sure calls are logged, not just possible.** The loop's audit trail should let someone reconstruct what it actually did without re-running it — which records it touched, what it changed, when. This matters for debugging and it's also exactly what a governance/security reviewer will ask for first (see `governance-and-security.md`).
- **Rotate or expire credentials on a schedule**, especially for anything meant to run unattended for months — a forgotten long-lived token is a bigger risk than the loop itself.
- **One narrow connector per concern beats one broad connector for everything.** It's easier for the agent to reason about a tool that does one thing well, and easier for a human to revoke just the piece that's misbehaving if something goes wrong.

## Step 3: Pick the verifier

This deserves its own step because it's the one most often skipped. Options, roughly in order of trust (and cost):

- **A second agent, different instructions, grading against the Step 1 definition of done.** Cheap, fast, works well when the definition of done is concrete and checkable.
- **A second agent on a stronger/different model.** Helps when the failure mode is the same blind spot showing up in both the doer and a same-model checker.
- **A programmatic check** (tests pass, schema validates, a count matches) where one exists. Always prefer this over an LLM judgment when the criterion is mechanically checkable — it's cheaper, faster, and doesn't share the model's blind spots.
- **A human approval gate.** Required whenever Step 0 flagged the action as high-stakes or hard to undo. Not a failure of the loop design — it's the correct call for the irreversible stuff.

Ask directly: "if this verifier says it's done, are you comfortable not looking at it?" If the answer is no, either the verifier isn't strong enough yet, or this step needs a human gate rather than an automated one.

### Tier the checks — cheap first, expensive last

Don't reach for an LLM judge as the first line of defense. A good verification stack runs deterministic checks first and only escalates what they can't resolve:

1. **Mechanical checks** (tests pass, schema validates, required fields present, no duplicate records) — fast, free of model blind spots, and catch the most common failures outright.
2. **A second agent grading against the definition of done** — for the judgment calls mechanical checks can't make (does this match intent, is this the right tone, is this explanation actually correct).
3. **A human gate** — for anything Step 0 flagged as high-stakes, or anything stage 1–2 flagged as low-confidence.

This tiering is also a cost lever: most runs should resolve at stage 1, which is nearly free, rather than paying for an LLM judgment on every single output.

### A grading rubric, by output type

A verifier needs to know *what* to check, not just *that* it should check. Tailor the rubric to what the loop actually produces:

- **Code:** tests pass, lint is clean, the diff matches the stated scope (no unrelated files touched), and adjacent tests don't regress.
- **Written content:** matches the brief's stated facts (not just "reads well"), tone consistent with examples given, no fabricated specifics — names, numbers, quotes — that weren't sourced from somewhere real.
- **Structured data / records:** schema valid, required fields populated, values within expected ranges, no duplicate or orphaned records created.
- **Ticket or triage decisions:** every field the definition of done requires is actually populated, and anything the agent wasn't confident about is flagged rather than guessed.

### Prove the verifier before you trust it

A verifier that's only ever seen correct output is unproven, not trustworthy. Before relying on it: feed it two or three outputs you already know are wrong, in different ways — one subtly wrong, one obviously wrong, one technically correct but violating an unstated rule — and confirm it catches each one. If it misses any, fix the verifier before fixing the doer.

For higher-stakes loops, chain stages rather than picking one: doer → automated check → second-agent review → human gate, escalating only what the previous stage couldn't resolve. This avoids paying for a human review on the large fraction of cases a cheap check already confirmed.

## Step 4: Pick the platform and wire it up

Once Steps 1–3 are settled, the platform choice is mostly "where does this work already live, and what does that platform make easy." Read `platforms.md` for the mapping of each block onto: Microsoft 365 / Copilot Studio, Jira, GitHub, Claude (Cowork, Claude Code, Claude Console / Managed Agents), n8n, and Oracle Fusion AI Agent Studio. Common pattern: pick the platform where the *trigger* and the *connectors* are most native — fighting a platform to reach a system it doesn't integrate with well is a common source of fragile loops. If the loop needs to span more than one platform, read the "Mixing platforms in one loop" section there too.

## Step 5: A trial run with the leash on

Before letting a loop run fully unattended:

- Run it once manually, end to end, and read every step it took, not just the final output.
- Run it on a small, low-stakes slice of real work (one ticket, one repo, one record) before pointing it at everything.
- Check that the verifier actually catches something — deliberately feed it a run you know is subtly wrong and confirm the checker flags it. A verifier that's never seen a failure is unproven.

## Step 6: Lengthen the leash (and budget the cost before you do)

Only after Step 5 holds up:

- Turn on the schedule/trigger for real.
- Set a review cadence proportional to risk — daily review of everything it did at first, tapering off as it earns trust, never zero for anything high-stakes.

### Budget the cost before you scale, not after

Token cost is the easiest part of a loop to get wrong precisely because it doesn't show up until the bill does. Before flipping the schedule on for real:

- **Estimate cost per run from actual data, not theory.** Run it manually 3–5 times against representative inputs and look at real token usage — multi-turn tool use and multi-stage verification both add up faster than the "one prompt, one response" mental model suggests.
- **Multiply by expected frequency** to get a daily or weekly number, and sanity-check it against what the task is actually worth. A loop that costs more per week than the time it saves isn't a win just because it's automated.
- **Set a hard ceiling** — a max-turn or max-cost cap per run — so a stuck loop fails loud and cheap instead of quietly burning budget for hours.
- **Watch actual spend against the estimate for the first couple of weeks.** Run-until-done patterns and multi-agent setups are the most likely to drift, especially when the agent hits something unexpected and starts retrying or over-exploring.
- **Decide in advance what happens when cost spikes** — throttle frequency, scale back which sub-agents run, or pause the loop entirely until someone looks at why. Deciding this after the spike means deciding it under pressure.

## A worked example end to end

*Goal: keep a backlog of incoming support tickets triaged without someone doing it manually each morning.*

1. **Done, in writing:** "Every ticket created in the last 24h has a category label, a severity, and either an assignee or a note explaining why none was obvious."
2. **Automation:** scheduled run, once every morning.
3. **Isolation:** not needed — tickets don't collide with each other in this task.
4. **Skills:** a short playbook describing the category taxonomy, severity definitions, and the team's current assignment rules (who owns what).
5. **Connectors:** the ticketing system's API (read tickets, write labels/assignee/comments), via a scoped service account limited to that one project.
6. **Sub-agents:** one agent drafts the categorization; a second agent checks each draft against the playbook's definitions before it's applied.
7. **Memory:** a daily run log (which tickets were touched, which were skipped and why) posted somewhere the team already looks.
8. **Verifier:** the second agent, checking against the playbook; anything it's not confident about gets flagged rather than guessed.
9. **Trial:** run once by hand against yesterday's tickets, read the output, deliberately include one ambiguous ticket to confirm it gets flagged rather than mis-triaged.
10. **Full run:** scheduled daily, with a human glancing at the run log each morning for the first two weeks, and a per-run cost ceiling set after measuring the trial run's actual token usage.

## Quick design template

Hand this to a user as a fill-in-the-blanks starting point:

```
Goal: <one sentence>
Definition of done (checkable): <specific, testable statement>
Trigger: <schedule / event / both>
Isolation needed?: <yes/no, and how>
Skill/playbook content needed: <what the agent would otherwise guess>
Connectors needed: <systems it must actually touch, scoped narrowly, via what identity>
Doer / checker split: <one agent or two; what the checker checks against; what tier each check runs at>
Memory location: <where state persists between runs>
Stakes if wrong: <low/recoverable vs high/hard to undo -> human gate needed?>
Estimated cost per run / cap: <measured from a trial run, plus a hard ceiling>
Platform: <see platforms.md>
```
