# Lesson 09 — Changing things safely

*The professional skill nobody teaches beginners: refactoring Terraform without destroying what it
manages.*

## Objective
Rename and restructure resources without triggering a destroy/recreate, adopt an existing
hand-created resource into management, and use lifecycle controls deliberately rather than as a
blunt "make the warning go away" tool.

## Principles in focus
- [**P2** — The plan is the product](principles.md#p2)

## Concepts

**`moved` blocks — rename/restructure with zero destroys.** Terraform identifies resources by
*address* (`docker_container.qdrant`), not by any underlying identity of its own. Rename that local
name to `docker_container.primary`, and — with no other guidance — Terraform sees "destroy the old
address, create the new one," even though nothing about the real container should change. A `moved`
block tells Terraform explicitly: this is the same resource, just at a new address; update state
accordingly, propose zero actual changes to the real infrastructure. This is the single most direct
demonstration in this curriculum of **P2** in practice — you can *watch* a destructive-looking rename
turn into "No changes" once the `moved` block is in place.

**`removed` blocks — stop managing without destroying.** The correct alternative to AP-04's
`terraform state rm` habit: when you genuinely want Terraform to stop managing something (without
destroying the real resource), a `removed` block makes that intent explicit and reviewable in a
`plan`, rather than an untracked, undiffed CLI command someone ran once.

**`import` blocks + `-generate-config-out` — adopt a resource created by hand.** Create a container
directly via the Docker CLI (outside Terraform entirely), then write an `import` block pointing your
module's resource address at its real ID, and run
`terraform plan -generate-config-out=generated.tf`. Terraform reads the real resource's current
configuration and generates matching `.tf` for you — review and reconcile it into your actual module
files, rather than adopting it blind.

**Lifecycle controls, used deliberately:**
- `create_before_destroy` — for a replace, create the new resource before destroying the old one
  (relevant once you care about avoiding downtime during a replace — matters more once you're on
  AWS in Part V, but the concept applies here too).
- `prevent_destroy` — hard-fails any plan that would destroy this resource. The correct home for this
  is exactly the volume from Lesson 4 — the thing you specifically don't want an accidental config
  change to be able to remove.
- `ignore_changes` — tells Terraform to stop reconciling specific attributes, because something
  *else* legitimately manages them (an autoscaler adjusting a desired count, for example). Scope it to
  the specific attribute. `ignore_changes = all` (AP-25) doesn't solve a specific problem — it
  silences *every* drift signal on that resource, including ones you'd actually want to know about.
- `replace_triggered_by` — force a replace when a referenced value changes, even if the resource's own
  arguments didn't (useful once you're baking config into an image/AMI in Part V).

**`taint` is deprecated.** Older material will mention `terraform taint`; the current mechanism is the
`-replace=<address>` flag on `plan`/`apply`, which achieves the same thing (force one specific resource
to be replaced on the next apply) without mutating state ahead of time the way `taint` did.

## Build

1. Rename `docker_container.qdrant` to something more descriptive (e.g. `docker_container.primary`),
   *without* a `moved` block first — run `plan`, observe the destroy/create it proposes.
2. Add the `moved` block (`moved { from = docker_container.qdrant, to = docker_container.primary }`),
   re-run `plan`, and confirm it now reports **"No changes."**
3. Add `lifecycle { prevent_destroy = true }` to the volume resource from Lesson 4.
4. Create a container by hand via the Docker CLI, then write an `import` block and use
   `-generate-config-out` to bring it under management; reconcile the generated config into the module.

## Break it on purpose

**Feel the transition in step 1→2 directly** — this is the whole lesson in one exercise. Run `plan`
right after the rename (before the `moved` block exists) and read the destroy/create output closely;
then add the `moved` block and run `plan` again. The jump from a destructive-looking plan to "No
changes," caused by four lines of config, is the clearest single demonstration in this curriculum of
why **P2** treats the plan as the real product: the same underlying intent (rename this resource) can
show up as either a dangerous plan or a safe one, entirely depending on whether you told Terraform
what you meant.

**Try to destroy the protected volume.** With `prevent_destroy = true` set, attempt
`terraform destroy` (or a change that would replace the volume). Confirm it hard-fails rather than
silently proceeding — this is the lifecycle control doing exactly its job.

## Pitfalls & antipatterns
- [**AP-03**](antipatterns.md#ap-03) — this
  lesson is the *correct* alternative to it, made concrete.
- [**AP-25**](antipatterns.md#ap-25)

## Checkpoint

1. Before the `moved` block existed, what did the plan for your rename actually propose, and why?
2. What's the difference in *intent* between a `removed` block and running `terraform state rm`?
3. Why does `ignore_changes = all` cause more problems than it solves, compared to scoping it to one
   attribute?
4. You're ready for Part III when: you've watched a destroy/create plan become "No changes" via a
   `moved` block, and successfully imported a hand-created resource with generated config you
   reviewed and understood.
