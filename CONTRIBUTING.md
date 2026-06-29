# Contributing — branching & merge rules

This product is built with the **Enate SDLC Factory**. Two branching rules apply, depending on
who is doing the work. Direct commits to `main` are blocked in both — `main` only ever moves
through a pull request that passes CI (and review, where required).

## 1. Human-in-the-loop (HITL) work

All human work — planning docs, fixes, skill changes, anything — happens on a **branch**, opened
as a **pull request** and merged into `main`. You cannot commit to `main` directly.

## 2. AFK — the orchestrator delivering Stories (current rule)

When the orchestrator delivers work autonomously it uses **one `story/<issue#>-<slug>` branch per
Story**. The agent works only inside that branch; the orchestrator owns the branch lifecycle (the
agent performs no branch operations). On `Approved`, the Story branch is **squash-merged into
`main`**. A single Story is in flight at a time.

---

> When the orchestrator's feature-branch integration buffer ships (a `feature/<id>-<slug>` branch
> with Story branches cut from it), **this note and the branch-protection rules must be updated**
> to match — see `enate-sdlc-orchestrator` Roadmap Feature 12 / ADR-0010. Until then, the two
> rules above are the whole story; do not pre-adopt the buffer model.
