# Building a Good Loop

This is the step-by-step process for going from "I want an agent to handle X automatically" to a working loop. Work through these in order — skipping straight to "which tool do I use" before nailing down the stopping condition is the single most common reason loops disappoint people.

## Contents
- Step 0: decide if a loop is even the right shape
- Step 1: define done, in writing
- Step 2: design the five-and-one blocks for this loop
- Step 3: pick the verifier
- Step 4: pick the platform and wire it up
- Step 5: a trial run with the leash on
- Step 6: lengthen the leash
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

## Step 3: Pick the verifier

This deserves its own step because it's the one most often skipped. Options, roughly in order of trust (and cost):

- **A second agent, different instructions, grading against the Step 1 definition of done.** Cheap, fast, works well when the definition of done is concrete and checkable.
- **A second agent on a stronger/different model.** Helps when the failure mode is the same blind spot showing up in both the doer and a same-model checker.
- **A programmatic check** (tests pass, schema validates, a count matches) where one exists. Always prefer this over an LLM judgment when the criterion is mechanically checkable — it's cheaper, faster, and doesn't share the model's blind spots.
- **A human approval gate.** Required whenever Step 0 flagged the action as high-stakes or hard to undo. Not a failure of the loop design — it's the correct call for the irreversible stuff.

Ask directly: "if this verifier says it's done, are you comfortable not looking at it?" If the answer is no, either the verifier isn't strong enough yet, or this step needs a human gate rather than an automated one.

## Step 4: Pick the platform and wire it up

Once Steps 1–3 are settled, the platform choice is mostly "where does this work already live, and what does that platform make easy." Read `platforms.md` for the mapping of each block onto: Microsoft 365 / Copilot Studio, Jira, GitHub, Claude (Cowork, Claude Code, Claude Console / Managed Agents), n8n, and Oracle Fusion AI Agent Studio. Common pattern: pick the platform where the *trigger* and the *connectors* are most native — fighting a platform to reach a system it doesn't integrate with well is a common source of fragile loops.

## Step 5: A trial run with the leash on

Before letting a loop run fully unattended:

- Run it once manually, end to end, and read every step it took, not just the final output.
- Run it on a small, low-stakes slice of real work (one ticket, one repo, one record) before pointing it at everything.
- Check that the verifier actually catches something — deliberately feed it a run you know is subtly wrong and confirm the checker flags it. A verifier that's never seen a failure is unproven.

## Step 6: Lengthen the leash

Only after Step 5 holds up:

- Turn on the schedule/trigger for real.
- Set a review cadence proportional to risk — daily review of everything it did at first, tapering off as it earns trust, never zero for anything high-stakes.
- Keep an eye on token/run cost for the first couple of weeks; loops can be far more expensive in practice than the single test run suggested, especially run-until-done patterns or anything with multiple sub-agents.

## A worked example end to end

*Goal: keep a backlog of incoming support tickets triaged without someone doing it manually each morning.*

1. **Done, in writing:** "Every ticket created in the last 24h has a category label, a severity, and either an assignee or a note explaining why none was obvious."
2. **Automation:** scheduled run, once every morning.
3. **Isolation:** not needed — tickets don't collide with each other in this task.
4. **Skills:** a short playbook describing the category taxonomy, severity definitions, and the team's current assignment rules (who owns what).
5. **Connectors:** the ticketing system's API (read tickets, write labels/assignee/comments).
6. **Sub-agents:** one agent drafts the categorization; a second agent checks each draft against the playbook's definitions before it's applied.
7. **Memory:** a daily run log (which tickets were touched, which were skipped and why) posted somewhere the team already looks.
8. **Verifier:** the second agent, checking against the playbook; anything it's not confident about gets flagged rather than guessed.
9. **Trial:** run once by hand against yesterday's tickets, read the output, deliberately include one ambiguous ticket to confirm it gets flagged rather than mis-triaged.
10. **Full run:** scheduled daily, with a human glancing at the run log each morning for the first two weeks.

## Quick design template

Hand this to a user as a fill-in-the-blanks starting point:

```
Goal: <one sentence>
Definition of done (checkable): <specific, testable statement>
Trigger: <schedule / event / both>
Isolation needed?: <yes/no, and how>
Skill/playbook content needed: <what the agent would otherwise guess>
Connectors needed: <systems it must actually touch, scoped narrowly>
Doer / checker split: <one agent or two; what the checker checks against>
Memory location: <where state persists between runs>
Stakes if wrong: <low/recoverable vs high/hard to undo -> human gate needed?>
Platform: <see platforms.md>
```
