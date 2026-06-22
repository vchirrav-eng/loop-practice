# loop-practice — A Closed-Loop Dependency-Update Agent

A hands-on practice repo for **loop engineering**: designing a small autonomous
system that *prompts the agent for you* instead of you prompting it turn by turn.

This repo implements one concrete use case end to end — an **automated dependency
update loop** built on Claude Code's `/goal` (closed-loop) primitive, with a
maker/checker split between two subagents.

> Background reading: [`loop-engineering-guide.md`](./loop-engineering-guide.md),
> based on Addy Osmani's [Loop Engineering](https://addyosmani.com/blog/loop-engineering/).

---

## The use case we implemented

**Goal:** keep dependencies current without a human babysitting each bump.

The loop runs until a *verifiable* condition holds:

> Pick one outdated dependency not already in `loop-state.md`, bump it in an
> isolated worktree, independently verify it, and stop when the verifier returns
> **PASS** and a branch `deps/<pkg>-<ver>` exists with passing tests — or when no
> eligible package remains.

### How it's wired together

| Piece | Role |
|---|---|
| **`dep-update` skill** (`.claude/skills/dep-update/`) | The procedure: one bump per run, patch > minor > major, branch naming, a do-not-auto-update list, and the definition of done. |
| **`dep-bumper` subagent** (`.claude/agents/dep-bumper.md`) | The **maker**. Bumps exactly one package in an *isolated git worktree*, updates the lockfile, runs tests + lint. Does not open a PR. |
| **`dep-verifier` subagent** (`.claude/agents/dep-verifier.md`) | The **checker**. Did *not* write the change. Independently re-confirms: exactly one dep changed, not on the block list, clean-install tests pass, lint passes. Outputs **PASS / FAIL**. |
| **`loop-state.md`** | The loop's memory. Each attempt is appended under Done / Skipped so the next iteration never repeats work. |
| **`/goal`** | The closed loop itself — a fresh verifier model grades the stop condition every turn and refuses to stop until it's genuinely met. |

### The core idea: the maker is not the checker

The agent that wrote the code is **not** the one that grades it. `dep-bumper`
produces the change; a fresh `dep-verifier` re-runs a clean install and the gates
from scratch and is told to be skeptical. That separation is what makes it safe to
walk away from the loop.

### Worked result

The first run bumped `lodash 4.17.20 → 4.18.1` on branch `deps/lodash-4.18.1`,
the verifier returned PASS on all five checks, and the result was recorded in
`loop-state.md`. (See that file for the running log.)

---

## Autonomy vs. token cost — the central trade-off

Loop engineering moves the leverage point from "write a good prompt" to "design a
good loop." But a closed loop runs a **verifier on every turn**, so autonomy is
bought with tokens. This is the trade-off this repo is meant to make tangible.

### Pros — what the autonomy buys you

- **Walk-away work.** The loop finds the work, does it, checks it, and records it
  without a human in the turn-by-turn position. Your job becomes *designing the
  loop*, not driving it.
- **Trustworthy "done."** Because a fresh verifier grades the stop condition,
  "done" means *proven*, not *claimed*. The maker/checker split catches the maker's
  blind spots (e.g. here it flagged that vendored `node_modules` was inconsistent
  with the manifest).
- **State that compounds.** `loop-state.md` means the loop never repeats a bump and
  can run again tomorrow picking up exactly where it left off.
- **Isolation by default.** Each bump happens in its own worktree, so a bad change
  never pollutes the main tree and failed attempts are cheap to discard.
- **Consistency.** The same skill/procedure is applied every time — no drift in how
  bumps are branched, tested, or recorded.

### Cons — what the autonomy costs you

- **Tokens scale with rigor.** Every turn runs the verifier; every bump spins up two
  subagents (maker + checker), each doing a full clean install and test run. A trivial
  one-line version bump can consume tens of thousands of tokens. The more skeptical the
  verifier, the more it costs.
- **Diminishing returns on stub projects.** Here `npm test`/`npm lint` are `echo`
  stubs — the verifier's clean-install-and-test ritual burns tokens to confirm
  something that can't actually fail. On real test suites the cost is justified; on
  toy ones it's pure overhead.
- **Unattended mistakes are still mistakes.** A loop running without you is also a
  loop *erring* without you. A weak stop condition or a verifier you don't truly trust
  turns autonomy into automated wrongness — and you pay tokens to produce it.
- **Hidden ceiling on coverage.** Loops silently bound their work (one bump per run,
  top-N selection). Without logging what was skipped, "the loop ran clean" can quietly
  mean "the loop ignored most of the backlog."
- **Setup cost is front-loaded.** Skills, subagent definitions, state files, and a
  precise stop condition all have to exist *before* the first useful turn. For a
  one-off task this overhead never pays back.

### Rule of thumb

Spend tokens on autonomy when **"done" has to mean something** (real tests must pass,
real risk if it's wrong) and the task **recurs** (the loop amortizes its setup). Keep
a human in the loop — or use a cheap open loop that only *reports* — when the work is a
one-off, the verifier can't actually distinguish success from failure, or the budget
matters more than walking away.

> The verifier is the bottleneck, not the model. The tokens it spends are the price
> of being able to trust the loop enough to leave it alone.

---

## Layout

```
.claude/skills/dep-update/   # the procedure (skill)
.claude/agents/dep-bumper.md  # maker subagent (runs in a worktree)
.claude/agents/dep-verifier.md# checker subagent (skeptical, independent)
loop-state.md                 # the loop's running memory
loop-engineering-guide.md     # deeper walkthrough of the concepts
package.json                  # the toy project the loop operates on
```

## Try it

```bash
# In Claude Code, set the closed-loop goal:
/goal Using the dep-update skill: pick one outdated dependency not already in
loop-state.md, dispatch dep-bumper to bump it in a worktree, then dispatch
dep-verifier to check it. Stop when dep-verifier returns PASS and a branch
deps/<pkg>-<ver> exists with passing tests; or when no eligible package remains.
```
