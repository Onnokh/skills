---
name: animation
description: Routes animation and motion work to the right Emil Kowalski skill — build, review, audit, find, remove, choreograph, performance, gestures, vocabulary — and supplies the shared design-engineering philosophy they all build on. Use for any animation, transition, motion, easing, spring, stagger, gesture or "make it feel better" task.
---

# Animation

## Route first

This skill is a router. Pick the task below, then **read the sub-skill's `SKILL.md` and follow it**. Each sub-skill lives in a sibling folder of this skill: `../<skill>/SKILL.md`, relative to this skill's base directory (globally `~/.agents/skills/<skill>/SKILL.md`). Read the file directly; do not depend on the Skill tool, because some sub-skills (such as `review-animations`) cannot be invoked by the model. Also read any file the sub-skill links (`RECIPES.md`, `STANDARDS.md`, `AUDIT.md`, `PLAN-TEMPLATE.md`).

| The user wants to… | Sub-skill | Edits code? |
| --- | --- | --- |
| Build or add one animation or transition | `animate` | Yes |
| Coordinate several elements that move together: stagger, hero entrance, view swap, multi-step flow | `choreograph` | Yes |
| Build drag, swipe, sheet, drawer, fling or hold interactions | `gestures` | Yes |
| Fix an animation that stutters or drops frames | `animation-performance` | Yes |
| Review specific animation code, a diff or a component strictly | `review-animations` | No |
| Audit all motion in a codebase and get prioritized fix plans | `improve-animations` | No |
| Find places that should animate but don't | `find-animation-opportunities` | No |
| Remove motion the interface doesn't need ("feels slow", "over-animated") | `remove-animations` | Only deletes |
| Name a motion effect they can only describe | `animation-vocabulary` | No |
| A general question about animation or UI polish | none: read [PHILOSOPHY.md](PHILOSOPHY.md) and answer | — |

Rules:

- Pick exactly one sub-skill. When a request spans two (for example build, then review), run them in sequence and say so.
- Ask one short question only when the request fits two rows equally well.
- The sub-skill's instructions, format and verdict win. Read [PHILOSOPHY.md](PHILOSOPHY.md) only to fill a gap the sub-skill leaves, never to override it.
- A sub-skill can point to a skill that is not installed (`animate-expo`, `pick-ui-library`). Say it is not installed and continue without it. Do not recreate it from memory.
- If a sub-skill folder is missing, say which one, then answer from [PHILOSOPHY.md](PHILOSOPHY.md).

## Initial Response

When this skill is first invoked without a specific request, respond only with a one-line greeting and the list of tasks from the routing table, then wait.
