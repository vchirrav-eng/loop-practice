# Loop Engineering: A Step-by-Step Guide

*A practical walkthrough based on Addy Osmani's [Loop Engineering](https://addyosmani.com/blog/loop-engineering/) (June 7, 2026). Worked example: a **dependency-update loop** in **Claude Code**.*

---

## 1. What loop engineering actually is

For two years, getting work out of a coding agent meant *you* prompting it: you type a thing, read what came back, type the next thing. You are holding the tool the entire time, one turn after another.

**Loop engineering replaces you as the prompter.** Instead of prompting the agent, you design a small system that finds the work, hands it out, checks it, writes down what's done, and decides the next thing — and you let *that* system poke the agent instead of you.

> Boris Cherny (head of Claude Code): "I don't prompt Claude anymore. I have loops running that prompt Claude... My job is to write loops."

The shift in one line: **the leverage point moved from writing good prompts to designing good loops.** The work didn't get easier — it got higher-leverage.

A useful mental picture: the agent already runs an *inner* loop every turn (reason → act → observe → reason again). Loop engineering wraps an *outer* loop around that: it runs on a timer, spawns helpers, and feeds itself.

---

## 2. Open loop vs. closed loop

This is the single most important design decision. It's borrowed from control-systems theory.

| | **Open loop** | **Closed loop** |
|---|---|---|
| **Definition** | Runs a task on a cadence; no feedback deciding whether it succeeded. | Keeps acting until a *verifiable condition* is true; a checker grades each turn. |
| **Claude Code primitive** | `/loop` (re-run on a schedule) | `/goal` (run until a stop condition holds) |
| **Stops when** | The schedule fires / the single run finishes. | A separate small model confirms the goal is met. |
| **Good for** | Discovery, triage, reports, reminders — "surface the work." | Doing the work to a standard — "fix it until tests pass." |
| **Risk** | Drifts; can repeat the same mistake unattended. | Costs more tokens (the verifier runs every turn); needs a *precise* stop condition. |

**Rule of thumb:** choose by *need-for-novelty × budget-you'll-risk*. Use an **open loop** to find and report (cheap, no judgment needed). Use a **closed loop** the moment "done" needs to actually *mean* something (the agent has to prove it, not claim it).

The key insight that makes closed loops trustworthy: **the agent that wrote the code is not the one that grades it.** `/goal` uses a fresh model as the verifier after every turn — the "maker/checker split" applied to the stop condition itself.

> In any loop, **the verifier is the bottleneck, not the model.** A loop running unattended is also a loop making mistakes unattended, so a verifier you genuinely trust is the only reason you can walk away.

---

## 3. The 6 building blocks of a good loop

Five primitives, plus a sixth thing (memory) that ties them together. Both Claude Code and Codex ship all of these now.

| # | Block | Job in the loop | Claude Code primitive |
|---|-------|-----------------|------------------------|
| 1 | **Automations** | Discovery + triage on a schedule. The *heartbeat* — it's what makes a loop a loop and not a one-off. | Scheduled tasks / cron, `/loop`, `/goal`, hooks, GitHub Actions |
| 2 | **Worktrees** | Isolate parallel work so two agents don't edit the same file and collide. | `git worktree`, `--worktree` flag, `isolation: worktree` on a subagent |
| 3 | **Skills** | Codify project knowledge so the agent stops guessing your conventions every session. | Agent Skills — a folder with `SKILL.md` + optional scripts/refs |
| 4 | **Plugins / connectors** | Connect the loop to your real tools (issue tracker, CI, Slack, DB) so it can *act*, not just suggest. | MCP servers; plugins to bundle/distribute |
| 5 | **Sub-agents** | Split the *maker* from the *checker* — one drafts, a different one verifies. | Subagents in `.claude/agents/`, agent teams |
| 6 | **State / memory** | Remember what's done and what's next, *outside* the conversation. The spine of the loop. | Markdown (`AGENTS.md`, progress files) or Linear via MCP |

**Why each matters in one line:**

- **Automations** turn intent into a recurring heartbeat — you stop being the one who goes around checking.
- **Worktrees** remove mechanical collisions, but *your review bandwidth is still the ceiling* on how many you can run.
- **Skills** are your intent written down on the outside, so the loop *compounds* instead of re-deriving your project from zero every cycle.
- **Connectors** are the difference between "here is the fix" and a loop that opens the PR, links the ticket, and pings the channel once CI is green.
- **Sub-agents** matter *specifically because the loop runs while you aren't watching* — a trusted verifier is what lets you walk away.
- **State** survives because the model forgets everything between runs. The agent forgets; the repo doesn't.

---

## 4. The canonical loop shape (from the article)

> An automation runs every morning. Its prompt calls a triage skill that reads yesterday's CI failures, open issues, and recent commits, and writes the findings to a markdown file or Linear board. For each finding worth doing, the thread opens an isolated worktree and sends a sub-agent to draft the fix; a second sub-agent reviews that draft against the project skills and existing tests. Connectors open the PR and update the ticket. Anything the loop can't handle lands in a triage inbox for you. The state file remembers what was tried, what passed, what's still open — so tomorrow's run picks up where today stopped.

You designed it **once**. You prompted **none** of those steps. That's the whole point.

---

## 5. Worked example — a dependency-update loop in Claude Code

**Goal:** every week, check for an outdated dependency, bump *one* of them in isolation, run the full test suite, and open a PR **only if it's green**. Otherwise record why it failed and move on. This is a *closed loop* (the stop condition is "tests pass"), driven by an *open-loop* schedule (weekly).

We'll build all 6 blocks. Adjust paths/commands to your stack (examples assume a Node repo; swap `npm` for `pip`/`cargo`/etc.).

### Step 0 — Prerequisites

- A repo with a working test suite and a lockfile.
- Claude Code installed and authenticated in that repo.
- A git remote you can push branches to.

### Step 1 — Block 6 first: the state file (memory)

Always build memory first; everything writes to it. Create `loop-state.md` at the repo root:

```markdown
# Dependency Update Loop — State

## In progress
- (none)

## Done
- (none)

## Skipped / blocked (with reason)
- (none)

## Last run
- timestamp: (none)
```

This lives on disk, *outside* any single conversation, so next week's run knows what already happened.

### Step 2 — Block 3: a Skill that codifies "how we update deps here"

Create `.claude/skills/dep-update/SKILL.md`. A tight, boring description makes the agent invoke it at the right time:

```markdown
---
name: dep-update
description: Update a single outdated dependency, run tests, and prepare a PR. Use for routine dependency bumps in this repo.
---

# Dependency Update Procedure

## Conventions
- Bump exactly ONE dependency per run (never batch).
- Prefer the lowest-risk update available: patch > minor > major.
- Never bump a package on the DO-NOT-AUTO-UPDATE list below.
- Branch naming: `deps/<package>-<new-version>`.

## Steps
1. Run the audit command to list outdated packages: `npm outdated --json`.
2. Pick ONE package not already in loop-state.md "Done" or "Skipped".
3. Update only that package and its lockfile.
4. Run the full test suite: `npm test`.
5. Run the linter: `npm run lint`.

## Do-not-auto-update
- (list fragile/pinned packages here, e.g. `react`, `webpack`)

## Definition of done
- The single bump is committed on its own branch AND `npm test` exits 0 AND `npm run lint` exits 0.
```

This is your *intent written down once*, so the loop doesn't re-derive your rules every week.

### Step 3 — Block 5: maker and checker sub-agents

Create two agents in `.claude/agents/` so the one who writes isn't the one who grades.

`.claude/agents/dep-bumper.md` (the maker):

```markdown
---
name: dep-bumper
description: Drafts a single dependency bump following the dep-update skill.
isolation: worktree
---
You bump exactly one dependency per the dep-update skill, in an isolated
worktree. Update the package + lockfile, then run the tests and lint.
Report: package, old version, new version, and test/lint results.
Do not open a PR. Do not touch loop-state.md.
```

`.claude/agents/dep-verifier.md` (the checker — ideally a stronger model on higher effort):

```markdown
---
name: dep-verifier
description: Independently verifies a dependency bump before it ships.
---
You did NOT write this change. Independently confirm:
1. Exactly ONE dependency changed (diff the lockfile).
2. The package is not on the do-not-auto-update list.
3. `npm test` passes from a clean install.
4. `npm run lint` passes.
Output a clear PASS or FAIL with the specific failing check. Be skeptical.
```

`isolation: worktree` on the maker gives it a fresh checkout (Block 2 — **worktrees**) that cleans itself up, so parallel runs can't collide.

### Step 4 — Block 4: connectors so the loop can act

Wire up the MCP connectors the loop needs to touch your real tools — e.g. a GitHub connector to open the PR and an issue-tracker connector to log it. With these, the loop *opens the PR and updates the ticket itself* instead of just telling you what it would do. Without them, the loop stops at "here's a green branch" and you do the rest by hand.

### Step 5 — Compose the closed loop with `/goal`

In a Claude Code session in the repo, run a closed loop whose stop condition is *verifiable*:

```
/goal Using the dep-update skill: pick one outdated dependency not already
in loop-state.md, dispatch dep-bumper to bump it in a worktree, then
dispatch dep-verifier to check it. Stop when dep-verifier returns PASS and
a branch deps/<pkg>-<ver> exists with passing tests; or when no eligible
package remains. After each attempt, append the result to loop-state.md.
If PASS, open a PR via the GitHub connector. If FAIL, record the reason in
loop-state.md "Skipped" and move on.
```

`/goal` keeps going across turns and, crucially, has a **separate model check whether the condition actually holds** after each turn — so "done" is graded by something other than the agent that did the work. (`/loop` would just re-run on a cadence without that check — that's the open-loop version.)

### Step 6 — Block 1: make it a heartbeat (schedule it)

Turn the one-off into a recurring loop. Either schedule it inside Claude Code (a scheduled/cron task running the prompt above weekly), or push it to **GitHub Actions** so it keeps running after you close the laptop. Findings and any PRs come to *you*; you're no longer the one kicking it off.

A minimal GitHub Actions cadence (sketch — fill in your runner/auth):

```yaml
# .github/workflows/dep-loop.yml
on:
  schedule:
    - cron: "0 9 * * 1"   # every Monday 09:00 UTC
jobs:
  dep-update-loop:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run dependency-update loop
        run: claude --headless --goal-file .claude/loops/dep-update.txt
```

### Step 7 — Close the human side of the loop

The loop now: wakes weekly → picks one outdated dep → bumps it in isolation → verifies independently → opens a PR only if green → records everything in state. **Your remaining job is the PR review.** That's deliberate — the loop surfaces a verified candidate; you stay the engineer who ships it.

---

## 6. How the 6 blocks map onto this example

| Block | Where it shows up above |
|-------|--------------------------|
| Automations | Step 6 — weekly schedule / GitHub Action (the heartbeat) |
| Worktrees | Step 3 — `isolation: worktree` on the bumper |
| Skills | Step 2 — `dep-update/SKILL.md` encodes your rules |
| Plugins / connectors | Step 4 — GitHub + issue-tracker MCP to open the PR |
| Sub-agents | Step 3 — `dep-bumper` (maker) vs `dep-verifier` (checker) |
| State / memory | Step 1 — `loop-state.md` carries context week to week |

---

## 7. What the loop still doesn't do for you

Three problems get *sharper* as the loop gets better, not easier:

1. **Verification is still on you.** "Done" is a claim, not a proof. The maker/checker split makes the claim stronger, but your job is still to ship code you confirmed works.
2. **Your understanding rots if you let it.** The faster the loop ships code you didn't write, the bigger the gap between what exists and what you actually understand ("comprehension debt"). Read what the loop made.
3. **The comfortable posture is the dangerous one.** It's tempting to stop having an opinion and take whatever it returns ("cognitive surrender"). Two people build the identical loop and get opposite results — one moves faster on work they understand deeply, the other avoids understanding the work at all. The loop doesn't know the difference. You do.

> **Build the loop. But build it like someone who intends to stay the engineer — not just the person who presses go.**

---

## Quick-start checklist

- [ ] Write `loop-state.md` (memory) first.
- [ ] Write a tight `SKILL.md` capturing your conventions and a precise *definition of done*.
- [ ] Split maker and checker into two sub-agents; give the maker `isolation: worktree`.
- [ ] Connect the real tools (MCP) the loop must act on.
- [ ] Wrap it in `/goal` with a **verifiable** stop condition (closed loop).
- [ ] Schedule it (the heartbeat); route results back to you.
- [ ] Keep reviewing the output. The verifier is the bottleneck — and so are you.

---

*Source: Addy Osmani, "Loop Engineering," https://addyosmani.com/blog/loop-engineering/ — the X thread you linked is a summary built on this article.*
