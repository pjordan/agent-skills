---
name: reflect
description: >
  After-action learning loop for a repo. Reads contribute's plan artifacts, git history, forge
  PR lifecycle (`gh`/`az`), CI results, and observe's baseline pages, then writes retro and
  playbook pages into agent-wiki. Use when: user says "retro"/"post-mortem"/"after-action"/"what
  did we learn"/"session summary"/"promote this to a playbook", or when a PR just merged on the
  current branch, contribute ran >=2 iterate rounds, or contribute tripped a scope cap — in
  those three cases suggest reflect and wait for confirmation (at most once per scope per
  session). Distinct from agent-wiki's `ingest`: reflect captures after-the-fact calibration
  against a plan, not in-the-moment codebase discoveries.
allowed-tools: Read Write Edit Glob Grep Bash
---

# Reflect

You close the learning loop. A contribution just shipped, an iteration just ended, a session
just wound down — reflect reads the plan that preceded it, the commits that implemented it, the
reviews and CI runs that gated it, and writes down what worked and what drifted from expectation.
Over time the pattern pages become playbooks the team can reuse.

You never mutate source code. You never open, comment on, or merge anything on the forge. You do
not invoke other skills — if reflect notices that observe's baselines look wrong, it records the
drift and suggests the user run `observe refresh`.

## Data flow

**Read-only inputs:**
- `<wiki-root>/plans/*.md` — contribute's plan artifacts (see
  [contribute § plan](../contribute/SKILL.md#plan--outline-the-approach-before-coding) for the
  filename grammar). The primary comparison point for "what did we expect vs. what actually
  happened."
- `git log`, `git show`, `git diff` on the branch or range under retro.
- Forge PR lifecycle. GitHub: `gh pr view <n> --json reviews,comments,statusCheckRollup`,
  `gh run view <run-id> --log-failed | tail -n 200`. Azure DevOps: `az repos pr show --id <n>`,
  `az pipelines runs show --id <run-id>`. Bound log reads — tail/grep first, widen only if the
  signal isn't obvious.
- `<wiki-root>/log.md` and observe's pages (`contributor`, `workflow`, `review-policy`,
  `team-dynamics`) as baseline priors for calibration (e.g. "contributor X's median review
  latency is 4h; on this PR it was 18h").
- `CLAUDE.md` at the repo root — **authoritative** project guidance; read, quote, never modify.
  Same posture as observe.

**Writes only to agent-wiki:** new pages of type `retro` or `playbook` under
`<wiki-root>/pages/`, plus updates to `<wiki-root>/index.md` and append-only entries in
`<wiki-root>/log.md`. See [agent-wiki § Storage location](../agent-wiki/SKILL.md#storage-location)
for how `<wiki-root>` is computed and the `<wiki-root>` convention.

**Never modify:** observe's pages (`contributor`, `workflow`, `review-policy`, `team-dynamics`),
the codebase, `CLAUDE.md`, `CONTRIBUTING.md`, or anything on the forge. The disjoint-write rule
is load-bearing: reflect writes `retro`/`playbook`; observe writes its four types; if both
skills wrote the same pages they would stomp each other across refresh cycles.

## Forge detection

See [observe § Forge detection](../observe/SKILL.md#forge-detection). Cache the detected forge
once per session and reuse it across every operation below.

## Page types

Two new page types extend agent-wiki's schema. On first invocation, append them to
`<wiki-root>/SCHEMA.md` if not already present.

- **retro** — one after-action report scoped to a single PR, issue, or session. Captures the
  plan-vs-actual diff and what the session teaches you about the repo or the team.
- **playbook** — a recurring pattern promoted from 3+ retros. Captures the situation, the
  evidence it keeps happening, and the recommended approach.

Frontmatter stays minimal — the body carries the structure:

```yaml
---
title: Retro — #482 auth timeout fix
type: retro
created: 2026-04-18
last_updated: 2026-04-18
tags: [retro, auth, iterate]
related: [contributor-alice, workflow-ci, playbook-flaky-e2e]
---
```

**`retro` body structure:**
- `## Context` — the task, PR number, issue number, scope of the retro.
- `## Timeline` — **include when the retro covers an incident, outage, or any event with a
  temporal sequence worth reconstructing.** A table with columns `Time | Event | Source` (use
  UTC or the team's timezone). Reconstruct from git timestamps, CI run start/end times, forge
  comments, deploy logs, and user-supplied details. If exact times are unavailable, use
  relative ordering (`T+0`, `T+5m`, etc.) and note the approximation. Skip this section for
  routine PR retros where the commit log already tells the story.
- `## Acceptance criteria — outcome` — one checklist item per criterion from the plan file's
  acceptance-criteria section (any header variant contribute emits — see
  [contribute § plan step 5](../contribute/SKILL.md#plan--outline-the-approach-before-coding)).
  Grade based on the criterion's type tag. Every grade ends with an evidence citation:

  | Type tag | Grade | Evidence format |
  |----------|-------|-----------------|
  | `[objective]` | `[x]` Met or `[ ]` Unmet | `sha:<short-sha>`, `pr:<url>`, or `test:<path>` |
  | `[proxy]` | `[x]` if metric improved in stated direction, `[ ]` if not | `delta:<before>→<after>` (e.g. `delta:steps 7→4`). Note: a proxy Met does not guarantee the subjective intent was achieved. |
  | `[subjective]` | `[?]` always — agent cannot verify | `human:<what-to-review>`. Describe what was implemented; flag for human review. Do not grade Met based on the agent's own judgment. |
  | untagged | `[x]` Met, `[ ]` Unmet, or `[?]` Unverifiable | `reason:<why>` for `[?]`. Legacy fallback for plans predating type tags. |

  If more than half of `[objective]` criteria grade Unmet, add a short note — that signals
  real drift. If most `[subjective]` criteria grade `[?]`, that is expected. If the plan file
  has no acceptance-criteria section, skip with: `_No plan-level criteria to grade._`
- `## Plan vs actual` — planned N lines / X files vs. actual M lines / Y files; iterate rounds;
  CI outcomes (pass/fail/flake). Quote the plan file's expected size verbatim.
- `## What went well` — concrete wins with SHA/PR links.
- `## What drifted` — signals that triggered during the session: scope caps tripped, iterate
  rounds consumed, protection refusals, flaky CI, review latency surprises.
- `## Calibration findings` — observed priors that look wrong. Name the baseline page, quote
  the prior value, cite the observed value and the source (SHA, PR URL).
- `## Lessons` — one-line takeaways, each with evidence link. These are the candidates playbook
  scans over later.

**`playbook` body structure:**
- `## Pattern` — the recurring situation in one paragraph.
- `## Evidence` — `[[wikilinks]]` to each retro page where the pattern showed up, with a one-line
  note per retro on how it manifested.
- `## Playbook` — the step-by-step recommended approach. Imperative voice. Cite the repo
  convention (`CLAUDE.md`, CONTRIBUTING, workflow page) any step leans on.
- `## Related` — cross-links to architecture, workflow, review-policy pages.

## Operations

Four operations. The user may invoke them explicitly; you may also **suggest** `retro` when the
trigger signals below fire, but you do not run it without confirmation.

---

### retro — Produce an after-action report

Run when the user says "retro this", "post-mortem on #482", "what did we learn", or "session
summary". **Also suggest** running `retro` (and stop, waiting for confirmation) when any of
these signals fire — at most once per scope per session (see the debounce rail in Principles):

- A PR just merged on the current branch.
- contribute finished `>= 2` iterate rounds.
- contribute tripped a scope cap during the session.

1. Scope the retro. One PR (`#N`), one session (time window), or one issue. If the user said
   "retro this" with no referent, ask which. Do not retro more than one scope at a time — the
   plan-vs-actual comparison degrades when scopes are mixed.

2. Locate the plan file. Glob `<wiki-root>/plans/*` for the issue number or slug matching the
   scope. **If absent**, note so in the retro's `## Context` and degrade gracefully: the retro
   still runs, but `## Plan vs actual` cites git+forge only and flags the missing plan as a
   process gap. If the plan file is found, also locate its `## Acceptance criteria` section for
   grading. If that section is absent, proceed — grading is skipped rather than blocking the
   retro.

3. Read git + forge + wiki — narrow queries only. Fetch each PR's forge lifecycle **at most
   once per session**; if you've already pulled `#N`'s reviews/CI in this session, reuse the
   result.
   - `git log --oneline <base>..<head>`, `git diff --stat <base>..<head>` for scope.
   - `gh pr view <n> --json reviews,comments,statusCheckRollup` or `az repos pr show --id <n>`.
   - `gh run view <run-id> --log-failed | tail -n 200` / `az pipelines runs show` only on
     failed or flaky checks; do not pull full CI logs into context.
   - Relevant observe pages (`contributor-*`, `workflow-*`, `review-policy-*`, `team-dynamics`)
     for baseline priors.

4. Write the retro page to `<wiki-root>/pages/retro-<short-slug>.md`. Slug rule: `retro-<issue>-<2-3 word kebab>`
   if there's an issue number, else `retro-<2-3 word kebab>-<YYYYMMDD>`. Every claim cites the
   SHA, PR URL, or observe page it came from.

5. Update `<wiki-root>/index.md` to list the new retro under the appropriate category. Append
   one `retro` entry to `<wiki-root>/log.md` with the scope and pages touched.

6. If forge CLIs are unauthenticated, run the retro from git alone and add a `## Limitations`
   section listing what you could not read and the command that would have unblocked you
   (`gh auth login`, `az login`).

---

### calibrate — Note drift from observe's priors

Run when, during a retro, you notice a baseline observe recorded that disagrees with what just
happened — contributor review latency, team-dynamics median cycle time, workflow required
checks, review-policy approval counts.

1. Identify the disagreeing baseline(s). Name the page (`contributor-alice`,
   `team-dynamics`, etc.), quote the prior value, cite the observed value.

2. Write the observation into the current retro page's `## Calibration findings` section with
   concrete evidence: commit SHA, PR URL, timestamps. One bullet per baseline.

3. **Append each finding to `<wiki-root>/calibration-queue.md`** so observe can discover drift
   without grepping retro pages. One line per finding, format:
   ```
   - **<page-name>** | <prior-value> → <observed-value> | [[retro-<slug>]] | <date>
   ```
   Create the file if it does not exist. Append-only — never edit or delete existing entries;
   observe drains the queue during refresh.

4. **Never modify observe's pages.** If the drift looks durable (not a one-off), note it in the
   queue entry. Reflect does not invoke other skills. The disjoint-write rule is a hard rail:
   two skills writing the same pages is the failure mode this avoids.

---

### playbook — Promote a recurring pattern

Run when a scan across existing retros shows the same pattern in 3+ retros, or when the user
says "promote this to a playbook".

1. Enumerate retros. Prefer a bounded scan: Grep `<wiki-root>/index.md` for retro entries with
   the candidate tag first, or limit to the N most recent retros (e.g. last 20). Only open the
   matching retros. Avoid Globbing and reading every retro on a mature wiki — that scales poorly.

2. Count occurrences. If fewer than 3, stop and tell the user the pattern isn't yet recurring
   enough to promote; list the retros you found so they can decide.

3. If 3 or more, draft a playbook page at `<wiki-root>/pages/playbook-<slug>.md` with the four
   body sections above. Cross-link each evidence retro with `[[wikilinks]]`.

4. **Show the draft to the user and wait for confirmation before writing.** Playbooks are
   compounding artifacts — they shape future sessions — and should be user-endorsed, not
   silently accreted.

5. On confirmation: write the page, update `<wiki-root>/index.md`, append a `playbook` entry
   to `<wiki-root>/log.md`, and add a backlink `related: [playbook-<slug>]` to each evidence
   retro's frontmatter.

---

### brief — Summarize what reflect knows

Run when the user says "brief me on retros" or needs a quick orientation to recent learning.

1. Read `<wiki-root>/index.md` and recent `retro` / `playbook` pages relevant to the user's
   framing (all of them if unspecified).

2. Synthesize a short answer — a few paragraphs, not a dump of the pages. Cover: what's been
   learned recently, which patterns are recurring, any calibration findings the team hasn't
   acted on yet.

3. **Writes nothing.** If synthesis surfaces a gap (a pattern that looks promotable, a
   calibration finding worth escalating), note it and suggest `playbook` or `observe refresh`.

---

## Principles

**Writes only to the wiki.** Never modifies observe's pages; never touches source code; never
makes forge mutations. The disjoint-write rule between reflect and observe is load-bearing.

**Degrade, don't fail.** If a plan file is missing, the retro still runs from git+forge and
flags the missing plan. If forge CLIs are unauthenticated, the retro runs from git alone with
a `## Limitations` section. A thin retro beats no retro.

**Evidence-linked.** Every claim in a retro or playbook cites the git SHA, PR URL, or wiki page
it came from. Uncited claims are not written.

**Compounding is opt-in.** Retro pages are written without confirmation (cheap to edit, easy
to delete). Playbook pages require user confirmation — they shape future sessions and deserve
an explicit nod.

**Retros are write-once forensic records.** Once written, a retro captures what was known at
that moment; do not retroactively edit it when later events contradict it (a revert, a follow-up
bug, a late user report that turns a `[?]` into a `[x]`, new CI evidence). Write a new retro
that cross-links the old one instead. This rule applies to every section of the retro,
including `## Acceptance criteria — outcome`.

**At most one retro suggestion per scope per session.** If the user already declined (or ran)
a retro for PR `#N` this session, don't re-surface the suggestion. Forge queries for lifecycle
data are non-trivial — avoid chatty re-suggestions.

**Does not invoke other skills.** When a baseline looks stale, reflect tells the user to run
`observe refresh` and stops. Skills coordinate through the wiki, not through calls.
