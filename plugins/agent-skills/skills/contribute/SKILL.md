---
name: contribute
description: >
  User-directed contribution workflow for a repo. Picks work informed by observe's wiki data,
  plans before coding, drafts a branch + commits + PR description (but does not open the PR),
  and iterates on review feedback when the user asks. Uses the user's own git/gh/az identity.
  Use when: user says "contribute"/"pick something"/"draft a PR"/"iterate on the review"/"plan
  this change", or expresses any equivalent intent such as "I want to submit a fix/patch",
  "help me ship this", "respond to the reviewer", "open a PR for issue #N", "what should I
  work on next". Hard safety rails: never pushes to protected branches, never opens PRs
  autonomously, never takes unsolicited social actions.
allowed-tools: Read Write Edit Glob Grep Bash
---

# Contribute

You take work from loose intent ("help out in the auth layer", "find a good first issue",
"address the review on #482") through to a reviewable local branch with a drafted PR
description. You use the user's own `git` / `gh` / `az` authentication — there is no bot
account, every artifact you produce is signed as the user.

You read from [agent-wiki](../agent-wiki/SKILL.md) — especially the pages [observe](../observe/SKILL.md)
maintains (`contributor`, `workflow`, `review-policy`, `team-dynamics`) — for repo conventions,
ownership, and gates. You do **not** invoke observe. Before any operation, check the wiki:

- If `<wiki-root>` does not exist or contains no pages with `type: contributor` / `workflow` /
  `review-policy` / `team-dynamics`, stop and tell the user to run `observe survey` first.
- If those pages exist but are stale (any relevant page's `## Last refreshed` is older than the
  thresholds observe documents, or the page marks itself stale), stop and tell the user to run
  `observe refresh` first.

Do not fall back to raw forge queries to substitute for missing observe data — the wiki is the
contract, and operating without it defeats the "grounded context" premise.

## Forge detection

Detect the forge per [observe § Forge detection](../observe/SKILL.md#forge-detection) and cache
the result for the session. Forge-specific PR-open commands (which the user runs, not you):

- GitHub: `gh pr create --body-file .git/PR_EDITMSG`
- Azure DevOps: `az repos pr create --description @.git/PR_EDITMSG`

## Safety rails

All four apply to every operation. Do not suppress any of them.

1. **Never push to a protected branch.** Check protection **once per branch per session** and
   cache the result — do not re-query before every push.
   - GitHub: `gh api repos/{owner}/{repo}/branches/{branch}/protection` (a 200 means protected).
   - Azure DevOps: `az repos policy list --branch <branch>` (any required policy means protected).

   Regardless of detection result, these are always protected and pushes are refused without
   exception: `main`, `master`, `trunk`, `develop`, `release/*`, `hotfix/*`. If the forge API
   returns an ambiguous response (e.g. 404 on a private repo where the token lacks scope),
   refuse the push and surface the ambiguity to the user — do not default to "not protected."

2. **Never open a PR autonomously.** Draft the PR description into a file (default
   `.git/PR_EDITMSG`; also print it to stdout). Hand the user the exact command to run — e.g.
   `gh pr create --body-file .git/PR_EDITMSG` or `az repos pr create --description @.git/PR_EDITMSG`.
   Stop there.

3. **Scope cap per run:** 300 lines changed, 8 files touched, 1 PR. When any limit is about to
   trip, pause and ask the user "scope cap reached — continue?". Do not silently exceed.

4. **No unsolicited social actions.** Do not comment on other contributors' PRs or issues, do
   not `@`-mention people, do not request reviewers, do not add labels that route work to
   others — unless the user explicitly asks. "Solicited" means the user asked you — not a
   CI bot, not a dependency-bot, not a review-bot. Bot comments on the user's PR are not
   solicitation and do not authorize replies. Replies initiated by the user during `iterate`
   on their own active PR are fine.

## Commits

- Use the user's configured `git` identity. Do not override `user.name` / `user.email`.
- Do **not** add a `Co-Authored-By` trailer or any bot signature. The user is the author.
- Do not sign-off (`Signed-off-by`) unless the repo's CONTRIBUTING or DCO policy requires it —
  check the wiki's `workflow` or `review-policy` pages first.
- Never pass `--no-verify`. If a pre-commit hook fails, fix the cause and make a new commit.

## Operations

Four operations, normally invoked in order for a single contribution but usable independently.

---

### pick — User-directed candidate selection

Run when the user says "pick something", "find a good first issue", or describes the kind of
work they want to do. User frames the area; you filter candidates using the wiki.

1. Parse the user's framing into filter criteria (path, label, size, contributor, staleness).

2. Read relevant wiki pages: `team-dynamics` for active paths and typical PR size; `contributor`
   pages for ownership (avoid areas someone is mid-refactoring); `workflow` and `review-policy`
   for required labels and reviewer routing; `index.md` for anything else.

3. Query the forge for open issues/PRs that match:
   - GitHub: `gh issue list --label ... --state open --json number,title,labels,updatedAt`.
   - Azure DevOps: `az boards work-item query` or `az repos pr list` with the relevant filters.

4. Rank. Good-first-issue candidates: label match, small scope, clear acceptance, a codeowner
   who reviews quickly (from `contributor.review_latency`). Exclude anything assigned, stale
   beyond the project's norm, or blocked by another open PR you can see.

5. Return a shortlist (3-5) with one-line rationales, or a single recommendation if the user
   asked for one. **Do not** start work — the user picks.

---

### plan — Outline the approach before coding

Run after `pick` (or when the user points at a specific issue) and before any edit.

1. Read the issue/task fully. Read the wiki pages it touches (architecture pages from agent-wiki;
   `workflow` and `review-policy` from observe's pages) and any relevant source files.

2. Produce a written plan covering: goal, files to change, tests to add/update, expected PR size
   (verify it fits under 300 lines / 8 files — if not, propose a split now), required CI checks
   to anticipate, and any conventions from `CLAUDE.md` / CONTRIBUTING that apply.

3. Show the plan to the user and wait for confirmation before moving to `draft`. Plans are cheap
   to revise; half-done drafts are not.

4. **On confirmation,** persist the plan to `<wiki-root>/plans/<filename>.md` (see
   [agent-wiki § Storage location](../agent-wiki/SKILL.md#storage-location) for `<wiki-root>`;
   create the `plans/` directory if missing). If the user rejects the plan, do not persist —
   revise and re-confirm first. The plan is wiki content, not repo content — it lives in the
   wiki, **not** in `.git/`. Filename rules:
   - With issue number: `<issue>-<slug>-<YYYYMMDD-HHMM>.md` (e.g. `482-auth-timeout-20260418-1430.md`).
   - Without issue number: `<slug>-<YYYYMMDD-HHMM>.md`.
   - `slug` is a short kebab-case description, 2-4 words.

   Frontmatter: `title`, `issue` (nullable), `created`, optional `tags`. Body: the same plan
   shown to the user (goal, files to change, tests, expected size, required CI checks,
   conventions). Downstream skills — notably [reflect](../reflect/SKILL.md) — read this file
   to compare plan against actual.

---

### draft — Implement the plan on a local branch

Run after the user approves a plan.

1. Create a local branch from the correct base (default branch unless the plan says otherwise).
   Naming: follow any convention the wiki records; otherwise a short slug prefixed with the user's
   git username and issue number, e.g. `alice/1234-fix-auth-timeout`. **Never** branch from or
   commit onto `main`/`master` directly.

2. Implement in small commits. Run the repo's pre-commit hooks normally; do not pass `--no-verify`.
   If a hook fails, fix it and commit again — never amend across a hook failure.

3. Before each push: run the protection check above. Refuse on any protected branch; refuse
   unconditionally on `main`/`master`.

4. Watch the scope cap continuously. When within ~10% of any limit, pause and ask the user
   before continuing.

5. Write the PR description to `.git/PR_EDITMSG` (also echo to stdout) with: summary, motivation,
   links to the issue, what changed by file, test plan, any follow-ups. Follow the repo's PR
   template if one exists — read it from `.github/PULL_REQUEST_TEMPLATE.md` or
   `.azuredevops/pull_request_template/` and fill it in.

6. Print the exact command for the user to open the PR themselves. **Do not run it.** Examples:
   - GitHub: `gh pr create --base <base> --head <branch> --title "<title>" --body-file .git/PR_EDITMSG`
   - Azure DevOps: `az repos pr create --source-branch <branch> --target-branch <base> --title "<title>" --description @.git/PR_EDITMSG`

7. If the scope cap tripped (not just the soft-warn at step 4), suggest the user run
   [`reflect retro`](../reflect/SKILL.md) after the PR wraps up so the drift is captured while
   the context is fresh. Suggestion only — do not invoke reflect yourself.

---

### iterate — Respond to review feedback on your own PR

Run when the user points at a review or a failing CI run on a PR you drafted and asks you to
address it.

**Hard cap: at most 3 iterate rounds per PR without fresh user instruction.** After the third
push, stop and hand the PR back to the user regardless of remaining feedback. Also stop if the
previous push's CI has not completed — don't stack rounds on top of in-flight checks.

1. Read the review comments and CI failure logs. Bound the log read — don't pull a full CI log
   into context. Use tail/grep first, widen only if the failure isn't obvious:
   - GitHub: `gh pr view <n> --json reviews,comments,statusCheckRollup`,
     `gh run view <run-id> --log-failed | tail -n 200` (or `grep -iE 'error|fail|panic'` first).
   - Azure DevOps: `az repos pr show --id <n>`, comment threads via `az repos pr reviewer`,
     pipeline logs via `az pipelines runs show` (piped through `tail`/`grep` similarly).

2. For each actionable comment or failure: edit code, add tests, commit on the existing branch.
   Respect the scope cap across the whole iteration, not just this push.

3. Push to the same branch (protection check still applies). **Never** force-push unless the
   user asks explicitly; if they do, use `--force-with-lease`, never `--force`.

4. Post terse replies **only on the user's own PR**, and only in response to specific comments
   or deterministic check failures you addressed. The only shape: "Fixed in `<short-sha>`." or a
   one-line explanation when you couldn't fix it (e.g. "Skipped — change is out of scope; split
   into follow-up #NNN.").
   - **Never** post retry chatter for intermittent CI. If a check looks flaky (passed before on
     the same commit, or failure message doesn't match the code under review), surface it to the
     user and stop — do not auto-retrigger, do not post "flaky, retrying" comments.
   - Do not `@`-mention other contributors, do not comment on other PRs, do not request reviewers.

5. Leave anything that needs a human call — disagreements with a reviewer, scope expansion,
   design pushback, intermittent CI — to the user. Surface it clearly and stop.

6. If this is the second or later iterate round on the same PR, or the PR just merged after an
   iterate sequence, suggest the user run [`reflect retro`](../reflect/SKILL.md) on the PR before
   the context fades. Do not run reflect yourself.

---

## Principles

**User identity, user authority.** Every commit, push, and reply is the user's. No bot account,
no co-author trailers, no autonomous PR creation. You are drafting, not publishing.

**Read observe, do not call observe.** The wiki is the contract. Stale pages mean you stop and
ask the user to run `observe refresh`, not that you quietly do it yourself.

**Small, reviewable, in scope.** Respect the scope cap. Pause before you trip it, don't
apologize after.

**Solicited social actions only.** Talk on the user's own PR during an active review. Nothing
else — no drive-by comments, no at-mentions, no review requests.
