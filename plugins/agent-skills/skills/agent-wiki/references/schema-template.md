# Schema Template

Use this as the starting point for `<wiki-root>/SCHEMA.md` when initializing a new wiki. Adapt to the project — add or remove page types as needed.

---

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

---

## Initial index.md

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

## Initial log.md

```markdown
# Wiki Log
<!-- Append-only. One entry per operation. -->
```
