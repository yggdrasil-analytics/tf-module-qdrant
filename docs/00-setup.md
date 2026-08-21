# Lesson 00 — Setup & mental model

## Objective
Understand what Terraform is (and isn't) well enough to place every subsequent lesson correctly, and
have a working, disciplined toolchain before writing a single resource.

## Principles in focus
- [**P1** — Declarative desired state](principles.md#p1)
- [**P6** — Immutability over mutation](principles.md#p6)

## Concepts

**What Terraform is.** A tool that reads a declarative description of infrastructure, compares it
against the real state of that infrastructure (via provider APIs), and computes + executes the
minimal set of changes to reconcile the two. It is not a scripting language, not a configuration-
management tool, and not a deployment tool for application code — those are adjacent, complementary
categories, and the most common category-confusion mistake beginners make is trying to use Terraform
for one of them.

**Placing it against tools you may already know**, because the boundary matters (**P6**):
- **Docker Compose** describes a set of containers on *one host*, for the lifetime of that host.
  Terraform describes resources against *any provider's API* — a container, a VPC, a DNS record — and
  tracks their state independently of any single machine being up.
- **Ansible** (and configuration management generally) reaches *inside* a running system and changes
  its configuration in place. Terraform's natural mode is to provision or replace, not to mutate what's
  running inside a resource.
- **A deploy script** for application code pushes a new build to already-existing infrastructure.
  Terraform is usually what created that infrastructure in the first place, and generally shouldn't be
  the tool pushing every application release.

Keeping these boundaries straight now will save you from writing "clever" Terraform later that's
really trying to be one of these other tools.

**Declarative vs. imperative**, concretely: an imperative script says *"check if a container exists;
if not, create it; then start it; then wait for health."* A declarative Terraform resource says *"a
container with these properties should exist."* Terraform computes the *how*. This is **P1**, and
it's worth sitting with before Lesson 1, because almost every early mistake is someone unconsciously
trying to write the imperative version in HCL syntax.

**The 2026 landscape.** In 2023, HashiCorp relicensed Terraform from MPL to the Business Source
License (BSL) — source-available, but with restrictions on competing commercial use. In response, a
group of major contributors forked the last MPL-licensed version and founded **OpenTofu** under the
Linux Foundation. As of 2026 the two are near-identical technically and mostly interchangeable for
learning purposes; this curriculum is written against Terraform 1.15.x (what's installed in this
environment), with any meaningful OpenTofu difference called out where it appears. Knowing this split
exists — and why — is genuinely table-stakes knowledge in the field now.

## Build

This lesson has no module code — it's entirely toolchain setup, done once:

1. **Version manager.** Install `tfenv` or `mise` rather than a single system-wide Terraform binary.
   You will end up needing different Terraform versions for different projects; pin the version this
   curriculum uses in a `.terraform-version` (tfenv) or `.tool-versions` (mise) file at the repo root.
2. **Editor tooling.** Install the Terraform/HCL language server for your editor (syntax highlighting,
   inline validation, go-to-definition across modules). Configure `terraform fmt` to run on save —
   `fmt` is opinionated and non-negotiable in professional codebases; let the tool handle it so it's
   never a topic of debate in review.
3. **`.gitignore`.** Before writing any config, create a `.gitignore` covering `.terraform/`
   (the local provider cache — large, regenerable, never committed), `*.tfstate*` (state and its
   backups — see **P3**, this is non-negotiable), `*.auto.tfvars` if it will ever hold anything
   sensitive, and `crash.log`.
4. **Docker.** Confirm Docker Engine is installed and running (`docker version`). Parts I–IV of this
   curriculum manage real Docker resources through Terraform — no cloud account needed yet.

## Break it on purpose

Verify the `.gitignore` actually works, rather than trusting that you typed it correctly: create an
empty `terraform.tfstate` file and attempt `git add .` — confirm it does *not* get staged. This is a
five-second check that has saved people from committing real secrets more than once. Do it now, while
the file is empty and harmless, so the habit of checking is in place before there's anything sensitive
in it.

**A known snag worth hitting deliberately rather than being surprised by later:** the `kreuzwerker/docker`
provider you'll use starting in Lesson 1 has had a signing-key issue on the official Terraform
Registry for versions ≥ 3.7.0 (the current maintainer isn't an org admin and can't rotate the GPG key
registries use to verify provider authenticity). If `terraform init` fails on a signature-verification
error in Lesson 1, the documented workaround is sourcing the provider from the OpenTofu registry
instead:

```hcl
terraform {
  required_providers {
    docker = {
      source  = "registry.opentofu.org/kreuzwerker/docker"
      version = "~> 3.0"
    }
  }
}
```

Don't treat this as a distraction from "real" Terraform — provider signing and registry trust *is*
part of reproducibility (**P4**), and this is exactly the kind of real-world snag a sanitized tutorial
would hide from you.

## Pitfalls & antipatterns
- [**AP-13**](antipatterns.md#ap-13) — mismatched
  CLI versions across a team. Solved here by adopting a version manager before you need one.

## Checkpoint

You're ready for Lesson 1 when you can answer, without looking back:

1. In one sentence, what's the difference between what Terraform does and what Ansible does?
2. Why does the `.gitignore` matter *before* you've written any resources, rather than once you have
   something worth protecting?
3. What's the practical difference between Terraform (BSL) and OpenTofu (MPL, Linux Foundation) in
   2026, and why might you encounter both in a real job?
4. `terraform fmt` runs on save in your editor — verify this is actually true, not assumed.
