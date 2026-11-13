# Skills

Personal agent skills.

## Install

```sh
# all skills
npx skills add Onnokh/skills

# a single skill
npx skills add Onnokh/skills --skill <skill-name>
```

## Design and animation

`design` and `animation` are routers. They load other skills, so install those
too. One command installs all of them, globally, for every detected agent:

```sh
npx skills add jakubkrehel/skills -g -y \
  -s better-accessibility -s better-layout -s better-writing \
  -s better-typography -s better-colors -s better-ui && \
npx skills add emilkowalski/skills -g -y \
  -s animate -s choreograph -s gestures -s animation-performance \
  -s review-animations -s improve-animations \
  -s find-animation-opportunities -s remove-animations \
  -s animation-vocabulary && \
npx skills add Onnokh/skills -g -y -s design -s animation
```

| Router | Based on | Loads |
| --- | --- | --- |
| `design` | `better-interface` from [jakubkrehel/skills](https://github.com/jakubkrehel/skills) | the six `better-*` skills |
| `animation` | `emil-design-eng` from [emilkowalski/skills](https://github.com/emilkowalski/skills) | the nine animation skills; Emil's full philosophy is in `PHILOSOPHY.md` |

Both are MIT-licensed; each folder keeps the original `LICENSE`.

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
