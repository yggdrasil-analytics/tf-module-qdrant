# Lesson 03 — Variables, outputs, locals

## Objective
Parameterize the Qdrant config properly, and be able to justify — for every value — whether it should
be a `variable`, a `local`, or hardcoded.

## Principles in focus
- [**P9** — A module is an API](principles.md#p9)
- [**P10** — Abstract on the rule of three](principles.md#p10)

## Concepts

**What to parameterize, and the judgment behind it.** Turn the image tag, container name, host port,
and Qdrant's storage path into `variable`s. For each one, this lesson's real question isn't "how do I
declare a variable" (that's mechanical) — it's **"will a second caller ever plausibly set this
differently?"** If yes, it's a variable. If no — if it's just a value this config happens to compute
or reuse internally — it's a `local`. Every variable you add is a promise you're committing to support
(**P9**); conflating "things a caller configures" with "things I compute for myself" is exactly how
modules end up with 80 variables nobody asked for (see AP-15, and Lesson 7 in full).

**Variable anatomy, and why each part earns its place:**
- `type` — a concrete type (`string`, `number`, `bool`, or a structured `object({...})`), not `any`.
  A concrete type is validated at plan time, before anything touches a real API.
- `description` — mandatory in this curriculum's style, not optional. It's the first thing a future
  caller (including you, in six months) reads.
- `default` — only when there's a genuinely sensible default; a required variable with no default is
  a legitimate, useful design choice.
- `nullable` — whether `null` is a valid value, distinct from "unset."
- `sensitive` — marks the value redacted from CLI/console output (not encrypted in state — see
  **P12**, and AP-20).
- `validation` — fails fast with a clear message if a caller passes something the module can't
  actually handle (e.g. a port outside the valid range).

**Outputs** are the module's other half of its public surface — what it exposes back to whatever
calls it. In this lesson, output the container's ID and the resolved host port.

**Variable precedence** — a genuine, hours-costing gotcha if you haven't internalized it. Terraform
resolves the same variable from multiple possible sources, in this order (later wins):
1. Environment variables (`TF_VAR_name`)
2. `terraform.tfvars`, if present
3. `*.auto.tfvars` files, in alphabetical order by filename
4. `-var` / `-var-file` flags on the CLI, in the order given
5. (A variable's `default` is the fallback only if none of the above set it.)

Deliberately set the same variable in three of these places at once, predict which value wins, then
check. Getting this wrong in a real environment looks like "I changed the file and nothing happened" —
usually because a `.auto.tfvars` or an env var was silently overriding it.

## Build

1. Convert the hardcoded image tag, container name, host port, and a storage path into `variable`s
   with full `type`/`description`/`default` (where sensible)/`validation`.
2. Add `output`s for the container ID and resolved host port.
3. Introduce one `local` — something computed from a variable (e.g. a derived container name prefix)
   — specifically so you can feel the difference between a `variable` (caller sets it) and a `local`
   (you compute it, caller never sees it directly).
4. Create a `terraform.tfvars` (gitignored if it will ever hold anything env-specific) and confirm
   `plan` picks it up.

## Break it on purpose

Set the host port variable four different ways simultaneously: a `default` in `variables.tf`, a value
in `terraform.tfvars`, a value in a new `port.auto.tfvars`, and a `-var` flag on the CLI. Before
running `terraform plan`, write down which value you expect to win, using the precedence order above.
Run it. If your prediction was wrong, find out why — this exercise is worth doing slowly exactly
once, because the failure mode in real life ("I changed the value and nothing happened") is subtle and
costs real debugging time when you haven't built the instinct for it.

## Pitfalls & antipatterns
- [**AP-15**](antipatterns.md#ap-15) — this lesson is where the discipline
  against it starts, even in a flat config.
- [**AP-20**](antipatterns.md#ap-20)
- [**AP-23**](antipatterns.md#ap-23)

## Checkpoint

1. For each variable you just created, state out loud the concrete reason it's a variable and not a
   local.
2. What's the actual difference between a variable's `default` and leaving it required with no
   default? When would you choose each?
3. Walk through the precedence order from memory — CLI flags, `.auto.tfvars`, `terraform.tfvars`, env
   vars, default. Which wins?
4. `sensitive = true` on an output — what exactly does it protect, and what does it *not* protect?
