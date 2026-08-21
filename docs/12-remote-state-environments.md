# Lesson 12 — Remote state, environments, layering

*The architectural core of this curriculum. Of everything you'll decide, state boundaries are among
the most expensive to change later — take this lesson slowly.*

## Objective
Move to a remote, locked backend; understand the real trade-offs between the common ways to isolate
environments; and be able to design a state-layering strategy for the Qdrant module's eventual AWS
deployment before you build it.

## Principles in focus
- [**P3** — State is the contract boundary](principles.md#p3)
- [**P8** — Blast radius is an architecture decision](principles.md#p8)

## Concepts

**Why local state fails a team on day one.** No locking (two people applying concurrently corrupt or
silently overwrite each other's changes — AP-02), no shared visibility (nobody else can even see
current state), no durability guarantee beyond one person's disk. This isn't a hypothetical
progression — it's the first thing that breaks the moment a second person needs to run `apply`
against the same infrastructure.

**Remote backends, and state locking today.** The `s3` backend is the canonical example (and what
Part V will use). As of Terraform 1.11, its **native locking** via `use_lockfile` (S3 conditional
writes) is generally available, and the older DynamoDB-table-based locking mechanism is now
**deprecated** — though you'll still encounter it in essentially every existing codebase you inherit,
so recognize both. Backend blocks have a real constraint worth knowing early: they **cannot reference
variables** — bucket names, keys, etc. must be literal. The way around this without hardcoding
per-environment values into the block itself is **partial backend configuration**: leave the block
mostly empty and supply the rest via `-backend-config=<file>` or `-backend-config="key=value"` flags
at `init` time, one file per environment.

**State isolation strategies — a real trade-off, not a rule.** Present these fairly; the "right"
answer depends on team size, blast-radius tolerance, and tooling maturity:

- **Directory-per-environment** (`environments/dev/`, `environments/staging/`, `environments/prod/`,
  each with its own backend config, each calling the same module). Fully explicit, fully separated —
  each environment's state, and its ability to diverge from the others, is visible directly in the
  filesystem. More duplication between environment directories, though the *module* is shared and
  the duplication is confined to root-level wiring.
- **Terraform workspaces** (`terraform workspace new staging`). One configuration, multiple named
  states, switched with a CLI command. Convenient for genuinely lightweight variants (e.g. short-lived
  feature-branch sandboxes) — but there is a well-known, widely-cited argument against using them for
  prod/staging separation specifically: workspaces share the *same backend and the same code path* by
  design, so any divergence between environments has to be encoded as conditionals inside that one
  shared configuration, and it's invisible from the outside which workspace you're currently pointed
  at unless you check. That combination is exactly the shape of mistake that leads to an accidental
  apply against production.
- **Terragrunt / HCP Terraform workspaces.** Purpose-built orchestration layers on top of Terraform,
  worth knowing exist and roughly what problem each solves (DRY environment configuration, remote
  execution with policy gates) — out of scope to build here, but you should recognize them by name.

This curriculum's default, consistent with Brikman's *Terraform: Up & Running* guidance: **directory-
per-environment** for anything where prod/staging separation matters, reserving workspaces for
genuinely lightweight, same-blast-radius variants.

**Layering by rate of change and blast radius.** Kief Morris's stack-sizing guidance, applied here:
don't put your VPC/network (changes rarely, huge blast radius if wrong) in the same state as your
application deployment (changes often, small blast radius). A representative layering for where this
module is headed:

```
bootstrap  →  network  →  data  →  platform  →  app
(state backend    (VPC, subnets,    (persistent    (EC2/ECS/EKS,     (the Qdrant
 itself)           SGs)              storage)        the runtime)     module call)
```

The monolithic stack — everything in one state — is the antipattern this whole lesson exists to
argue against (AP-05): a typo in an unrelated resource's tag becomes a plan that can propose changes
across your entire infrastructure at once.

**Passing data between layers.** `terraform_remote_state` data sources can read another layer's
outputs directly, but create tight coupling — a renamed output in the network layer breaks every
layer reading it, invisibly, until their next `plan` (AP-06). Prefer narrower interfaces where
practical: passing specific IDs in as explicit variables, reading a well-defined `data` source, or
publishing values to SSM Parameter Store for other layers to read — the infrastructure equivalent of
dependency inversion: depend on a narrow, stable interface, not on another layer's entire internal
output surface.

## Build

This lesson is primarily architectural design work, applied to the module's current local-Docker form
so the pattern is already in place before Part V adds real cost and real stakes:

1. Restructure into `environments/dev/` (and optionally a second, e.g. `environments/local-demo/`),
   each with its own root config calling `modules/qdrant/`.
2. Configure a remote backend appropriate to what's available in this environment (even a
   locally-hosted, locking-capable backend for practice, since real S3 isn't needed until Part V) using
   partial configuration (`-backend-config`).
3. Write out, in a short design note, the intended layering for the eventual AWS deployment
   (bootstrap/network/data/platform/app) — this becomes the actual plan Lesson 13 executes against.

## Break it on purpose

**Simulate a lock conflict.** Start an `apply` that will run for a few seconds (or introduce an
artificial delay), and from a second terminal, immediately try to run `plan` or `apply` against the
same state. With a locking backend configured correctly, the second command should fail immediately
with a clear "state is locked" error, rather than silently racing the first. Confirm you understand
what would have happened instead against local, unlocked state (AP-02).

**Feel the monolithic-state failure mode directly.** Temporarily combine two conceptually separate
layers (e.g., a "network" resource and the Qdrant deployment itself) into one state, then make an
unrelated, trivial change to one (a tag) and run `plan`. Notice that the blast radius of that plan now
includes everything in the combined state, even though your change touched one small thing (AP-05).
Split them back apart.

## Pitfalls & antipatterns
- [**AP-02**](antipatterns.md#ap-02)
- [**AP-05**](antipatterns.md#ap-05)
- [**AP-06**](antipatterns.md#ap-06)
- [**AP-07**](antipatterns.md#ap-07)

## Checkpoint

1. Why can't a backend block reference a variable, and what's the actual mechanism (partial
   configuration) for still varying it per environment?
2. State the specific, named argument against using Terraform workspaces for prod/staging separation
   — not just "some people don't like them," the actual mechanism.
3. Sketch your intended layering (bootstrap/network/data/platform/app) for the AWS deployment before
   reading Lesson 13 — you'll compare notes against it there.
4. You're ready for Part V when: you can explain, in concrete terms, why splitting state *after* it's
   grown large and monolithic is expensive real work, not a quick refactor.
