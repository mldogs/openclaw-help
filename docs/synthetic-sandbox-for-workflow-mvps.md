# Synthetic sandbox for workflow-heavy MVPs

Short practical note on adding a lightweight sandbox to an app with long stateful workflows.

## When this becomes necessary

A sandbox becomes worth it when your product is no longer one CRUD object and starts becoming a chain like:

- need
- match
- approval
- intro
- outcome
- learning/reliability update

At that point, manual clicking stops being a serious test strategy.

## The failure mode without a sandbox

Without a sandbox, teams start doing one of these bad things:

- checking only the last thing they changed
- re-testing flows manually in the UI
- relying on memory for previous states and guard conditions
- shipping workflow changes without regression coverage

That is how long workflows rot.

## Minimal structure

The smallest useful sandbox usually has:

- a dedicated `sandbox/` directory
- a separate database from production/dev data
- synthetic fixtures
- a runner script
- a machine-readable report

Example shape:

```text
sandbox/
  fixtures.js
  run.js
  README.md
  reports/latest.json
```

## The important design rule

Do **not** test a toy reimplementation of your logic if you can avoid it.

Prefer this:
- exercise the **real API**
- create the **real workflow objects**
- hit the same transitions the app uses in normal operation

Why:
- fake local logic drifts
- API-driven sandbox tests catch integration mistakes, not just local helper mistakes

## What to isolate

A separate database matters.

Use a sandbox-specific DB so that:
- test runs do not pollute the real app state
- fixtures can be deterministic
- reruns are cheap
- cleanup stays simple

## What to test first

For workflow-heavy apps, the first scenarios should be:

1. **Happy path**
   - the core flow actually completes

2. **Missing authority blocks action**
   - protected steps do not silently bypass permission checks

3. **Stale state requires refresh**
   - old data cannot glide through critical transitions

4. **Private data stays private**
   - non-disclosable claims do not leak into outward actions

5. **Outcome changes learning**
   - downstream reliability/feedback state really updates

## Why this is worth it early

A lightweight sandbox gives you:
- fast regression checks after each slice
- confidence to add workflow steps incrementally
- a reproducible artifact (`latest.json`) instead of vague "seems fine"
- less UI babysitting

## Practical workflow

After each meaningful slice:

1. run quick syntax/static checks
2. run sandbox scenarios
3. inspect failures by scenario name
4. only then commit/push

This is much cheaper than discovering broken transitions later through manual clicks.

## Keep the scenarios honest

Do not add twenty weak scenarios just to get a bigger number.

Prefer a small set of scenarios that each protect a real invariant:
- authority gating
- disclosure control
- status transitions
- outcome propagation
- ranking/selection assumptions

## Main lesson

> If the product is a workflow, build the sandbox early. Otherwise you end up developing against memory, and memory is a garbage test harness.
