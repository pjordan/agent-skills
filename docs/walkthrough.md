# Walkthrough: From Install to Feedback Loop

This walkthrough shows the full agent-skills lifecycle across two sessions: bootstrapping context, shipping a contribution, learning from the outcome, and using calibrated data in the next session.

## Phase 1: Install and Build Context (Session 1)

```console
~/git/acme-api $ /plugin marketplace add pjordan/agent-skills
 ✓ Source added: pjordan/agent-skills

~/git/acme-api $ /plugin install agent-skills@pjordan-agent-skills
 ✓ Installed agent-skills v1.4.0 (4 skills: agent-wiki, observe, contribute, reflect)

~/git/acme-api $ /reload-plugins
 ✓ Plugins reloaded. 4 new skills available.
```

Initialize the wiki and survey the repository:

```console
> init the wiki for this project

  Initializing wiki at $CLAUDE_PLUGIN_DATA/wikis/-Users-pat-git-acme-api/ ...
  Created SCHEMA.md, index.md, log.md
  Wiki ready — 0 pages.

> observe survey

  Detecting forge ... github.com (gh authenticated as @pat)
  Scanning git history (1,247 commits, 14 contributors) ...
  Reading CLAUDE.md, CONTRIBUTING.md, CODEOWNERS ...
  Querying gh: PRs, reviews, branch protection, workflows ...
  Writing pages:
    pages/contributor-alice-chen.md      47 commits in src/auth/, src/payments/
    pages/contributor-bob-kumar.md       63 commits in src/api/, src/models/
    pages/contributor-pat.md             19 commits in src/cli/, tests/
    pages/workflow-ci.md                 3 required checks, branch protection on main
    pages/review-policy-main.md          1 approval required, CODEOWNERS active
    pages/team-dynamics.md               median cycle time 1.4d, 82 PRs in 90d
  Survey complete — 6 pages written.
```

The wiki now contains structured, quantitative data about the repository — ownership areas, review latency, CI configuration, and team dynamics. All sourced from git history and forge metadata.

## Phase 2: Ship a Contribution (Session 1 continued)

```console
> pick something to work on — ideally in the auth layer

  Reading wiki ... contributor-alice-chen.md, workflow-ci.md, review-policy-main.md
  Querying open issues: gh issue list --label="good first issue" ...
  Candidates:
    #214  Fix token refresh race in AuthMiddleware    (auth, bug, good-first-issue)
    #201  Add rate-limit headers to /v2/sessions      (auth, enhancement)
    #198  Flaky test in test_oauth_callback            (auth, test, flaky)
  Alice owns src/auth/ (68% blame share, 3.2h median review latency).
  Recommend: #214 — scoped bug, 1 file, clear repro in the issue.

> go with 214

  Planning ... wrote plans/214-fix-token-refresh-20260418-1430.md
  Scope estimate: ~45 lines, 2 files
  Acceptance criteria:
    - [ ] Race condition in concurrent refresh no longer reproduces
    - [ ] Regression test covers the concurrent-refresh path
  Branch: pat/214-fix-token-refresh-race

  Drafting ...
    src/auth/middleware.ts               +32 -8
    tests/auth/middleware.test.ts        +12 -0
  2 commits: d3f7a01 fix: guard refresh with mutex
             a8c2e49 test: add concurrent-refresh case
  PR description written to .git/PR_EDITMSG

  Ready for you to open the PR:
    gh pr create --title "fix: guard token refresh race (#214)" --body-file .git/PR_EDITMSG
```

The user opens the PR themselves — contribute never opens PRs autonomously:

```console
~/git/acme-api (pat/214-fix-token-refresh-race) $ gh pr create --body-file .git/PR_EDITMSG
 ✓ Created pull request #221
```

## Phase 3: Learn from the Outcome (Session 2)

After the PR merges, the next session begins:

```console
~/git/acme-api (main) $ claude
 ✓ Loaded agent-skills (4 skills)

> PR #221 was merged yesterday — let's retro it

  Reading plan: plans/214-fix-token-refresh-20260418-1430.md
  Comparing plan vs. actual:
    Planned: 2 files, ~45 lines    Actual: 2 files, 44 lines     on target
    Planned reviewer: Alice        Actual: Alice, 4.1h latency   (baseline 3.2h)
    CI: all green on first push    Actual: lint failed once, fixed in follow-up commit
  Acceptance criteria:
    - [x] Race condition no longer reproduces                    sha:d3f7a01
    - [x] Regression test covers concurrent-refresh path         sha:a8c2e49

  Calibration drift detected:
    workflow-ci.md lists eslint but omits the --fix flag from CONTRIBUTING.md
  Appended 1 finding to calibration-queue.md

  Retro written to pages/retro-221-token-refresh.md
```

The retro captures what went well, what drifted, and what the wiki got wrong. The calibration finding is queued for observe to process.

## Phase 4: Compounding — Calibrated Next Session (Session 2 continued)

```console
> pick more work

  Checking wiki ... calibration-queue.md has 1 unconsumed entry.
  Run observe refresh first so baselines are corrected.

> observe refresh

  Reading calibration-queue.md (1 entry):
    workflow-ci.md: adding eslint --fix note from CONTRIBUTING.md
  Refreshing forge metadata (HEAD:f91b3c0, +3 commits since last survey) ...
  Updated pages:
    pages/workflow-ci.md                 +lint --fix guidance, commit count 1250
    pages/contributor-pat.md             +2 merged PRs, updated ownership areas
  calibration-queue.md drained — 0 entries remaining.
  Refresh complete — 2 pages updated.

> now pick work

  Reading wiki (calibration queue empty) ...
  Candidates:
    #198  Flaky test in test_oauth_callback    (auth, test) — pat now has ownership signal
    #227  Deprecation warning on Node 22       (infra, chore)
    #201  Add rate-limit headers               (auth, enhancement)
  Retro for #221 noted lint-on-save prevents CI churn — factoring that in.
  Recommend: #198 — flaky test, small scope, builds on auth context from #214.
```

The second pick is informed by the first session's work: ownership has shifted, the CI lint finding has been corrected, and the retro's lessons influence candidate ranking. This is the feedback loop in action — each session leaves the wiki more accurate than it found it.
