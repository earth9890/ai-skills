# ai-skills

Private Claude Code plugin marketplace.

## Install

```
/plugin marketplace add earth9890/ai-skills
/plugin install bsf --scope project
```

## Plugins

### `bsf`

| Skill | What it does |
| --- | --- |
| `pr-review` | Reviews a pull request — or a coordinated pair of PRs across two repos — for correctness, layer-boundary violations, SSOT/reusability, regressions, security, and end-to-end feature-flow integrity. |

## Layout

```
.claude-plugin/marketplace.json   # marketplace manifest
bsf/
  .claude-plugin/plugin.json      # plugin manifest
  skills/pr-review/SKILL.md       # the skill
```

Add a skill by dropping `bsf/skills/<name>/SKILL.md` — `plugin.json` points at the
whole `./skills/` directory, so no manifest edit is needed.
