# Grill-driven OpenSpec

A custom [OpenSpec] schema that runs a change through a **grill-first** pipeline
and delegates each stage's *technique* to the [Matt Pocock skill collection] —
`grilling`, `tdd`, `code-review`, and friends — instead of re-describing them.
The schema keeps only the OpenSpec-specific mechanics (the capability contract,
delta-spec format, checkbox tracking); the skills own the "how."

The core idea: **don't write code until the ambiguity is gone.** Every change
starts with a relentless interview (the "grill") that resolves scope,
constraints, vocabulary, and definition-of-done before a single artifact is
written. Only once that's settled does the pipeline flow forward:

```
grill → proposal → specs + design → tasks → apply → archive
```

[OpenSpec]: https://github.com/Fission-AI/OpenSpec
[Matt Pocock skill collection]: https://github.com/mattpocock/skills

## Quick start

Starting from scratch? You need four things: the OpenSpec CLI, Claude Code, the
Matt Pocock skills, and this schema.

### Prerequisites

- **[Node.js]** 18+ (provides `npm`)
- **[Claude Code]** — the agent that runs the skills and `/opsx:*` commands

[Node.js]: https://nodejs.org
[Claude Code]: https://claude.com/claude-code

### 1. Install the OpenSpec CLI

```bash
npm install -g @fission-ai/openspec
```

This gives you the `openspec` command. To wire it into a project (creating the
`openspec/` directory and installing the `/opsx:*` slash commands for Claude
Code), run once inside the project root:

```bash
openspec init
```

### 2. Install the Matt Pocock skill collection

The recommended path is the Claude Code plugin marketplace:

```bash
claude plugin marketplace add mattpocock/skills
claude plugin install mattpocock-skills@mattpocock
```

Or, from inside Claude Code:

```
/plugin marketplace add mattpocock/skills
/plugin install mattpocock-skills@mattpocock
```

Prefer editable skills copied into your project instead? Use the `skills`
installer:

```bash
npx skills@latest add mattpocock/skills
```

### 3. Configure the skills for your repo (one-time)

Run this once per repository to scaffold the config the engineering skills
assume (issue tracker, triage labels, domain-doc layout):

```
/setup-matt-pocock-skills
```

### 4. Add the grill-driven schema and select it

This repo **is** the schema — it lives at `openspec/schemas/grill-driven/`. To
use it in another project, copy that folder into your project's
`openspec/schemas/`. Then validate it and make it the default:

```bash
openspec schema validate grill-driven
```

```yaml
# openspec/config.yaml
schema: grill-driven
```

With the default set, every `openspec new change` and `/opsx:*` command uses the
grill-driven pipeline automatically. (You can also pick it per-change with
`--schema grill-driven` instead of setting the default.)

> If the skills aren't installed the pipeline still works — each stage names the
> method it needs, so you can follow it by hand — but installing them is the point.

## How the pipeline works

```
grill → proposal → specs + design → tasks → apply
```

Each artifact `requires` the one(s) before it: no `proposal` until `grill.md`
exists, no `specs`/`design` until `proposal` exists, no `tasks` until **both**
`specs` and `design` exist. `specs` and `design` don't depend on each other, so
you generate them one after another off the same proposal — eyeball both before
`tasks`, since nothing forces them to agree.

## Stage by stage

Each stage below names the skill that powers it.

**grill.md** — `grilling` (+ `research`, `domain-modeling`). Start every change
here. The agent interviews you one question at a time, always proposing its own
recommended answer so you confirm or correct rather than write essays, walking
scope, affected surfaces, constraints, edge cases, and definition of done. Two
things layer on: factual questions get looked up (codebase, or `research` for
primary sources) instead of guessed at, and new or fuzzy terms get sharpened
into a Vocabulary section via `domain-modeling` so every later artifact uses the
same words. Don't move on while `Open branches` is non-empty — that's the signal
the interview isn't done.

**proposal.md** — Built strictly from what's resolved in `grill.md`, nothing new
introduced. This is where you name the capabilities that become spec files; the
Capabilities section is the contract the specs phase reads.

**specs/** — One spec file per capability, in OpenSpec delta format (ADDED /
MODIFIED / REMOVED / RENAMED, scenarios at exactly four hashtags). Each scenario
is a testable case the apply phase can turn into a test.

**design.md** — `codebase-design` (+ `design-an-interface`, `prototype`).
Always produced (tasks depend on it), scaled to the change. Decisions use
deep-module vocabulary (small interface, hidden complexity); when a module's
interface shape is the crux, it's designed several ways and compared; and any
open question cheaper to build than to argue about gets a throwaway prototype
first, with the result written down instead of debated in prose.

**tasks.md** — Checkbox list the `apply` phase tracks progress against. For
bigger changes, task groups declare explicit blocking edges instead of relying
on numbering order, so what's actually parallelizable is visible.

**apply** — `tdd` (+ `diagnosing-bugs`, `resolving-merge-conflicts`,
`code-review`). Each task is driven test-first (agree seams → red → green). If a
bug resists a quick fix, it switches to the diagnosis loop — build a tight
feedback loop first, then reproduce → minimise → hypothesise → instrument → fix
→ regression-test — rather than guessing. A merge/rebase conflict is resolved
hunk by hunk, never aborted. Before the change is called done, `code-review` runs
a two-axis pass over the diff: **Standards** (repo conventions + a code-smell
baseline) and **Spec** (does it match proposal/specs/design), reported
separately so one doesn't water down the other.

**archive** — Once apply and the close-out review are clean, archive as usual;
specs get merged into `openspec/specs/`.

## Day-to-day commands

The core loop — one artifact at a time, stopping to look at each:

```bash
openspec new change <name> --schema grill-driven   # start (or /opsx:new)
```

```
/opsx:continue    # step through grill → proposal → specs/design → tasks
/opsx:apply       # implement with the TDD / diagnosis / review loop above
/opsx:archive     # finish; merge specs into openspec/specs/
```

Use `/opsx:continue` (one artifact at a time), **not** `/opsx:propose` (which
generates every artifact in one shot). The grill needs a real back-and-forth, so
you want to stop and look at each artifact rather than fast-forwarding past the
interview that the whole workflow is built around.

Handy OpenSpec CLI commands alongside the loop:

```bash
openspec list                # active changes, most-recently-modified first
openspec status <change>     # artifact completion status for a change
openspec view                # interactive dashboard of specs and changes
openspec validate <change>   # check a change's structure
```

### Matt Pocock helper commands

These aren't part of the pipeline — they're standalone skills you invoke by hand
whenever they're useful during a change:

| Command | What it does |
| --- | --- |
| `/handoff` | Compact the current conversation into a handoff document another agent (or a later session) can pick up. |
| `/claude-handoff` | Same, but launches a fresh background agent seeded with the summary so it continues the work immediately. |
| `/grill-me` | A one-off relentless interview to sharpen a plan or design — the grill outside of a change. |
| `/ask-matt` | Router over the collection: describe your situation and it points you at the skill or flow that fits. |
| `/research` | Investigate a question against high-trust primary sources and write the findings to a Markdown file in the repo. |
| `/code-review` | Two-axis review (Standards + Spec) of the diff since a fixed point, on demand rather than only at close-out. |

The pipeline stages fire the rest of the collection (`grilling`, `tdd`,
`diagnosing-bugs`, `codebase-design`, `design-an-interface`, `prototype`,
`domain-modeling`, `resolving-merge-conflicts`) automatically — see
[Stage by stage](#stage-by-stage).
