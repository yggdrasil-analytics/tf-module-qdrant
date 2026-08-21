# Antipatterns (AP-01–AP-36)

Nobody adopts an antipattern on purpose. Each entry here includes *why it's tempting* — the
reasonable-sounding shortcut that leads there — because knowing the temptation is what lets you
resist it later, under deadline pressure, once the lesson that introduced it has faded. Each entry
also names the *symptom* you'll actually observe, so you can recognize one in the wild even if you've
forgotten its number.

Lessons cite these by ID (e.g. `AP-22`). Principle cross-references point back to
[`principles.md`](principles.md).

---

## State (AP-01 – AP-07)

<a id="ap-01"></a>

**AP-01 — Committing `*.tfstate` to git.**
*Tempting because:* it's right there, it feels like "just another file to version," and it makes
state visible in code review.
*Symptom:* secrets in git history forever (rotating the secret doesn't remove it from history);
constant merge conflicts in a large JSON blob nobody can meaningfully diff.
*Fix:* `.gitignore` it from day one (Lesson 0); use a remote backend (Lesson 12). See **P3**, **P12**.

<a id="ap-02"></a>

**AP-02 — Local state for team work, with no locking.**
*Tempting because:* it's the zero-config default, and it works fine solo.
*Symptom:* two people `apply` around the same time; state gets corrupted or one person's changes are
silently lost.
*Fix:* a remote backend with locking as soon as more than one person touches the config (Lesson 12).

<a id="ap-03"></a>

**AP-03 — Hand-editing state JSON instead of `moved`/`import`/`state mv`.**
*Tempting because:* it looks like "just data," and the fix seems obvious when you're staring at the
JSON.
*Symptom:* a state file that no longer matches the schema Terraform expects; corrupted state; a
resource silently orphaned or double-managed.
*Fix:* use `terraform state mv`, `moved` blocks, or `import` blocks — all of which validate as they
go (Lesson 9).

<a id="ap-04"></a>

**AP-04 — `terraform state rm` as "fix the error."**
*Tempting because:* it makes an annoying error go away immediately.
*Symptom:* Terraform stops managing something that's still real — the resource keeps running,
untracked, until someone rediscovers it the hard way (a surprise bill, an orphaned security group).
*Fix:* understand *why* the error is happening before removing something from state. `state rm` means
"stop managing this," not "resolve this conflict."

<a id="ap-05"></a>

**AP-05 — Monolithic state for the whole estate.**
*Tempting because:* one state is simpler to reason about than several, at first.
*Symptom:* 20-minute plans; a typo in an unrelated resource's tag proposing changes across your
entire infrastructure; everyone blocked on the same state lock.
*Fix:* layer state by blast radius and rate of change (Lesson 12). See **P8**.

<a id="ap-06"></a>

**AP-06 — Spaghetti `terraform_remote_state` webs.**
*Tempting because:* it's the first tool you reach for once you've split state and need to pass data
between layers.
*Symptom:* tight coupling between layers; changing one layer's outputs breaks three others in ways
that are invisible until the next `plan`; upgrade ordering nobody can reconstruct.
*Fix:* prefer narrower interfaces — explicit IDs passed as variables, SSM parameters, data sources —
over broad `remote_state` reads (Lesson 12).

<a id="ap-07"></a>

**AP-07 — No backup or versioning on the state backend.**
*Tempting because:* it's an easy checkbox to skip when you're focused on getting `apply` to work.
*Symptom:* a bad apply or accidental `state rm` has no way back.
*Fix:* enable versioning on the state bucket/backend from the start (Lesson 12).

---

## Determinism & versioning (AP-08 – AP-13)

<a id="ap-08"></a>

**AP-08 — Unpinned providers (`>=` in a root module).**
*Tempting because:* it means you never have to manually bump a version constraint.
*Symptom:* a routine `init` on a new machine — or CI cache miss — pulls a newer provider with
different defaults, and your plan changes without you having changed anything.
*Fix:* pin with `~>` in root configs; commit the lock file (Lesson 5). See **P4**.

<a id="ap-09"></a>

**AP-09 — *Over*-pinning (`=`) inside a reusable child module.**
*Tempting because:* it feels like the safe, deterministic choice.
*Symptom:* a module that requires exactly `4.2.0` of a provider becomes uncallable by anyone whose
root module needs `4.3.0` for an unrelated reason — the constraints can't both be satisfied.
*Fix:* modules declare a *range* (`~>`); only root configurations pin tightly, and even then normally
via the lock file rather than an exact `=` constraint (Lesson 5, Lesson 6).

<a id="ap-10"></a>

**AP-10 — Not committing `.terraform.lock.hcl`, or committing one locked to a single platform.**
*Tempting because:* it looks like a generated/cache file, similar to `.terraform/`.
*Symptom:* teammates on a different OS get "no compatible package" errors on `init`; or, if not
committed at all, everyone silently gets whatever the constraint currently resolves to.
*Fix:* commit the lock file; run `terraform providers lock -platform=...` for every platform your
team actually uses (Lesson 5).

<a id="ap-11"></a>

**AP-11 — `image = "qdrant/qdrant:latest"`.**
*Tempting because:* you always want the newest version, right up until it breaks something at 2am
without you having changed a line of config.
*Symptom:* the exact same configuration behaves differently on different days, with no diff to point
to — the classic **P4** violation.
*Fix:* pin to a specific tag or digest; bump deliberately, as a reviewed change (Lesson 1).

<a id="ap-12"></a>

**AP-12 — Sourcing modules from `main` instead of a pinned `?ref=<tag>`.**
*Tempting because:* it's one character shorter and always "up to date."
*Symptom:* the module's maintainer pushes a breaking change to `main`, and every consumer's next
`init -upgrade` pulls it in, unreviewed.
*Fix:* always pin `?ref=<tag>`; bump deliberately (Lesson 6, Lesson 14).

<a id="ap-13"></a>

**AP-13 — Mismatched Terraform CLI versions across a team.**
*Tempting because:* nobody thinks to check until something breaks.
*Symptom:* one teammate's `apply` silently upgrades the state file's format to a newer version; state
format upgrades are one-way, and teammates on an older CLI can no longer read that state at all.
*Fix:* a version manager (`tfenv`/`mise`) plus `required_version` in config, enforced from Lesson 0.

---

## Module design (AP-14 – AP-21)

<a id="ap-14"></a>

**AP-14 — `provider` blocks inside a reusable module.**
*Tempting because:* it's the most direct way to make a module "just work" without asking the caller
to configure anything.
*Symptom:* the module becomes uncallable with `for_each`/`count`; removing the module cleanly becomes
difficult; HashiCorp explicitly documents this as unsupported for exactly these reasons.
*Fix:* modules inherit providers from the caller; use `configuration_aliases` if a module genuinely
needs more than one provider configuration (Lesson 6).

<a id="ap-15"></a>

**AP-15 — Kitchen-sink modules.**
*Tempting because:* exposing every possible argument feels maximally flexible and future-proof.
*Symptom:* 80 variables, most unused by any real caller, most undocumented in practice, and a module
with no real opinion about how it should be used.
*Fix:* a small, opinionated required surface with deliberate escape hatches — justify every variable
against a real caller need (Lesson 7). See **P9**, **P10**.

<a id="ap-16"></a>

**AP-16 — Thin wrapper modules.**
*Tempting because:* "wrap everything in a module" sounds like good practice in the abstract.
*Symptom:* a module that passes every argument straight through to one resource, adding a layer of
indirection and a version to track, without adding any actual value (validation, sane defaults,
composition of multiple resources).
*Fix:* only modularize where you're adding real value — composition, defaults, validation, a
narrower interface (Lesson 7).

<a id="ap-17"></a>

**AP-17 — Deep module nesting (more than 2–3 levels).**
*Tempting because:* each individual level of composition seems reasonable in isolation.
*Symptom:* resource addresses become long and unreadable (`module.a.module.b.module.c.aws_instance.x`);
plans become hard to interpret; refactoring any inner module risks breaking everything above it.
*Fix:* keep module hierarchies shallow; prefer composition at the root over deep nesting (Lesson 7).
See **P11**.

<a id="ap-18"></a>

**AP-18 — Premature abstraction: a "reusable" module with exactly one caller.**
*Tempting because:* it feels more professional to build it "properly" from the start.
*Symptom:* speculative variables nobody uses; an interface shaped by guesses rather than evidence,
usually wrong in both directions once a real second caller shows up.
*Fix:* the rule of three (Lesson 7). See **P10**.

<a id="ap-19"></a>

**AP-19 — No `examples/` directory.**
*Tempting because:* you already know how to call your own module, so it doesn't feel necessary.
*Symptom:* the module has never actually been exercised the way a real consumer would use it; the
first person to try it hits friction the author never saw.
*Fix:* an `examples/basic/` from the moment the module exists, kept plan-clean as a living test
(Lesson 6, Lesson 10).

<a id="ap-20"></a>

**AP-20 — Outputs without `sensitive = true` on credential-shaped values.**
*Tempting because:* it's easy to forget on a value that doesn't look like a "password" field.
*Symptom:* a generated API key or connection string appears in plain text in console output and CI
logs.
*Fix:* mark anything credential-shaped `sensitive = true`, and remember that this redacts *console
output only* — it does not encrypt the value in state (Lesson 7). See **P12**.

<a id="ap-21"></a>

**AP-21 — A module that configures its own backend or reads its own remote state.**
*Tempting because:* it seems convenient for the module to be "self-contained."
*Symptom:* the module can no longer be composed cleanly — callers can't control where its state
lives, and the module silently depends on infrastructure it never declared as an input.
*Fix:* modules take inputs and produce outputs; backend configuration belongs to the root that calls
them (Lesson 6).

---

## HCL & language (AP-22 – AP-27)

<a id="ap-22"></a>

**AP-22 — `count` over a list where an element can be removed from the middle.**
*Tempting because:* `count` is the first iteration construct people learn, and a list feels like the
natural way to hold "a few of these."
*Symptom:* removing or reordering any element but the last shifts every subsequent index, and
Terraform proposes destroying and recreating everything after that point — resources you never meant
to touch. **This is the single most common self-inflicted Terraform outage.**
*Fix:* `for_each` over a map (or a set of stable keys) whenever items can be added, removed, or
reordered independently (Lesson 8).

<a id="ap-23"></a>

**AP-23 — `type = any`, no `description`, no `validation` on variables.**
*Tempting because:* it's faster to write and the module "works" either way.
*Symptom:* callers pass in something the module wasn't designed for and only find out at apply time,
with an unhelpful error deep inside the module's internals.
*Fix:* concrete types, a `description` on every variable, and `validation` blocks that fail fast with
a clear message (Lesson 3, Lesson 7).

<a id="ap-24"></a>

**AP-24 — `dynamic` block abuse.**
*Tempting because:* it collapses repetitive-looking config into fewer lines.
*Symptom:* configuration nobody can review at a glance; a plan diff that's harder to interpret than
the duplication it replaced.
*Fix:* reach for `dynamic` only when the *structure itself* varies by caller, not merely to avoid
minor repetition (Lesson 8). See **P11**.

<a id="ap-25"></a>

**AP-25 — `lifecycle { ignore_changes = all }`.**
*Tempting because:* it makes an annoying, persistent drift warning disappear immediately.
*Symptom:* Terraform stops noticing *any* drift on that resource, including drift you actually cared
about — the antipattern silences the signal instead of resolving its cause.
*Fix:* scope `ignore_changes` to the specific attribute that's expected to be managed elsewhere (e.g.
an autoscaler-managed `desired_count`); never reach for `all` without a documented reason (Lesson 9).

<a id="ap-26"></a>

**AP-26 — `depends_on` where an implicit reference already exists.**
*Tempting because:* it feels like being extra explicit and safe.
*Symptom:* the dependency graph gets more serialized than it needs to be, applies get slower, and the
real coupling between resources becomes harder to spot because there are now two places claiming to
express it.
*Fix:* let attribute references do the work; reserve `depends_on` for dependencies with no visible
attribute link (Lesson 4). See **P7**.

<a id="ap-27"></a>

**AP-27 — A `data` source looking up a resource managed in the same configuration.**
*Tempting because:* it's a quick way to "get the ID" without threading a reference through.
*Symptom:* a race condition — the data source may read stale information from before the managed
resource's `apply` completes, especially on first apply.
*Fix:* reference the managed resource directly (`resource_type.name.attribute`); reserve `data`
sources for things genuinely outside this configuration's management (Lesson 4).

---

## Workflow & process (AP-28 – AP-32)

<a id="ap-28"></a>

**AP-28 — `-auto-approve` as a habit in interactive work.**
*Tempting because:* it removes a keystroke from a loop you run often.
*Symptom:* eventually, an apply goes through that you would have caught by reading the plan.
*Fix:* reserve `-auto-approve` for scripted CI applies that already gated on a reviewed plan — never
for interactive work (Lesson 1). See **P2**.

<a id="ap-29"></a>

**AP-29 — `-target` as routine practice.**
*Tempting because:* it's a fast way to apply just the one thing you're working on.
*Symptom:* the rest of the configuration silently drifts out of sync with state, because routine
applies stopped reconciling the whole graph; the next full apply produces a surprising, large diff.
*Fix:* treat `-target` as an emergency recovery tool only — for routine work, apply the whole
configuration (Lesson 4).

<a id="ap-30"></a>

**AP-30 — Applying from laptops with no CI gate.**
*Tempting because:* it's faster than waiting on a pipeline.
*Symptom:* no audit trail for who ran what; concurrent applies from different people conflict; "who
actually ran this?" becomes an unanswerable question after an incident.
*Fix:* plan-on-PR, apply-on-merge, gated by review (Lesson 11). See **P14**.

<a id="ap-31"></a>

**AP-31 — Storing or attaching plan files.**
*Tempting because:* a saved plan file seems like a convenient, precise artifact to share or archive.
*Symptom:* the plan file contains fully resolved values, including secrets — attaching it to a PR or
ticket publishes those secrets exactly as if they'd been committed.
*Fix:* never commit or attach a `.tfplan` binary; if a plan needs to be shared for review, share its
rendered text output with secrets redacted, generated by your CI pipeline (Lesson 11). See **P12**.

<a id="ap-32"></a>

**AP-32 — ClickOps: manual console changes alongside Terraform-managed infrastructure.**
*Tempting because:* it's the fastest way to make one small emergency change.
*Symptom:* Terraform's next plan either silently reverts the manual change (surprising whoever made
it) or fights it forever if the change was never reconciled back into config — genuine drift, either
way.
*Fix:* if it's managed by Terraform, it changes through Terraform; if an emergency console change was
unavoidable, bring it back into config immediately afterward (Lesson 2, Lesson 15).

---

## AWS-specific (AP-33 – AP-36)

*(Introduced in Part V, once real AWS resources enter the curriculum.)*

<a id="ap-33"></a>

**AP-33 — Long-lived IAM access keys in CI instead of OIDC federation.**
*Tempting because:* pasting an access key into a CI secret is the fastest way to get `apply` running
in a pipeline.
*Symptom:* a credential that doesn't expire, that can leak from CI logs or a compromised runner, and
that has to be manually rotated forever.
*Fix:* OIDC federation from CI to an assumed role, with no long-lived keys stored anywhere (Lesson 11).

<a id="ap-34"></a>

**AP-34 — `aws_ami` with `most_recent = true`.**
*Tempting because:* it means you never have to manually update an AMI ID.
*Symptom:* a new AMI publishes upstream (sometimes outside your control, e.g. a base-image update),
and your very next `plan` — with no config change on your part — proposes replacing running
production instances.
*Fix:* pin to a specific, known-good AMI ID; update deliberately as a reviewed change (Lesson 13).
See **P4**.

<a id="ap-35"></a>

**AP-35 — Secrets in `user_data`.**
*Tempting because:* it's the simplest way to hand an instance a credential at boot.
*Symptom:* the secret is readable via the instance metadata service by anything running on that
instance, *and* stored in plaintext in Terraform state — a double exposure.
*Fix:* fetch secrets at boot from Secrets Manager/SSM Parameter Store using the instance's IAM role,
rather than baking them into `user_data` (Lesson 13). See **P12**.

<a id="ap-36"></a>

**AP-36 — Security groups open to `0.0.0.0/0` "because it's just dev."**
*Tempting because:* it's the fastest way to stop fighting a connectivity issue while testing.
*Symptom:* for Qdrant specifically, this is not abstract — Qdrant's HTTP/gRPC API has **no
authentication by default**, so an open security group is a live, unauthenticated path to read and
write every vector in the database.
*Fix:* scope ingress to known CIDRs or a bastion/VPN; if the API must be internet-reachable, put an
API key or auth proxy in front of it (Lesson 13).
