# Lesson 06 — Root vs child module

*The conceptual pivot of the whole curriculum. Part I built working config; Part II turns it into
something else can safely depend on.*

## Objective
Move the Qdrant config into a proper reusable module, call it from a separate root example, and
understand exactly what did — and didn't — change in the process.

## Principles in focus
- [**P9** — A module is an API](principles.md#p9)

## Concepts

**What a module *is*, mechanically: deliberately anticlimactic.** A module is a directory of `.tf`
files, called from somewhere else via a `module` block with a `source`. That's the entire mechanical
definition — there's no special manifest, no registration step. What makes a module *meaningful* is
what it *means*: the moment something else calls it, its variables and outputs become a promise you
now maintain (**P9**), not just files you organized for your own convenience.

**The standard module structure** (HashiCorp's convention, worth following even for something this
small, because it's what every experienced reader will expect):
```
modules/qdrant/
  main.tf
  variables.tf
  outputs.tf
  versions.tf
examples/
  basic/
    main.tf        # calls module "qdrant" { source = "../../modules/qdrant" ... }
```

**Module sources.** A `source` can be a local relative path (what you'll use for now), a Git URL with
`?ref=<tag>` (what you'll use once this module is consumed from *outside* this repo, in Lesson 14), or
a registry address. Whichever it is, **always pin** — sourcing from a branch like `main` means every
consumer's next `init -upgrade` can silently pull in a breaking change the module's author pushed
without any coordination (AP-12).

**The rule you must not break: no `provider` blocks inside a reusable module.** Providers are
configured once, by whatever calls the module, and inherited implicitly. A module that configures its
own `provider` block breaks `for_each`/`count` on the *module call itself* — HashiCorp's tooling
explicitly can't reconcile multiple provider configurations with multiple module instances — and it
makes the module harder to remove or restructure cleanly later. If a module genuinely needs to work
against more than one provider configuration (multi-region, for example), the mechanism is
`configuration_aliases`, passed in explicitly by the caller — not a `provider` block declared inside
the module itself. This is AP-14, and it's one of the few rules in this curriculum with essentially no
legitimate exception for a reusable module.

## Build

1. Create `modules/qdrant/` and move the current root config's resources, variables, and outputs into
   it verbatim (network, volume, container — everything from Lessons 1–5).
2. Create `examples/basic/main.tf` that configures the `docker` provider (this now lives at the
   *root*, not inside the module) and calls `module "qdrant" { source = "../../modules/qdrant" ... }`,
   passing through the variables you designed in Lesson 3.
3. From `examples/basic/`, run the full loop: `init`, `plan`, `apply`, verify, `destroy`.
4. Confirm the module itself contains zero `provider` blocks.

## Break it on purpose

**Violate the rule, on purpose, to feel why it exists.** Temporarily add a `provider "docker" {}`
block inside `modules/qdrant/`. Then, in `examples/basic/main.tf`, try calling the module with
`for_each` over a small map (e.g., to stand up two independently-named instances). Read the error
Terraform gives you. This is AP-14 made concrete rather than a rule you're just asked to trust — once
you've seen the actual failure, you won't need to be reminded of it. Remove the `provider` block from
the module afterward.

## Checkpoint tag
Tag this lesson: `git tag lesson-06`. Then run `git diff lesson-05..lesson-06` and read it end to end.
Confirm you can narrate, out loud, exactly what moved and why — the resources themselves shouldn't
have changed at all; only their location and how they're invoked should have.

## Pitfalls & antipatterns
- [**AP-12**](antipatterns.md#ap-12)
- [**AP-14**](antipatterns.md#ap-14)
- [**AP-19**](antipatterns.md#ap-19)
- [**AP-21**](antipatterns.md#ap-21)

## Checkpoint

1. What, precisely, makes a directory of `.tf` files "a module" versus just "some Terraform files"?
2. Why does a `provider` block inside a reusable module break `for_each` on the module call — walk
   through the mechanism, not just the rule.
3. What actually changed in `git diff lesson-05..lesson-06`, and — just as importantly — what didn't?
4. Why does `examples/basic/` matter even though you, the module's author, already know how to call
   your own module?
