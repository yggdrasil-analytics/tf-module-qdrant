# Terraform from Zero to Proficient, via a Qdrant Module

This repo is two things at once: a **learning path** for Terraform, and the **real, reusable module**
that path produces. You're not building throwaway exercises — every lesson leaves the actual
`tf-module-qdrant` module a little further along.

## Who this is for

Someone who has just installed the Terraform CLI and wants to become genuinely proficient — not
someone who wants to copy-paste a working `main.tf`. If you finish this and can't explain *why* a
piece of Terraform is structured the way it is, the curriculum failed, regardless of whether the
code runs.

## How to use it

1. Read [`docs/principles.md`](docs/principles.md) once, up front, even though most of it won't
   make sense yet. Re-read it after Part II — it will suddenly make a lot more sense.
2. Work the lessons in order: `docs/00-setup.md` → `docs/15-day-two.md`. Each one builds on the
   module state left by the last.
3. Do the **"Break it on purpose"** section in every lesson. Do not skip it because it feels like a
   detour. It is the highest-value part of the lesson — most of what separates someone who's used
   Terraform for a year from someone who's used it for five years is how many failures they've
   personally caused and diagnosed, on purpose, in a low-stakes setting.
4. Tag your work at each checkpoint: `git tag lesson-04` once Lesson 4 is done and destroyed cleanly.
   `git diff lesson-03..lesson-04` then shows you exactly what that lesson changed — read that diff
   before moving on. It's often more instructive than the lesson text.
5. Keep [`docs/antipatterns.md`](docs/antipatterns.md) and [`docs/glossary.md`](docs/glossary.md)
   open as references, not reading assignments. Lessons cite them by number (`AP-22`) and term.

## Prerequisites

- Terraform CLI installed (this curriculum was written against 1.15.x; see `docs/00-setup.md` for
  version-manager setup).
- Docker Engine running locally (Parts I–IV use the real `docker` provider — no cloud account, no
  cost, no credentials, until Part V).
- Comfort with a terminal and a text editor. No prior IaC experience assumed.
- An AWS account is **not** needed until Part V (Lesson 13).

## Structure of the repo as it evolves

```
CURRICULUM.md              ← you are here
docs/
  principles.md            # P1–P14 — the transferable judgment. Read first, re-read after Part II.
  antipatterns.md          # AP-01–AP-36 — grouped by what they damage, cited from every lesson.
  glossary.md              # terms defined once, precisely, linked from lessons.
  reading.md               # annotated external sources — who to follow, and what they each argue.
  00-setup.md  … 15-day-two.md
```

The module code itself grows in place as you work through the lessons:

- **L01–L05**: a flat root configuration (`main.tf`, `variables.tf`, `outputs.tf`, `versions.tf`)
  that runs Qdrant in Docker.
- **L06+**: the same logic, moved into `modules/qdrant/`, called from `examples/basic/`.
- **L10**: `tests/` — native `terraform test` files.
- **L11**: `.github/workflows/` — CI.
- **L13**: `modules/qdrant-aws/` — the same module, ported to EC2.

## The learning path

| Part | Lessons | What it answers | Cost |
|---|---|---|---|
| **I — Foundations** | 00–05 | What does Terraform actually do, resource by resource? | Free (local Docker) |
| **II — Becoming a module** | 06–09 | How do I turn working config into a reusable, safe-to-change API? | Free |
| **III — Confidence** | 10–11 | How do I know it works before it's live, and how does a team run this? | Free |
| **IV — Teams & architecture** | 12 | Where do state boundaries go, and why is that decision expensive to change? | Free |
| **V — AWS** | 13–15 | What transfers to a real cloud, what doesn't, and what does year two look like? | A few dollars, destroyed after each session |

Full lesson list:

| # | Lesson | Core question |
|---|---|---|
| [00](docs/00-setup.md) | Setup & mental model | What *is* Terraform, and what is it not? |
| [01](docs/01-first-resource.md) | Your first resource | What does the core loop actually do? |
| [02](docs/02-state.md) | State | What is Terraform's memory, and why is it dangerous? |
| [03](docs/03-variables-outputs-locals.md) | Variables, outputs, locals | How do I parameterize without over-parameterizing? |
| [04](docs/04-graph.md) | Multiple resources & the graph | How does Terraform decide order? |
| [05](docs/05-layout-versions-lockfile.md) | Layout, versions, lock file | How do I make this reproducible next year? |
| [06](docs/06-root-vs-child-module.md) | Root vs child module | What actually changes when code becomes a module? |
| [07](docs/07-module-as-api.md) | The module as API | How do I design an interface I won't regret? |
| [08](docs/08-iteration-expressions.md) | Iteration & expressions | `count` or `for_each`, and why the answer matters |
| [09](docs/09-changing-safely.md) | Changing things safely | How do I refactor without destroying production? |
| [10](docs/10-testing-pyramid.md) | The testing pyramid | How do I know it works before it's live? |
| [11](docs/11-ci-linting-policy.md) | CI, linting, policy | How does this become a team's workflow? |
| [12](docs/12-remote-state-environments.md) | Remote state, environments, layering | Where do I draw state boundaries? |
| [13](docs/13-porting-to-aws.md) | Porting to AWS | What transfers, and what was Docker-specific? |
| [14](docs/14-publishing-versioning.md) | Publishing & versioning | What do I owe the people who consume this? |
| [15](docs/15-day-two.md) | Day 2 | What does year two of this module look like? |

## The two spines

Every lesson in this curriculum touches two catalogs, and cites them by number:

- **[Principles (P1–P14)](docs/principles.md)** — the reusable judgment. Terraform's syntax will
  change over your career; these won't.
- **[Antipatterns (AP-01–AP-36)](docs/antipatterns.md)** — the specific, named ways people get hurt,
  grouped by what they damage (state, versioning, module design, HCL, workflow, AWS). Each entry
  includes *why it's tempting* — nobody adopts an antipattern on purpose, and knowing the temptation
  is what lets you resist it later, under deadline pressure, when the lesson has faded.

By the end, the goal isn't that you've memorized 50 rules. It's that when you see something wrong in
someone else's Terraform, you can say *"that's AP-22, and here's why it'll bite you"* — not just
*"that looks wrong somehow."*

## A note on "no-frills"

Lesson 1 deploys one Qdrant container with almost no configuration. That's deliberate, not a
placeholder for something more impressive. Every later lesson adds exactly one new capability or
concept and explains why it's earned its place. If at any point you feel like the module is more
complex than it needs to be for what it does *right now*, that feeling is the antipattern detector
working (see **P10**, and AP-15/AP-18) — flag it and check the relevant lesson's reasoning.
