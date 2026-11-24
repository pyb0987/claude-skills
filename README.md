# claude-skills

Claude Code skills I use day to day.

| Skill | What it does |
|---|---|
| [`review-fix-loop`](./review-fix-loop/SKILL.md) | Reviews a PR with fresh subagents, verifies every finding adversarially, fixes the confirmed ones test-first, and re-reviews until clean (max 3 rounds). Bugs and structure/performance run as separate lanes. On a teammate's PR it posts verified review comments instead of fixing. |

## Install

```bash
git clone https://github.com/pyb0987/claude-skills.git ~/claude-skills
ln -s ~/claude-skills/review-fix-loop ~/.claude/skills/review-fix-loop
```

## Project-specific overlay

Each skill reads an optional `local.md` beside its `SKILL.md`: spec lookups, team comment
language, approval rules, extra smells. It is gitignored, so company details stay on your
machine. So is `runs.jsonl`, the per-run log the skill appends to.
