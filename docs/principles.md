# Principles (P1–P14)

Terraform's syntax will change over your career. HCL will get new features; the CLI will grow new
subcommands; the provider you're using today may not exist in five years. These fourteen principles
are the part that doesn't change — they're judgments about how to work with *any* declarative
infrastructure tool, and they transfer to Pulumi, CloudFormation, or whatever comes next.

Every lesson in this curriculum cites the principles in play. Read this document once up front, then
come back to it — it will mean more each time, especially after Part II.

---

<a id="p1"></a>

### P1 — Declarative desired state

You describe the *destination*, not the *route*. You write "a container running image X, publishing
port Y" — you do not write "check if a container exists, if not run `docker create`, then
`docker start`, then verify it's healthy." Terraform's job is to compute the difference between
what you declared and what exists, and to execute the minimal set of API calls to close that gap.

**Why it matters:** the moment you start writing Terraform *procedurally* — chaining
`local-exec` provisioners, using `depends_on` to force an order that should be implicit, encoding
"first do this, then do that" — you are fighting the tool. Declarative thinking is the single
biggest mental shift for people coming from scripting, and almost every early Terraform mistake
traces back to not having made it yet.

**Watch for:** yourself narrating *steps* instead of *end states* when you describe what you're about
to write.

---

<a id="p2"></a>

### P2 — The plan is the product

`terraform plan` is not a preview you skim past to get to `apply`. It is the actual deliverable —
the thing a reviewer approves, the thing that gets attached to a change record, the artifact that
answers "what is about to happen and why." An `apply` you didn't carefully read the plan for is a
production change you didn't actually authorize; you just hoped it matched what you intended.

**Why it matters:** the canonical Terraform horror story — "why did it just destroy the database" —
is, almost without exception, a story about a plan someone glanced at instead of read. `-/+ replace`
looks similar to `~ update in place` at a skim. It is not similar in consequence.

**Watch for:** running `apply` immediately after `plan` without reading the summary line
(`N to add, N to change, N to destroy`) and checking that the destroy count is what you expect —
usually zero.

---

<a id="p3"></a>

### P3 — State is the contract boundary

The state file defines what Terraform believes it owns. If a resource is in state, Terraform will
manage it — including destroying it when your config no longer describes it. If a resource is *not*
in state, Terraform doesn't know it exists, no matter how real it is. State is also where every
attribute of every managed resource ends up recorded — including anything sensitive — in plain,
readable JSON.

**Why it matters:** state isn't an implementation detail you can ignore. It's the actual boundary of
Terraform's authority and a security-sensitive artifact you're now responsible for protecting, backing
up, and never hand-editing casually.

**Watch for:** treating `terraform.tfstate` as disposable, or as something you can safely commit to
git "just this once."

---

<a id="p4"></a>

### P4 — Reproducibility is engineered, not free

Nothing about Terraform automatically guarantees that running the same config twice, a year apart,
produces the same infrastructure. That guarantee only exists if you pin provider versions, commit the
lock file, pin image/AMI references to immutable identifiers, and pin module sources to tags. Every
unpinned reference is a place where "it worked last week" can silently stop being true.

**Why it matters:** reproducibility is what makes infrastructure-as-code actually *code* — reviewable,
diffable, revertible. Without it you have a script that happens to work today.

**Watch for:** `:latest` tags, `most_recent = true` data sources, unconstrained provider version
requirements, module sources pointed at `main`/`master`.

---

<a id="p5"></a>

### P5 — Idempotency and convergence

Applying the same configuration twice in a row should change nothing the second time. This isn't a
nice-to-have — it's the actual test of whether your configuration is *correct*. If a second,
immediate `apply` proposes changes, either something outside Terraform is fighting it (drift), or
your configuration has a bug (a value that's non-deterministic, an attribute Terraform can't
represent stably).

**Why it matters:** idempotency is what lets you run `apply` in CI on every merge without fear, and
it's the property that makes "just re-run it" a safe recovery strategy after a transient failure.

**Watch for:** a config that reports changes on a second, back-to-back apply. Treat that as a bug to
fix immediately, not noise to ignore. See it drilled explicitly in every lesson's checkpoint.

---

<a id="p6"></a>

### P6 — Immutability over mutation

Terraform's natural mode is to *provision* — create, or replace-and-recreate — not to reach inside a
running resource and patch it in place like a configuration-management tool would. When you find
yourself wanting to SSH in and change something, or chain a `remote-exec` provisioner to install
software after the fact, that's usually a sign the resource should be replaced wholesale (a new
container, a new AMI) rather than mutated.

**Why it matters:** mutable infrastructure drifts. Two servers that were "the same" at creation time
and have each been patched by hand five times are no longer the same, and nothing can tell you how
they differ. Immutable infrastructure — replace, don't patch — is what keeps environments
reproducible over time (this is also why **P13** works).

**Watch for:** reaching for `local-exec`/`remote-exec` as a substitute for baking configuration into
the artifact (the image, the AMI) itself.

---

<a id="p7"></a>

### P7 — The graph, not the script

You declare *edges* — "this resource's ID feeds into that resource's argument" — and Terraform
builds a dependency graph and decides both the order of operations and what can safely run in
parallel. You do not decide execution order yourself.

**Why it matters:** `depends_on` exists for the rare case where a dependency isn't visible through
any attribute reference (e.g., IAM eventual consistency). Reaching for it out of habit, when a real
reference already implies the ordering, serializes work that could have run in parallel and — worse —
hides the actual coupling from anyone reading the config later.

**Watch for:** a `depends_on` next to an argument that already references the same resource.

---

<a id="p8"></a>

### P8 — Blast radius is an architecture decision

Every resource you put in the same Terraform state as every other resource is a resource that a
mistake — a bad `plan`, a typo, a bug in someone else's PR — can potentially destroy in the same
`apply`. Where you draw state boundaries (one state per environment? per layer? per team?) is not a
cosmetic choice; it's a direct decision about how much can go wrong at once.

**Why it matters:** the monolithic state — everything, in one `terraform.tfstate` — is the single
most consequential architecture mistake in this curriculum's antipattern catalog, and it's expensive
to unwind later (splitting state after the fact is real, careful surgery). This is worth deliberating
before you have hundreds of resources, not after.

**Watch for:** "let's just put it all in one state for now, we'll split it later." Later rarely comes
cheaply. See Lesson 12.

---

<a id="p9"></a>

### P9 — A module is an API

The moment you extract a `modules/qdrant/` directory that something else calls, its variables,
outputs, and defaults become a promise to whoever calls it — including future-you. Changing a
variable's meaning, tightening a default, or renaming a resource inside the module can break every
caller, silently, on their next `apply`.

**Why it matters:** treating a module as "just some files I organized" instead of "a versioned
interface I now support" is how modules become impossible to change safely. Every variable you add is
a variable you're committing to keep working.

**Watch for:** adding a variable "just in case," without a concrete caller need in front of you —
that's **P10**'s territory too.

---

<a id="p10"></a>

### P10 — Abstract on the rule of three

Don't build a reusable abstraction for one caller, speculating about what a second caller might
need. Duplication between two similar-but-not-identical configs is cheap — it's visible, it's easy to
diff, and it's easy to later refactor into a real abstraction once you actually have three examples
in front of you showing what varies and what doesn't. A premature abstraction, built on guesses, is
usually wrong in ways that are much more expensive to unwind than duplication ever was.

**Why it matters:** most kitchen-sink Terraform modules (80 variables, a flag for every conceivable
combination) exist because someone tried to abstract before they had evidence of what actually needed
to vary.

**Watch for:** a module with exactly one caller that already has five conditional code paths for
callers that don't exist yet.

---

<a id="p11"></a>

### P11 — Optimize for the reader and the diff

Terraform configuration is read far more often than it's written, usually by someone (including
future-you) who is debugging something at an inconvenient hour and needs to understand a `plan`
output fast. Clever HCL — deeply nested `dynamic` blocks, elaborate `for` expressions doing five
things at once — is a liability precisely when you need it least.

**Why it matters:** the plan output is also a reading exercise. A resource address that's opaque or a
diff that's hard to interpret at 3am costs real time and increases the odds of an approved-without-
understanding `apply` (see **P2**).

**Watch for:** yourself feeling clever about a one-liner. That feeling is often a warning sign, not a
compliment.

---

<a id="p12"></a>

### P12 — Secrets never live in code, state, plans, or logs

Assume all four of those are, at some point, going to be read by more people than you intended: code
in a git history that outlives the secret's rotation, state in a backend someone gets read access to,
a plan file attached to a PR, log output captured by CI. None of them are an acceptable home for a
credential, and Terraform's `sensitive = true` only redacts *console output* — it does not encrypt
the value in state.

**Why it matters:** this is the principle most often violated by well-meaning shortcuts ("I'll just
hardcode it for now") that quietly become permanent.

**Watch for:** any literal-looking secret in a `.tf` file, and any `terraform.tfstate` your backend
doesn't encrypt at rest.

---

<a id="p13"></a>

### P13 — If you can't destroy and rebuild it, you don't have IaC

If your environment has drifted — hand-edited, patched outside Terraform, dependent on manual setup
steps nobody wrote down — to the point that `terraform destroy && terraform apply` wouldn't recreate
it faithfully, then what you have is documentation that happens to be executable, not actual
infrastructure-as-code. The whole value proposition of IaC is that the config *is* the source of
truth; if it isn't, you've lost the thing you adopted the tool for.

**Why it matters:** this is why every lesson in this curriculum ends with a working `destroy` and a
verified rebuild. It's a habit, not a formality — and it's the habit that also protects you from
runaway cloud bills once Part V starts costing real money.

**Watch for:** an environment nobody's comfortable destroying, "just to be safe." That discomfort is
diagnostic.

---

<a id="p14"></a>

### P14 — The pipeline is the only path to production

Humans should review *plans*; machines should execute *applies*. The moment production changes can
come from someone's laptop, running with their personal credentials, outside any review, you've lost
the audit trail, you've introduced the possibility of concurrent conflicting applies, and you've made
"who actually ran this" a real question you can't answer after an incident.

**Why it matters:** this is where the earlier principles converge into a team practice — **P2**'s
plan-as-artifact only works as a control if it's the plan a reviewer saw, not a different one applied
minutes later from someone's terminal.

**Watch for:** any workflow where "just run it locally, it's faster" is the default for anything
beyond your own local Part I–IV Docker sandbox.
