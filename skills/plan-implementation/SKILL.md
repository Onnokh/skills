---
name: plan-implementation
description: >-
  Plan an implementation run before executing it: gather the current state
  from the issue tracker and local repositories, then emit an ordered plan
  with delivery and caveats and wait for go. Use when the user types
  /plan-implementation, gives a Linear link or a hint of where work resides
  and wants a plan first, or asks for a pre-flight summary of a backlog.
---

# Plan implementation

Produce an implementation plan for a body of work, then stop for approval.
Planning only: until the user says go, the only output is the plan message —
every file, branch, ticket, and deployment stays untouched.

## Step 1 — Resolve the scope

The argument is a Linear URL or a hint.

- **Linear URL** (project or issue): fetch it and its child issues.
- **Hint** (project name, repo path, feature phrase): search Linear and the
  local filesystem for the matching work. Two or more plausible matches:
  list them and ask which one.
- **No argument**: ask what to plan.

Use whichever Linear access is connected (MCP server, executor, or CLI).

## Step 2 — Gather the current state

Legwork, not summary-from-memory — every fact below comes from a live lookup:

- **Linear**: every issue in scope with status, priority, labels, and parent.
  Split them: done / in review / open.
- **Repositories**: for each repo the work touches — remotes, current branch,
  dirty state, submodules, and any open MR/PR tied to an in-review issue.
- **Domain language**: read the CONTEXT.md of each repo in scope and write
  the plan in its ubiquitous language.

## Step 3 — Load the execution contract

Silently read the `implement` skill (the `SKILL.md` in the `implement`
folder beside this skill's own folder in the installed skills directory).
The plan must be executable under that contract (TDD seams, typechecking
cadence, code review, commit target). The user sees the resulting plan,
never this read. If no `implement` skill is installed, plan against the
repository's own test and review conventions and say so in Caveats.

## Step 4 — Emit the plan

Exactly four sections, then the go question:

1. **Current state** — issue counts by status, each repo with its branch,
   and anything already in review.
2. **Plan** — every open issue exactly once, in dependency order, with its
   ID. Group issues that form a natural sequence.
3. **Delivery** — the fixed layout, stated as such even when the repository's
   history shows another pattern: one integration branch per affected
   repository, branched from the fetched remote default branch; one commit
   per issue; one MR per repository, opened when the full feature is
   complete. Name the local test gate that runs before the MRs open.
   The user's message can change this layout; repository convention cannot.
4. **Caveats** — every step that touches live infrastructure, spends money,
   or needs credentials the agent may lack — each with how the plan
   handles it.

Close with one question: proceed?

The plan is complete when every open issue in scope appears exactly once
and every caveat names its handling.

## After go

Execute the plan items in order under the contract from Step 3. For each
issue:

- **Pick-up** — set the Linear issue to In Progress before its first change.
- **Landing** — when its commit lands and its checks pass, leave a Linear
  comment naming what changed, the commit hash, and how it was verified,
  then move the issue to the workflow's next status.
- **Chat** — report in chat after each issue lands: one or two lines on what
  landed and what is next. Surface a blocker the moment it stops progress.

When the MRs open, comment each MR link on every issue it delivers.
