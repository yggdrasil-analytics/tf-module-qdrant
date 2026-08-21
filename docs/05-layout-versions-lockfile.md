# Lesson 05 — Layout, versions, lock file

## Objective
Split the config into conventional files, pin versions properly, and understand exactly what the lock
file guarantees (and what it doesn't).

## Principles in focus
- [**P4** — Reproducibility is engineered, not free](principles.md#p4)
- [**P11** — Optimize for the reader and the diff](principles.md#p11)

## Concepts

**File layout is convention, not requirement.** Terraform concatenates every `.tf` file in a
directory into one logical configuration — `main.tf`, `variables.tf`, and `outputs.tf` are not special
filenames the tool treats differently; they're a convention for *human* readers (**P11**). Split the
current single-file config into:
- `versions.tf` — `terraform { required_version, required_providers }`
- `variables.tf` — all `variable` blocks
- `outputs.tf` — all `output` blocks
- `main.tf` — the resources themselves

**Version constraint operators**, and why the choice of operator matters:
- `~> 3.0` — allows `3.x`, blocks `4.0` (the conventional choice for most root and module
  constraints — room to pick up fixes, not breaking changes).
- `>= 3.0` — allows anything `3.0` or newer, including major versions with breaking changes. This is
  AP-08 in a root config: the classic "it worked yesterday, an unrelated `init` today pulled something
  newer and different" mistake.
- `= 3.2.1` — exact pin. Appropriate rarely, and specifically *not* inside a reusable child module,
  where it can make the module's provider requirement unsatisfiable for a caller who needs a
  different exact version elsewhere (AP-09).

**The lock file (`.terraform.lock.hcl`).** Generated/updated by `init`, it records the *exact*
provider version and per-platform checksums that satisfied your constraints on that run. This is what
actually delivers **P4**'s promise — the version constraint says a *range* is acceptable, but the lock
file pins the *specific build* everyone's `init` will resolve to, until someone deliberately runs
`init -upgrade`. Commit it. On a team with mixed operating systems, run
`terraform providers lock -platform=linux_amd64 -platform=darwin_arm64 ...` for every platform anyone
actually develops on — a lock file generated on only one platform will fail `init` for everyone else
(AP-10).

**Naming conventions**, from HashiCorp's official style guide: `snake_case` throughout; don't repeat
the resource type in the resource's local name (`docker_container.qdrant`, not
`docker_container.qdrant_container`); use `this` as the local name when a resource type appears
exactly once in a module and there's no more descriptive name that adds information.

**Static checks**, run before every `plan` from here on: `terraform fmt -check` (formatting),
`terraform validate` (syntactic and internal-reference correctness, no provider calls), and `tflint`
(a broader linter catching provider-specific mistakes `validate` doesn't).

## Build

1. Split into `versions.tf` / `variables.tf` / `outputs.tf` / `main.tf`.
2. Set `required_version` for Terraform itself and `~>` constraints for the `docker` provider.
3. Run `terraform init`, confirm `.terraform.lock.hcl` is generated, and commit it.
4. Rename any resource whose local name currently repeats its type or is otherwise non-conventional.
5. Run `fmt -check` and `validate`; install and run `tflint`.

## Break it on purpose

**Loosen a constraint and feel the drift.** Change the provider constraint from `~> 3.0` to `>= 3.0`,
delete `.terraform/` and the lock file, and run `init -upgrade`. If a newer major version is
available, observe it get pulled in — an upgrade you didn't deliberately choose (AP-08). Restore the
`~>` constraint and re-init to pin back to a known-good range.

**Delete the lock file and feel reproducibility disappear.** Remove `.terraform.lock.hcl`, re-run
`init`, and diff the regenerated file against what git had. If it differs at all — even a checksum —
that's the exact gap between "my config has a version constraint" and "my config is actually
reproducible" (AP-10). Restore it from git afterward.

## Pitfalls & antipatterns
- [**AP-08**](antipatterns.md#ap-08)
- [**AP-09**](antipatterns.md#ap-09)
- [**AP-10**](antipatterns.md#ap-10)

## Checkpoint

1. Why does Terraform not care which file a given block lives in — and why do you, as the reader,
   care anyway?
2. What's the practical difference between `~>`, `>=`, and `=`, and which is correct for a *reusable
   module's* provider requirement versus a *root* configuration's?
3. The lock file and a version constraint both relate to "which provider version." What does each one
   actually guarantee that the other doesn't?
4. You're ready for Part II when: `fmt -check`, `validate`, and a basic `tflint` run all pass cleanly,
   and the lock file is committed.
