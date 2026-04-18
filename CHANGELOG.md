# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.2.0] - 2026-04-18

### Added
- **observe** skill: context-builder for a repo, its team, and its workflow. Reads git,
  GitHub (`gh`) / Azure DevOps (`az`) forge metadata, `CLAUDE.md`, `CONTRIBUTING.md`,
  `CODEOWNERS`, issue/PR templates, and ADRs; writes structured pages into agent-wiki using
  four new page types: `contributor`, `workflow`, `review-policy`, `team-dynamics`.
  Self-registers the new types in `<wiki-root>/SCHEMA.md` on first survey. Operations:
  `survey` / `refresh` / `brief` / `investigate`. Refresh auto-triggers at >7 days or
  >50 commits since last survey. Graceful degrade when forge CLIs are missing or
  unauthenticated.
- **contribute** skill: semi-autonomous contribution workflow. Reads observe's wiki pages
  for repo context, then takes a task through `pick` (user-directed selection) → `plan`
  (written approach, user confirms) → `draft` (local branch + commits + PR description in
  `.git/PR_EDITMSG`; user opens the PR) → `iterate` (review comments + CI failures).
  Uses the user's own git/`gh`/`az` identity — no bot account, no `Co-Authored-By`.
  Safety rails: always-protected branches include `main`, `master`, `trunk`, `develop`,
  `release/*`, `hotfix/*`; no autonomous PR creation; 300 lines / 8 files / 1 PR per run
  with confirm gate; max 3 iterate rounds per PR; no unsolicited social actions (solicited
  replies on the user's own active PR are fine).

## [1.1.0] - 2026-04-18

### Changed
- **agent-wiki**: Moved wiki storage from `.wiki/` in the project repo to
  `${CLAUDE_PLUGIN_DATA}/wikis/<project-key>/` under the user's Claude data directory.
  `<project-key>` is the resolved absolute project path (git toplevel when available) with
  `/` replaced by `-`. Keeps the project tree clean and removes the need to gitignore `.wiki/`.
  The skill's user-facing operations (`init`/`ingest`/`query`/`lint`) are unchanged, but on-disk
  layout is different — see the migration snippet below for preserving prior content.
- **agent-wiki**: Added `Bash` to `allowed-tools` so the skill can resolve `<project-key>`
  via `git rev-parse --show-toplevel` / `pwd -P`.
- **agent-wiki**: Reframed the README to make the per-user, per-machine scope explicit —
  the wiki does not travel with `git clone` or sync across teammates.

### Migration
Existing `.wiki/` directories are not auto-migrated. To preserve prior content, run this from
the project root in a Claude Code session so `$CLAUDE_PLUGIN_DATA` is populated:

```bash
KEY=$({ git rev-parse --show-toplevel 2>/dev/null || pwd -P; } | sed 's|/|-|g')
DEST="${CLAUDE_PLUGIN_DATA:?run this inside a Claude Code session}/wikis/$KEY"
mkdir -p "$DEST" && cp -R .wiki/. "$DEST/"
```

## [1.0.0] - 2026-04-11

### Changed
- **agent-wiki**: Refactored SKILL.md following Anthropic's skill best practices
  - Tightened description for better triggering accuracy
  - Added `allowed-tools` frontmatter (Read, Write, Edit, Glob, Grep)
  - Extracted schema template and init boilerplate to `references/schema-template.md`
  - Added concrete input/output examples in `references/examples.md`
  - Leaner SKILL.md body with progressive disclosure to reference files
- Bumped plugin version to 1.0.0

## [0.1.0] - 2026-04-11

### Added
- Initial release as a Claude Code plugin
- **agent-wiki** skill: persistent knowledge base for coding agents
  - `init` operation to bootstrap wiki from codebase scan
  - `ingest` operation to file discoveries with cross-references
  - `query` operation to consult accumulated knowledge
  - `lint` operation for wiki health checks
  - Based on [Andrej Karpathy's llm-wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
- Plugin manifest and marketplace configuration
