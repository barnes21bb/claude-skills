# Governance and Security, From the Builder's Side

This is not the InfoSec audit. It's the thinking a builder does *before* that conversation, so the review goes faster and turns up fewer surprises. Use this with someone who has a loop designed (or already running) and wants to walk into a security/governance review having already answered the obvious questions, rather than discovering them live in the meeting.

The frame matters: every item below is phrased as something the builder decides and can explain, not as a control someone else imposes on them. If a question here is hard to answer, that's useful information *now* — better to find the gap while it's still cheap to fix.

## Contents
- Why this belongs to the builder, not just InfoSec
- The eight questions
- Mapping to what each platform already gives you
- A self-score table
- Walking into the actual review

## Why this belongs to the builder, not just InfoSec

The person who designed the loop knows things no reviewer can infer from a config alone: why a connector needs the scope it has, what happens if a run goes wrong at 2am, whether the data it touches is sensitive. Showing up to a review without having thought about this shifts that work onto the reviewer, who has to ask basic questions before they can ask the interesting ones — and it makes "no" the easy default answer when something's unclear.

Showing up having already worked through the questions below does the opposite: it turns the review into "here's what I considered, here's why I made these calls, here's what I'd want confirmed" — a much shorter and more credible conversation, and one far more likely to end in approval.

## The eight questions

Work through these for the loop as actually designed (see `building-loops.md` Steps 2–3 for the design itself) — not as a retrofit.

### 1. What data does this loop actually touch?

List the systems and data types it reads and writes, not just the systems it's "connected to" in principle. Flag anything that's personal data, financial data, health data, HR data, or anything else regulated where you operate — these categories almost always change the answer to questions 2–4 below. If you're not sure whether something counts as sensitive, that uncertainty is itself worth surfacing rather than assuming it's fine.

### 2. How reversible are its actions?

Walk through the actual action list and sort each one: fully reversible (a comment, a draft, a label), reversible with effort (a merged PR, a sent-but-recallable message), or effectively irreversible (an email that's left the building, a deleted record, a payment, a permissions change). Anything in the last bucket needs a human approval gate — this is the same test as Step 0 in `building-loops.md`, applied with a security reviewer's level of skepticism rather than an optimistic one.

### 3. What identity does it act as, and is that identity scoped tightly?

Is the loop running as a dedicated service account or a person's own credentials? (It should be the former — see the connector-scoping checklist in `building-loops.md` Step 2.) Can you state, in one sentence, exactly what that identity can and can't do? If the honest answer is "it has broad access because that was easier to set up," that's the question a reviewer will ask first, so it's worth tightening before they do.

### 4. What's the credential's lifecycle?

Where does the credential/token live, who else can see it, does it expire or rotate, and what happens the day someone forgets to rotate it? A token that's been live for a year with nobody tracking when it expires is a more common real-world failure than anything exotic — have an answer ready, even if the answer is "we should set up rotation and haven't yet."

### 5. Can you reconstruct what it did, after the fact, without re-running it?

Pull up the actual logs/audit trail for a real run and check: do they show what was touched, what changed, and when, in a form someone other than you could read? "I could explain it if asked" is not the same as "there's a record." If the answer is no, this is usually the fastest gap to close, and reviewers will ask for it directly.

### 6. Is there a kill switch, and have you actually used it?

Can the loop be stopped — not "in theory, by deleting the automation" but with an action someone can take in under a minute while something is actively going wrong? Have you tested it, or only assumed it works? A kill switch nobody has pulled is a kill switch you don't actually know works.

### 7. Who is the human owner of record?

If this loop does something wrong at 2am, whose problem is it, and do they know that? "Whoever's on call" is not an answer — name the person or role, make sure they know they own it, and make sure that ownership is written down somewhere other than this conversation.

### 8. What's the worst plausible single failure, concretely?

Not "something could go wrong" — write the actual sentence: "it sends an incorrect refund to a customer," "it merges a change that fails in production," "it discloses an internal document to the wrong channel." A concrete worst case is something you and a reviewer can actually evaluate together; a vague one just produces a vague, slower conversation.

## Mapping to what each platform already gives you

Most platforms already have governance hooks built in — the gap is usually that nobody turned them on or pointed at them, not that they don't exist. From `platforms.md`:

| Platform | Native governance hook to use |
|---|---|
| Microsoft 365 / Copilot Studio | Agent 365 registration, permissioning, and audit logging — register the agent here rather than treating it as an afterthought |
| Jira | Permission schemes scoping what the automation/integration can touch, plus the issue's own audit log |
| GitHub | Branch protection, required reviewers, CODEOWNERS, and Actions run logs — wire the loop's stopping condition into these instead of a parallel system |
| Claude Cowork / Code | Connected-app/connector scopes, and session transcripts as the run record |
| Claude Console / Managed Agents | Session tracing and integration analytics — built for exactly this kind of "what happened, who can see it" question |
| n8n | Credential vaults (don't inline secrets in a workflow) plus per-execution logs |
| Oracle Fusion AI Agent Studio | Built-in auditability and ROI tracking, plus the platform's native human-oversight checkpoints in workflow orchestration |

If you answered question 5 or 6 above with "not really," check this table first — there's a reasonable chance the platform already has the hook and it just isn't turned on for this loop.

## A self-score table

Score each as **Solid / Needs work / Haven't checked**, the same style as `evaluation-rubric.md`, so this can sit next to that review if you're doing both at once.

```
| Question | Status | Note |
|---|---|---|
| Data sensitivity identified | Solid / Needs work / Haven't checked | |
| Action reversibility mapped | Solid / Needs work / Haven't checked | |
| Identity scoped tightly | Solid / Needs work / Haven't checked | |
| Credential lifecycle defined | Solid / Needs work / Haven't checked | |
| Audit trail reconstructable | Solid / Needs work / Haven't checked | |
| Kill switch tested | Solid / Needs work / Haven't checked | |
| Human owner named | Solid / Needs work / Haven't checked | |
| Worst-case failure written down | Solid / Needs work / Haven't checked | |
```

A few "Needs work" entries going into a review isn't a failure — it's exactly what this exercise is for. Going in with zero "Haven't checked" entries is the actual goal; an honest gap is a faster conversation than an unexamined one.

## Walking into the actual review

- Bring the self-score table filled out, including the gaps — a reviewer trusts "here's what's not solved yet" far more than a clean sheet that turns out to have unconsidered holes.
- Lead with question 8 (the worst plausible failure) and question 2 (reversibility) — these are almost always what a reviewer cares about most, so don't bury them under connector details.
- If the loop touches regulated data (question 1), expect the review to focus there disproportionately — that's appropriate, not an obstacle, and having already named the data type means you're not discovering the conversation's actual topic live.
- Treat anything you marked "Needs work" as a thing to fix or explicitly accept, not something to talk around — reviewers can tell the difference, and "we know, here's the plan" lands better than hoping it doesn't come up.
