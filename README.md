# agent-skills

[![Claude Code Plugin](https://img.shields.io/badge/Claude_Code-Plugin-blueviolet?logo=anthropic)](https://code.claude.com/docs/en/skills)
[![Version](https://img.shields.io/badge/version-1.3.0-green)](CHANGELOG.md)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![GitHub stars](https://img.shields.io/github/stars/pjordan/agent-skills)](https://github.com/pjordan/agent-skills/stargazers)
[![GitHub issues](https://img.shields.io/github/issues/pjordan/agent-skills)](https://github.com/pjordan/agent-skills/issues)

A collection of skills for coding agents during long-running autonomous development sessions. Installable as a [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin.

## Installation

In Claude Code, add the marketplace and install the plugin:

```
/plugin marketplace add pjordan/agent-skills
/plugin install agent-skills@pjordan-agent-skills
```

Then reload to activate:

```
/reload-plugins
```

## Usage

Once installed, skills are available automatically — Claude invokes them when relevant. You can also trigger a skill explicitly:

```
/agent-skills:agent-wiki init the wiki for this project
```

Or just ask naturally: "initialize the wiki", "ingest this finding", "lint the wiki".

## Skills

| Skill | Description |
|-------|-------------|
| [agent-wiki](plugins/agent-skills/skills/agent-wiki/) | Persistent, compounding knowledge base. Maintains a per-project wiki of cross-referenced markdown pages across sessions, stored under the user's Claude data directory. |
| [observe](plugins/agent-skills/skills/observe/) | Context-builder. Reads git, `gh`/`az` forge metadata, and project docs to file structured contributor/workflow/review-policy pages into agent-wiki. |
| [contribute](plugins/agent-skills/skills/contribute/) | Semi-autonomous contribution workflow. Picks work, plans, drafts a local branch + PR description, iterates on reviews. Hard safety rails; user opens the PR. |
| [reflect](plugins/agent-skills/skills/reflect/) | After-action learning loop. Reads plans, commits, reviews, and CI; writes retro and playbook pages into agent-wiki so priors compound across sessions. |

## Repo Structure

```
agent-skills/                            # Marketplace root
├── .claude-plugin/
│   └── marketplace.json                 # Marketplace definition
├── plugins/
│   └── agent-skills/                    # Plugin
│       ├── .claude-plugin/
│       │   └── plugin.json              # Plugin manifest
│       └── skills/
│           ├── agent-wiki/              # Persistent knowledge base
│           ├── observe/                 # Repo + team + workflow context-builder
│           ├── contribute/              # Semi-autonomous contribution workflow
│           └── reflect/                 # After-action learning loop
├── CONTRIBUTING.md
├── CHANGELOG.md
└── LICENSE
```

The repo is a marketplace containing one plugin. Each skill lives under `plugins/agent-skills/skills/` with a `SKILL.md` (agent instructions) and `README.md` (human overview).

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on adding new skills or improving existing ones.

## License

[MIT](LICENSE)
