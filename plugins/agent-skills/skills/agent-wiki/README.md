# agent-wiki

A persistent, compounding knowledge base that coding agents maintain during development sessions. Plain markdown files in a per-project wiki directory (stored under the user's Claude data directory) that grow richer over time — so agents stop rediscovering the same things.

## Why

Agents working on a codebase across multiple sessions waste time rediscovering the same things: that the auth middleware silently swallows certain errors, that CI tests are flaky because of a container timing issue, that a particular dependency has a quirky API. Each session starts from scratch.

agent-wiki fixes this by giving agents a structured place to file discoveries and consult prior knowledge. The core insight (from [Andrej Karpathy's llm-wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)) is that agents are good at exactly the bookkeeping that makes knowledge bases useful — cross-referencing, indexing, flagging contradictions — the tedious work humans abandon.

## Operations

| Operation | What it does |
|-----------|-------------|
| **init** | Bootstrap the wiki for a new repo — scans the codebase and creates foundational pages |
| **ingest** | File a discovery — bug root causes, architecture insights, dependency quirks, failed approaches |
| **query** | Consult the wiki for context before making decisions |
| **lint** | Health check — find contradictions, stale pages, orphans, and gaps |

## Where the wiki lives

Wikis live under `${CLAUDE_PLUGIN_DATA}/wikis/<project-key>/`, where `<project-key>` is the resolved project path with `/` replaced by `-`. `$CLAUDE_PLUGIN_DATA` is set by Claude Code and typically resolves under `~/.claude/plugins/data/...`. See [SKILL.md § Storage location](SKILL.md#storage-location) for the exact path-computation rules.

```
<wiki-root>/
├── SCHEMA.md                  # Page type definitions and conventions
├── index.md                   # Categorized catalog of all pages
├── log.md                     # Chronological activity log
└── pages/
    ├── project-overview.md    # Architecture and structure
    ├── auth-middleware.md      # How auth works, JWT config
    ├── flaky-ci-tests.md      # Bug: pool timeout too short for CI
    └── silent-403-gotcha.md   # Gotcha: auth errors become 500s
```

Each page has YAML frontmatter (title, type, tags, related pages) and `[[wikilinks]]` connecting related topics into a navigable knowledge graph.

**Scope**: the wiki is per-user and per-machine — it does not travel with `git clone`, does not sync across teammates, and is lost if the plugin is uninstalled. Think "personal cross-session notes for this checkout," not "team-shared documentation."

## How it works in practice

**Session 1** — Agent initializes the wiki, scans the codebase, creates foundational pages about the project architecture, auth system, and database layer.

**Session 2** — Agent debugs flaky tests, discovers the DB pool timeout is too short for CI. Files the discovery: creates a bug page, a gotcha page about CI container timing, updates the database page with a cross-reference. Three pages touched from one finding.

**Session 3** — New agent starts work. Queries the wiki before touching the auth code. Learns about the silent 403 issue immediately instead of hitting it 30 minutes into debugging.

## Design

- **Plain markdown** — no databases, no embeddings, no special tooling. Works with any LLM agent.
- **Per-user, off-repo** — stored under `${CLAUDE_PLUGIN_DATA}` so the repo stays clean; keyed by resolved project path.
- **Agent-first** — written for agents to read and maintain, though humans can browse it too.
- **Compounding** — every session builds on prior knowledge through cross-references and incremental ingests.

## Credits

Based on [Andrej Karpathy's llm-wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f), adapted from a personal knowledge base for humans to a development knowledge base for coding agents.
