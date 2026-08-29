# Code Quality Check

A Claude Code plugin that adds the **`/compress`** skill — it shrinks a bloated code diff
down to the minimal, reviewable change. A PR should read as one idea; `/compress` audits
your branch's diff for change scope, test bloat, and over-long comments, then trims it to
the smallest change that still fully accomplishes the intent.

This repository doubles as a **plugin marketplace**, so anyone can install it directly.

## Install

In an interactive Claude Code session, run:

```
/plugin marketplace add evolvingkid/code-quality-agent
/plugin install code-quality-check@code-quality-agent
```

## Usage

Run it after a multi-file change — or right before committing / opening a PR:

```
/compress
```

What it does:

1. **Establishes the true base** — fetches `origin` and diffs against the remote base
   (never a stale local branch), then sanity-checks the diff before trusting it.
2. **Runs three audits in parallel** — change scope (incidental file edits), test bloat
   (redundant/unrelated tests), and comment length (the 3-line law).
3. **Presents a compression plan** with a stable ID per finding and applies **nothing**
   until you approve — everything, a subset, or nothing.
4. **Applies approved fixes** behind a safety snapshot, then runs the test suite/build as
   a hard gate and reports before/after plus how to roll back.

The intended change is treated as sacred; only incidental churn is a candidate for
removal.

## Local development

To test changes before pushing to GitHub, add the marketplace from your local path:

```
/plugin marketplace add /Users/fayez/Documents/claude plugin/code-quality-check
/plugin install code-quality-check@code-quality-agent
```

Re-run `/plugin marketplace update code-quality-agent` after editing files to pick up
changes.

## Repository layout

```
.
├── .claude-plugin/
│   ├── marketplace.json    # marketplace manifest (lists installable plugins)
│   └── plugin.json         # plugin manifest (metadata)
├── skills/
│   └── compress/
│       └── SKILL.md        # the /compress skill
├── README.md
└── LICENSE
```

## Extending

Add more skills by dropping additional `<name>/SKILL.md` directories in `skills/` — they're
auto-discovered. To host more than one plugin from this marketplace, put each plugin in
its own subfolder and add an entry to the `plugins` array in
`.claude-plugin/marketplace.json` with `"source": "./that-folder"`.

## License

MIT — see [LICENSE](LICENSE).
