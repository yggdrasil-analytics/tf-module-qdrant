# Glossary

Terms defined once, precisely, here — lessons link to this file rather than re-explaining a term
every time it comes up. If a lesson uses a term you don't recognize, check here first.

**Provider**
A plugin that implements CRUD operations against a specific API (AWS, Docker, GitHub, ...) and
exposes them to Terraform as resource and data source types. `docker` and `aws` are both providers;
your configuration declares which ones it needs in a `required_providers` block and configures them
in a `provider` block.

**Resource**
A block declaring one thing Terraform should create, update, or destroy to match your configuration —
`docker_container.qdrant`, `aws_instance.qdrant`. Terraform tracks each resource's real-world state
in the state file and reconciles it against your config on every `plan`.

**Data source**
A read-only lookup against a provider's API — for information Terraform doesn't manage but needs to
reference (e.g. an existing VPC's ID). Unlike a resource, Terraform never creates, changes, or
destroys what a data source reads. See AP-27 for the pitfall of reading a *managed* resource this
way.

**State (state file)**
Terraform's record of what it currently believes it manages and each resource's last-known
attributes. It's how Terraform maps your configuration to real-world objects and computes diffs. See
[`principles.md` P3](principles.md#p3) and Lesson 2.

**Backend**
Where the state file lives and how it's locked — `local` (a file on disk, the default), `s3`,
`azurerm`, `gcs`, etc. A backend is configured once, in a `backend` block, and generally cannot use
variables (see Lesson 12 on partial configuration with `-backend-config`).

**State locking**
A mechanism preventing two `apply`/`plan` operations from writing to the same state concurrently.
Local state has no locking. Remote backends typically support it (e.g. the S3 backend's native
`use_lockfile` option).

**Drift**
Any difference between what's in state (what Terraform believes exists) and what's actually real
(what the provider's API reports). Caused by manual changes outside Terraform, resources deleted
out-of-band, or attributes that changed on their own. Detected by `plan` (and specifically by
`plan -refresh-only`, which refreshes state without proposing config changes).

**Root module**
The Terraform configuration in the directory you run `terraform init`/`plan`/`apply` from. Every
Terraform run has exactly one root module.

**Child module**
A module called by another module (via a `module` block) rather than run directly. The reusable
`modules/qdrant/` this curriculum builds is a child module; `examples/basic/` is the root module that
calls it.

**Module**
A directory of `.tf` files, called via a `module` block with `source`. Deliberately not a special
kind of file or a registered artifact by itself — what makes a module meaningful is that it's called
with a defined set of inputs (`variable`s) and produces a defined set of outputs (`output`s). See
[`principles.md` P9](principles.md#p9).

**Dependency graph**
The directed acyclic graph (DAG) Terraform builds from every reference between resources
(`aws_instance.x` referencing `aws_security_group.y.id`, for example). Terraform uses this graph to
decide both correct ordering and what can safely run in parallel. See
[`principles.md` P7](principles.md#p7) and `terraform graph`.

**Plan**
The output of `terraform plan`: a proposed set of actions (create / update in place / destroy /
replace) that would bring real infrastructure in line with configuration, computed by diffing config
against refreshed state. Nothing changes until `apply` executes a plan.

**Apply**
Executes a plan's proposed actions against the real provider APIs, and updates state to match the
result.

**Lock file (`.terraform.lock.hcl`)**
Records the exact provider versions (and their checksums, per platform) that satisfied your
`required_providers` constraints on the most recent `init`. Committed to version control so every
`init` — on any machine, at any later date — resolves to the identical provider build. Distinct from
*state locking*, which is a different mechanism entirely; the shared word "lock" refers to two
unrelated things.

**Idempotent / idempotency**
A configuration is idempotent if applying it repeatedly, with no changes in between, produces no
further changes after the first apply. See
[`principles.md` P5](principles.md#p5).

**Immutable infrastructure**
An approach where resources are replaced wholesale rather than modified in place — a new container
from a new image, rather than patching a running one. See
[`principles.md` P6](principles.md#p6).

**`count` / `for_each`**
The two constructs for declaring multiple instances of a resource or module from one block. `count`
indexes by position (0, 1, 2, ...); `for_each` indexes by a stable key from a map or set. See AP-22
and Lesson 8 for why this distinction is a common source of outages.

**`moved` block**
Configuration that tells Terraform a resource has been renamed or relocated within the config, so it
updates state accordingly instead of destroying the old address and creating a new one. See Lesson 9.

**`import` block**
Configuration that brings an already-existing, unmanaged real-world resource under Terraform's
management, mapping it to a resource address in your config. See Lesson 9.

**Lifecycle block**
A nested block (`lifecycle { ... }`) inside a resource controlling how Terraform handles its
create/update/destroy behavior — `create_before_destroy`, `prevent_destroy`, `ignore_changes`,
`replace_triggered_by`. See Lesson 9 and AP-25.

**Blast radius**
How much can be affected by a single mistake in a single `apply` — directly determined by what's
sharing one state file. See [`principles.md` P8](principles.md#p8).

**Workspace (Terraform workspace)**
A mechanism for maintaining multiple named state files from one configuration directory. Distinct
from — and often confused with — separating environments by directory. See Lesson 12 for the
trade-off and the well-known argument against using workspaces for prod/staging separation.

**HCL (HashiCorp Configuration Language)**
The `.tf` file syntax Terraform configuration is written in. Terraform also accepts a JSON
equivalent (`.tf.json`), rarely used by hand.
