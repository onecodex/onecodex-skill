# onecodex-skill

A plugin for AI coding agents (Claude Code, GitHub Copilot CLI, VS Code Copilot Chat, OpenAI Codex) providing the `onecodex` skill: a reference for the [One Codex Python client](https://github.com/onecodex/onecodex) (samples, analyses, querying, plotting) plus authoring and debugging custom per-sample Workflows on the One Codex platform — `shell_script` and `nextflow` job types, runtime contract (env vars, filesystem layout, `deinterleave` for paired-end FASTQs), dependencies, assets, and gotchas.

The skill follows the [Agent Skills](https://code.visualstudio.com/docs/agent-customization/agent-skills) open standard, so the same `SKILL.md` works across tools.

## Examples

Some of the tasks you can perform using this skill with an AI coding agent:

- "What's the abundance of *E. coli* across the samples in my `gut-microbiome` project?"
- "I want to run Kraken2 with a custom database on every sample I upload."
- "My Nextflow Workflow runs locally but the platform job exits 1 — what am I missing?"
- "Pull all samples uploaded in the last 30 days and dump filename, size, and read count to a CSV."
- "Tag every sample in this project whose top hit is *Salmonella enterica* with `flagged-salmonella`."

## Install

### Claude Code

```text
/plugin marketplace add onecodex-collaborations/onecodex-skill
/plugin install onecodex@onecodex
```

Update with `/plugin marketplace update onecodex`.

### GitHub Copilot (CLI + VS Code)

Requires the GitHub Copilot CLI (`npm install -g @github/copilot`).

```bash
copilot plugin install onecodex-collaborations/onecodex-skill:plugins/onecodex
```

Installed globally for your user — available in Copilot CLI and VS Code Copilot Chat across all workspaces. Reload the Copilot Chat view in VS Code after install.

### OpenAI Codex CLI

Requires OpenAI Codex CLI ≥ 0.142 (the `plugin` subcommand isn't in older releases — upgrade with `npm install -g @openai/codex@latest`).

```bash
codex plugin marketplace add onecodex-collaborations/onecodex-skill
codex plugin add onecodex@onecodex
```

Restart `codex` and check `/skills`. (Or do it interactively: `codex` → `/plugins` → install.) The OpenAI Codex desktop app picks it up after restart.

### Other tools

The skill at `plugins/onecodex/skills/onecodex/SKILL.md` is plain markdown with frontmatter. Drop it into any tool that reads `~/.claude/skills/`, `~/.copilot/skills/`, or `~/.agents/skills/`.

## Local development

Claude Code:

```text
/plugin marketplace add /path/to/onecodex-skill
/plugin install onecodex@onecodex
```

Copilot CLI:

```bash
copilot plugin install ./plugins/onecodex
```

Update Claude with `/plugin marketplace update onecodex`; reinstall the Copilot plugin after edits.
