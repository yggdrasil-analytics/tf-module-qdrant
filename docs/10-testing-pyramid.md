# Lesson 10 — The testing pyramid

## Objective
Build a layered set of tests for the Qdrant module — static, contract, unit, and a real integration
test against Docker — and understand why the layering, not any single layer, is what gives you
confidence.

## Principles in focus
- [**P13** — If you can't destroy and rebuild it, you don't have IaC](principles.md#p13)

## Concepts

**Cheapest-first, in order:**

1. **Static** — `terraform fmt -check`, `terraform validate`, `tflint`, and a security/config scanner
   (Trivy or Checkov, either works). Catches syntax errors, unresolved references, and known
   misconfiguration patterns without ever touching a real provider. Runs in seconds; run it on every
   save, not just before committing.
2. **Contract** — the `validation` blocks and `check` block from Lesson 7. These assert the module's
   *interface* is being honored (valid inputs) and that its *observable behavior* is correct
   (Qdrant actually answers `/healthz`) — without being a separate test suite; they run as a normal
   part of `plan`/`apply`.
3. **Unit** — native `terraform test` (`.tftest.hcl` files under `tests/`), run with `command = plan`
   and a `mock_provider` block standing in for the real `docker` provider. This checks your module's
   *logic* — does a given set of inputs produce the plan you expect — without needing Docker running
   at all, and without any cost. Fast enough to run on every commit.
4. **Integration** — `terraform test` again, but with `command = apply`, against the **real** `docker`
   provider. This is the payoff of building this curriculum against local Docker instead of toy
   providers or AWS from the start: a genuine create → assert → destroy integration test that runs in
   a few seconds, costs nothing, and needs no cloud account or credentials at all. Most people never
   get to write a true integration test early in their Terraform learning, because their first project
   was already against a paid cloud provider.

**`examples/` as tests, not just documentation.** Every example directory should `plan` clean at all
times — wire this into whatever you run before considering a change done. An example that's silently
gone stale is worse than no example at all, because it actively misleads the next reader.

**What "confidence" actually means here.** No single layer proves the module is correct — static
checks don't know if Qdrant actually starts; a unit test with a mocked provider doesn't know if the
real Docker API accepts your config; an integration test alone is slow enough that you won't run it on
every keystroke. The pyramid works *because* it's layered: fast, cheap checks catch most mistakes
immediately, and the slow, expensive integration test exists specifically to catch what the cheaper
layers structurally can't.

## Build

1. `tests/plan.tftest.hcl` — a unit test using `mock_provider "docker" {}`, asserting the module
   produces the expected plan (right number of resources, expected attribute values) for a couple of
   representative variable inputs, including at least one that should fail your `validation` blocks.
2. `tests/apply.tftest.hcl` — an integration test with `command = apply` against the real provider:
   apply the module, assert the container is actually running and Qdrant answers health checks, then
   let the test framework's automatic cleanup destroy everything.
3. Run both. Confirm the integration test genuinely leaves nothing behind afterward
   (`docker ps -a`/`docker volume ls` clean) — this is **P13** enforced by the test suite itself, not
   just by discipline.

## Break it on purpose

**Write a test that asserts implementation instead of contract.** Write a first draft of the plan
test that checks an internal, incidental detail (e.g., the exact number of arguments Terraform passed
to a provider call) rather than the module's actual observable contract (the right container exists
with the right published port). Then make a harmless internal refactor to the module — reorder a
block, rename a `local` — that doesn't change behavior at all, and watch the test fail anyway. This is
what a test that asserts implementation rather than contract costs you: it fails on changes that
shouldn't matter, and that failure teaches you nothing. Rewrite it against the actual contract and
confirm the same refactor now passes.

**Break the cleanup, on purpose, once.** Comment out (or misconfigure) the integration test's
teardown, run it, and then check `docker ps -a` — confirm you can see the leaked container sitting
there. This is what a test suite that doesn't honor **P13** looks like in the wild, and it's worth
having seen once so you recognize the smell (a test suite that leaves things running) immediately in
someone else's code.

## Pitfalls & antipatterns
No numbered antipatterns from the catalog are specific to testing, but two related failure modes are
worth naming directly (see above): tests that assert implementation rather than contract, and tests
that leak real resources by not destroying them.

## Checkpoint

1. Name the four layers of the pyramid in order, and what each one can catch that the layer below it
   can't.
2. Why does a mocked-provider unit test matter even though it can't tell you whether the real
   provider will accept your config?
3. What made the "implementation instead of contract" test brittle, specifically?
4. You're ready for Lesson 11 when: all four layers pass, and the integration test's teardown leaves
   `docker ps -a` and `docker volume ls` completely clean afterward.
