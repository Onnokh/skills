# Skills

Personal agent skills for Claude Code (and other agents).

## Install

```sh
# all skills
npx skills add Onnokh/skills

# a single skill
npx skills add Onnokh/skills --skill example-skill
```

## Adding a skill

Create `skills/<skill-name>/SKILL.md`:

```markdown
---
name: skill-name
description: What it does and when the agent should use it. This line drives triggering — be specific.
---

Instructions for the agent go here.
```

Optional extras per skill: a `references/` folder with supporting docs the
skill can point to, and `scripts/` for helper scripts.
