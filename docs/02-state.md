# Lesson 02 — State

*The most important lesson in Part I. Take your time with it.*

## Objective
Understand precisely what Terraform's state file is for, see drift happen and get detected, and be
able to cleanly answer "what's the difference between what's in state, what's in config, and what's
actually real?"

## Principles in focus
- [**P3** — State is the contract boundary](principles.md#p3)
- [**P12** — Secrets never live in code, state, plans, or logs](principles.md#p12)

## Concepts

**Why state exists at all.** Terraform needs a way to map your configuration's resource addresses
(`docker_container.qdrant`) to real-world object IDs (a specific Docker container ID), to remember
attributes the provider doesn't return cheaply on every call, and to compute dependency ordering
without re-querying every resource's full detail on every run. That mapping and metadata is
`terraform.tfstate` — a JSON file, readable by any text editor.

**Open it and read it.** After Lesson 1's `apply` (before you `destroy`), open `terraform.tfstate`
directly. Find your container's entry. Notice: it's plain, unencrypted JSON, and it contains every
attribute the provider returned — which, for some resources, includes things you'd consider
sensitive (**P12**). This single observation is why state is treated as sensitive by default in every
serious Terraform setup, not a paranoid overreaction.

**Two inspection commands**, distinct from editing:
- `terraform state list` — every resource address currently tracked.
- `terraform state show <address>` — full recorded attributes for one resource.

**The drift lab.** This is the core exercise of the lesson — the moment Terraform stops looking like
magic and starts looking like a tool with a specific, learnable model:

1. With Qdrant running (`apply` from Lesson 1's config), run `docker stop <container>` **outside**
   Terraform, from the Docker CLI directly.
2. Run `terraform plan`. Terraform refreshes state against Docker's real API, notices the container is
   no longer running, and proposes a change to reconcile it. This is drift detection: state said
   "running," reality said "stopped," and `plan` is what surfaces the gap.
3. Now `docker rm` the container entirely, outside Terraform. `plan` again — this time it proposes a
   full recreate, because the object state pointed to no longer exists at all.
4. Finally, change the container's published port using the Docker CLI directly (`docker update` or
   equivalent), then run `terraform plan -refresh-only`. This refreshes state to match reality
   *without* proposing any config-driven changes — a way to see drift in isolation from "what would
   Terraform do about it."

Sit with what just happened: state is Terraform's *belief* about the world, config is what you've
*declared* you want, and the real Docker daemon is what's *actually true*. Three different things,
and `plan` is the tool that reconciles all three every time you run it.

**Backups and locking**, introduced conceptually here and built for real in Lesson 12: state can be
corrupted or lost, so a serious backend versions it; state can be written concurrently by two people,
so a serious backend locks it. Local state (what you're using through Lesson 11) has neither — that's
fine solo, and a real problem the moment a second person touches this config.

## Build

No net-new resources this lesson. The "build" *is* the drift lab above, run against Lesson 1's
config.

## Break it on purpose

**Commit state, then walk it back.** `git add terraform.tfstate` and imagine committing it (don't
actually push if this were a shared repo — but do stage it and look at `git diff --staged`). Read
through what you just staged: full container configuration, IDs, and — depending on what the resource
type returns — potentially credentials. This is AP-01, made concrete rather than abstract. Unstage it,
confirm your `.gitignore` from Lesson 0 actually catches it going forward.

**`state rm` as a trap.** With Qdrant running, run `terraform state rm docker_container.qdrant`. Now
run `docker ps` — the container is still there, still running, completely real. But `terraform plan`
now shows it as something to *create*, because Terraform has forgotten it manages the existing one.
You've just orphaned a real resource that Terraform no longer tracks (AP-04). This is exactly how
"forgotten" cloud resources rack up surprise bills in real environments — a `state rm` used to silence
an error, rather than to deliberately stop managing something. Re-import it (or just `destroy` and
`apply` fresh) before moving on, and don't leave orphaned resources lying around.

## Pitfalls & antipatterns
- [**AP-01**](antipatterns.md#ap-01)
- [**AP-03**](antipatterns.md#ap-03)
- [**AP-04**](antipatterns.md#ap-04)

## Checkpoint

1. **The core question of this lesson:** what is the difference between what's in state, what's in
   config, and what's real? Answer this cleanly before moving on — everything else in Part I builds on
   it.
2. Why does `plan -refresh-only` exist as a separate command from `plan`? What would you lose if it
   didn't?
3. You ran `state rm` on a running container. What's now true about that container from Terraform's
   perspective, and what's still true about it in reality?
4. Why is state treated as sensitive by default, even for a resource type that "obviously" has no
   secrets in it?
