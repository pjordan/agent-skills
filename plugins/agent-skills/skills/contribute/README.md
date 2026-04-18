# contribute

User-directed contribution workflow that takes a task from "pick something to do" through to a reviewable local branch and a drafted PR description — without ever opening the PR itself or taking unsolicited social actions. Reads context from [agent-wiki](../agent-wiki/) and the pages [observe](../observe/) maintains.

## Why

Autonomous contribution is easy to get wrong in ways that waste real people's time: a bot opens a PR nobody asked for, tags three maintainers, pushes to a branch with protection rules, and expands a "small fix" into a 900-line refactor. contribute is designed around the opposite defaults — the user directs, the user publishes, and the agent stops at the last safe moment before any of those failure modes can trigger.

It does the tedious middle: reading the issue, consulting the wiki for conventions, planning, coding on a local branch with appropriate commit hygiene, and writing a proper PR description. Then it hands the user a single command to open the PR themselves.

## Operations

| Operation | What it does |
|-----------|-------------|
| **pick** | User says what area or kind of work; contribute filters candidates using observe's wiki data and returns a recommendation or shortlist. |
| **plan** | Outlines the approach in writing before any edit — files, tests, size, conventions, CI gates. User confirms. |
| **draft** | Creates a local branch, implements the plan in small commits, writes the PR description to `.git/PR_EDITMSG`, prints the command for the user to open the PR. Does not open it. |
| **iterate** | On the user's own PR, reads review comments and CI failures, pushes fixes, and posts terse replies only to the specific items being addressed. |

## Where data comes from

contribute reads agent-wiki at `${CLAUDE_PLUGIN_DATA}/wikis/<project-key>/pages/` — especially the `contributor`, `workflow`, `review-policy`, and `team-dynamics` pages that observe writes. It does not call observe; if those pages look stale, contribute tells the user to run `observe refresh` and stops. See [SKILL.md](SKILL.md) for the operation details and safety rails.

## Safety rails

All four are enforced on every operation:

1. **No pushes to protected branches.** Checks `gh api .../branches/{b}/protection` or `az repos policy list`. A fixed set of common branch names (including `main`, `master`, `release/*`, and others — see [SKILL.md § Safety rails](SKILL.md#safety-rails)) is always refused regardless of detection.
2. **No autonomous PRs.** The PR description lands in `.git/PR_EDITMSG`; the user runs `gh pr create` / `az repos pr create`.
3. **Scope cap per run:** 300 lines, 8 files, 1 PR. Agent pauses and asks before exceeding.
4. **No unsolicited social actions.** No comments on others' PRs, no `@`-mentions, no review requests — unless the user explicitly asks. Solicited replies on the user's own active PR during iterate are fine.

## How it works in practice

**Picking work** — User says "find me a good first issue in the repo". contribute reads `team-dynamics` and the codeowner pages, queries `gh issue list --label "good first issue" --state open`, and returns three candidates ranked by label match, size, and the codeowner's median review latency. The user picks one; contribute moves to plan.

**Drafting a change** — For the chosen issue, contribute reads the relevant architecture pages from agent-wiki plus the `review-policy` and `workflow` pages observe wrote, outlines a 120-line change across 3 files, confirms with the user, branches from `main`, commits, writes a PR body that fills in the repo's template, and prints:

```
gh pr create --base main --head alice/482-fix-timeout \
  --title "Fix auth middleware timeout on cold-start" \
  --body-file .git/PR_EDITMSG
```

The user inspects the branch and runs the command themselves.

**Iterating on review** — After a reviewer leaves two comments and one CI check fails, the user says "address the review". contribute reads the comments and the failed check's log, pushes a fix commit (non-force) to the same branch, and posts "Fixed in `abc1234`." on each addressed thread. It does not answer the reviewer's design question, which is a call for the user — it surfaces it and stops.

## Design

- **User identity end-to-end.** User's git, gh, and az auth. No bot account, no `Co-Authored-By` trailer on commits, no autonomous publishing.
- **Pre-commit hooks respected.** Never passes `--no-verify`. On hook failure, fix and commit anew — no amending across failures.
- **Force-pushes opt-in.** Only when the user asks, and then with `--force-with-lease`.
- **Reads observe's output, does not call observe.** Stale wiki data is surfaced to the user as a prerequisite, not silently refreshed.
- **Hard scope caps.** 300 lines / 8 files / 1 PR per run, with a pause-and-confirm gate before exceeding.
