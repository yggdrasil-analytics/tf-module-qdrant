# Lesson 11 — CI, linting, policy

## Objective
Turn the local testing pyramid from Lesson 10 into a team-usable workflow: pre-commit hooks locally,
and a CI pipeline that plans on every PR and applies only on merge, behind review.

## Principles in focus
- [**P2** — The plan is the product](principles.md#p2)
- [**P14** — The pipeline is the only path to production](principles.md#p14)

## Concepts

**`pre-commit` hooks.** Wire `terraform fmt`, `terraform validate`, `tflint`, and `terraform-docs`
into a `pre-commit` config so the static layer from Lesson 10 runs automatically before a commit can
even be made — catching the cheapest class of mistake before it ever reaches a PR, let alone CI.

**The pipeline shape:** lint → validate → test (Lesson 10's suite) → **plan on PR, posted as a PR
comment** → apply on merge, gated by required review. This is **P14** operationalized: the plan a
human reviews and approves in the PR is — by construction of the pipeline — the same plan CI executes
on merge, not a different one someone re-generates from their laptop minutes later. Posting the
rendered plan as a PR comment is what makes it the actual reviewable artifact **P2** describes,
instead of something only the pipeline operator ever sees.

**Never commit or attach the plan *file*.** The rendered plan *text*, with any secret values redacted,
is what belongs in a PR comment. The binary `.tfplan` file itself contains fully resolved values —
including secrets — and attaching or committing it publishes those secrets exactly as if they'd been
committed directly (AP-31, **P12**). This distinction — rendered summary vs. raw plan file — is easy
to blur when you're first wiring up CI; get it right from the start.

**Policy-as-code**, introduced conceptually: tools like OPA/Conftest or Sentinel evaluate a plan
against organizational rules *before* apply is permitted — "no security group may allow ingress from
`0.0.0.0/0`," for example (directly relevant once Lesson 13 introduces AWS). Where it fits in the
pipeline: after `plan`, before the apply gate, as an additional automated check alongside human
review — not a replacement for either the tests or the review.

**Credentials in CI: OIDC, never long-lived keys.** Once this pipeline needs real cloud credentials
(starting in Part V), the correct mechanism is OIDC federation — CI proves its identity to the cloud
provider per-run and receives a short-lived, scoped credential — never a long-lived access key pasted
into a CI secret store. This is set up here, conceptually, so it's already in place by the time
Lesson 13 needs it for real (AP-33).

## Build

1. `.pre-commit-config.yaml` wiring `fmt`, `validate`, `tflint`, `terraform-docs`.
2. `.github/workflows/terraform.yml` (or your CI system's equivalent): a job running lint → validate →
   `terraform test` on every PR; a separate step running `plan` and posting its rendered output as a
   PR comment; a merge-gated job running `apply` only after the PR is approved and merged.
3. Confirm the pipeline actually blocks merge on a failing test, and confirm the plan comment appears
   and is readable, not just "pipeline passed."

## Break it on purpose

**Let a bad plan through by skipping the read.** Open a PR with a change you know would `destroy` a
resource unintentionally (reuse one of the earlier lessons' sabotage drills — e.g. the unpinned
`count` cascade from Lesson 8). Let the pipeline post its plan comment, and — deliberately, once, to
feel the cost — approve and merge *without reading it closely*. Then look at what actually got
destroyed. This is the entire justification for **P2** and for gating apply behind a *read* plan, not
just an *existing* plan: the pipeline can put the right information in front of you, but it can't make
you read it.

**Try to sneak a plan file into a PR.** Save a `.tfplan` binary and attempt to attach it to a PR
comment or commit it. Open it in a text editor first and search for anything that looks like a
resolved secret or sensitive value. This is the concrete version of AP-31 — confirm your `.gitignore`
and PR template steer people away from doing this by accident.

## Pitfalls & antipatterns
- [**AP-28**](antipatterns.md#ap-28)
- [**AP-30**](antipatterns.md#ap-30)
- [**AP-31**](antipatterns.md#ap-31)

## Checkpoint

1. Why does the plan posted as a PR comment need to be the *same* plan that later gets applied on
   merge, rather than a fresh one generated at merge time?
2. What's the practical difference between the rendered plan output and the raw `.tfplan` file, and
   why does only one of them belong in a PR?
3. Where does policy-as-code fit in the pipeline relative to human review — replacing it, or alongside
   it?
4. You're ready for Part IV when: a PR against this repo produces a readable plan comment, and a
   deliberately bad change gets caught by either the test suite or a reviewer actually reading the
   plan — ideally both.
