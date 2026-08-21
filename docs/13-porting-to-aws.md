# Lesson 13 — Porting the module to AWS

*Part V begins here. From this point on, applying real resources costs real (small) money — destroy
after every session, without exception (P13).*

## Objective
Port the Qdrant module to run on EC2, and see directly that the Terraform you already know didn't
change — only the provider did.

## Principles in focus
- [**P1** — Declarative desired state](principles.md#p1)
- [**P6** — Immutability over mutation](principles.md#p6)

## Concepts

**The mapping, made real.** Every local-Docker concept from Parts I–IV has a direct AWS counterpart —
this is exactly why this curriculum started local:

| Local (Docker) | AWS | What's actually the same |
|---|---|---|
| `docker_image` | AMI (pinned ID, or a launch template) | An immutable, versioned artifact reference — pin it, don't chase `latest`/`most_recent` (**P4**, AP-34) |
| `docker_container` | `aws_instance` | The compute unit; replace-vs-update behavior works identically |
| `docker_volume` | `aws_ebs_volume` | A stateful resource with a lifecycle independent of compute — the Lesson 4 lesson, unchanged |
| `docker_network` | VPC + subnet + security group | The network boundary; still an implicit dependency via references |
| port binding | security group ingress rule | The exposure surface — still a least-privilege decision |

Build `modules/qdrant-aws/` with this mapping directly: `aws_instance` (pinned AMI), `aws_security_group`
(scoped ingress, not `0.0.0.0/0` — see below), `aws_ebs_volume` attached to the instance, Qdrant
started via Docker inside `user_data`. The variables and outputs you designed in Lesson 7 should
require only modest changes — that continuity is the point.

**AWS authentication, done properly, from the start.** Never hardcoded access keys. Use an AWS SSO
profile (or assume-role from an existing session) locally; this repo will use OIDC federation in CI
once Lesson 11's pipeline needs real credentials (AP-33). Set this up before writing any `aws_*`
resource, not as an afterthought.

**`default_tags` on the provider.** Set once, applied to every resource the provider creates —
`Environment`, `ManagedBy = "terraform"`, `Module = "qdrant"`. A tagging strategy isn't decoration; in
a real account it's how cost gets attributed and how "what is this resource and can I delete it"
gets answered six months from now.

**`user_data` vs. a baked AMI — a direct application of P6.** `user_data` runs once, at first boot,
to configure the instance (install Docker, pull the Qdrant image, run it) — it's the pragmatic choice
for this lesson's scope. The more production-shaped alternative is baking a custom AMI with Packer
that already has everything installed, so the instance boots ready rather than configuring itself —
worth naming as the next step this module *could* take, without building it here (see **P10**: don't
add that complexity until a real need justifies it).

**Qdrant-specific, and genuinely important: its API has no authentication by default.** This isn't a
generic "secure your resources" reminder — it's a concrete, live risk specific to this exact module.
An `aws_security_group` with ingress open to `0.0.0.0/0` on Qdrant's ports is a direct,
unauthenticated path for anyone on the internet to read and write every vector in the database
(AP-36). Scope ingress to known CIDRs (your IP, a VPN range, a bastion) by default. If the API
genuinely needs to be internet-reachable, an API key or auth proxy in front of it is required, not
optional — and that key belongs in Secrets Manager, fetched at boot via the instance's IAM role, never
baked into `user_data` (which is both readable via the instance metadata service *and* stored in
plaintext in state — AP-35).

**Cost discipline.** Set a budget alarm before your first real `apply`. `terraform destroy` at the end
of every session — this is **P13**, now with a dollar cost attached to skipping it. Consider
`infracost` in the CI pipeline from Lesson 11 to surface the cost of a plan before it's applied.

## Build

1. `modules/qdrant-aws/`: `aws_instance` (pinned AMI, not `most_recent`), `aws_security_group` scoped
   to a specific CIDR, `aws_ebs_volume` attached and mounted, `user_data` installing Docker and running
   Qdrant, `default_tags` on the provider.
2. `examples/aws-basic/` calling it, following the same example-first discipline from Lesson 7.
3. Set a budget alarm in the AWS account before the first `apply`.
4. Apply, verify Qdrant answers over the instance's IP (from an allowed CIDR only), then `destroy`
   immediately — every single session, no exceptions.

## Break it on purpose

**Change `user_data`, read the plan, and reason about the consequence.** Modify the `user_data`
script (even trivially — a comment, or a version bump in the install command) and run `plan`.
`user_data` changes force instance replacement by default — read the plan and confirm it proposes a
full `-/+ replace`, not an in-place update. Now connect that to Lesson 4's compute-vs-data lesson:
because your EBS volume is a *separate resource* from the instance, the replacement destroys and
recreates the compute, but — if wired correctly — the volume and its data survive. If it doesn't
survive in your current config, that's the bug to find and fix here, before it's a real incident.

**Try the open security group, on purpose, in a disposable sandbox only.** Temporarily set an ingress
CIDR to `0.0.0.0/0`, apply, and — from outside the security group's intended access, if you have a way
to test that safely — confirm you can reach Qdrant's unauthenticated API. This is AP-36 made concrete
rather than abstract. Destroy this configuration immediately afterward and never leave it applied.

## Pitfalls & antipatterns
- [**AP-33**](antipatterns.md#ap-33)
- [**AP-34**](antipatterns.md#ap-34)
- [**AP-35**](antipatterns.md#ap-35)
- [**AP-36**](antipatterns.md#ap-36)

## Checkpoint

1. Walk through the full local-Docker-to-AWS mapping table from memory before checking it against
   this lesson.
2. Why does a `user_data` change force a full instance replacement, and what has to be true about
   your volume's configuration for that replacement to be safe?
3. Concretely — not generically — why is an open security group on Qdrant's ports worse than on many
   other services?
4. You're ready for Lesson 14 when: you've applied and destroyed this module against real AWS at
   least once, confirmed the budget alarm is in place, and left nothing running afterward.
