---
name: agent-wiki
description: >
  A persistent, compounding knowledge base that coding agents maintain during development sessions.
  Adapted from Andrej Karpathy's llm-wiki pattern for autonomous agent workflows. The wiki lives
  as plain markdown in a `.wiki/` directory and compounds learnings across sessions — architecture
  insights, bug investigations, decision records, dependency gotchas, failed approaches, and patterns.
  Use this skill whenever the user mentions "wiki", "agent-wiki", "init wiki", "ingest", "query wiki",
  "lint wiki", or when starting a long development session. Also use it when the agent discovers
  something significant worth persisting — a tricky bug root cause, an architecture decision,
  a dependency quirk — even if the user doesn't explicitly ask. The wiki prevents agents from
  rediscovering the same things across sessions.
---

# Agent Wiki

A persistent development knowledge base that you maintain as plain markdown files. The core insight
(from Karpathy's llm-wiki pattern) is that the tedious part of maintaining a knowledge base isn't
the reading or thinking — it's the bookkeeping: updating cross-references, flagging contradictions,
keeping the index current. You're good at that. Humans aren't. So you own the wiki layer entirely.

The wiki lives in `.wiki/` at the project root. It compounds across sessions — each time you or
another agent works in this repo, the wiki gets richer. Instead of rediscovering that "the auth
middleware silently swallows 403s" for the third time, it's already documented with context.

## Three Layers

1. **Raw sources** (read-only): The codebase, error messages, stack traces, docs, READMEs. You read
   these but never modify them as wiki content.
2. **The wiki** (you own this): `.wiki/pages/` — markdown files you create, update, and cross-reference.
3. **The schema**: `.wiki/SCHEMA.md` — conventions that govern how you maintain the wiki.

## Operations

There are four operations. The user may invoke them explicitly ("init the wiki", "ingest this finding")
or you may perform them naturally as part of your work (e.g., ingesting a discovery mid-debugging).

---

### init — Bootstrap the wiki

Run this when `.wiki/` doesn't exist yet. It sets up the directory structure and does an initial
scan of the codebase to create foundational pages.

**Steps:**

1. Create the directory structure:
   ```
   .wiki/
   ├── SCHEMA.md
   ├── index.md
   ├── log.md
   └── pages/
   ```

2. Write `SCHEMA.md` with the conventions below (adapt to the project):

   ```markdown
   # Wiki Schema

   ## Page Types
   - **architecture**: How the system is structured — components, data flow, boundaries
   - **pattern**: Recurring code patterns, conventions, idioms used in this codebase
   - **decision**: Why something was done a particular way (ADR-lite)
   - **bug**: Bug investigations — root cause, symptoms, fix, and what made it tricky
   - **dependency**: External libraries/services — versions, quirks, upgrade notes
   - **gotcha**: Things that surprised you — subtle behaviors, footguns, non-obvious constraints
   - **concept**: Domain concepts that aren't obvious from the code alone

   ## Frontmatter
   Every page has YAML frontmatter:
   - title, type, created (YYYY-MM-DD), last_updated, tags (list), related (list of page names)

   ## Cross-references
   Use [[page-name]] wikilinks to connect related pages. When creating or updating a page,
   check if related pages need a backlink added.

   ## Index
   index.md is a categorized catalog. Update it on every ingest.

   ## Log
   log.md is append-only. One entry per operation with date, operation type, and summary.
   ```

3. Initialize `index.md` with empty category sections:
   ```markdown
   # Wiki Index

   ## Architecture
   ## Patterns
   ## Decisions
   ## Bugs
   ## Dependencies
   ## Gotchas
   ## Concepts
   ```

4. Initialize `log.md`:
   ```markdown
   # Wiki Log
   <!-- Append-only. One entry per operation. -->
   ```

5. Scan the codebase and create foundational pages:
   - Read the project's README, package.json/Cargo.toml/pyproject.toml/go.mod (whatever exists)
   - Read the top-level directory structure
   - Create 3-5 foundational pages covering: project overview, key architecture, main dependencies
   - Update `index.md` and `log.md` accordingly

The goal of the initial scan is a useful starting point, not exhaustive documentation. Capture the
big picture — what this project is, how it's structured, what it depends on. Depth comes later
through incremental ingests.

---

### ingest — File a discovery into the wiki

This is the most common operation. When you learn something significant during development — a bug
root cause, an architecture insight, a dependency quirk, a failed approach — file it into the wiki.

A single discovery often touches multiple pages. Finding a bug might reveal an architecture pattern
and a dependency gotcha. That's fine — create or update as many pages as the discovery warrants.

**Steps:**

1. **Decide what pages to create or update.** Read `index.md` to see what already exists. If a
   relevant page exists, update it. If not, create a new one.

2. **Write or update pages in `.wiki/pages/`.** Use this frontmatter format:
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
   Keep pages focused. A page about the auth middleware shouldn't also cover the database schema.
   If content spans multiple concerns, split it into pages and cross-reference them.

3. **Add cross-references.** Use `[[page-name]]` wikilinks. When you add a link from page A to
   page B, check if page B should link back to A. The goal is a navigable knowledge graph.

4. **Update `index.md`.** Add new entries or update summaries under the appropriate category:
   ```markdown
   - [Auth Middleware](pages/auth-middleware.md) — Express middleware handling JWT validation and role-based access | tags: auth, express
   ```

5. **Append to `log.md`:**
   ```markdown
   ## [2026-04-11] ingest | Discovered auth middleware silently swallows 403 errors
   Updated: pages/auth-middleware.md, pages/error-handling.md
   Created: pages/silent-403-gotcha.md
   Cross-refs added: auth-middleware <-> error-handling, auth-middleware <-> silent-403-gotcha
   ```

**When to ingest (even without being asked):**
- You found the root cause of a non-trivial bug
- You discovered an architectural boundary or pattern that wasn't obvious
- An approach failed and the reason is worth remembering
- A dependency behaves unexpectedly
- You figured out something about the build, deploy, or test setup that took effort

The bar is "would another agent (or future-me) waste time rediscovering this?" If yes, ingest it.

---

### query — Consult the wiki

When you need context about the codebase — before starting work, when investigating a bug, when
making a design decision — check the wiki first.

**Steps:**

1. Read `.wiki/index.md` to find relevant pages.
2. Read the relevant pages.
3. Synthesize what you learned with whatever you already know.
4. If the query produces new insight (e.g., you connect two things that weren't linked before),
   file it back via ingest. This is how the wiki compounds — queries can create new knowledge.

Querying is lightweight. Don't make it a ceremony. It's just "read the index, read relevant pages."

---

### lint — Health check the wiki

Run this periodically (at the start of a session, or when the user asks) to keep the wiki healthy.

**Check for:**

- **Contradictions**: Page A says X, page B says not-X. Flag them and resolve if you can, or mark
  them as open questions.
- **Stale pages**: Pages whose `last_updated` is old and whose content may no longer reflect the code.
  Read the relevant code and update or archive the page.
- **Orphan pages**: Pages with no inbound `[[wikilinks]]` from other pages. Either add cross-references
  or consider if the page is still relevant.
- **Missing pages**: `[[wikilinks]]` that point to pages that don't exist. Either create them or
  remove the broken links.
- **Gaps**: Important concepts or components that should have pages but don't. Suggest creating them.
- **Index drift**: Pages that exist in `.wiki/pages/` but aren't in `index.md`, or vice versa.

**Output:** Summarize what you found and what you fixed. For things you couldn't resolve, add them
as open questions on the relevant pages.

---

## Principles

These aren't rules to memorize — they're the reasoning behind the design. When in doubt, think
about what makes the wiki most useful for the next agent session.

**Compound, don't duplicate.** The whole point is that knowledge accumulates. If a page exists,
update it rather than creating a parallel one. If two pages cover the same thing, merge them.

**Focused pages over comprehensive ones.** A page about "the auth system" that covers middleware,
JWT config, role definitions, and session handling is hard to update and hard to link to. Split
it into focused pages and cross-reference them.

**Write for future agents, not humans.** The primary reader is an LLM agent in a future session.
Be precise and concrete. Include file paths, function names, error messages. Skip the prose.

**Low ceremony.** Don't agonize over perfect categorization or formatting. A rough page that
captures a key insight is infinitely better than no page. You can always lint and improve later.

**Cross-reference aggressively.** The value of the wiki grows with its connections. Every page
should link to related pages. When you update one page, check if related pages need updates too.

---

## File Layout Reference

```
.wiki/
├── SCHEMA.md          # Conventions and page type definitions
├── index.md           # Categorized catalog of all pages
├── log.md             # Append-only chronological activity log
└── pages/
    ├── project-overview.md
    ├── auth-middleware.md
    ├── database-schema.md
    ├── ci-pipeline.md
    └── ...
```
