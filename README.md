# claude-skills

A personal collection of Claude Skills — reusable knowledge packages that can be loaded into Claude Code, Claude Cowork, or any Claude Agent SDK session to give the model deep, structured guidance on a specific topic.

## Skills

| Skill | Description |
|-------|-------------|
| [loop-engineering](skills/loop-engineering/) | Designing systems that schedule and supervise AI agents instead of prompting turn-by-turn |
| [context-engineering](skills/context-engineering/) | Deciding what belongs in an AI agent's context window (prompt vs. file vs. retrieved vs. memory) and fixing context rot in long-running or automated work |

## How to install a skill

1. **Package it** — zip the skill folder so that `SKILL.md` is at the root of the archive, e.g.:
   ```
   cd skills/loop-engineering && zip -r ../../loop-engineering.skill . && cd ../..
   ```
2. **Add via Settings** — in the Claude desktop app, go to **Settings → Capabilities → Skills** and upload the `.skill` file.
3. **Point Claude Code / Cowork at the folder directly** — pass the folder path when starting a session, or reference it in your project's `.claude/settings.json`:
   ```json
   {
     "skills": ["./skills/loop-engineering"]
   }
   ```

Once loaded, the skill's `SKILL.md` is automatically injected as context and the `references/` files are available for the model to fetch on demand.
