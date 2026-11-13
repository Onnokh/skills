# Skills

Personal agent skills.

## Install

```sh
# all skills
npx skills add Onnokh/skills

# a single skill
npx skills add Onnokh/skills --skill <skill-name>
```

## Reference

- **[design](skills/design)** — Design new screens, flows and components, or review existing ones, with the `better-*` skills from [jakubkrehel/skills](https://github.com/jakubkrehel/skills) for accessibility, layout, writing, typography, color and UI polish.
- **[animation](skills/animation)** — A router for [Emil Kowalski's animation skills](https://github.com/emilkowalski/skills): build, review, audit, choreograph, gestures, performance and more, on top of his design engineering philosophy.
- **[effect-patterns](skills/effect-patterns)** — Effect-TS conventions for services, layers, schemas, errors, config, retries, caching, streams and HTTP clients.
- **[plan-implementation](skills/plan-implementation)** — Gather the current state from the issue tracker and local repositories, then give an ordered plan and wait for go before any work starts.
- **[rankstas-paradise](skills/rankstas-paradise)** — Query and update SEO plans, keyword demand and Search Console results for Sleevy, Missingmounts, Printfeest and Shadertown via the Ranksta's Paradise MCP.

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
