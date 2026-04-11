---
name: agent-wiki
description: >
  Persistent, compounding knowledge base for coding agents. Maintains a .wiki/ directory of
  cross-referenced markdown pages — architecture, bugs, decisions, gotchas, patterns, dependencies.
  Use when: user says "wiki"/"init wiki"/"ingest"/"lint wiki", starting a long dev session,
  or the agent discovers something worth persisting (bug root causes, architecture insights,
  failed approaches). Prevents rediscovering the same things across sessions.
allowed-tools: Read Write Edit Glob Grep
---

# Agent Wiki

You maintain a persistent knowledge base as plain markdown in `.wiki/` at the project root. The
core insight: the tedious part of a knowledge base is the bookkeeping — cross-references, index
updates, consistency checks. You're good at that. So you own the wiki layer entirely.

The wiki compounds across sessions. Instead of rediscovering that "the auth middleware silently
swallows 403s" for the third time, it's already documented with context.

## Three Layers

1. **Raw sources** (read-only): The codebase, error messages, stack traces, docs. You read these
   but never modify them as wiki content.
2. **The wiki** (you own this): `.wiki/pages/` — markdown files you create, update, cross-reference.
3. **The schema**: `.wiki/SCHEMA.md` — conventions governing how you maintain the wiki.

## Operations

Four operations. The user may invoke them explicitly or you may perform them naturally during work.

For concrete input/output examples of each operation, see [references/examples.md](references/examples.md).

---

### init — Bootstrap the wiki

Run when `.wiki/` doesn't exist. Sets up the structure and scans the codebase for foundational pages.

1. Create `.wiki/` with `SCHEMA.md`, `index.md`, `log.md`, and `pages/` directory.
   Use the templates in [references/schema-template.md](references/schema-template.md).

2. Scan the codebase — read the README, manifest files (package.json, Cargo.toml, pyproject.toml,
   go.mod), and top-level directory structure.

3. Create 3-5 foundational pages covering: project overview, key architecture, main dependencies.

4. Update `index.md` and `log.md`.

The goal is a useful starting point, not exhaustive documentation. Capture the big picture. Depth
comes later through incremental ingests.

---

### ingest — File a discovery into the wiki

The most common operation. When you learn something significant — a bug root cause, an architecture
insight, a dependency quirk, a failed approach — file it.

A single discovery often touches multiple pages. Finding a bug might reveal an architecture pattern
and a dependency gotcha.

1. Read `index.md` to see what exists. Update existing pages or create new ones.

2. Write pages in `.wiki/pages/` with YAML frontmatter:
   ```yaml
   ---
   title: Auth Middleware Architecture
   type: architecture
   created: 2026-04-11
   last_updated: 2026-04-11
   tags: [auth, middleware, express]
   related: [jwt-config, session-handling]
   ---
   ```
   Keep pages focused — one topic per page. Split and cross-reference when content spans concerns.

3. Add `[[page-name]]` wikilinks. When linking A to B, check if B should link back to A.

4. Update `index.md` with new/changed entries under the appropriate category.

5. Append to `log.md` with date, operation, summary, and pages touched.

**When to ingest without being asked:**
- Found a non-trivial bug's root cause
- Discovered a non-obvious architectural boundary or pattern
- An approach failed and the reason is worth remembering
- A dependency behaves unexpectedly
- Figured out something about build/deploy/test that took effort

The bar: "would another agent waste time rediscovering this?" If yes, ingest it.

---

### query — Consult the wiki

When you need codebase context — before starting work, investigating a bug, making a design
decision — check the wiki first.

1. Read `.wiki/index.md` to find relevant pages.
2. Read the relevant pages.
3. Synthesize with what you already know.
4. If the query produces new insight, ingest it back. This is how the wiki compounds.

Querying is lightweight. Don't make it a ceremony.

---

### lint — Health check the wiki

Run periodically (start of session, or when asked) to keep the wiki healthy.

**Check for:**
- **Contradictions**: Page A says X, page B says not-X
- **Stale pages**: `last_updated` is old and content may not reflect current code
- **Orphan pages**: No inbound `[[wikilinks]]` from other pages
- **Missing pages**: `[[wikilinks]]` pointing to nonexistent pages
- **Gaps**: Important concepts or components that should have pages but don't
- **Index drift**: Pages in `pages/` not in `index.md`, or vice versa

Fix what you can, flag what you can't as open questions on the relevant pages.

---

## Principles

**Compound, don't duplicate.** If a page exists, update it. If two pages overlap, merge them.

**Focused pages over comprehensive ones.** One topic per page. Split and cross-reference.

**Write for future agents.** Be precise — include file paths, function names, error messages. Skip prose.

**Low ceremony.** A rough page capturing a key insight beats no page. Lint and improve later.

**Cross-reference aggressively.** The wiki's value grows with connections. Update related pages too.

---

## File Layout

```
.wiki/
├── SCHEMA.md          # Conventions and page type definitions
├── index.md           # Categorized catalog of all pages
├── log.md             # Append-only chronological activity log
└── pages/
    ├── project-overview.md
    ├── auth-middleware.md
    └── ...
```
