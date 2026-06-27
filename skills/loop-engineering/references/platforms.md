# Loop Engineering by Platform

How the five-and-one building blocks (automation, isolation, skills, connectors, sub-agents, memory — see `concepts.md`) map onto each platform a user is likely to actually have access to. Read only the section(s) relevant to the user's stack. Product details change quickly — if something here seems off relative to what the user sees on screen, trust what they're looking at and tell them you may be out of date.

## Contents
- Quick mapping table
- Microsoft 365 / Copilot Studio
- Jira
- GitHub
- Claude: Cowork
- Claude: Claude Code
- Claude: Console / Managed Agents
- n8n
- Oracle Fusion AI Agent Studio
- Mixing platforms in one loop

## Quick mapping table

| Block | M365 / Copilot Studio | Jira | GitHub | Claude Cowork | Claude Code | Claude Console / Managed Agents | n8n | Oracle Fusion AI Agent Studio |
|---|---|---|---|---|---|---|---|---|
| Automation | Triggers (event/schedule) in Agent Builder / Copilot Studio | Automation rules on issue events | Actions workflows, scheduled or event-triggered; Copilot coding agent assigned to issues | Scheduled tasks | Scheduled tasks, hooks, `/loop`-style cadence | Scheduled agent runs (per-session) | Schedule/Webhook trigger nodes | Workflow orchestration triggers |
| Isolation | Separate agent sessions per workflow | N/A (issues are already isolated units) | Branches / PR-per-issue; Actions runners | Separate sessions/worktrees per task | `git worktree`, `--worktree`, subagent `isolation: worktree` | Separate sandboxed sessions per run | Separate workflow executions | Separate agent/workflow instances |
| Skills/playbooks | Knowledge sources + instructions in Agent Builder | Space-level guidance, custom fields | Repo instructions, custom agent configs, MCP-fed context (e.g. Confluence) | Skills (SKILL.md) | Skills (SKILL.md), CLAUDE.md | System prompt + bundled skills on the agent definition | Sub-workflows encoding reusable logic | Knowledge sources / content intelligence |
| Connectors | Power Platform connectors, Graph API, plugins | Native Jira actions; "Use GitHub Copilot" automation action | GitHub API, MCP servers, integrations (e.g. Jira) | MCP servers, connected apps | MCP servers | Tools + MCP servers defined on the agent | Hundreds of app nodes + HTTP Request + MCP | Oracle Fusion data + partner/external agent connections |
| Sub-agents | Multiple agents orchestrated via Agent 365 / multi-agent workflows | Coding agent + automation rule logic (limited native sub-agent split) | Copilot coding agent (doer) + required human/CI review (checker) | Task tool subagents, agent teams | Subagents in `.claude/agents/`, agent teams | Multiple managed agent definitions calling each other | AI Agent node calling tool sub-workflows; can chain multiple agent nodes | Multi-agent workflow orchestration with rules between steps |
| Memory/state | Agent 365 session + Dataverse/CRM records | The Jira issue itself + comments | The PR/issue thread; repo files (e.g. progress notes) | Memory files, markdown state files, connected boards | CLAUDE.md, progress markdown files, Linear/board via MCP | Vaults (credentials) + external state store you wire in | Workflow execution data, a connected data store, or a memory node | Contextual memory feature; enterprise data store |

## Microsoft 365 / Copilot Studio

Copilot Studio's Agent Builder lets you describe an agent in plain language and it configures knowledge sources, triggers, and actions for you. The relevant shift for a loop is that agents can now run **proactively on triggers** (an email arrives, a record is created, a schedule fires) rather than only responding when a person messages them — that's the automation block, native to the platform.

- **Automation:** define triggers in Agent Builder/Copilot Studio — event-based (record created/updated, email received) or schedule-based.
- **Skills:** attach knowledge sources and written instructions to the agent; this is the M365 equivalent of a SKILL.md.
- **Connectors:** Power Platform connectors and the Graph API are the main way the agent reaches Outlook, SharePoint, Teams, Dynamics, etc. Scope connector permissions tightly — this is also where governance review happens.
- **Sub-agents:** multi-agent workflows are coordinated through Agent 365, which is also where governance, registration, and audit logging for autonomous agents live. Treat registering an agent in Agent 365 as part of the loop design, not an afterthought — persistent permissions on a proactive agent are exactly the kind of "hard to undo" risk flagged in `building-loops.md` Step 0.
- **Memory:** typically a Dataverse/CRM record, a SharePoint list, or the session itself, depending on what the agent is updating.
- **What's distinctive here:** the governance layer is more built-in than on most other platforms — lean on Agent 365's registration/audit features as the verifier/guardrail layer rather than building one from scratch.

## Jira

Jira's own automation rules are the trigger layer, and the most direct path to a coding loop is the GitHub Copilot integration.

- **Automation:** Automation rules fire on issue events (created, transitioned, labeled). A rule can include a "Use GitHub Copilot" action that hands the issue to GitHub's coding agent.
- **Connectors:** this integration is bidirectional — Copilot's coding agent reads the issue description/comments for context, and posts status updates back into the issue's agent panel in real time, so the loop's progress is visible where the team already looks.
- **Skills:** space-level guidance and custom fields let you encode project-specific conventions that the coding agent reads as context.
- **Sub-agents/verification:** Jira itself doesn't deeply split doer/checker — the draft PR that comes back from GitHub is the checkpoint, and human/CI review on the GitHub side is the verifier. Don't treat "Copilot opened a PR" as the stopping condition; the stopping condition is the PR passing review and CI.
- **Memory:** the issue itself (description, comments, status) is the state file — which is convenient, since the whole team already watches it.
- **What's distinctive here:** Jira is usually the *trigger and tracking* layer of a loop, not the place doer/checker logic lives — pair it with GitHub for the actual build-and-verify work (see GitHub section).

## GitHub

GitHub is usually where the actual "doer" work happens in a software loop, with Actions as the scheduler and the Copilot coding agent as an asynchronous, autonomous executor.

- **Automation:** GitHub Actions workflows (scheduled via cron syntax, or triggered on issue/PR events) are the heartbeat. The Copilot coding agent can also be assigned directly to an issue as a trigger.
- **Isolation:** every coding agent run and CI job effectively gets its own checkout/branch; PR-per-issue is the natural isolation unit.
- **Connectors:** GitHub's own API plus MCP servers give the agent reach into other systems (Jira, Confluence, internal tools) from inside the same run.
- **Skills:** repository-level instructions and custom agent configuration files tell the coding agent project conventions, similar in spirit to a SKILL.md.
- **Sub-agents/verification:** the coding agent is the doer; CI checks and required human review are the checker. This is one of the cleanest natural doer/checker splits available on any platform — use it. Don't merge on "CI is green" alone if the definition of done included things CI doesn't check (e.g. "matches the design," "doesn't regress UX") — add a review step for those.
- **Memory:** the PR and issue threads are the durable record; for multi-step work, a progress file committed to the repo survives better than relying on conversation history alone.
- **What's distinctive here:** GitHub's strength is verification infrastructure that already exists (CI, required reviewers, branch protection) — wire the loop's stopping condition into these rather than inventing a parallel one.

## Claude: Cowork

Cowork is aimed at non-developer, file-and-task workflows on a person's own machine/files, with scheduled tasks, skills, and subagents available as building blocks.

- **Automation:** scheduled tasks (see the `schedule` skill) cover the heartbeat — "every morning," "every Monday," "in an hour."
- **Skills:** Cowork skills (this skill is one) are exactly the "written-down knowledge" block — anything the user would otherwise have to re-explain belongs in a skill.
- **Connectors:** MCP connectors and connected apps (Slack, email, project trackers, etc., depending on what's connected) give Cowork reach into real systems; the connector registry can be searched to find what's available before assuming something needs to be built by hand.
- **Sub-agents:** the Agent/Task tool spawns subagents for research, verification, or parallel work — use a dedicated subagent as the checker on anything that writes a file or takes an action, rather than having the same conversation grade its own output.
- **Memory:** Cowork's persistent memory system (user/feedback/project/reference memories) plus files saved to the user's connected folder are the durable state.
- **What's distinctive here:** Cowork is explicitly built for people who aren't necessarily engineers — when designing a loop here, keep the definition-of-done language plain and verifiable rather than assuming the user wants to write a formal spec.

## Claude: Claude Code

Claude Code is the most "native" loop-engineering environment among the Claude products — most of the five-and-one blocks ship as first-class features.

- **Automation:** scheduled tasks/cron, hooks that fire at points in the agent's lifecycle, and run-until-done patterns that keep working against a stated goal with a separate check on whether it's actually done.
- **Isolation:** `git worktree`, a `--worktree` flag for an isolated session, and an `isolation: worktree` setting on subagents so each helper gets a clean, disposable checkout.
- **Skills:** `SKILL.md` files (project-level or user-level) plus `CLAUDE.md` for standing project context.
- **Connectors:** MCP servers, the same protocol used across most of these platforms, so a connector built once tends to be reusable elsewhere.
- **Sub-agents:** subagents defined under `.claude/agents/`, with the classic split of one that explores, one that implements, one that verifies against a spec — and the same pattern applies to the stopping condition itself: a fresh agent/model judging "is this actually done" rather than the one that did the work.
- **Memory:** markdown progress files, `CLAUDE.md`, or an external board reached via MCP.
- **What's distinctive here:** because the primitives are this exposed, it's easy to over-build — remind users that wiring up worktrees and three subagents for a task that doesn't need parallelism is pure overhead. Match the blocks used to what Step 0 in `building-loops.md` actually calls for.

## Claude: Console / Managed Agents

Managed Agents (on the Claude Developer Platform / Console) are composable APIs for deploying cloud-hosted agents at scale, aimed at production deployments rather than a single person's session.

- **Automation:** agents can run on a schedule, with each scheduled firing starting a fresh session — no separate scheduler service to maintain.
- **Connectors/tools:** you define the model, system prompt, tools, and MCP servers once when creating the agent, then reference it by ID across every run.
- **Memory:** vaults handle credential/secret storage for scheduled runs; durable task state beyond a single session needs to be wired to an external store or tracker — Managed Agents themselves don't impose a default state file the way a repo's markdown file does.
- **Sub-agents/verification:** built by defining multiple agent IDs and having one call another, or by routing output to a separate verification agent; session tracing and integration analytics built into the Console are the observability layer that lets a team actually trust a checker over time rather than just once.
- **What's distinctive here:** this is the platform built for "run this in production, at volume, for other people," not for a single person's workflow — governance, tracing, and cost visibility matter more here than on Cowork or Claude Code, so push harder on Step 3 (the verifier) and on monitoring spend before scaling up run frequency.

## n8n

n8n is a general workflow-automation tool with a dedicated AI Agent node, which makes it a strong fit when the loop needs to touch many different SaaS tools rather than just code/issue-tracker systems.

- **Automation:** Schedule/Cron and Webhook trigger nodes are the heartbeat; a typical pattern is trigger -> fetch data -> AI Agent node -> output/action nodes.
- **The AI Agent node itself:** runs an internal reasoning loop (assemble context, call the model, call tools, repeat) until the model stops or hits a step cap — this is the harness, one level down from the workflow-level loop.
- **Skills/playbooks:** encoded as reusable sub-workflows, or as detailed system-message content on the Agent node, since n8n doesn't have a separate skill-file concept the way Claude products do.
- **Connectors:** n8n's main strength — hundreds of pre-built app integrations plus generic HTTP Request and MCP nodes, so the loop can reach almost any SaaS tool without custom code.
- **Sub-agents/verification:** achievable by chaining a second AI Agent node configured as a checker, or by adding a deterministic IF/Switch node that validates structured output before it's allowed to proceed — prefer the deterministic check whenever the criterion is mechanically checkable (see Step 3 in `building-loops.md`).
- **Memory:** a memory sub-node attached to the Agent node for within-session context, plus whatever external system (a database node, a spreadsheet, a tracker) holds state across runs.
- **What's distinctive here:** n8n makes it very easy to wire up the automation and connector blocks and very easy to forget the verifier block, because the canvas naturally ends at "and then post the result" — explicitly ask what checks the output before it's treated as final.

## Oracle Fusion AI Agent Studio

AI Agent Studio is the agent-building layer inside Oracle Fusion (ERP/HCM/SCM/CX), aimed at orchestrating agents across enterprise business processes rather than code or tickets.

- **Automation:** workflow orchestration triggers tied to business events inside Fusion applications (a record changing state, a process step completing).
- **Connectors:** the Agentic Applications Builder lets you compose workflows from Oracle, partner, and external agents against enterprise data, without traditional coding.
- **Skills:** content intelligence brings unstructured data (documents, communications) together with transactional data so agents have the equivalent of project knowledge already in context.
- **Sub-agents:** workflow orchestration is explicitly multi-agent and multi-step, with rules controlling how work moves between steps and **built-in human oversight** points — this is the platform's native version of the doer/checker split, and it's designed in rather than bolted on.
- **Memory:** contextual memory lets agents retain and share relevant information across workflow steps and across runs.
- **What's distinctive here:** governance, observability, and ROI measurement are first-class (built-in auditability and ROI tracking), reflecting that this platform is built for regulated, high-stakes enterprise processes — use the native human-oversight points rather than skipping them for speed, especially for anything touching finance, HR, or customer data, where Step 0's "hard to undo" flag almost always applies.

## Mixing platforms in one loop

Real loops often span platforms — e.g., a Jira automation rule triggers GitHub's coding agent, which is verified by CI, with Slack or Teams as the notification layer. Cross-platform loops fail most often at the seams, not within any one platform's own logic — the work below is about making the seams explicit instead of implicit.

### Build an ownership matrix first

Before wiring anything, write down which single platform owns each of these — not "could check," but **authoritative source**:

| Concern | Owned by (pick one) |
|---|---|
| Definition of done | |
| Memory / current state | |
| Trigger for the next step | |
| Verifier of record | |
| Human approval gate (if any) |  |

If two platforms could plausibly answer "is this done yet," that's the bug, not a detail to sort out later — pick one, and have every other platform defer to it rather than keeping its own parallel answer. The same goes for state: the moment two systems each think they hold the current status, they will eventually disagree, and the loop will act on stale information from whichever one nobody's looking at.

### A handoff checklist

For every point where work crosses from one platform to another, confirm:

- **The handoff is itself observable.** Something visible changes (a status field, a posted comment, a label) at the moment work moves — not just a side effect a person has to infer happened.
- **The receiving platform has enough context, not just a pointer.** A link to "see the Jira ticket" is weaker than the receiving agent actually being fed the relevant fields/comments at handoff time — don't assume the next platform will go fetch context on its own.
- **There's a timeout or stuck-state alarm.** If the receiving platform never picks up the handed-off work (an integration is down, a queue backs up), something should notice and surface that — silently waiting forever is a common cross-platform failure mode.
- **Failure on one side doesn't get silently swallowed by the other.** If GitHub's coding agent fails, does Jira's automation rule know, or does the ticket just sit there looking "in progress" forever? Decide what failure propagation looks like before relying on the chain.
- **Identity is consistent across the seam.** If platform A acts as a service account and platform B logs actions under a person's name, an audit trail reconstructed later will be confusing or wrong — keep the acting identity traceable end to end (see `governance-and-security.md`).

### Worked example: Jira → GitHub → Slack/Teams

A common chain, walked through against the checklist above:

1. **Jira** owns the definition of done (the ticket's acceptance criteria) and the current state (ticket status). An automation rule fires the "Use GitHub Copilot" action when a ticket is labeled `ready-for-agent`.
2. **Handoff to GitHub:** the rule passes the issue's description and comments as context, not just a link — Copilot's coding agent reads that content directly rather than re-deriving intent from a bare issue number.
3. **GitHub** owns execution and the verifier of record: CI plus required review on the resulting PR. GitHub posts status updates back into the Jira issue's agent panel as it works, so the handoff is observable in both directions, not just outbound.
4. **Handoff to Slack/Teams:** a notification fires on specific state changes (PR opened, PR merged, CI failed) — not a duplicate of everything Jira/GitHub already show, just the events a human actually needs to act on.
5. **Stuck-state check:** if no GitHub activity appears within a defined window after the handoff (say, a few hours), the automation rule (or a separate watcher) flags the ticket back to a human rather than leaving it silently "in progress." This is the timeout alarm from the checklist — without it, a failed handoff looks identical to a slow one.
6. **Closing the loop:** once the PR merges, GitHub (not a person, and not Slack) is the system of record that transitions the Jira ticket — keeping "what actually happened" anchored to the platform that owns verification, consistent with the ownership matrix decided up front.

MCP is the common thread across most of these (GitHub, Claude products, n8n) — when a connector needs to be built once and reused across a chain like this, building it as an MCP server is usually the most portable choice.
