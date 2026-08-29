# Code Quality Check

A Claude Code plugin that adds a `/code-quality-check` slash command to review your
current code changes for quality issues — naming, complexity, duplication, dead code,
error handling, and test coverage.

This repository doubles as a **plugin marketplace**, so anyone can install it directly.

## Install

In an interactive Claude Code session, run:

```
/plugin marketplace add evolvingkid/code-quality-agent
/plugin install code-quality-check@code-quality-agent
```

## Usage

Review the current change set (git diff):

```
/code-quality-check
```

Focus on a specific file or directory:

```
/code-quality-check src/auth
```

The command produces findings grouped by severity (**Blocking**, **Recommended**,
**Nitpick**), each with a `file:line` reference and a concrete fix, plus a short
overall assessment. It reviews only — it won't edit files unless you ask it to.

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
├── commands/
│   └── code-quality-check.md   # the /code-quality-check slash command
├── README.md
└── LICENSE
```

## Extending

Add more commands by dropping additional `.md` files in `commands/` — they're
auto-discovered. To host more than one plugin from this marketplace, put each plugin in
its own subfolder and add an entry to the `plugins` array in
`.claude-plugin/marketplace.json` with `"source": "./that-folder"`.

## License

MIT — see [LICENSE](LICENSE).
