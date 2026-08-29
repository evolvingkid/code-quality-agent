# /compress

Shrinks a bloated code diff down to the minimal, reviewable change. A PR should read as
one idea; `/compress` audits your branch's diff and trims it to the smallest change that
still fully accomplishes the intent.

## Usage

Run it after a multi-file change — or right before committing / opening a PR:

```
/compress
```

## What it does

1. **Establishes the true base** — fetches `origin` and diffs against the remote base
   (never a stale local branch), then sanity-checks the diff before trusting it.
2. **Builds a change brief** — intent, what's in scope, and what's explicitly out of
   scope — and shares it with every audit agent.
3. **Runs three audits in parallel** — change scope (incidental file edits), test bloat
   (redundant/unrelated tests), and comment length (the 3-line law).
4. **Presents a compression plan** with a stable ID per finding and applies **nothing**
   until you approve — everything, a subset, or nothing.
5. **Applies approved fixes** behind a safety snapshot, then runs the test suite/build as
   a hard gate and reports before/after plus how to roll back.

The intended change is treated as sacred; only incidental churn is a candidate for
removal.

## Reference

The full operating procedure lives in [`SKILL.md`](SKILL.md) — that file is the prompt
Claude loads when the skill runs. This README is the human-facing summary.
