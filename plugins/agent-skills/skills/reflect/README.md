# reflect

After-action learning loop for a repo. Reads the plan that went in, the commits and reviews that came out, and the CI that gated them; writes retro and playbook pages into [agent-wiki](../agent-wiki/) so the next session starts with calibrated priors instead of fresh guesses.

## Why

Sessions end, and the thing that actually got learned — the scope cap you tripped, the reviewer whose latency was nothing like the observed median, the third time a flaky e2e cost an iterate round — evaporates unless something writes it down. reflect is that something. It sits downstream of [contribute](../contribute/) (which produces a plan artifact) and alongside [observe](../observe/) (which maintains baseline priors), and its job is to compare expectation to outcome and file the delta. Over a handful of retros, recurring deltas promote into playbooks the team can actually reuse.

reflect is strictly write-after-the-fact. It never mutates source code, never opens or comments on anything on the forge, and never modifies observe's pages — the disjoint-write rule between reflect and observe keeps both skills honest across refresh cycles.

## Operations

| Operation | What it does |
|-----------|-------------|
| **retro** | Produces an after-action report for one PR, session, or issue — plan vs. actual, what went well, what drifted, lessons |
| **calibrate** | During a retro, flags observe baselines that disagree with observed reality and records the drift (without editing observe's pages) |
| **playbook** | Promotes a pattern seen in 3+ retros into a standalone playbook page; requires user confirmation |
| **brief** | Synthesizes a compact summary of recent retros and playbooks on demand; writes nothing |

## Where data lives

reflect writes into agent-wiki, under `<wiki-root>/pages/`, using two new page types — `retro` and `playbook` — appended to `<wiki-root>/SCHEMA.md` on first invocation. Plans read during `retro` come from `<wiki-root>/plans/`, which contribute writes during its `plan` operation. See [SKILL.md](SKILL.md) for the exact data flow and page-body structure, and [agent-wiki/SKILL.md § Storage location](../agent-wiki/SKILL.md#storage-location) for how `<wiki-root>` is computed.

## Forge support

reflect auto-detects the forge per [observe § Forge detection](../observe/SKILL.md#forge-detection) — GitHub via `gh`, Azure DevOps via `az`, anything else via bare git with a `## Limitations` note on the retro page.

## How it works in practice

**PR just merged** — User merges `#482`. reflect notices the merge signal, suggests `reflect retro #482`, and stops. On confirmation, it reads `<wiki-root>/plans/482-auth-timeout-*.md`, diffs planned scope (300 lines / 6 files) against actual (420 lines / 9 files — the scope cap tripped), pulls the PR's review thread and two failed CI runs, and writes `retro-482-auth-timeout.md` with what went well, what drifted, and one calibration finding: the codeowner's median review latency on this one was 18h vs. the 4h observe has recorded.

**Third retro with the same pattern** — Scanning retros, reflect notices three separate entries mention a flaky Playwright e2e costing an iterate round. It drafts `playbook-flaky-e2e.md` cross-linking the three retros, shows the draft to the user, and on confirmation writes the page and adds backlinks into each evidence retro's frontmatter.

**Mid-session orientation** — User asks "what have we learned about releases lately?" Agent runs `reflect brief`, reads recent `retro` and `playbook` pages tagged `release`, and answers in a paragraph with citations. Writes nothing.

## Design

- **Writes only to the wiki** — never mutates source code, observe's pages, or anything on the forge.
- **Evidence-linked** — every claim cites a SHA, PR URL, or wiki page; uncited claims are not written.
- **Degrades, doesn't fail** — missing plan file, unauthenticated forge CLI, thin git history: reflect still runs and marks the gap with `## Limitations`.
- **Compounding is opt-in** — retros write freely; playbooks require user confirmation because they shape future sessions.
- **Does not invoke other skills** — when observe's baselines look stale, reflect records the drift and tells the user to run `observe refresh`.
