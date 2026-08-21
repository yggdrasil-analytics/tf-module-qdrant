# Lesson 01 — Your first resource

## Objective
Deploy one Qdrant container with Terraform, and understand exactly what each command in the core loop
(`init` → `plan` → `apply` → `destroy`) actually does.

## Principles in focus
- [**P1** — Declarative desired state](principles.md#p1)
- [**P2** — The plan is the product](principles.md#p2)
- [**P4** — Reproducibility is engineered, not free](principles.md#p4)

## Concepts

**One file, no frills.** At repo root, `main.tf` declares: the `docker` provider, a `docker_image`
resource that pulls a pinned Qdrant image tag, and a `docker_container` resource that runs it,
publishing container port 6333 (Qdrant's HTTP API) to a host port. That's the entire module for this
lesson — no variables, no modules, no networks. Resist the urge to add more; every later lesson adds
exactly one new idea, deliberately, and explains why.

**The core loop, slowly:**

- **`terraform init`.** Reads `required_providers`, resolves version constraints against the
  registry, downloads the matching provider plugin into `.terraform/`, and writes (or verifies)
  `.terraform.lock.hcl`. It also initializes the backend (local, by default, until Lesson 12). This is
  not "just installing dependencies" in the npm sense — it's the step that pins *which exact provider
  build* every later command in this directory will use.
- **`terraform plan`.** Refreshes state against the real provider (for existing resources), then
  diffs your configuration against that refreshed state, and prints a proposed set of actions. Learn
  to actually read the symbols, because they carry real weight:
  - `+` create
  - `-` destroy
  - `~` update in place
  - `-/+` destroy and recreate (**replace** — read this one especially carefully)
  - `<=` read (a data source)
  This is **P2** in practice: the plan is the thing you're actually reviewing, not a formality on the
  way to `apply`.
- **`terraform apply`.** Re-runs `plan` (state can move between commands), shows you the same
  proposal, and — after you type `yes` — executes it against the real provider API, then updates
  state to match the result.
- **`terraform destroy`.** Plans and executes the removal of everything currently in state. You'll
  run this at the end of every lesson in this curriculum (**P13**) — get comfortable with it now.

**A green apply is not a working service.** `terraform apply` succeeding tells you the Docker API
accepted your container's configuration. It does not tell you Qdrant actually started, bound its
port, and is answering requests. The gap between "apply succeeded" and "the thing works" is where
most real production incidents live — closing that gap yourself, by hand, every time, is the habit
this lesson is building.

## Build

1. Declare the `docker` provider (see Lesson 0's snag note if `init` fails on signature verification).
2. `docker_image` resource, pinned to a specific Qdrant tag (not `:latest` — see below).
3. `docker_container` resource referencing that image, publishing port 6333 to a host port of your
   choice.
4. `terraform init`, then `terraform plan` — read every line before moving to `apply`.
5. `terraform apply`, then verify by hand: `curl localhost:<host-port>/healthz` (or the port you
   chose). Confirm you get a real response, not just a successful `apply`.
6. `terraform destroy`. Confirm with `docker ps -a` that nothing Qdrant-related is left.

## Break it on purpose

**Part one — `:latest`.** Set the image tag to `qdrant/qdrant:latest`, `apply`, and then try to answer:
*what version of Qdrant is actually running right now?* You can't answer precisely — `:latest` is a
moving target, and your config no longer describes a specific thing (AP-11). Now pin it to a specific
tag, `plan`, and notice the difference: the plan now shows exactly what's changing and why, and if you
ran this same config again in six months you'd get the identical image.

**Part two — read the plan for real.** Change the published host port, run `plan`, and *before*
running `apply`, write down in one sentence which action symbol you expect to see and why. Then check
yourself against the actual output. If you were wrong, figure out why before moving on — this is the
skill **P2** is built on, and it's worth being deliberately slow about it exactly once.

## Pitfalls & antipatterns
- [**AP-11**](antipatterns.md#ap-11) — `:latest` and non-deterministic
  images.
- [**AP-28**](antipatterns.md#ap-28) — don't reach for
  `-auto-approve` here, even though it's tempting in a fast local loop. Build the habit of reading
  the plan every single time, starting now.

## Checkpoint

1. What does `terraform init` actually download, and where does it record what it downloaded?
2. In your own words, what's the difference between `~ update in place` and `-/+ replace` — and why
   does that difference matter for a stateful resource?
3. Why is "the apply succeeded" not sufficient evidence that Qdrant is working?
4. You're ready for Lesson 2 when: you've run the full loop (`init` → `plan` → `apply` → verify →
   `destroy`) at least twice, once with `:latest` and once pinned, and can explain the difference in
   what each `plan` told you.
