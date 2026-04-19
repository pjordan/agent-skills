---
name: observe
description: >
  Context-builder for a repo, its team, and its workflow. Scans git history, forge metadata
  (GitHub via `gh`, Azure DevOps via `az`), CLAUDE.md, CONTRIBUTING.md, CODEOWNERS, PR/issue
  templates, and ADRs, then writes structured pages into agent-wiki. Use when: user says
  "observe"/"survey"/"refresh observations"/"brief me"/"investigate <topic>", starting work on
  an unfamiliar repo, or before a session that needs contributor/workflow context. Produces
  grounded context for contribute and other downstream skills.
allowed-tools: Read Write Edit Glob Grep Bash
---

# Observe

You build the contextual picture of a repository that coding agents need before acting: who owns
what, how reviews happen, what the CI gates are, what the project's own guidance says. You read
public git + forge data and the repo's own docs, then file structured pages into the
[agent-wiki](../agent-wiki/SKILL.md).

Downstream skills (notably `contribute`) read the wiki pages you produce. They do not invoke you;
if their data looks stale they instruct the user to run `observe refresh` and stop.

## Data flow

**Read-only inputs:**
- `git log`, `git blame`, `git log --follow`, tracked files
- GitHub forge via `gh` (issues, PRs, reviews, branch protection, workflows, CODEOWNERS, labels)
- Azure DevOps forge via `az repos`, `az pipelines`, `az boards`, `az repos policy`
- `CLAUDE.md` at the repo root — **authoritative** project guidance; read, quote, never modify
- `CONTRIBUTING.md`, `CODEOWNERS`, `.github/` templates, `.azuredevops/` templates, `docs/adr/`, `docs/decisions/`

**Writes only to agent-wiki:** pages are written under agent-wiki's storage path
`${CLAUDE_PLUGIN_DATA}/wikis/<project-key>/pages/` (see [agent-wiki § Storage
location](../agent-wiki/SKILL.md#storage-location) for the path-computation rules and the
`<wiki-root>` convention). On the first survey, append the page types defined below to
`<wiki-root>/SCHEMA.md` so other skills and future sessions know what they mean.

**Never modify:** the codebase, `CLAUDE.md`, `CONTRIBUTING.md`, or anything else in the tree.

## Forge detection

Detect the forge once per session from the origin URL and reuse the result:

```bash
git remote -v | head -1
```

- Contains `github.com` → GitHub. Use `gh` (e.g. `gh pr list`, `gh api repos/{owner}/{repo}/branches/{b}/protection`).
- Contains `dev.azure.com` or `visualstudio.com` → Azure DevOps. Use `az repos`, `az pipelines`, `az boards`, `az repos policy list`.
- Neither → bare git only. Degrade gracefully (see below).

If the forge CLI is installed but unauthenticated, run the survey with what git alone provides
and add a `## Limitations` section to each affected page listing exactly what you could not
read and which command would have unblocked you (e.g. `gh auth login`, `az login`).

## Page types

These four page types extend agent-wiki's schema. Append them to `<wiki-root>/SCHEMA.md` on the
first survey if not already present:

- **contributor** — a named individual. Files owned (top paths by commits/blame share),
  review latency (median hours from PR ready to first review on PRs they review), review volume,
  areas of activity. Public git/forge data only. **Do not** record communication style,
  tone, personality, or anything that reads as social profiling.
- **workflow** — a named development or release workflow. CI jobs, required checks, branch
  protection, merge strategy, release cadence, environment gates.
- **review-policy** — how code review works here. Required reviewers, CODEOWNERS mapping,
  approval counts, stale-review rules, auto-merge behavior.
- **team-dynamics** — repo-level patterns across contributors: active set over the last N days,
  review network (who reviews whom, from PR metadata), hand-off patterns, typical PR size
  and cycle time. Aggregates only; no individual social judgments.

Frontmatter stays minimal — let the body carry the structure:

```yaml
---
title: Review Policy — main branch
type: review-policy
created: 2026-04-18
last_updated: 2026-04-18
tags: [review, branch-protection]
related: [workflow-ci, team-dynamics]
---
```

## Operations

Four operations. Users invoke them explicitly; you may also run `refresh` automatically when
the wiki's staleness markers trip (see below).

---

### survey — First-time scan

Run when no observe pages exist in the wiki, or when the user says "survey this repo".

1. Resolve `<wiki-root>` per agent-wiki's rules. If `<wiki-root>` does not exist, stop and ask
   the user to run agent-wiki's `init` first — observe writes into an existing wiki, it does
   not bootstrap one.

2. Detect the forge (see above). Cache the result for the rest of the operation.

3. Append the four page types above to `<wiki-root>/SCHEMA.md` if missing.

4. Read the authoritative docs in this order and quote relevant passages verbatim in the pages
   you write: `CLAUDE.md`, `CONTRIBUTING.md`, `CODEOWNERS`, `.github/PULL_REQUEST_TEMPLATE.md`,
   `.github/ISSUE_TEMPLATE/*`, `.azuredevops/pull_request_template/*`, ADR directories.

5. Gather contributor data. Default window is the last **90 days**; only widen to a year or more
   if the user explicitly asks for a full-history scan.
   - `git shortlog -sne --since="90 days ago"` for the active set.
   - For each top contributor, compute top owned paths via `git log --author=... --name-only`
     (or `git blame` aggregated by path) and record the top 5-10 paths by share.
   - From forge PRs, record review counts and median review-latency (hours from PR ready-for-review
     to first review) over the same window. GitHub: `gh pr list --state merged --limit 200 --json ...`
     (closed-but-unmerged PRs rarely inform latency and triple the cost). Azure: `az repos pr list --status completed`.

6. Gather workflow data:
   - CI: `.github/workflows/*.yml` or Azure Pipelines YAML; list required checks.
   - Branch protection: `gh api repos/{owner}/{repo}/branches/{b}/protection` or
     `az repos policy list --branch <b>`. Record which branches are protected and what's required.
   - Merge strategy: squash/merge/rebase policy from repo settings or CONTRIBUTING.
   - Release cadence: inferred from tag history (`git tag --sort=-creatordate`).

7. Write pages (one per contributor for the top ~5-10 active contributors; one `workflow-*` per
   distinct workflow; one `review-policy-*` per protected branch or global policy; one
   `team-dynamics` page). Add `[[wikilinks]]` between them and to existing architecture pages
   where relevant. Update `<wiki-root>/index.md` and append a single `survey` entry to
   `<wiki-root>/log.md`.

8. Each page gets a short `## Sources` section listing the commands and file paths the content
   came from, and a `## Last refreshed` line with today's date plus the current `HEAD` SHA and
   commit count (`git rev-list --count HEAD`). These drive staleness detection.

---

### refresh — Re-run against latest state

Auto-trigger when any observe page has `last_refreshed > 7 days ago` **or** the repo has more
than 50 commits since the recorded `HEAD` at last refresh. Manual invocation always works.

1. Read existing observe pages once. Collect their `last_refreshed` dates and SHAs into a single
   in-memory map; do not re-read per check.

2. Compute `git rev-list --count <recorded-sha>..HEAD` once for the repo's current HEAD and
   reuse it across every page's staleness check. If a page is stale (by either threshold) or the
   user asked for a full refresh, re-run the relevant survey steps for that page's scope.

3. Before recomputing a page's baseline, check for drift notes from `reflect`. Bound the scan:
   read recent `retro-*.md` pages whose `related:` frontmatter names this page (use Grep for
   `related:.*<page-name>` on `<wiki-root>/pages/retro-*.md`), and within those read the
   `## Calibration findings` section. Treat findings that name this page as evidence the prior
   has drifted — weight recomputation toward observed reality rather than stale averages.
   [reflect § calibrate](../reflect/SKILL.md#calibrate--note-drift-from-observes-priors)
   produces these findings; observe consumes them.

4. Update pages in place — do not create duplicates. Preserve `created`; bump `last_updated` and
   `## Last refreshed`. If a contributor has dropped out of the active set, mark the page
   `## Status: inactive since <date>` rather than deleting it.

5. Append a `refresh` entry to `log.md` summarizing what changed.

---

### brief — Compact on-demand summary

Run when the user says "brief me on this repo" or needs a quick orientation.

1. Read `<wiki-root>/index.md` and the observe pages relevant to the user's framing
   (all of them if unspecified).

2. Synthesize a short answer — a few paragraphs, not a dump of the pages. Cover: what the repo
   is, who's active, how review works, what the CI gates are, any `CLAUDE.md` rules the agent
   must honor, and any `## Limitations` that shaped the picture.

3. Do not write new pages during a brief. If synthesis surfaces a gap, note it and suggest
   `investigate <topic>` or `refresh`.

---

### investigate <topic> — Deep dive one area

Run when the user asks for detail beyond what the current pages cover — e.g. "investigate the
release workflow", "investigate alice's review patterns", "investigate how auth changes get
reviewed".

1. Identify the single focused topic. Resolve it to a target page (existing or new).

2. Run the minimum set of commands to answer it. Prefer narrow git queries and forge queries
   scoped to the topic over broad rescans.

3. Update the target page or create a new one of the appropriate type. Cross-reference into
   `team-dynamics` and any related `workflow` / `review-policy` pages.

4. Append an `investigate` entry to `log.md` with the topic and pages touched.

---

## Principles

**Public data only.** Git and forge metadata that any contributor can already see. No
communication-style or personality profiling — files owned and review latency, not tone.

**CLAUDE.md is authoritative.** When observed practice diverges from `CLAUDE.md` guidance, quote
`CLAUDE.md` verbatim and document both the stated policy and the observed configuration in a
`## CLAUDE.md compliance` section on the relevant page. For example, if CLAUDE.md says "all PRs
require two approvals" but the repo's branch protection only requires one, record both values.
This prevents downstream skills from enforcing rules the repo does not actually follow.
Never edit `CLAUDE.md`.

**Degrade, don't fail.** If `gh` or `az` is missing or unauthenticated, emit what git alone can
tell you and list the blocked command under `## Limitations` on each affected page.

**Write once, refresh in place.** Pages are long-lived. Update the same page across refreshes;
never shadow an old page with a near-duplicate new one.

**Do not act.** You read, you summarize, you file. Code changes, PRs, and comments belong to
contribute or the user.

**Quantify, don't narrate.** Prefer concrete metrics over qualitative summaries. Instead of
"Alice is active in the auth module," write "Alice: 47 commits in `src/auth/` (68% of path),
median review latency 3.2h (n=12 PRs)." Instead of "the team reviews PRs quickly," write
"median PR cycle time: 18h (p90: 42h, n=31 merged PRs in 90d)." Downstream skills make
decisions based on these numbers — vague characterizations force them to guess.
