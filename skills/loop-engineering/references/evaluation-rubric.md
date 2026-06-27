# Evaluating an Agent or Loop

Use this when a user wants a review of something they've already built — a loop, an automation, a single agent config — rather than help designing one from scratch. The goal of a review is always a short, prioritized list of specific changes, not a grade.

## Contents
- Before scoring anything, get the real artifact
- The seven dimensions
- How to score and report
- Common diagnoses and what to recommend
- Review template

## Before scoring anything, get the real artifact

Loops fail in their details, and details don't survive a verbal description. Before evaluating, ask to see the actual thing: the system prompt, the workflow export, the automation rule config, the repo with the agent's config files, screenshots of the platform setup, or a transcript of a real run. If the user only has a verbal description, evaluate what's possible from that but flag clearly which dimensions below you couldn't actually check.

If they can share a run that already happened (logs, a transcript, a PR it opened, a ticket it touched), read that in preference to the config alone — the config says what it's supposed to do, the run shows what it actually did.

## The seven dimensions

Score each as **Solid / Needs work / Missing**, with a one-line reason. These map directly onto the concepts and design steps in `concepts.md` and `building-loops.md`.

1. **Definition of done.** Is there a written, checkable stopping condition, or does the loop run until something vague like "looks finished"? Missing or vague definitions of done are the single most common root cause of bad loop output — start here.
2. **Verification.** Is there an actual checker (sub-agent, programmatic check, human gate) distinct from whatever produced the output? Does the checker check against the definition of done from #1, or against something looser? Would the user actually trust this checker enough to not look at its output?
3. **Context hygiene.** Across a multi-turn or multi-run task, is the agent's context staying relevant, or accumulating stale tool output, dead ends, and contradictions? Is project knowledge written down once (a skill/playbook) rather than re-explained, or re-guessed, every time?
4. **Tool/connector design.** Can the agent actually *do* the thing, or only describe it? Are the tools/connectors scoped narrowly enough to reason about, or is access broader than the task needs? Do failures show up as clear errors, or fail silently/ambiguously?
5. **Isolation.** If more than one run can happen concurrently, is there a mechanism (worktree, branch, sandbox, lock) preventing collisions? If only one run ever happens at a time, this dimension may legitimately be N/A — don't penalize a loop for skipping isolation it doesn't need.
6. **Memory/state.** Is there a durable record outside the model of what's done and what's next? Would the next run pick up where the last one left off, or start cold every time?
7. **Stakes and oversight match.** Does the level of human oversight (none / async review / hard approval gate) match how reversible the agent's actions actually are? An agent that can email customers, spend money, change permissions, or delete data with no gate is a red flag regardless of how good the other six dimensions are. If this dimension scores anything below Solid, or the user mentions an upcoming security/InfoSec review, point them to `governance-and-security.md` for the fuller builder-side prep.

## How to score and report

- Lead with the **highest-impact gap**, not the alphabetically first dimension. A missing definition of done matters more than slightly loose tool scoping, and the report should reflect that, not just list seven scores in order.
- For each gap, give a **specific, actionable fix** tied to the user's actual setup — "add a sub-agent that checks the output against [the spec you already have in the ticket]" beats "improve verification."
- Where something is genuinely solid, say so — a review that's all criticism is less useful (and less trusted) than one that's calibrated.
- If the user's setup has no real loop yet (e.g. it's a single agent run they want to make autonomous), say that plainly rather than scoring nonexistent dimensions — point them to `building-loops.md` instead.

## Common diagnoses and what to recommend

| Symptom the user describes | Likely root cause | What to recommend |
|---|---|---|
| "It says it's done but it isn't" | No real definition of done, or the checker is the same agent that did the work | Write a checkable definition of done; add an independent verifier against it |
| "It works in testing but goes off the rails when left alone" | No verifier was ever tested against a known-bad run | Deliberately feed the verifier a wrong output and confirm it catches it before trusting it unattended |
| "It's burning way more tokens/cost than expected" | Run-until-done loop with no cap, or multiple sub-agents added without checking the cost tradeoff | Add a max-turn/step cap as a backstop; measure token cost per run before scaling frequency |
| "Two runs stepped on each other / overwrote each other's work" | No isolation mechanism for concurrent runs | Add worktree/branch/sandbox isolation per the platform (see `platforms.md`) |
| "It keeps re-asking things it should already know" | No skill/playbook — context isn't persisting project knowledge | Write the recurring knowledge down once as a skill/playbook the loop reads every run |
| "Overnight it produced a pile of stuff I now have to review" | Loop is producing faster than the human review loop can absorb, or stakes/oversight mismatch | Throttle frequency or batch size to match review bandwidth; add a gate for anything irreversible |
| "It can describe the fix but can't actually do it" | Missing or misconfigured connector | Identify the specific system it needs to touch and wire up the narrowest connector that reaches it |

## Review template

```
## Loop review: <name/description>

**Highest-impact issue:** <one sentence>

| Dimension | Status | Note |
|---|---|---|
| Definition of done | Solid / Needs work / Missing | |
| Verification | Solid / Needs work / Missing | |
| Context hygiene | Solid / Needs work / Missing | |
| Tool/connector design | Solid / Needs work / Missing | |
| Isolation | Solid / Needs work / Missing / N/A | |
| Memory/state | Solid / Needs work / Missing | |
| Stakes/oversight match | Solid / Needs work / Missing | |

**Recommended changes, in priority order:**
1. ...
2. ...
3. ...

**What's already working well:** ...
```
