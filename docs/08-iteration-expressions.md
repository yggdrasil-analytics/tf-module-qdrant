# Lesson 08 — Iteration & expressions

## Objective
Support multiple Qdrant instances from one module call, and be able to explain — from having caused
it once — why `count` over a list is the single most common self-inflicted Terraform outage.

## Principles in focus
- [**P11** — Optimize for the reader and the diff](principles.md#p11)

## Concepts

**The demo that makes it permanent.** This is worth doing exactly as described, even though you
already know the "correct" answer going in — reading about this failure and *causing* it yourself
produce very different levels of caution:

1. Define three Qdrant containers using `count`, iterating over a plain list of names:
   `["alpha", "beta", "gamma"]`, with each container's name derived from `var.names[count.index]`.
   Apply it — three containers exist, indexed 0, 1, 2.
2. Now remove `"beta"` from the *middle* of the list, leaving `["alpha", "gamma"]`. Run `terraform
   plan` and read it closely. You did not ask to touch `alpha` or the container previously at index
   2 — but because `count` indexes purely by *position*, everything after the removed element shifts
   index, and Terraform proposes destroying and recreating resources you never intended to touch.
3. Rebuild using `for_each` over a **map** instead — `{ alpha = {...}, beta = {...}, gamma = {...} }`
   — where each instance's key is now the stable identity, not its position in a list. Remove `beta`'s
   entry from the map this time. `plan` now shows *exactly one* destroy — the one you actually meant —
   and nothing else moves.

That difference — one unintended cascade of destroy/recreate vs. one precise, intended destroy — is
the entire lesson, and it's why `for_each` over a map (or a set of stable string keys) is the default
choice for this curriculum whenever instances can be added, removed, or reordered independently of
each other. `count` remains legitimate for a genuinely homogeneous, positionally-meaningless set (N
identical instances where "which one" truly doesn't matter) — but that case is rarer than people
initially reach for it.

**Other expression constructs, applied where they earn their place:**
- `for` expressions — transforming a list/map into another list/map inline, e.g. deriving a map of
  container names from a set of instance configs.
- `dynamic` blocks — generate repeated *nested* blocks from a collection. Reach for these only when
  the structure itself needs to vary per caller, not merely to avoid writing two similar-looking
  blocks by hand — the latter is AP-24, and it trades a small amount of duplication for configuration
  that's genuinely harder to read at a glance (**P11**).
- `try()`, `coalesce()`, `lookup()` — safe fallbacks when a value might be absent, instead of letting
  an expression error out.
- `optional()` inside an `object({...})` type constraint — lets a structured variable have some
  fields that aren't required from every caller, with a default.

**A real error you'll likely hit, and the fix.** `for_each` requires a value known *before* apply
(it can't depend on another resource's computed attribute, because Terraform needs the set of keys to
plan the graph itself). If you try to `for_each` over something derived from a resource's output,
you'll get an explicit error naming this constraint. The fix is almost always restructuring so the
keys come from configuration/variables, not from something only known after another resource applies.

## Build

1. Convert the module to support N named Qdrant instances via `for_each` over a map variable (each
   key a stable name, each value an object of that instance's settings — host port, storage size,
   etc., using `optional()` for anything with a sensible default).
2. Confirm removing one entry from the middle of the map produces exactly one destroy in `plan`.
3. Use at least one `for` expression to derive an output (e.g., a map of instance name → resolved
   host port) from the `for_each` result.

## Break it on purpose

Do the full `count`-over-a-list demo described above, in this exact module, before converting to
`for_each` — don't skip straight to the "correct" version. Watch the unintended destroy/recreate cascade
happen to *your* Qdrant instances, including one that held data you'd want to keep. Then do the
`for_each` version and watch the same removal produce exactly the diff you intended. This before/after
contrast, felt once, is worth more than any explanation of why `for_each` is generally preferred.

## Pitfalls & antipatterns
- [**AP-22**](antipatterns.md#ap-22) — the demo above, in full.
- [**AP-24**](antipatterns.md#ap-24)

## Checkpoint

1. In your own words, why does removing a middle element from a `count`-indexed list cascade into
   unrelated destroys? Trace the mechanism, not just the rule.
2. When, if ever, is `count` still the right choice over `for_each`?
3. What does "known before apply" mean in the context of `for_each`'s keys, and what error do you get
   when you violate it?
4. You're ready for Lesson 9 when: you've caused the `count` cascade yourself, converted to
   `for_each`, and can explain the difference to someone else without notes.
