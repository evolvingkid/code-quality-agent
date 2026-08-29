# Code Quality Check

A Claude Code plugin — and installable marketplace — providing focused code-quality
skills. Each skill keeps its own docs in its folder, so this file stays a short index.

## Install

In an interactive Claude Code session, run:

```
/plugin marketplace add evolvingkid/code-quality-agent
/plugin install code-quality-check@code-quality-agent
```

## Skills

| Skill | Command | What it does | Docs |
|-------|---------|--------------|------|
| compress | `/compress` | Shrinks a bloated code diff down to the minimal, reviewable change via three parallel audits and an approved plan. | [skills/compress/README.md](skills/compress/README.md) |

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
│       ├── SKILL.md        # the /compress prompt Claude loads
│       └── README.md       # human-facing docs for this skill
├── README.md               # this index
└── LICENSE
```

## Extending

Each skill lives in its own `skills/<name>/` folder with two files: `SKILL.md` (the prompt
Claude loads) and `README.md` (human docs). Add a new skill by creating that folder and
adding one row to the **Skills** table above — the root README stays an index rather than
growing without bound. To host more than one plugin from this marketplace, put each plugin
in its own subfolder and add an entry to the `plugins` array in
`.claude-plugin/marketplace.json` with `"source": "./that-folder"`.

## License

MIT — see [LICENSE](LICENSE).
