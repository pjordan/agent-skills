# agent-skills

[![Claude Code Plugin](https://img.shields.io/badge/Claude_Code-Plugin-blueviolet?logo=anthropic)](https://code.claude.com/docs/en/skills)
[![Version](https://img.shields.io/badge/version-1.0.0-green)](CHANGELOG.md)
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
| [agent-wiki](skills/agent-wiki/) | Persistent, compounding knowledge base. Maintains a `.wiki/` directory of cross-referenced markdown pages across sessions. |

## Repo Structure

```
agent-skills/
├── .claude-plugin/
│   ├── plugin.json              # Plugin manifest
│   └── marketplace.json         # Marketplace index
├── skills/
│   └── agent-wiki/
│       ├── README.md            # Human-facing overview
│       ├── SKILL.md             # Agent-facing instructions
│       └── references/          # Supporting docs (loaded on-demand)
├── CONTRIBUTING.md
├── CHANGELOG.md
└── LICENSE
```

Each skill lives in its own directory under `skills/` with a `SKILL.md` (agent instructions) and `README.md` (human overview).

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on adding new skills or improving existing ones.

## License

[MIT](LICENSE)
