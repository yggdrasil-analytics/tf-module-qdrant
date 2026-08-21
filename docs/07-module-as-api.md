# Lesson 07 — The module as API

## Objective
Design the Qdrant module's public surface deliberately — not just make it work, but be able to
justify every variable and output it exposes.

## Principles in focus
- [**P9** — A module is an API](principles.md#p9)
- [**P10** — Abstract on the rule of three](principles.md#p10)

## Concepts

**README-driven development.** Before touching the module's internals further, write the usage
example in `examples/basic/`'s README (or its `main.tf`, commented) *as if the module were already
done* — the exact `module` block a caller would write, with exactly the variables you imagine them
setting. If that call site feels awkward — too many required arguments, an argument whose meaning
isn't obvious from its name, a nested object type nobody could guess the shape of — that's the
interface telling you something, and it's far cheaper to fix before the implementation exists than
after.

**The core tension of this lesson: P9 vs. P10, and how to resolve it in practice.** A module rigid
enough to do exactly one thing well often can't be extended for a caller with slightly different
needs (AP-18's underlying failure mode, in module-design terms). A module flexible enough to handle
every conceivable future need becomes the 80-variable kitchen sink (AP-15) — maximally flexible and,
in practice, unusable, because no one can tell which combination of the 80 arguments is actually
supported together. The resolution is not "pick a side" — it's a **small, opinionated required
surface, with deliberate, narrow escape hatches** where a real need justifies one. For each variable
you're tempted to add, the test is the same one from Lesson 3, sharpened: *would you be comfortable
writing this variable's description in the README, right now, pointing at a real reason it exists?*
If you can't, it doesn't belong yet.

**Contracts beyond types.** `validation` blocks fail a `plan` immediately, with a message you write,
if a caller passes something the module fundamentally can't handle — much cheaper than failing deep
inside a provider's API at apply time. `precondition`/`postcondition` blocks (inside a resource's or
output's `lifecycle`) assert something must be true before or after an operation. `check` blocks are a
newer, standalone construct for exactly this lesson's needs: an assertion evaluated on every `plan`
and `apply`, independent of any one resource — write one that curls Qdrant's `/healthz` endpoint (via
a `data "http"` source or similar) and fails loudly if it doesn't respond, closing the "green apply
isn't a working service" gap from Lesson 1 for good.

**`terraform-docs`.** A tool that generates an accurate, always-current input/output table directly
from your `variable`/`output` blocks and their descriptions — install it now and regenerate the
module's README section from it, so the documented interface can never silently drift from the actual
one.

## Build

1. Write `examples/basic/`'s intended usage first, then reconcile the module's actual variables
   against it — delete or rename anything that made the call site awkward.
2. Add `validation` blocks to at least two variables where an invalid value would otherwise fail
   confusingly deep inside the provider.
3. Add a `check` block asserting Qdrant's health endpoint responds after apply.
4. Install `terraform-docs`, generate the module's input/output table, and commit it into
   `modules/qdrant/README.md`.

## Break it on purpose

**Speculative variables, and the discipline of deleting them.** Add five variables you can imagine
*someone* wanting eventually — a custom log driver, a restart policy override, a resource-limits
object, whatever comes to mind — without a concrete caller in front of you needing them. For each one,
try to write its README description honestly. You'll find at least a few you can't justify beyond "it
seemed thorough" — delete those. This is the rule of three (**P10**) as a felt exercise rather than an
abstract rule: the module you'll actually maintain well is the one whose every variable you can defend
on demand.

## Pitfalls & antipatterns
- [**AP-15**](antipatterns.md#ap-15)
- [**AP-16**](antipatterns.md#ap-16)
- [**AP-17**](antipatterns.md#ap-17)
- [**AP-18**](antipatterns.md#ap-18)

## Checkpoint

1. For every variable currently in the module, can you state — out loud, in one sentence each — the
   real reason it exists?
2. What's the difference between a `validation` block and a `check` block, and when would you reach
   for each?
3. You wrote the usage example before the implementation. Did anything about writing it change your
   mind about the module's shape? If not, try it again on your next module — it usually does.
4. You're ready for Lesson 8 when: `examples/basic/` plans clean, the `check` block passes against a
   real apply, and the generated docs table matches the module exactly.
