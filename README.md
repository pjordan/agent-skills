# agent-skills

[![Claude Code Plugin](https://img.shields.io/badge/Claude_Code-Plugin-blueviolet?logo=anthropic)](https://code.claude.com/docs/en/skills)
[![Version](https://img.shields.io/badge/version-1.0.0-green)](CHANGELOG.md)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![GitHub stars](https://img.shields.io/github/stars/pjordan/agent-skills)](https://github.com/pjordan/agent-skills/stargazers)
[![GitHub issues](https://img.shields.io/github/issues/pjordan/agent-skills)](https://github.com/pjordan/agent-skills/issues)
[![Inspired by llm-wiki](https://img.shields.io/badge/inspired_by-llm--wiki-orange)](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)

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

Once installed, skills are available automatically. Claude will invoke them when relevant based
on your prompt. You can also trigger a skill explicitly:

```
/agent-skills:agent-wiki init the wiki for this project
```

Or just ask naturally — "initialize the wiki", "ingest this finding", "lint the wiki" — and
Claude will use the skill when the context matches.

## Skills

### [agent-wiki](skills/agent-wiki/SKILL.md)

A persistent, compounding knowledge base that coding agents maintain as plain markdown files. Adapted from [Andrej Karpathy's llm-wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) for autonomous agent workflows.

The wiki lives in a `.wiki/` directory in your project and supports four operations:

| Operation | What it does |
|-----------|-------------|
| **init** | Bootstrap the wiki for a new repo — scans the codebase and creates foundational pages |
| **ingest** | File a discovery into the wiki — bug root causes, architecture insights, dependency quirks |
| **query** | Consult the wiki for context before making decisions |
| **lint** | Health check — find contradictions, stale pages, orphans, and gaps |

The core insight: agents are good at the bookkeeping that makes knowledge bases useful (cross-referencing, indexing, consistency checks). The wiki compounds across sessions so agents stop rediscovering the same things.

## Repo Structure

```
agent-skills/
├── .claude-plugin/
│   ├── plugin.json          # Plugin manifest
│   └── marketplace.json     # Marketplace index
├── skills/
│   └── agent-wiki/
│       └── SKILL.md         # Skill definition
├── CONTRIBUTING.md
├── CHANGELOG.md
└── LICENSE
```

Each skill lives in its own directory under `skills/` with a `SKILL.md` file containing the skill's instructions and metadata.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on adding new skills or improving existing ones.

## License

[MIT](LICENSE)
