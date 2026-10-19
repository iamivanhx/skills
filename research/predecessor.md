# Why the Predecessor got messy

Research for [#3](https://github.com/iamivanhx/skills/issues/3), a child of the map [#1](https://github.com/iamivanhx/skills/issues/1).

**Question.** What does the Predecessor ([ivanhxlabs/skills](https://github.com/ivanhxlabs/skills)) contain, and what made it messy? What is worth keeping, and which structural rules would have prevented the mess?

**Source.** The Predecessor repo itself (private), cloned at `main` = [`323936c`][tree] (2026-10-18), its full git history (414 commits, 2026-09-14 to 2026-10-18), and its GitHub issues. Every claim below cites a file at that commit, a commit, or an issue. Commit counts come from `git log --no-merges`, so they cover the 314 non-merge commits.

## Short answer

The Predecessor started as a **fork of an Inspiration** (mattpocock/skills) and never became its own Plugin. Over five weeks it grew in three directions at once: it kept absorbing upstream proposals, it took on whole new domains (product strategy, product verification), and it ran a heavy spec, ticket, and review process over **prose**. Most of the history is agents polishing the wording of rules that were copied across skills, then building checks to keep those copies in step. Size was measured but never cut, and the evals that would have made cuts safe were deferred ([#104][i104], [#105][i105]).

## Inventory

| Area | Contents | Size |
| --- | --- | --- |
| Skills | 31, split into `engineering/` (24) and `productivity/` (7), listed one by one in [`plugin.json`][pluginjson] | 8.8k lines in total, about 6.8 KB per `SKILL.md` ([#105][i105]) |
| Subagents | 6 in [`agents/`][agents]: designer, explorer, implementer, researcher, reviewer, verifier. A seventh, `merger`, was added and later removed | 11 to 17 lines each |
| Per-skill adapters | 28 `agents/openai.yaml` files for Codex | 3 to 5 lines each |
| Scripts | [`check-docs.mjs`][checkdocs] (links, manifest list, "canaries", banned phrases, size budgets), [`release.mjs`][release] (conventional-commit tags), `validate-plugin.mjs`, each with tests | 1.4k lines |
| CI | [`ci.yml`][ci]: runs the checks on every PR and push, then tags a release | 109 lines |
| Docs | [`GLOSSARY.md`][glossary] (287 lines, 7 domains), 5 ADRs, 4 specs, `docs/agents/*` (tracker, labels, domain), [`upstream-evidence.md`][upstream] (258-line audit table) | |
| Packaging | The repo is its own marketplace (`ivanhxlabs`) with one plugin (`hx-skills`), installed over SSH. pi installs it through `npx skills`. [`README.md`][readme] is 164 lines, nearly all install, update and rollback steps for two harnesses | |
| Hooks, commands, output styles, MCP | None | |

Biggest skills (bytes of `SKILL.md`, plus the other `.md` files in the same directory): `wayfinder` 21 KB (39 KB), `code-review` 18 KB, `ask-matt` 17 KB, `maintain-verification-skill` 17 KB, `idea` 16.5 KB (**82 KB**), `implement-spec` 16 KB, `retro` 13 KB.

## How it grew

| Dates | Commits | What happened |
| --- | --- | --- |
| 09-14 to 09-15 | 88 | Fork, then "spec 2": 24 upstream issues from mattpocock/skills turned into 18 tickets and merged in one day, each with a review-fix pass ([`upstream-evidence.md`][upstream]). Packaging, CI, and releases ("spec 30") were built the same day. |
| 09-16 to 10-04 | 54 | `idea` skill, subagent roles, a prompt audit, `merger` role retired, "review follow-ups are filed" spec, rules moved into shared files, `CONTEXT.md` renamed to `GLOSSARY.md`. |
| **10-17 to 10-18** | **272** | Two-day burst. `idea` grows into a product-strategy tool (market fit, red team, probes, internal-tool track, triage inbox). A "verify the running product" spec adds `create-verification-skill`, `maintain-verification-skill`, the `verifier` role, and proof levels running through `implement` and `implement-spec`. `retro` gets rebuilt. Size budgets and canaries are added to `check-docs`. |

Churn by file (non-merge commits touching it): `implement-spec` 48, `idea/SKILL.md` 46, `GLOSSARY.md` 41, `implement` 39, `idea/MARKET-FIT-FORMAT.md` 33, `implement/SHARED-RULES.md` 31, `idea/FEATURES-FORMAT.md` 25, `ask-matt` 22, `upstream-evidence.md` 20, `engineering/README.md` 19.

Commit types: 193 `fix`, 69 `feat`, 37 `docs`, 11 `refactor`. So **61% of commits are fixes**, and nearly all of them reword skill prose, for example "name the red-team's well reasoned and could not assess lists the same way throughout (#68)". 61 commit subjects say "review". Of 86 commits scoped to `idea`, 72 are fixes. At least 82 commits are marked co-authored by Claude.

## Signs of mess

### 1. A fork that never became its own thing

- Upstream names stayed: [`ask-matt`][askmatt] and [`setup-matt-pocock-skills`][setup] in a repo that replaced Matt's plugin.
- Day two went into porting 24 upstream issue fixes, with a 258-line audit table whose quotes `check-docs` verifies as exact substrings ([`upstream-evidence.md`][upstream], [`check-docs.mjs`][checkdocs] lines 4-6). The repo kept carrying upstream's ideas instead of choosing its own.
- Later imports followed the same pattern: upstream's `pr` and `retro` skills ([`06247f5`][c06247f5]), and two open specs that "close the gaps" with pstack's originals (open issues #174, #175).

### 2. Scope creep into other domains

- `idea` is a product-strategy and market-research workflow (market fit, founder edge, wedge, durability, red team, probes, internal-tool track), with its own repo layout and 6 format files. It holds 82 KB of the repo's markdown ([`idea/SKILL.md`][idea]).
- About 145 of the glossary's 287 lines (the "Product design" and "Features" sections) exist only for `idea` ([`GLOSSARY.md`][glossary]). One glossary serves seven domains: Product design, Features, Planning, Delegation, Review, Verification, Retrospective.
- `teach` (a multi-session tutor) and `to-questionnaire` sit next to the development Workflow but are not part of it.

### 3. A spec process run over prose, enforced by copying

- Skill changes went through spec, locked acceptance criteria, tickets, `implement-spec`, `code-review`, then fix commits. The criteria were about text ("The `implement` skill states that…"), so moving text later needed an ADR to say that was allowed ([`20261017-shipped-criteria-bind-behavior.md`][adr1017]).
- Rules were deliberately repeated across skills and then **guarded by "canaries"**: a check that a 260-character sentence appears word for word in five places ([`check-docs.mjs`][checkdocs], `CANARIES`). That makes duplication a contract instead of removing it.
- Rule homes kept moving: `SHARED-RULES.md`, then `FILING.md`, then `JOURNEY-FORMAT.md` as plain files inside one skill's directory, linked from others, because a user-invoked skill cannot load another one ([`SHARED-RULES.md`][shared] intro). The verifier procedure had already drifted in four places before it was consolidated ([`20261017` ADR][adr1017]).

### 4. Review loops that polish wording

A typical feature PR is one `feat` commit followed by 5 to 10 `fix(...): name X the same way / drop the restated Y / keep Z once` commits from agent review (for example #68: [`c2663cb`][cc2663cb] then 9 fixes). About 41 subjects are about restating, naming or wording. The reviews found inconsistencies that the duplication had created, so effort went into keeping copies consistent instead of making the skills better.

### 5. Size and sprawl measured but never cut

- The repo's own `writing-for-agents` names sprawl as a failure mode ([`writing-for-agents`][wfa] line 43, § Pruning).
- The size budget was set to the largest existing file (`skillBytes: 21020`, "wayfinder today"), and the description budget to the longest existing description ([`08706ea`][c08706ea]). It stops further growth but never drove a cut.
- The compaction work ([#105][i105]) and the eval harness that would make it safe ([#104][i104]) are both open and deferred. No skill change was ever checked against runs, only by reading.

### 6. Duplicate and alias skills

- Three grilling entry points: `grilling`, `grill-me` (one line: call `grilling`), and `grill-with-docs` (one line: call `grilling` and `domain-modeling`) ([`grill-me`][grillme], [`grill-with-docs`][gwd]).
- `ask-matt` is a 17 KB router (plus a 55-line `PHASE-BOUNDARIES.md`) that describes the flow in prose again. It was changed in 22 commits to keep up with the other skills.
- The skill list appears in five places that must agree: `plugin.json`, `engineering/README.md`, `productivity/README.md`, `ask-matt`, and the top-level `README.md`. The `idea` entry in `engineering/README.md` alone is one sentence of about 250 words ([`engineering/README.md`][engreadme]).

### 7. Packaging for several harnesses

- Two install routes (the Claude Code plugin, and `npx skills` for pi) need an explicit skill list, a check that the list matches the `SKILL.md` files, and README sections on duplicate installs, leftover renamed skills, and symlinks ([`README.md`][readme], [install ADR][adrinstall]).
- The ADR says Codex is out of scope, yet 28 `agents/openai.yaml` Codex adapters are still shipped ([install ADR][adrinstall]).
- Splitting skills into `engineering/` and `productivity/` is what made the explicit list necessary, and also caused a trap: a top-level `skills/` directory would ship unlisted skills ([install ADR][adrinstall]).
- Setup templates support GitHub, GitLab and a local tracker (8 commits each), even though only GitHub was used. `docs/agents/issue-tracker.md` and its template in `setup-matt-pocock-skills` have drifted apart.

### 8. Quick reversals and docs out of step

- The `merger` role was added on 09-16 and retired on 09-24 ([`20260924` ADR][adrmerge]). That ADR still says "The plugin ships five roles", but there are six now that `verifier` exists.
- The verification skill path was set by a spec on 10-17 and reversed by an ADR on 10-18 ([`20261018` ADR][adr1018]).
- `CONTEXT.md` was renamed to `GLOSSARY.md` partway through ([`366e1bd`][c366e1bd]).
- A temporary `IMPLEMENTATION_PLAN.md` was committed and then deleted (224 lines of churn).

## Worth keeping

These are lessons and mechanisms, not text to copy. The glossary describes the Predecessor as something studied for lessons, not borrowed from.

- **Repo as its own marketplace, versioned by commit SHA.** No `version` field, so every commit on `main` is an update. Releases are tags only, created by CI from conventional commits, for reading changes and rolling back ([install ADR][adrinstall], [`release.mjs`][release]). This is useful input for the "Versioning and updates" decision.
- **Facts about auto-updates.** Background marketplace pulls disable git credential helpers, so a *private* repo needs SSH. Setting `CLAUDE_CODE_PLUGIN_KEEP_MARKETPLACE_ON_FAILURE=1` keeps the last good copy when a pull fails. `claude --plugin-dir <worktree>` tests a branch for one session ([`README.md`][readme]). The new repo is public, so the SSH point may not apply.
- **Facts about subagents in plugins.** Plugin agents ignore permission mode, hooks and MCP fields. They cannot override built-ins like `Explore`. `isolation: worktree` branches from the default branch, which breaks stacked work. Choose a model per role on purpose ([subagent ADR][adragents]).
- **A small `check-docs`**: relative links resolve, and the manifest matches the skills that exist ([`check-docs.mjs`][checkdocs]). Drop the canaries and the upstream-quote checks.
- **The eval design in [#104][i104]**: fixtures with hidden checks, headless `claude -p --plugin-dir`, two arms (main against the branch), grading from transcripts, self-tested graders. It was never built, and it is the missing piece.
- **Ideas from individual skills**: grilling in numbered rounds with recommended answers ([`grilling`][grilling]); retro's ladder of rules and enforcers; `writing-for-agents` § Pruning (single source of truth, no-ops, sediment); GitHub issue limits (65,536-character body, 100 sub-issues) in the tracker doc.

## Rules that would have prevented it

Input for the Workflow and repo-layout decisions on the map.

1. **Own the Plugin from commit one.** Write skills in your own voice and name them yourself. Credit Inspirations in one file. Don't track or port their issue backlogs.
2. **Keep a scope fence.** The Plugin covers the software-development Workflow. A new domain (product strategy, tutoring) goes in a different plugin or repo, or isn't built. A new skill has to name the Workflow step it serves.
3. **One meaning, one home, and no copies to guard.** When two skills need the same rule, the rule moves into one model-invocable skill or one reference file that both load. Never copy text and add a check to keep the copies equal. If a canary-style check looks necessary, the layout is wrong.
4. **Specs for behaviour, not for wording.** Skill text is changed directly and judged by evals, not by locked "the skill states" criteria and rounds of review. Process tools (spec, tickets, implement-spec) are for code. A skill change is one PR with an eval comparison.
5. **Evals before growth, budgets that bite.** Build the eval harness early. Set size budgets as targets below today's sizes, not at the current maximum. A skill that grows must show it got better.
6. **One flat `skills/` directory, one harness.** Use Claude Code's default layout so no explicit list or sync check is needed. Ship no other harness's adapters until that harness is actually in use.
7. **Few entry points.** No alias skills that only call another skill, no prose router that repeats the skills' own descriptions. Discovery comes from the descriptions plus one generated or short index.
8. **The glossary is for the Plugin's domain only.** Each skill's own terms live in that skill. Keep the root glossary short.
9. **ADRs record decisions that last.** Settle the decision before building on it. A reversal edits or supersedes the old ADR in the same change, so no ADR describes a state that no longer exists.
10. **Slow down.** Avoid 270-commit bursts of agent-on-agent review. Merge a skill change once it passes evals and a human has read it, not once every reviewer nit is fixed.

[tree]: https://github.com/ivanhxlabs/skills/tree/323936c8df51e1d76e5c3bb2d25565e1f0966221
[pluginjson]: https://github.com/ivanhxlabs/skills/blob/323936c8df51e1d76e5c3bb2d25565e1f0966221/.claude-plugin/plugin.json
[agents]: https://github.com/ivanhxlabs/skills/tree/323936c8df51e1d76e5c3bb2d25565e1f0966221/agents
[checkdocs]: https://github.com/ivanhxlabs/skills/blob/323936c8df51e1d76e5c3bb2d25565e1f0966221/scripts/check-docs.mjs
[release]: https://github.com/ivanhxlabs/skills/blob/323936c8df51e1d76e5c3bb2d25565e1f0966221/scripts/release.mjs
[ci]: https://github.com/ivanhxlabs/skills/blob/323936c8df51e1d76e5c3bb2d25565e1f0966221/.github/workflows/ci.yml
[glossary]: https://github.com/ivanhxlabs/skills/blob/323936c8df51e1d76e5c3bb2d25565e1f0966221/GLOSSARY.md
[upstream]: https://github.com/ivanhxlabs/skills/blob/323936c8df51e1d76e5c3bb2d25565e1f0966221/docs/agents/upstream-evidence.md
[readme]: https://github.com/ivanhxlabs/skills/blob/323936c8df51e1d76e5c3bb2d25565e1f0966221/README.md
[engreadme]: https://github.com/ivanhxlabs/skills/blob/323936c8df51e1d76e5c3bb2d25565e1f0966221/engineering/README.md
[askmatt]: https://github.com/ivanhxlabs/skills/blob/323936c8df51e1d76e5c3bb2d25565e1f0966221/engineering/ask-matt/SKILL.md
[setup]: https://github.com/ivanhxlabs/skills/blob/323936c8df51e1d76e5c3bb2d25565e1f0966221/engineering/setup-matt-pocock-skills/SKILL.md
[idea]: https://github.com/ivanhxlabs/skills/blob/323936c8df51e1d76e5c3bb2d25565e1f0966221/engineering/idea/SKILL.md
[shared]: https://github.com/ivanhxlabs/skills/blob/323936c8df51e1d76e5c3bb2d25565e1f0966221/engineering/implement/SHARED-RULES.md
[wfa]: https://github.com/ivanhxlabs/skills/blob/323936c8df51e1d76e5c3bb2d25565e1f0966221/productivity/writing-for-agents/SKILL.md
[grilling]: https://github.com/ivanhxlabs/skills/blob/323936c8df51e1d76e5c3bb2d25565e1f0966221/productivity/grilling/SKILL.md
[grillme]: https://github.com/ivanhxlabs/skills/blob/323936c8df51e1d76e5c3bb2d25565e1f0966221/productivity/grill-me/SKILL.md
[gwd]: https://github.com/ivanhxlabs/skills/blob/323936c8df51e1d76e5c3bb2d25565e1f0966221/engineering/grill-with-docs/SKILL.md
[adrinstall]: https://github.com/ivanhxlabs/skills/blob/323936c8df51e1d76e5c3bb2d25565e1f0966221/docs/adr/20260915-install-from-the-repo.md
[adragents]: https://github.com/ivanhxlabs/skills/blob/323936c8df51e1d76e5c3bb2d25565e1f0966221/docs/adr/20260916-subagent-roles-ship-in-the-plugin.md
[adrmerge]: https://github.com/ivanhxlabs/skills/blob/323936c8df51e1d76e5c3bb2d25565e1f0966221/docs/adr/20260924-merge-in-the-parent.md
[adr1017]: https://github.com/ivanhxlabs/skills/blob/323936c8df51e1d76e5c3bb2d25565e1f0966221/docs/adr/20261017-shipped-criteria-bind-behavior.md
[adr1018]: https://github.com/ivanhxlabs/skills/blob/323936c8df51e1d76e5c3bb2d25565e1f0966221/docs/adr/20261018-verification-works-in-any-harness.md
[c06247f5]: https://github.com/ivanhxlabs/skills/commit/06247f5
[c08706ea]: https://github.com/ivanhxlabs/skills/commit/08706ea
[c366e1bd]: https://github.com/ivanhxlabs/skills/commit/366e1bd
[cc2663cb]: https://github.com/ivanhxlabs/skills/commit/c2663cb
[i104]: https://github.com/ivanhxlabs/skills/issues/104
[i105]: https://github.com/ivanhxlabs/skills/issues/105
