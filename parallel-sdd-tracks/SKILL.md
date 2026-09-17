---
name: parallel-sdd-tracks
description: Use when an implementation plan covers several independent features or areas and the user wants them built in parallel with subagent-driven development, or asks for per-track progress percentages while subagents run unattended
---

# Parallel SDD Tracks

## Overview
Extends superpowers:subagent-driven-development (SDD). The plan is split into
**tracks**. Each track gets its own git worktree, branch, plan file and ledger.
Tasks inside a track stay strictly sequential with implement → review. Tracks run
at the same time. The controller merges them back and posts a progress table
every time a task subagent finishes.

**REQUIRED SUB-SKILL:** superpowers:subagent-driven-development (per-track loop).
**REQUIRED SUB-SKILL:** superpowers:writing-plans (master plan + track plans).

Running tracks in parallel overrides SDD's "never dispatch implementers in
parallel" rule. That is safe only because each track has its own working tree.
Record this as a ruling in the master ledger. Everything this skill does not
override follows SDD: fix-round escalation and cap, model choice, review package,
ledger format.

## When to Use
- The spec has 2+ areas whose files barely overlap.
- The user asks for "tracks", "in parallel" or "progress %".

When NOT to use: the tasks share most of their files, or they have to land in a
fixed order (for example schema → repo → UI). Keep those in one track.

**Cross-track dependency** ("C needs B's task 3"): put C in B's track after B3.
If C is small and the dependency is a thin seam, move that seam into T0 instead.

## Paths used below
- `<R>`: the main checkout.
- `<F>`: the feature worktree, on the feature branch. The controller works here.
- `<Tn>`: the track worktrees, siblings of `<F>`, on branches `<feature>-tN`.

If no `<F>` exists yet, create it from `<R>` with `git worktree add` (or
superpowers:using-git-worktrees). Copy the untracked config into it from `<R>`
(`.env`, `local.env`, …). A worktree never carries untracked files.

## 1. Planning: master plan + track plans

The master plan (`docs/superpowers/plans/<date>-<topic>.md`) contains:
1. **Global constraints**: the project rules every task must follow (lint, test,
   format, commit trailer, files never to stage, baseline test count).
2. **Track map** table: Track | plan file | spec § | branch / worktree path.
3. **File ownership** per track: the paths each track may touch. List every
   **shared, merge-managed** file and a rule that keeps its merge trivial:
   - Structured shared file (JSON/YAML/ARB): T0 seeds one empty block or
     namespace per track, and each track writes only inside its own block. With
     no seed, commas and neighbouring lines cause conflicts.
   - Plain generated files (codegen output, l10n classes): never merged by
     hand. Take either side, then regenerate.
   - **Ordered** generated files (DB migrations, numbered files): each track
     generates only for the apps or modules it owns. If two tracks need the same
     one, move that change into T0. After a merge, if there are two leaves, the
     later-merged track deletes its unshipped migration and regenerates it.
   - Test catalog / changelog: each track edits only its own entries, under a
     heading T0 made.
   - Shared source/test file: one track reworks it, the others only append a
     self-contained block at the end.
4. **Shared runtime resources**: parallel test runs collide on the test DB
   name, ports and caches. Give each `<Tn>` its own values in its untracked
   config (for example `DB_NAME=app_t1`).
5. **T0**: seeds and seams that land on the feature branch first. It is
   required whenever a structured shared file needs seeded blocks. T0 runs
   sequentially with normal SDD, and its implementers work in `<F>` under the
   same pinning rules as the tracks. So `ExitWorktree` and
   `implementer-instructions.md` come before T0.
6. **Fan-out task** (controller only): section 2.
7. **Merge task** (controller only): section 5.
8. **Wrap-up task**: cleanups the tracks deferred, format, full lint and
   test run, docs.

One plan file per track, `<date>-<topic>-tN-<slug>.md`. Planners can be
parallel subagents. Then run one plan-review subagent per track plan before
execution, and patch the plans from its findings.

**Commits:** implementers commit after every task. If the project's
CLAUDE.md requires asking before commits, get one blanket approval for the
whole run before T0. It covers task, fix, merge and wrap-up commits. If the
user says no, do not use this skill.

Check that `<R>` git-ignores the worktree folder and `.superpowers/`. If not,
put a `.gitignore` containing `*` in `<F>/.superpowers/sdd/` as well.

## 2. Fan-out

```bash
cd <F> && git worktree add ../<topic>-t1 -b <topic>-t1 HEAD   # repeat per track
cp <R>/.env ../<topic>-t1/.env        # untracked config; adjust per-track values (DB name, port)
(cd ../<topic>-t1 && <install deps / venv> && <codegen> && <baseline test>)
mkdir -p ../<topic>-t1/.superpowers/sdd && echo '*' > ../<topic>-t1/.superpowers/sdd/.gitignore
```
The controller creates, in `<F>/.superpowers/sdd/`:
- one ledger per track, `<track-plan-name>/progress.md`, whose header holds
  `<Tn>`, the branch and the BASE sha (task briefs go in the same folder);
- the master ledger, `<master-plan-name>/progress.md`;
- `implementer-instructions.md`, holding the rules below.

On Windows, run these in the Bash tool (Git Bash). PowerShell needs different syntax.

### Worktree pinning (Claude Code)
Subagents inherit the controller's worktree pin, and `EnterWorktree(path)`
does **not** lift it for writes. If the controller is inside an
`EnterWorktree` session, every track implementer comes back BLOCKED.
- Controller: `ExitWorktree` with keep **before** the first dispatch. After
  that the session's working directory is `<R>`, so start every controller
  command with `cd <F> && ...`.
- Implementers: do NOT call EnterWorktree. Instead:
  - start every shell command with `cd <Tn> && ...`;
  - use absolute paths for Read/Edit/Write;
  - make `git branch --show-current` the first command. If it does not print
    the track branch, stop and report BLOCKED;
  - never touch `<R>`, `<F>` or another track's worktree, except to READ their
    brief and constraints from `<F>/.superpowers/sdd/`.

## 3. Dispatch shape (per task)

The implementer prompt contains, in this order:
1. Track + task title, and a one-line purpose.
2. **Where you work**: `<Tn>`, the branch, and the path to
   `implementer-instructions.md`.
3. **Requirements**: the brief file, the track constraints, and the project's
   CLAUDE.md.
4. **Context**: the baseline (lint clean, N tests), the files that must not
   change, the shared-file rules for this track, and a note that other tracks
   run elsewhere so it stays in its own files.
5. Job: TDD → focused tests → full lint + test → commit → self-review.
6. "You do not dispatch subagents."
7. **Report**: the full report goes to
   `<Tn>/.superpowers/sdd/<track-plan-name>/task-K-report.md`. The reply is
   ≤15 lines: Status (DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT),
   commits, test summary vs baseline, concerns, report path.

Subagent lifecycle:
- Every task gets a **fresh** implementer subagent and a **fresh** reviewer
  subagent. Never reuse one task's implementer for the next task.
- A fix round resumes that task's implementer (SendMessage), then a **fresh**
  reviewer does the scoped re-review (`FIX_BASE..HEAD`). SDD's escalation still
  applies: a fresh, more capable implementer in later rounds, and the round cap.
- A BLOCKED attempt that produced nothing: read its report and fix the cause,
  then dispatch a **fresh** implementer. If every track is BLOCKED, suspect the
  worktree pin. If only one is, suspect that track's brief, config or deps.

Reviews: run SDD's `review-package` script from `<F>`
(`cd <F> && bash <SDD skill dir>/scripts/review-package <track-plan> <BASE> <track-branch>`).
After each review, append the result to the track ledger, then dispatch the
next task with the new BASE.

Tracks skip SDD's per-plan final review and workspace cleanup. The wrap-up
does one whole-branch review instead, over `git merge-base <R-branch> HEAD..HEAD`
in `<F>`, given the master plan, the spec and the pooled deferred minors.

## 4. Progress table (after every task subagent finishes)

**Units** (a T0 row, if T0 exists, uses the same units):
- Each task = 2 (implement + review).
- When a review returns issues, add 2 to that track's total (fix +
  re-review) at that moment. The review that found the issues still counts as done.
- A BLOCKED attempt that produced nothing counts 0 and does not change the total.
- Post a table after every implementer, reviewer and wrap-up subagent, BLOCKED
  ones included. Planners and plan reviewers do not count and do not trigger one.
- Once T0 is done, its row shows ✅ in the next table and is then dropped.

`% = done / total`, rounded half up. "Now running" names the step in flight.

| Track | Done / total | % | Now running |
|---|---|---|---|
| T1 multi-plan | 4 / 22 | 18% | Task 2 fix round 1 |
| T2 annotations | 8 / 10 | 80% | Task 5 implement |
| T3 send dialog | 8 / 8 | 100% ✅ merged | — |
| Wrap-up | 2 / 5 | 40% | final review; then fix wave + re-review |

Post it in the same short message as the usual status line. Add a **Wrap-up**
row once all tracks are merged:
- **Starting units, 5:** wrap-up task + its review (2), final whole-branch
  review (1), final fix wave + its re-review (2).
- If the whole-branch review is clean, drop the fix wave: total 3.
- Each extra final fix round adds 2.

## 5. Merge-back

**Order:** merge a track as soon as its ledger says
`TRACK Tn COMPLETE: <base>..<head>`. The only exception is a merge order fixed
in the master plan, for example by migrations. If several tracks are complete
at the same time, merge the one with the most units last. On a tie, merge the
one with the larger diff last.

```bash
cd <F> && git merge-tree --write-tree --name-only HEAD <topic>-tN   # preview conflicts
cd <F> && git merge --no-ff --no-commit <topic>-tN
cd <F> && <regenerate generated files> && <migration check> && <lint> && <full test>
cd <F> && git add <regenerated files> && git commit -m "Merge track TN: <title>"   # only when green
```
If the checks are red, fix the problem inside the merge, or run
`git merge --abort` and send a fix task to the track.
Before removing the worktree, copy the deferred minors from
`<Tn>/.superpowers/sdd/` into the track ledger in `<F>`. `git worktree remove`
deletes those files. Then:
```bash
cd <F> && git worktree remove ../<topic>-tN && git branch -d <topic>-tN
```
When every track is merged, gather the deferred minors from all ledgers into
one list. That list feeds the whole-branch review and its fix wave. Finish with
superpowers:finishing-a-development-branch.

## Common Mistakes
| Mistake | Fix |
|---|---|
| Controller still inside EnterWorktree → all implementers BLOCKED | ExitWorktree (keep) before the first dispatch; then `cd <F> &&` |
| Two tracks edit the same file freely | File-ownership list + seeded blocks / append rules in the master plan |
| Hand-merging generated files | Take either side, then regenerate |
| Two tracks generating migrations for the same app | Move that change into T0, or fix the merge order |
| Parallel test runs sharing one DB or port | Per-track values in each worktree's untracked config |
| `cp .env` from `<F>` when EnterWorktree never copied it | Copy untracked config from `<R>` |
| Reviewer reused for the re-review | Every review and re-review gets a fresh reviewer |
| Progress % drops with no explanation | Say "fix round N" in "Now running"; the total grew |
| Counting BLOCKED attempts | They count 0 units |
| Losing minors on `git worktree remove` | Copy them into the ledger in `<F>` first |
