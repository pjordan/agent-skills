# observe

Context-builder for a repo, its team, and its workflow. Reads public git and forge data plus the project's own guidance, then files structured pages into [agent-wiki](../agent-wiki/) so coding agents start sessions with grounded context instead of guesses.

## Why

Every coding session on an unfamiliar repo starts the same way: skim the README, squint at the PR history, guess at the review rules, miss the CODEOWNERS file, and push something that trips a required check nobody mentioned. observe does that reconnaissance once, writes it down, and keeps it fresh so the next agent — or the next operation of the same agent — starts with the picture already drawn.

It is deliberately read-only on the repo. Everything it learns lands in the per-user agent-wiki; the source tree is never modified.

## Operations

| Operation | What it does |
|-----------|-------------|
| **survey** | First-time scan — reads git history, forge metadata, CLAUDE.md, CONTRIBUTING, CODEOWNERS, templates, ADRs; writes contributor, workflow, review-policy, and team-dynamics pages |
| **refresh** | Re-runs the survey against current state; auto-triggers when pages are older than 7 days or more than 50 commits behind |
| **brief** | Synthesizes a compact summary of what observe knows, on demand |
| **investigate** | Deep-dives a specific topic (a workflow, a contributor's review patterns, a file's ownership) and updates or adds one page |

## Where data lives

observe writes into agent-wiki, under `${CLAUDE_PLUGIN_DATA}/wikis/<project-key>/pages/`. It extends agent-wiki's schema with four new page types — `contributor`, `workflow`, `review-policy`, `team-dynamics` — and appends them to `<wiki-root>/SCHEMA.md` on first survey. See [SKILL.md](SKILL.md) for the exact data flow and page-type definitions, and [agent-wiki/SKILL.md § Storage location](../agent-wiki/SKILL.md#storage-location) for how `<wiki-root>` is computed.

## Forge support

observe auto-detects the forge from `git remote -v`:

- **GitHub** (`github.com`) — uses `gh` for PRs, reviews, branch protection, CODEOWNERS, workflows.
- **Azure DevOps** (`dev.azure.com`, `visualstudio.com`) — uses `az repos`, `az pipelines`, `az boards`, `az repos policy`.
- **Other / offline** — falls back to git alone and records what it could not reach under a `## Limitations` section on each page.

## How it works in practice

**Day 1 on a new repo** — User runs `observe survey`. observe reads `CLAUDE.md` (quoting the rules verbatim), walks CONTRIBUTING and CODEOWNERS, pulls the last 90 days of commits and merged PRs, computes ownership and review latency for the top ten contributors, and writes about a dozen pages plus a `team-dynamics` overview. Total time: a few minutes; the wiki is now primed for any skill that reads it. (Full-history scans are available on explicit request.)

**Two weeks later** — A contributor has onboarded, a new required check has appeared, and the release branch rules changed. observe notices the staleness markers on next invocation and runs `refresh` automatically: contributor pages get new ownership shares, `review-policy-main` picks up the new required check, `log.md` gets one entry summarizing the diff.

**Mid-session** — User asks "how do releases actually ship here?" The agent runs `observe brief`, reads the `workflow-release` and `review-policy` pages, and answers in a paragraph with citations. If the answer is thin, the agent follows up with `observe investigate release workflow` to deepen that one page rather than re-surveying everything.

## Design

- **Public data only** — git and forge metadata any contributor can already see. No communication-style or personality profiling.
- **CLAUDE.md is authoritative** — observe quotes it, never edits it; deviations are recorded as deviations.
- **Graceful degradation** — missing `gh`/`az` auth produces a narrower page with a `## Limitations` section, not a failure.
- **Writes only to agent-wiki** — the repo stays clean; the learned context is per-user and per-machine.
- **Does not invoke other skills** — observe only reads the world and writes pages. Downstream skills read those pages; they do not call back into observe.
