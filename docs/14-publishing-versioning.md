# Lesson 14 — Publishing & versioning

## Objective
Version the Qdrant module properly, understand what actually counts as a breaking change for a
Terraform module (it's less obvious than it sounds), and ship changes in a way that protects
consumers.

## Principles in focus
- [**P9** — A module is an API](principles.md#p9)

## Concepts

**SemVer for infrastructure modules, and the hard question underneath it.** Standard SemVer
(`MAJOR.MINOR.PATCH`) applies, but "what counts as breaking" is genuinely less obvious for a Terraform
module than for a library's function signatures. All of the following are breaking changes, and none
of them look like one at a glance:
- Renaming a variable, even if you keep the old default behavior.
- Changing a variable's *default* in a way that forces resource replacement on the caller's next
  `apply` — the interface signature didn't change, but a caller who never touched that variable now
  gets a surprise destroy/recreate.
- Restructuring resources internally such that their addresses change — without a `moved` block
  shipped *in the module itself* to cover it (see below), this destroys and recreates real
  infrastructure for every consumer on their next upgrade.
- Changing an output's type or removing one, even one that seemed unused.

The unifying question, worth asking of every change before tagging a release: **"if I only changed
this module's version pin, could this ever produce a `plan` a caller didn't expect?"** If yes, it's
breaking, regardless of how it looks in a diff.

**Registry naming and pinning.** If this module is ever published to a registry (public or private),
the convention is `terraform-<provider>-<name>` (e.g. `terraform-aws-qdrant` for the AWS variant).
Whether registry-published or just Git-sourced, every consumer's `source` should pin `?ref=<tag>` —
this is the consumer's half of the versioning contract (AP-12), just as maintaining accurate SemVer is
the maintainer's half.

**Shipping `moved` blocks *inside* the module.** When Lesson 9's rename/restructure technique is
applied to a module that already has external consumers, the `moved` block belongs in the module's
*own* source, shipped as part of the release — so that when a consumer bumps their `ref` to the new
tag, Terraform reconciles the address change automatically, with zero destroy/recreate on their end.
This is the difference between a breaking change and a transparent internal refactor, from the
consumer's point of view.

**The formal deprecation mechanism (new in Terraform 1.15).** Rather than silently removing a
variable or output in a major bump, mark it deprecated first, with a message pointing at its
replacement — giving consumers a `plan`-time warning and a migration window before the next major
version actually removes it. Use this for any variable/output you're phasing out.

**CHANGELOG and upgrade guides.** A CHANGELOG entry per release, and — for any major version bump — an
explicit upgrade guide describing exactly what a consumer needs to check or change before bumping
their `ref`. This is the artifact that makes "what counts as breaking" from above a documented
decision, not something a consumer has to reverse-engineer from a diff.

## Build

1. Tag the current state of `modules/qdrant/` (and `modules/qdrant-aws/`) as `v1.0.0`.
2. Write a CHANGELOG entry format and start one.
3. Make one deliberate, real change to the module — ideally something you've been meaning to clean
   up from an earlier lesson — and classify it honestly against the question above: patch, minor, or
   major? Tag accordingly.
4. If the change involved any renaming/restructuring, ship a `moved` block inside the module itself
   covering it, and verify a consumer bumping from the old tag to the new one gets "No changes" (or
   only the intended changes) rather than an unexpected destroy/recreate.

## Break it on purpose

**Ship a "harmless" change and discover it wasn't.** Change a variable's default value to something
that forces a resource replacement (e.g., a default that changes an immutable attribute), tag it as a
patch release (`v1.0.1`), and then — from a separate example calling the module pinned to the old tag
— bump only the `ref` to the new patch tag and run `plan`, having changed nothing else. Watch it
propose a destroy/recreate you, as the consumer, never asked for and had no reason to expect from a
patch bump. This is the exact failure mode the SemVer discipline above exists to prevent — feeling it
once, from the consumer's side, is what makes "is this actually breaking?" a real question you ask
before every release rather than a rule you nod along with.

## Pitfalls & antipatterns
- [**AP-12**](antipatterns.md#ap-12) — the
  consumer's half of this lesson's contract.

## Checkpoint

1. Give an example of a change that looks minor in a diff but is actually a major version bump, and
   explain why using the "would a version-pin-only change produce an unexpected plan" test.
2. Why does a `moved` block belong *inside* the module's own source for a breaking restructure,
   rather than being something each consumer has to write themselves?
3. What does the new deprecation mechanism buy a consumer that simply removing a variable in a major
   bump doesn't?
4. You're ready for Lesson 15 when: you've caused an unexpected-consumer-side plan from a
   mis-classified "patch," understood why, and fixed the versioning to reflect it as the major/minor
   bump it actually was.
