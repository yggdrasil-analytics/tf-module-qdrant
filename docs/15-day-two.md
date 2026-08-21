# Lesson 15 — Day 2

*The final lesson, and the one with no clean endpoint — infrastructure code is never actually done.*

## Objective
Build the operational habits and recovery skills that matter after a module is "finished" and
running in production: drift detection, upgrades, incident recovery, and safe decommissioning.

## Principles in focus
- All fourteen, in practice, at once — this lesson is where they stop being individually citable and
  just become how you work.

## Concepts

**Scheduled drift detection.** A recurring (e.g. nightly) CI job running `plan -refresh-only`
against production state, on a schedule, independent of any code change — surfacing manual changes
(AP-32) or provider-side drift before they compound into a surprising diff the next time someone
actually needs to `apply` something real. `check` blocks (Lesson 7) can run as part of this same job
for behavioral assertions, not just configuration drift.

**Terraform and provider upgrades — the one-way door.** Running a newer Terraform CLI against
existing state can upgrade the state file's format; that upgrade is generally **not reversible** —
older CLI versions can't read a state file a newer one has touched. This is why Lesson 0's version-
manager discipline and Lesson 5's `required_version` constraint matter for the entire life of the
project, not just onboarding: an uncoordinated upgrade by one team member can lock everyone else out
of `apply` until they upgrade too. Treat CLI and provider upgrades as deliberate, scheduled,
team-wide events — plan them, don't let them happen by accident on someone's laptop.

**An incident playbook**, written down before you need it, not improvised during an outage:
- **State backup and recovery.** Confirm your backend's versioning (Lesson 12, AP-07) actually lets
  you restore a prior state version, and practice doing it once, deliberately, before you ever need to
  under pressure.
- **`import` for adopted resources.** When something real exists but Terraform doesn't know about it
  (created out-of-band during an incident, or recovered by hand), Lesson 9's `import` block workflow
  is how it comes back under management — reviewed, not guessed.
- **`state mv` — when it's legitimate, and when it's AP-03.** Legitimate: correcting a known,
  understood state/address mismatch, with a specific plan for what the result should look like — the
  kind of surgery Lesson 9's `moved` blocks usually handle better, but `state mv` remains a valid
  manual tool when a `moved` block isn't practical. AP-03: reaching for it to make a *confusing* error
  go away without first understanding why state and reality disagree. The line between the two is
  whether you can state, in advance, exactly what state should look like after the operation — if you
  can't, you're not ready to run it yet.

**Decommissioning.** When a module or environment is genuinely being retired — not just refactored —
`removed` blocks (Lesson 9) let you stop managing resources deliberately and reviewably, distinct from
actually destroying them, when that distinction matters (e.g., handing something off rather than
deleting it).

**What 1.14/1.15 added for exactly this kind of work**, worth knowing by name even if you don't use
every feature immediately: **list resources** (defined in `*.tfquery.hcl` files) and the
**`terraform query`** command let you enumerate real, existing infrastructure matching filters — and
optionally generate config for it — which is directly useful for discovering resources that drifted
out of management entirely. **`actions` blocks** let a provider expose operations outside the normal
create/read/update/delete lifecycle (invoking a Lambda, invalidating a CloudFront distribution) —
relevant once your operational needs go beyond what CRUD naturally expresses.

## Build

This lesson has no single "final module" deliverable — it's the operational layer around everything
built in Lessons 1–14:

1. A scheduled CI job running `plan -refresh-only` (and the Lesson 7 `check` block) against your
   deployed environment, on a cadence.
2. A written incident playbook: how to restore a prior state version, how to `import` an
   out-of-band resource, and the specific criteria (from above) for when `state mv` is appropriate.
3. Practice the state-restore step once, deliberately, in a non-production sandbox — confirm you can
   actually do it before you'd ever need to.

## Break it on purpose

**Force real drift, on a schedule, and let the detection job catch it — not you.** Make a manual,
out-of-band change to something the module manages (adjust a setting via the Docker or AWS console/CLI
directly, outside Terraform), and then let the scheduled drift-detection job run on its normal cadence
rather than checking manually. Confirm it surfaces the drift on its own. This is the actual test of
whether the operational layer you just built works — not whether you, the author, can find drift when
you're looking for it, but whether the system finds it when nobody's looking.

**Attempt a state restore under (mildly) realistic pressure.** Deliberately corrupt or lose a bit of
local state in a sandbox (delete the state file, or revert to an earlier backend version), and time
how long it takes you to fully recover using your written playbook, without improvising. If it takes
longer than you'd want during a real incident, that's useful information about the playbook, not about
you — fix the playbook.

## Pitfalls & antipatterns
- [**AP-32**](antipatterns.md#ap-32) — what scheduled drift detection exists to catch.
- [**AP-03**](antipatterns.md#ap-03) — revisit
  it here specifically in light of when `state mv` is legitimate recovery versus a shortcut around
  understanding.

## Checkpoint — and the end of the curriculum

1. Why is a state-format upgrade a one-way door, and what does that imply about how a team should
   coordinate Terraform CLI upgrades?
2. State, precisely, the criterion that separates a legitimate `state mv` from AP-03.
3. What's the difference between what `list resources`/`terraform query` can discover and what a
   normal `plan` can tell you?
4. **The real final checkpoint:** hand `examples/basic/` (or `examples/aws-basic/`) to someone who has
   never seen this repository. If they can stand up Qdrant successfully from the README alone, with no
   help from you, the module's interface (**P9**) is genuinely done — not just working, but actually
   usable by someone other than its author. That's the bar this whole curriculum has been building
   toward.

Infrastructure code doesn't have a "finished" state the way a script does — it has an ongoing
relationship with reality that drifts, ages, and needs tending. The habits in this lesson — scheduled
verification, deliberate upgrades, a rehearsed incident playbook, and reviewable decommissioning — are
what keep that relationship honest over years, not just through the first successful `apply`.
