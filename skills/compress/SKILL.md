---
name: compress
description: Shrink a bloated code diff down to the minimal, reviewable change. Use this skill whenever the user runs /compress, asks to "clean up", "slim down", "shrink", or "minimize" a diff or PR, complains that Claude changed too many files, wrote too many tests, or left long comments — and also proactively after finishing any multi-file code change, right before committing or opening a PR. Runs three parallel audits (change scope, test bloat, comment length) on the branch's diff against the fetched remote base, then presents a compression plan for approval before touching anything.
---

# Compress

A PR should read as one idea. Every extra file touched, every redundant test, every
five-line comment dilutes that idea and costs reviewer attention. This skill audits the
current branch's diff with three parallel agents, presents a compression plan, and — only
after the user approves — trims the diff down to the smallest change that still fully
accomplishes the intent.

Two hard rules frame everything below:

- **The intended change is sacred.** Everything else is a candidate for removal.
- **Nothing is modified without approval.** The audits produce a plan; the user decides
  what gets applied. This matters because the skill's actions — reverting files, deleting
  tests — are exactly the kind that destroy work when a premise (like the base commit)
  is wrong.

## Step 0 — Establish the TRUE base (get this wrong and everything below is dangerous)

A stale local `main` is the single worst failure mode of this skill: if local `main` is
behind the remote, the merge-base lands far in the past, the "diff" balloons to include
every PR merged since, and the scope auditor will happily propose reverting *other
people's merged work*. So:

1. **Fetch first.** Run `git fetch origin`. If the fetch fails (offline, no remote),
   say so explicitly and treat every conclusion below as suspect — flag it in the plan.
2. **Base off the remote ref, never the local branch.** Prefer, in order: a base the
   user named; the repo's default from `git symbolic-ref refs/remotes/origin/HEAD`
   (e.g. `origin/main`); `origin/main`; `origin/master`.
   ```bash
   BASE=$(git merge-base origin/main HEAD)
   git diff --stat $BASE     # "before" diffstat
   ```
3. **Sanity-check the diff before believing it.** Look at
   `git log --oneline $BASE..HEAD` and the diffstat, and compare against what this
   branch is supposed to be (from the conversation, branch name, commit messages). Red
   flags: commits by other authors, merge commits pulling in mainline, or a file count
   far beyond what the intent could plausibly touch (a "add one field" branch showing
   300 changed files). If anything looks inflated, STOP — show the user the diffstat and
   commit list, explain the base looks wrong, and do not audit, let alone edit.
4. **Write a short change brief** — the shared context every agent receives. Draw it
   from the conversation, the branch name, the commit messages
   (`git log $BASE..HEAD`), and any linked PR/issue description, and capture:
   - **Intent** — one or two sentences on what this branch is *for*.
   - **In scope** — the files/areas the intent is expected to touch.
   - **Out of scope** — anything the user said to leave alone, plus follow-ups
     deliberately deferred to a later PR.

   This brief is how the agents tell essential changes from incidental ones — the
   richer it is, the fewer false reverts. If any part is genuinely unknown, say so in
   the brief rather than guessing (an unknown intent makes every revert suspect).

If the diff is empty, say so and stop.

## Step 0.5 — Detect PR state

Check whether the branch is published: does `git rev-parse --abbrev-ref @{u}` resolve,
or does `origin/<branch>` exist? If yes — especially if a PR is open or under review —
say so in the plan and include this warning: applying rewrites will require a re-push,
which re-triggers CI and invalidates any in-flight review comments. That cost is the
user's to weigh, not the skill's. Never amend or force-push published commits; leave all
committing and pushing to the user.

## Step 1 — Three audits in parallel (analysis only)

Spawn all three agents **in a single message** so they run concurrently. They analyze
only — no edits. Give every agent: the **change brief** from Step 0 (intent + in-scope +
out-of-scope), the validated `$BASE`, the commit list (`git log --oneline $BASE..HEAD`),
and instructions to read the diff themselves (`git diff $BASE`) plus surrounding code for
context.

Every finding must record the **exact current text of the lines it targets** (file,
line numbers, verbatim content). This is the staleness mechanism used in Step 3: a fix
is only applied if its recorded lines still match the file at apply time.

### Agent 1 — Scope auditor (unnecessary file changes)

Ground every judgment in the change brief. Use it as the yardstick:

- **Essential** — the change traces to the stated intent, or is a direct, unavoidable
  consequence of it (a call-site updated for a changed signature, an import the new code
  needs, a type the new code forces).
- **Incidental** — it has no line back to the intent. If a file is not on the brief's
  **in-scope** list, treat its hunks as incidental until proven otherwise; if it matches
  the **out-of-scope** list, it's incidental with high confidence.
- **Unknown intent** — if the brief couldn't pin down the intent, do not propose reverts
  on that basis; only flag changes that are incidental under *any* reasonable intent
  (pure formatting, unrelated files) and say the rest is unassessable.

Classify every changed file and hunk on that basis. Incidental changes to hunt for:

- Drive-by refactors and renames in code the change didn't need to touch
- Formatting/whitespace/import-reordering churn with no behavior change
- Opportunistic cleanups ("while I was here...")
- New helper files or abstractions the change doesn't actually require
- README/changelog/config edits nobody asked for

Return `{file, hunk-or-whole-file, verdict, reason, action, target-lines}` where action
is `revert-file` or `revert-hunk`. When genuinely unsure whether something is essential,
keep it — a false revert breaks the build; a kept extra line just costs review time.

### Agent 2 — Test auditor (test bloat)

The tests **under audit** are only those this branch added or modified. But to judge
them, also **scan the pre-existing test files around them** (same test file, sibling
files for the same module) — a new test can only be recognized as redundant, or as a
consolidation candidate, by comparing it against what already exists on `$BASE`. Flag:

- New tests that don't exercise the intended change (unrelated code, re-testing
  framework/library behavior, trivial getters/setters)
- New tests for **obviously-correct trivial code** — logic so simple it can be verified
  by reading it: a one-line arithmetic change (`total + fee`), a plain pass-through or
  delegation, a constant/field rename. If the code has no branches, no error paths, and
  no edge cases, a dedicated test adds maintenance cost without catching anything —
  flag it for deletion. This only applies when the logic is genuinely trivial: any
  conditional, loop, boundary, or failure mode means the test stays
- New tests whose coverage an **existing** test already provides — same code path,
  same assertion shape, different names
- Clusters of near-identical **new** tests that should collapse into one parameterized /
  table-driven test
- A new test that repeats an existing test's setup/logic and adds one extra check —
  e.g. an existing test verifies value A, the new test runs the same logic and verifies
  B: consolidate by extending the existing test to assert A **and** B, and drop the new
  one
- Excessive edge-case padding far beyond what the change warrants

Hard limits on touching pre-existing tests: they may only be **extended** (absorbing a
new test's assertions into their existing setup) — never deleted, and never have any
existing assertion weakened or removed. If a consolidation would require restructuring
an existing test beyond adding assertions, don't propose it; keep the tests separate.

Return `{file, test-name(s), verdict, action, target-lines}` where action is `delete`,
`merge-into-one` (collapse new tests, with the merged test sketched out), or
`merge-into-existing` (name the existing `$BASE` test, sketch the extended version, and
record target-lines for **both** the new test to remove and the existing test to
extend). The goal is the smallest test set that still meaningfully covers the intended
change — coverage of the actual change must never drop, only redundancy.

### Agent 3 — Comment auditor (the 3-line law)

Scan every comment and doc comment in the diff and enforce:

- **No comment or doc comment may exceed 3 lines.** This is a strict law. Longer
  comments get rewritten — compress with bullet points or arrows (`->`) to preserve the
  full meaning in fewer lines. Never drop information a future reader needs; compress
  the phrasing, not the content.
- **No inline comments** (trailing a line of code). Move the content above the function,
  or into the file-top comment, whichever fits.
- Delete comments that merely restate the code (`// increment i`).

Return `{file, line, original-comment, rewritten-comment, target-lines}` — the rewrite
ready to paste, already ≤3 lines and positioned above the function or file.

## Step 2 — Present the compression plan and get approval

Assemble every finding into one plan, grouped by auditor, each item with its one-line
reason and a stable ID:

```
## Compression plan (nothing applied yet)
Base: origin/main @ abc1234 (fetched 2 min ago) | Branch: pushed, PR #142 under review ⚠ re-push will re-trigger CI

Scope   S1. revert src/utils/format.ts   — formatting-only churn
        S2. revert README.md             — unrelated edit
Tests   T1. merge 4 near-identical priority tests -> 1 parameterized
        T2. delete 'listTasks returns priorities' — duplicates existing coverage
Comments C1. tasks.js:12 compress 7-line block -> 2 lines
         C2. cli.py:8 move inline comment above function
```

Ask the user what to apply: everything, a subset (by ID), or nothing. **Do not edit
anything before this approval — including comment rewrites.** If no one is around to
answer (unattended run), save the plan to a file, report where it is, and stop; a plan
that waits is worth more than an unapproved edit.

## Step 3 — Apply approved fixes (snapshot first, sequentially, staleness-checked)

1. **Safety snapshot before the first edit**, so every state is recoverable:
   ```bash
   git branch compress-backup-$(git branch --show-current | tr '/' '-')   # committed state
   git stash store -m "pre-compress snapshot" "$(git stash create)"       # uncommitted state, keeps tree intact
   ```
   (Skip the stash line if `git stash create` outputs nothing — clean tree.)
2. Apply approved items in order: **scope reverts -> test fixes -> comment rewrites**.
   Scope first means later steps don't waste work on code that's about to disappear.
3. **Staleness check on every item**: before applying, re-read the file and compare
   against the finding's recorded target lines. If they no longer match (an earlier
   revert removed or shifted them), skip the item and note it in the report — never
   apply a fix to lines it wasn't written for.

## Step 4 — Verify (hard gate) and report

1. Run the **full test suite** and the build/typecheck if one exists. This is a gate,
   not a suggestion: the branch must not be left broken.
   - A failure means the compression went too far — restore the specific change that
     broke it and re-run.
   - If it can't be fixed that way, restore the snapshot entirely (backup branch +
     stash) and report which items caused the breakage.
   - If no test suite exists or it can't run, say so plainly and mark the result
     "unverified" — never imply verification that didn't happen.
2. Confirm the intent statement still holds: the feature/fix the branch exists for is
   intact.
3. Report before/after, every applied item, every skipped-stale item, and how to roll
   back (the backup branch and stash names). End by offering to restore anything the
   user wants back.
