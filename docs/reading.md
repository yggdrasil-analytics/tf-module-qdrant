# Reading (annotated)

These sources don't all agree with each other, and the disagreements are instructive — knowing
*whose* opinion you're adopting on a given point (workspaces vs directories, how big a module should
be) matters more than treating any one of them as gospel.

- **HashiCorp** — the official [Style Guide](https://developer.hashicorp.com/terraform/language/style),
  [Standard Module Structure](https://developer.hashicorp.com/terraform/language/modules/develop/structure),
  and ["Opinionated Terraform Best Practices and Anti-Patterns"](https://www.hashicorp.com/en/resources/opinionated-terraform-best-practices-and-anti-patterns).
  Start here for anything syntax- or convention-level; it's the closest thing to a spec for "idiomatic."

- **Yevgeniy Brikman (Gruntwork)** — *Terraform: Up & Running* (3rd ed.), plus Gruntwork's
  reusable-modules blog series and production-readiness checklists. The most complete single source
  on module design and team workflows; makes the case for directory-per-environment over workspaces,
  which this curriculum follows in Lesson 12.

- **Kief Morris** — *Infrastructure as Code* (2nd ed., O'Reilly). The best source on *stack sizing*
  — how to decide what belongs in one deployable unit versus another — and on the monolithic-stack
  antipattern this curriculum's **P8** is built on. Less Terraform-specific, more broadly applicable
  to IaC as a discipline.

- **Anton Babenko** — maintainer of `terraform-aws-modules` (the most widely used community AWS
  module collection) and `awesome-terraform-compliance`. A good source for what a real, heavily-used,
  publicly-consumed module's interface design looks like in practice — read a module or two of his
  after Lesson 7 and compare its variable surface to your instincts.

- **Rosemary Wang** — *Essential Infrastructure as Code* (O'Reilly). Strong on composability and
  applying dependency-injection thinking to infrastructure, and on the testing pyramid this
  curriculum's Lesson 10 is modeled on.

- **Google Cloud Foundation Toolkit** — ["Best practices for using Terraform"](https://cloud.google.com/docs/terraform/best-practices/general-style-structure).
  Strong specifically on directory structure and state layout; useful even though this curriculum
  targets AWS, because the layering advice is cloud-agnostic.

- **OpenTofu** — the Linux Foundation–governed fork of Terraform, created after HashiCorp's 2023
  license change to the Business Source License (BSL). Near-equivalent to Terraform technically as of
  2026; worth understanding the split exists and why, since you'll encounter both in the wild. See
  Lesson 0.
