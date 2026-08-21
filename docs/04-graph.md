# Lesson 04 — Multiple resources & the dependency graph

## Objective
Add a network and a persistent volume to the Qdrant deployment, understand how Terraform decides
execution order without being told, and observe firsthand that compute and data have different
lifecycles.

## Principles in focus
- [**P7** — The graph, not the script](principles.md#p7)
- [**P8** — Blast radius is an architecture decision](principles.md#p8)

## Concepts

**Implicit dependencies.** Add a `docker_network` and a `docker_volume` (mounted into Qdrant's storage
path), and wire both into the `docker_container` resource by *referencing* their attributes
(`docker_network.qdrant.name`, `docker_volume.qdrant_storage.name`) rather than hardcoding names.
Every such reference is an edge in Terraform's dependency graph — you never declared "create the
network first"; Terraform inferred it, because the container's config literally can't be evaluated
until the network's `id`/`name` exists. Run `terraform graph` and look at the actual DAG it built from
nothing but your attribute references.

**When `depends_on` is real, not habitual.** The overwhelming majority of dependencies are implicit,
via references, and that's correct — it lets Terraform parallelize everything that doesn't actually
depend on something else (**P7**). `depends_on` exists for the rare case where a real dependency isn't
visible through any attribute (e.g., a resource whose side effects another resource needs, with no
attribute link between them). Reaching for it reflexively, next to a reference that already implies
the same ordering, is a real antipattern (AP-26) — it serializes work that could run in parallel and
duplicates, confusingly, what the reference already says.

**Resource addressing.** Every resource has an address (`docker_volume.qdrant_storage`) usable with
`-target`, `state show`, `import`, and `moved` blocks. Understand *why* `-target` is described as an
emergency-only tool in this curriculum (see AP-29): using it to apply just one resource, routinely,
means the rest of the config quietly stops being reconciled — state and reality can drift apart in
ways a later full apply then surfaces as a surprisingly large diff.

**The lesson that seeds all of Part IV.** With everything applied, destroy just the container
(`terraform destroy -target=docker_container.qdrant` — one of the legitimate emergency/exploratory
uses of `-target`), then `terraform apply` again. The container comes back; the volume was never
touched, and your data survived. **Compute and data have different lifecycles.** This single, felt
observation is the seed of blast-radius thinking (**P8**) — it's *why* production architectures
separate a stateless compute layer from a stateful data layer, often into entirely different
Terraform state (Lesson 12), and it's *why* `prevent_destroy` exists as a lifecycle safeguard
specifically for things like volumes (Lesson 9).

## Build

1. `docker_network` resource for Qdrant's network.
2. `docker_volume` resource for its storage, referenced by path in the container's mount config.
3. Update `docker_container` to reference both by attribute, not by hardcoded name/path.
4. `terraform graph` — inspect the DAG, confirm it matches your mental model of what depends on what.
5. Apply everything. Then destroy and recreate *just the container* and confirm the volume (and your
   data in it) is untouched.

## Break it on purpose

**Force serialization, then feel the cost.** Add a `depends_on = [docker_network.qdrant]` to the
container resource, even though it already references the network's attribute directly — a redundant
dependency (AP-26). It won't change correctness here (the graph was already going to order it that
way), but note in your own words why this is still wrong: it adds a second, separate place claiming to
express a relationship the code already expresses once, and in a larger graph it can genuinely
serialize work that didn't need to be serial. Remove it.

**Circular dependency.** Temporarily make the network resource reference something about the
container (e.g., a fabricated attribute dependency), creating a cycle. Run `plan` and read the error
message closely — this is what a graph the tool genuinely cannot resolve looks like, and recognizing
that error on sight will save you real confusion later in a larger config.

## Pitfalls & antipatterns
- [**AP-26**](antipatterns.md#ap-26)
- [**AP-27**](antipatterns.md#ap-27)
- [**AP-29**](antipatterns.md#ap-29)

## Checkpoint

1. Without running `terraform graph`, sketch the dependency graph for your current config from
   memory. Then run it and check yourself.
2. In one sentence, when is `depends_on` actually necessary rather than redundant?
3. You destroyed and recreated the container but not the volume. What did that prove about how you
   should think about state boundaries later?
4. Why is `-target` described as an emergency tool rather than routine practice?
