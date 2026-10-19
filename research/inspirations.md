# Inspirations compared: Workflows, skill inventories, packaging, token techniques, licences

Research for [Compare the Inspirations' workflows, skill inventories and licences](https://github.com/iamivanhx/skills/issues/2), a child of the map in [issue #1](https://github.com/iamivanhx/skills/issues/1).

Everything below comes from the repos' own files (SKILL.md files, manifests, hook scripts, licences), read at these commits on 2026-10-19:

| Inspiration | Repo | Commit read | Version |
| --- | --- | --- | --- |
| mattpocock/skills | [mattpocock/skills](https://github.com/mattpocock/skills) | [`49dd158`](https://github.com/mattpocock/skills/tree/49dd158d1076134a641b33efb035946536778336) (2026-10-09) | `mattpocock-skills` 1.3.1 |
| pstack | [cursor/plugins/pstack](https://github.com/cursor/plugins/tree/main/pstack) | [`d457567`](https://github.com/cursor/plugins/tree/d457567332aa6989e753ea289f30ea2bfbc80bc0/pstack) (2026-10-19) | `pstack` 0.15.28 |
| ponytail | [DietrichGebert/ponytail](https://github.com/DietrichGebert/ponytail) | [`09c434a`](https://github.com/DietrichGebert/ponytail/tree/09c434ab73ca43761c5d8b0c5d341926c2fba710) (2026-10-18) | `ponytail` 5.3.2 |

Link shorthands used below: **MP** = `https://github.com/mattpocock/skills/blob/49dd158d1076134a641b33efb035946536778336/`, **PS** = `https://github.com/cursor/plugins/blob/d457567332aa6989e753ea289f30ea2bfbc80bc0/`, **PT** = `https://github.com/DietrichGebert/ponytail/blob/09c434ab73ca43761c5d8b0c5d341926c2fba710/`. Citations name files relative to those roots.

---

## 1. mattpocock/skills

### Workflow

The router skill `ask-matt` spells out the whole Workflow ([MP `skills/engineering/ask-matt/SKILL.md`]). It has one **main flow** and three **on-ramps**:

| Stage | Skill | Hands off | Artifact it leaves |
| --- | --- | --- | --- |
| 0. Precondition, once per repo | `/setup-matt-pocock-skills` | (all engineering skills read its output) | `docs/agents/issue-tracker.md`, `triage-labels.md`, `domain.md`, plus an `## Agent skills` block in CLAUDE.md/AGENTS.md |
| 1. Sharpen the idea | `/grill-with-docs` (= `grilling` + `domain-modeling`) | to 2 or 3 | `GLOSSARY.md`, ADRs in `docs/adr/` |
| 2. Optional detour | `/handoff` out, then `/prototype`, then `/handoff` back | back to 1 | handoff file in `$TMPDIR`; prototype kept on a `prototype/<name>` branch |
| 3a. Multi-session build | `/to-spec`, then `/to-tickets` | to `/implement` per ticket (with `/clear` between tickets) or `/implement-spec` | spec issue (`ready-for-agent`); one ticket per vertical slice, with native blocking links |
| 3b. Build | `/implement` (drives `tdd`, ends with `code-review`, commits) or `/implement-spec` (task graph, parallel implementer subagents in worktrees, one integration branch) | to `pr` when a PR goes up | commits, integration branch, PR body |
| 4. Close the loop | `/retro` | (changes to the environment) | suggested checks, coding standards, steering-file edits |
| On-ramp: incoming issues | `/triage` | to `/implement` | agent-ready issues with triage labels |
| On-ramp: hard bug | `diagnosing-bugs` | to `/retro`, maybe `/improve-codebase-architecture` | regression test plus fix |
| On-ramp: huge, foggy effort | `/wayfinder` | to `/to-spec` | a map issue plus decision tickets (`wayfinder:research/prototype/grilling/task`), research findings on `research/<name>` branches |

Context hygiene is part of the flow: keep steps 1 to 3 in one context window, then start each `/implement` fresh. `PHASE-BOUNDARIES.md` gives an ordered tree for each phase boundary: Continue, then `/clear`, then `/handoff`, then a subagent, then `/compact` as the last resort ([MP `skills/engineering/ask-matt/PHASE-BOUNDARIES.md`]).

### Skill inventory and triggering

The Plugin ships 27 skills: the **promoted** buckets `engineering/` and `productivity/`, listed one by one in `plugin.json` ([MP `.claude-plugin/plugin.json`]). The buckets `misc/`, `in-progress/` and `deprecated/` stay out of the Plugin.

The repo splits skills on one axis, **invocation** ([MP `.agents/invocation.md`]):

- **User-invoked** (`disable-model-invocation: true`, with a short description written for a person): `ask-matt`, `grill-with-docs`, `triage`, `improve-codebase-architecture`, `setup-matt-pocock-skills`, `to-spec`, `to-tickets`, `implement`, `implement-spec`, `wayfinder`, `retro`, `grill-me`, `handoff`, `teach`, `to-questionnaire`, `wait-what`. These **orchestrate**.
- **Model-invoked** (description written for the model, with trigger phrases): `prototype`, `diagnosing-bugs`, `research`, `tdd`, `domain-modeling`, `codebase-design`, `code-review`, `pr`, `wizard`, `grilling`, `writing-for-agents`. These hold the **reusable discipline**.
- **Chaining rule**: a skill calls another with an explicit instruction, "Call the Skill tool with "grilling"", never a `../other/FILE.md` link and never a bare `/name`. A user-invoked skill may call model-invoked skills, but no skill can reach a user-invoked one. When the dependency is user-invoked, the skill tells the human to run it (for example, "tell the user to run `/setup-matt-pocock-skills`").
- Wrapper skills can be tiny. `grill-with-docs` has one line in its body: "Call the Skill tool twice, for "grilling" and "domain-modeling"" ([MP `skills/engineering/grill-with-docs/SKILL.md`]).
- ADR 0001 separates **hard** dependencies on the setup config (`to-spec`, `to-tickets`, `triage` name the setup skill) from **soft** ones (`tdd`, `diagnose` refer to "the project's glossary" in vague prose), so that soft skills stay light ([MP `.agents/adr/0001-explicit-setup-pointer-only-for-hard-dependencies.md`]).

### Repo structure and packaging

- `.claude-plugin/plugin.json` lists each promoted skill path in a `skills` array. `.claude-plugin/marketplace.json` makes the repo its own one-plugin marketplace (`mattpocock`). Claude Code users install through `claude-plugins-official`, which **pins a commit sha**, so a release reaches users only when that pin moves ([MP `.agents/adr/0002-ship-as-a-claude-code-plugin.md`]).
- Namespace: the plugin is `mattpocock-skills`, and skills show as `mattpocock-skills:<name>`.
- Versioning: changesets plus `scripts/sync-plugin-version.mjs`, which copies the `package.json` version into `plugin.json` (it has a `--check` mode for CI) ([MP `package.json`], [MP `scripts/sync-plugin-version.mjs`]).
- Each skill has an `agents/openai.yaml` beside it for Codex (display name, plus `allow_implicit_invocation: false` for user-invoked skills). The same tree serves Codex and Copilot.
- Shared reference lives **inside the skill that owns it** (`domain-modeling/ADR-FORMAT.md`, `GLOSSARY-FORMAT.md`; `codebase-design/DEEPENING.md`). Other skills reach it by calling that skill, not by a link across folders ([MP `.agents/invocation.md`]).
- Repo governance: `CLAUDE.md` holds the invariants (each promoted skill appears in the README, in `plugin.json`, in a docs page and in the `ask-matt` router; no em-dashes). `SCOPE.md` plus `.out-of-scope/*.md` record rejected ideas, and no new skills are accepted ([MP `CLAUDE.md`], [MP `SCOPE.md`], [MP `.out-of-scope/new-skills.md`]).
- Dev loop: `scripts/link-skills.sh` symlinks the skills into `~/.claude/skills`, so a `git pull` updates them.
- Plugin components: **skills only**. The repo ships no hooks, agents, output styles or MCP servers. A hook script exists only in the unshipped `misc/git-guardrails-claude-code`.

### Context and token techniques

- **User-invoked by default.** A user-invoked skill has "zero context load" because its description never enters the model's context. The cost moves to the human, who must remember the skill exists, and the `ask-matt` router reduces that cost ([MP `skills/productivity/writing-for-agents/SKILL-MECHANICS.md`]).
- **Progressive disclosure** through sibling files reached by pointers (`PHASE-BOUNDARIES.md`, `tdd/tests.md`, `tdd/mocking.md`, `prototype/LOGIC.md`/`UI.md`, `triage/AGENT-BRIEF.md`). The rule: "inline what every branch needs, and push behind a pointer what only some branches reach" ([MP `skills/productivity/writing-for-agents/SKILL.md`]).
- **Leading words** (_tracer bullet_, _frontier_, _fog of war_, _red_, _seam_) that recruit what the model already knows, which costs fewer tokens. Also no-op pruning, a single source of truth, and stating rules positively instead of as negations (same file).
- **Subagents**: `code-review` runs its Standards and Spec axes as parallel subagents. `research` runs as a background agent. `implement-spec` talks to its subagents through **context pointers** (spec, tickets, notes), not pasted content. `grilling` sends fact-finding to subagents ([MP `skills/engineering/implement-spec/SKILL.md`], [MP `skills/productivity/grilling/SKILL.md`]).
- **Ubiquitous language** in `GLOSSARY.md` gives shorter replies and less thinking, a selling point in the README ([MP `README.md`]).
- **Evals**: none. "There is no automated eval. You check by running the document by hand" ([MP `docs/productivity/writing-for-agents.md`]).

### Licence

MIT, "Copyright (c) 2026 Matt Pocock" ([MP `LICENSE`]). The `pr` skill also contains text copied "almost word for word" from Dex Horthy's `show-me` skill (Humanlayer). It credits this in `pr/CREDITS.md` and in a `metadata.credits` frontmatter block ([MP `skills/engineering/pr/CREDITS.md`]). If you copy `pr`, that upstream text carries its own licence terms too.

---

## 2. pstack (cursor/plugins)

### Workflow

pstack has no tracker-driven pipeline. It has **one front door**, `/poteto-mode`, which sorts each task into one of **23 playbooks** and copies that playbook's steps verbatim into a todo list ([PS `pstack/skills/poteto-mode/SKILL.md`]). The guide orders the stages as follows ([PS `pstack/docs/guide/README.md`]):

| Stage | Skills / playbooks | Artifacts |
| --- | --- | --- |
| Setup | `/setup-pstack` | an always-on rule `pstack-models` mapping roles to model and effort (`~/.claude/rules/pstack-models.md` in Claude Code) |
| Understand | Investigation playbook, `how`, `why`, `teach`, `recall` | cited explanations |
| Design | `architect` (sketch types and signatures, design it twice through `arena`), `arena`, `swarm`, `interrogate`, Prototype playbook | rationale doc, chosen sketch |
| Build and clean | Feature / Bug fix / Refactoring / Perf playbooks; `tdd`, `unslop`, `no-comments` (spawns the `comment-sicko` agent) | "throughput checkpoint" todos; code from `poteto-agent` delegates |
| Verify and ship | `create-verification-skill` (generates a project-local control skill), formal verification skills, `benchmark-checklist`, Opening a PR, Babysit, Shipping | PRs (stacked, Conventional Commits, fixed body sections), verified stack |
| Overnight | Autonomous run, Orchestrate, Autopilot, `show-me-your-work` | TSV decision log |
| Make it yours | `automate-me` (mines transcripts into a personal `-mode` skill), `reflect` (3 reviewer subagents, then a synthesizer, then skill edits), `correct` (make repeated mistakes impossible: architecture, then types, then lint, then test, then docs) | edits to skills and checks |

Every playbook ends in "Opening a PR" ([PS `pstack/skills/poteto-mode/playbooks/opening-a-pr.md`]). The Feature playbook shows the shape: `how`, then `architect`, then a four-part throughput checkpoint, then a delegated implementation (through `arena` when several shapes are valid), then verification on the real surface, then small ordered commits, `interrogate` if contested, and a PR ([PS `pstack/skills/poteto-mode/playbooks/feature.md`]).

### Skill inventory and triggering

56 skills ([PS `pstack/skills/`]):

- **1 router/mode**: `poteto-mode` (149 lines, about 20 KB), with 23 playbooks in `playbooks/*.md`, and `poteto-help` (Q&A over pstack, does not start work).
- **24 `principle-*` leaf skills** (9 to 34 lines each): laziness protocol, model the domain, prove it works, guard the context window, and so on. `poteto-mode` keeps a one-line index of them and says "Read the leaf skill in full for any principle you apply."
- **30 workflow and utility skills**: `architect`, `arena`, `swarm`, `interrogate`, `how`, `why`, `teach`, `recall`, `reflect`, `correct`, `automate-me`, `figure-it-out`, `show-me-your-work`, `tdd`, `unslop`, `no-comments`, `technical-writing`, `blast-radius`, `benchmark-checklist`, `bro`, `ketchup`, `sharpen`, `in-your-own-words`, `typescript-best-practices`, verification skill generators and maintainers, `make-bot-ui`, `setup-pstack`.
- **2 subagent definitions**: `poteto-agent` (a delegate that reads `poteto-mode` in full before it starts work) and `comment-sicko` ([PS `pstack/agents/`]).
- **Triggering**: **55 of 56 skills set `disable-model-invocation: true`**. Only `setup-pstack` can be invoked by the model. Even so, descriptions keep trigger phrases ("Use for /architect, 'architect this'..."). Skills chain by **naming** another skill ("the **how** skill") or by a relative path (`../<name>/SKILL.md`). In Claude Code, the adapter says the Skill tool refuses most pstack skills, so the agent must **`Read` the SKILL.md and follow it** ([PS `pstack/docs/harness/claude-code.md`]).
- One skill uses `paths: ["**/*.ts", "**/*.tsx"]` frontmatter (`typescript-best-practices`).
- There is a cross-plugin dependency: `poteto-mode` sends work to `deslop`, `control-cli` and `control-ui` from the separate `cursor-team-kit` plugin.

### Repo structure and packaging

- pstack is written **for Cursor first**: `pstack/.cursor-plugin/plugin.json` (with `"skills": "./skills/"` and `"agents": "./agents/"`) and an entry in the monorepo's `.cursor-plugin/marketplace.json`.
- **Claude Code gets a generated tree**: `harness/pstack/claude-code/`, built by `scripts/generate-pstack-harness-trees.mjs` (CI fails if it is stale). Each generated SKILL.md gains one inserted line: "In Claude Code, read the Claude Code adapter once per session" ([PS `harness/pstack/README.md`]). The repo-level `.claude-plugin/marketplace.json` points `pstack` at that tree. Install: `/plugin marketplace add cursor/plugins` then `/plugin install pstack@cursor-plugins`. Namespace: `/pstack:<name>`, agents `pstack:poteto-agent`.
- The **adapter doc** (`docs/harness/claude-code.md`) is a mapping table from Cursor to Claude Code: `Task` becomes `Agent`, `readonly: true` becomes `Explore`, `environment: "cloud"` becomes `isolation: "worktree"`, model slugs become `model` + `effort`, `.mdc` rules become `~/.claude/rules/`. Cursor-only steps are skipped with a single line.
- Versioning: a `version` field in both manifests (0.15.28). The repo has no changelog tooling.
- Shared reference: per-skill `references/` folders (reviewer prompts, rubrics, templates), `playbooks/`, and `scripts/` (TypeScript tools such as `watch-pr`, `orch`, `check-plan`, each with tests).

### Context and token techniques

- **Nearly every skill is user-invoked**, so 55 descriptions stay out of the model's context. The single router loads leaf skills and playbooks **by path, on demand**. This is progressive disclosure at the scale of the whole Plugin.
- **Playbooks as disclosed files**: `poteto-mode` keeps only a one-line index entry per playbook and principle.
- **Heavy subagent delegation** with fixed defaults: background subagents, "file pointers not inlined context", **fresh subagents by default** (resuming one "silently drops directives"), and the parent reviews the diff and writes its own summary ([PS `pstack/skills/poteto-mode/SKILL.md`, "Subagents"]).
- **Model tiering per role**: a fast model for mechanical code, the strongest model for the hardest changes and for judgment, with effort turned up or down per job. Multi-model panels (`arena`, `interrogate`, `reflect`) spread judgment across different models.
- `principle-guard-the-context-window`: "Route bulk to subagents; keep summaries in the main thread, not raw payloads."
- **Evals**: an Eval playbook. Blinded candidates get an organic prompt (no words like `eval`, `test` or `judge` that would reveal the test), run on several models, and a judge from a different model family scores them against a hidden rubric. Whether a candidate followed the skill chain is graded from **transcripts**, not from what the candidate claims ([PS `pstack/skills/poteto-mode/playbooks/eval.md`]).

### Licence

MIT, "Copyright (c) 2026 Lauren Tan", in `pstack/LICENSE`, and the same file is copied into the generated harness tree. **The cursor/plugins repo itself has no root licence** (GitHub reports none), so only the `pstack/` folder is clearly MIT. Other plugins in that monorepo (such as `cursor-team-kit`, which pstack depends on) need their own check before you borrow from them.

---

## 3. ponytail

### Workflow

ponytail is not a Workflow. It is an **always-on behaviour mode** plus a few one-shot report commands. The README puts it this way: "Ponytail is one prompt" ([PT `README.md`]). Its "workflow" is the **ladder** inside every coding task: read the task and everything the change touches, then take the first rung that fully works (does it need to exist? is it already in the codebase? stdlib or platform? an installed dependency? one line? otherwise the minimum code), leave a small test for risky logic, and end each reply with what was skipped and the risks ([PT `skills/ponytail/SKILL.md`]). The hand-offs are to the user, through `/ponytail-review` (a diff), `/ponytail-audit` (the whole repo) and `/ponytail-debt` (collects `shortcut:` comments into a ledger).

### Skill inventory and triggering

6 skills, all **model-invoked**, with long trigger-rich descriptions ([PT `skills/`]):

| Skill | Owns | Trigger |
| --- | --- | --- |
| `ponytail` | the ruleset; levels `lite`/`full`/`ultra`/`off` (`argument-hint`) | any coding task, "be lazy", "yagni"; **also injected by hooks** |
| `ponytail-review` | review of a change: correct, safe, load, tested, fast, lean; reads callers outside the diff | "review this", "review my PR" |
| `ponytail-audit` | ranked audit of the whole repo, changes nothing | "audit this codebase", "find bloat" |
| `ponytail-debt` | ledger of `shortcut:` comments | "what did ponytail defer" |
| `ponytail-gain` | shows the benchmark scoreboard | "ponytail impact" |
| `ponytail-help` | quick reference | "ponytail help" |

### Repo structure and packaging

This is the richest Plugin of the three. It uses **every component type except agents and MCP servers** ([PT `.claude-plugin/plugin.json`]):

- **Hooks** (`hooks/claude-codex-hooks.json`):
  - `SessionStart` (startup/resume/clear/compact) injects the ruleset, filtered to the active level, as hidden context. It adds a **codebase map** of top-level functions and exports, capped at 2,000 characters, built by `git ls-files` plus regexes, so "reuse first" needs no search. It also nudges the user once to set up the statusline. After a compaction it keeps the current level ([PT `hooks/ponytail-activate.js`], [PT `hooks/ponytail-map.js`]).
  - `SubagentStart` injects the same rules into subagents, which SessionStart context never reaches. `PONYTAIL_SUBAGENT_MATCHER` limits this to some agent types ([PT `hooks/ponytail-subagent.js`]).
  - `UserPromptSubmit` watches for `/ponytail <level>` and records the mode in a flag file ([PT `hooks/ponytail-mode-tracker.js`]).
- **Statusline** script (`ponytail-statusline.sh`/`.ps1`). The hook copies it into `~/.claude/` because the versioned plugin cache directory is deleted on update.
- **A Claude Code UI module**, the "band": `plugin.json` names a second hooks file, `hooks/claude-band.json`, whose `"modules": ["./band/register.tsx"]` imports from `'claude-code'`. It shows a live cost, token, reuse and test band above the prompt, and a local board page. This needs Claude Code 2.1.290 or later and the terminal or desktop app ([PT `hooks/band/register.tsx`], [PT `README.md`]).
- **`userConfig`** in `plugin.json`: plugin settings, including a `sensitive` webhook token kept in the keychain.
- **Marketplace**: the repo is its own marketplace `ponytail` (`/plugin install ponytail@ponytail`). Namespace: `ponytail:<name>`.
- **Many hosts from one source**: `skills/ponytail/SKILL.md` is canonical. `AGENTS.md` is the compact copy, and seven rule-file copies (Cursor, Windsurf, Cline, Copilot, Kiro, Qoder, `.agents`) are **checked by a script** to stay identical (`scripts/check-rule-copies.js`). The repo also has manifests for Codex, Devin, Grok, Kimi, Gemini (`commands/*.toml`), OpenCode, pi and Qoder ([PT `docs/agent-portability.md`]).
- Versioning: semver in `package.json` and the manifests, `scripts/check-versions.js`, and an npm publish workflow.

### Context and token techniques

- The whole product is a **token-saving behaviour**. The measured result is -45% output tokens, -26% cost, -41% time and -53% lines of code against the same agent with no skill, while 98% of risky logic ships with a test (68% without the skill). The numbers come from 39 tasks, Opus 5.5 and 5 runs each ([PT `benchmarks/README.md`], [PT `benchmarks/results/2026-10-07-agentic.md`]).
- **Level filtering**: the hook sends only the table rows and examples for the active level ([PT `hooks/ponytail-instructions.js`]).
- **Codebase map** under a fixed character budget, so the agent does not need exploratory searches.
- The ruleset is about 50 lines. The trade-off: it is **always on**, so it costs context in every session and every subagent.
- **Evals**: the most serious method of the three. A promptfoo single-shot suite compares no skill, caveman and ponytail. An **agentic benchmark** runs a headless Claude Code session per task on a seeded repo and scores the files it leaves behind: diff lines, plus hidden safety checks that run adversarial input. Its arms include `yagni` one-liner prompts as a control. There are also `node --test` unit tests that prove the graders can tell good output from bad ([PT `benchmarks/agentic/README.md`], [PT `tests/behavior.test.js`]).

### Licence

MIT, "Copyright (c) 2026 DietrichGebert" ([PT `LICENSE`]). The `ponytail` SKILL.md also says `license: MIT` in its frontmatter.

---

## 4. What MIT allows (all three)

All three Inspirations are MIT (pstack only for its `pstack/` folder). MIT lets us **use, copy, modify, merge, publish, distribute, sublicense** the material. The one condition: "The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software."

In practice:

- **Ideas, structure and vocabulary** (a router skill, an invocation split, playbooks, hooks that inject rules, the eval method) are not covered by copyright. We can borrow them freely and credit them as Inspirations.
- **Copying or closely adapting text** (a SKILL.md body, a hook script) needs that Inspiration's copyright line and the MIT notice kept with it, for example in a `CREDITS.md` or `THIRD_PARTY_NOTICES` file. mattpocock's `pr/CREDITS.md` plus `metadata.credits` frontmatter is a good pattern to follow.
- **Watch for nested sources**: mattpocock's `pr` skill contains Humanlayer text, and pstack's routes point at `cursor-team-kit`, which is not covered by pstack's licence.

---

## 5. Side by side

| | mattpocock/skills | pstack | ponytail |
| --- | --- | --- | --- |
| What it is | a Workflow that runs through an issue tracker | an engineering style plus a task router | an always-on minimalism mode |
| Front door | `ask-matt` router (hints only); you type each stage | `/poteto-mode` (does the routing itself) | none needed; hooks turn it on |
| Workflow shape | idea, grill, spec, tickets, implement (TDD), review, PR, retro; wayfinder for foggy efforts | task type, then playbook, then todos, then delegated build, verify, PR, babysit, ship | ladder within each task; review/audit on demand |
| Durable artifacts | GLOSSARY.md, ADRs, spec and ticket issues, research branches | PRs, decision-log TSV, generated verification skills | `shortcut:` comments, a task ledger CSV |
| Skills | 27 shipped | 56 | 6 |
| Invocation | deliberate split: 16 user-invoked orchestrators, 11 model-invoked disciplines | 55 of 56 user-invoked; the router reads files by path | all 6 model-invoked, plus hook injection |
| Chaining | "Call the Skill tool with X" | name the skill or `Read ../<name>/SKILL.md` | n/a |
| Components beyond skills | none | 2 subagents | hooks (3 events), statusline, UI module, userConfig |
| Claude Code packaging | native; own marketplace plus official marketplace (sha-pinned) | generated tree plus Cursor-to-Claude adapter doc | native; own marketplace |
| Versioning | changesets plus version-sync script | manual version field | semver plus version-check script plus npm |
| Token techniques | user-invoked by default, disclosure, leading words, subagents, context pointers, glossary | router plus on-demand leaves, background fresh subagents, model tiering | less output by design, level filtering, budgeted codebase map |
| Evals | none (manual) | blinded multi-model eval playbook | headless Claude Code benchmark with hidden checks, plus grader unit tests |
| Tracker | GitHub, GitLab or local, through setup | none (PRs only) | none |
| Licence | MIT | MIT (`pstack/` only) | MIT |

### Where they agree

- **Verify against reality**: TDD at agreed seams (mattpocock), "prove it works" on the real surface (pstack), "lazy code without its check is unfinished" (ponytail).
- **Less code, deeper modules**: mattpocock's deep modules and `codebase-design`; pstack's laziness protocol and subtract-before-you-add; ponytail's whole ladder.
- **Subagents for isolation**: review axes or reviewers run in separate contexts in both mattpocock and pstack, and ponytail goes as far as injecting its rules into subagents.
- **Encode lessons in the environment, not in more prose**: mattpocock `/retro` ("mechanical mistakes become deterministic checks"), pstack `/correct` and `principle-encode-lessons-in-structure`.
- **One skill as the source of truth for each concept**, reached by name or path, never duplicated (ponytail even enforces this with a script).

### Where they differ

- **Who drives**: in mattpocock the human types each stage and the human is the index. pstack asks for one command and the agent picks the playbook. ponytail needs no command at all.
- **Planning versus doing**: mattpocock invests heavily before code (grilling, glossary, spec, tickets, wayfinder). pstack invests in design exploration and verification, and plans through todos and PRs rather than a tracker. ponytail scopes each task on the spot.
- **How the user-invoked flag is used**: mattpocock treats it as a hard wall (no skill can call a user-invoked one). pstack uses it only to keep descriptions out of context, then gets around the wall by reading files by path.
- **Components**: mattpocock and pstack are almost all prose. ponytail is mostly code (hooks, map, statusline, UI).
- **Evals**: from none, to a judged procedure, to a reproducible benchmark.

### Ideas most worth borrowing

1. **The invocation split and the "Call the Skill tool with X" convention** (mattpocock). This gives orchestrators zero context load and keeps disciplines discoverable. Add pstack's **read-by-path** fallback for reference that only user-invoked skills need.
2. **One router skill** (`ask-matt` style) as the human's index, kept accurate by a rule in CLAUDE.md.
3. **The mattpocock Workflow spine** (grill with glossary and ADRs, spec, tickets, implement with TDD, two-axis review, PR, retro) plus **wayfinder** for foggy efforts. It matches the "GitHub Issues only" preference directly.
4. **pstack's subagent defaults**: fresh by default, context pointers instead of pasted content, the parent reviews the diff, and model and effort chosen per role. These map cleanly onto Claude Code's `Agent` `model`/`effort`/`isolation`.
5. **pstack's playbook pattern**: a one-line index in the router and full steps in disclosed files copied into the todo list. Worth it if the Workflow grows task types beyond feature work (bug fix, perf, refactor).
6. **ponytail's hooks**: `SessionStart` (including `compact`) to restore state, and `SubagentStart` to give subagents the standing rules, since SessionStart context never reaches them. Also a statusline copied out of the versioned cache dir, and the budgeted **codebase map**. These are the strongest leads for the "Plugin components beyond skills" ticket.
7. **ponytail's eval method**: headless `claude -p` sessions on a seeded repo, with and without the skill, hidden deterministic checks, and a trivial-prompt control arm. Combine it with pstack's **blinding rules and transcript-based chain grading**. This answers "skill evals matter" better than mattpocock's manual approach.
8. **Repo hygiene that keeps a repo from getting messy** (relevant to the Predecessor lesson): mattpocock's invariants in CLAUDE.md, `SCOPE.md` with `.out-of-scope/`, a version-sync script, and ponytail's copy-check script. Decisions get written down and drift gets checked by a script.
9. **Credits pattern** for anything copied: a per-skill `CREDITS.md` plus `metadata.credits` frontmatter, and a top-level notice file listing the copyright lines.

[MP `skills/engineering/ask-matt/SKILL.md`]: https://github.com/mattpocock/skills/blob/49dd158d1076134a641b33efb035946536778336/skills/engineering/ask-matt/SKILL.md
[MP `skills/engineering/ask-matt/PHASE-BOUNDARIES.md`]: https://github.com/mattpocock/skills/blob/49dd158d1076134a641b33efb035946536778336/skills/engineering/ask-matt/PHASE-BOUNDARIES.md
[MP `.claude-plugin/plugin.json`]: https://github.com/mattpocock/skills/blob/49dd158d1076134a641b33efb035946536778336/.claude-plugin/plugin.json
[MP `.agents/invocation.md`]: https://github.com/mattpocock/skills/blob/49dd158d1076134a641b33efb035946536778336/.agents/invocation.md
[MP `skills/engineering/grill-with-docs/SKILL.md`]: https://github.com/mattpocock/skills/blob/49dd158d1076134a641b33efb035946536778336/skills/engineering/grill-with-docs/SKILL.md
[MP `.agents/adr/0001-explicit-setup-pointer-only-for-hard-dependencies.md`]: https://github.com/mattpocock/skills/blob/49dd158d1076134a641b33efb035946536778336/.agents/adr/0001-explicit-setup-pointer-only-for-hard-dependencies.md
[MP `.agents/adr/0002-ship-as-a-claude-code-plugin.md`]: https://github.com/mattpocock/skills/blob/49dd158d1076134a641b33efb035946536778336/.agents/adr/0002-ship-as-a-claude-code-plugin.md
[MP `package.json`]: https://github.com/mattpocock/skills/blob/49dd158d1076134a641b33efb035946536778336/package.json
[MP `scripts/sync-plugin-version.mjs`]: https://github.com/mattpocock/skills/blob/49dd158d1076134a641b33efb035946536778336/scripts/sync-plugin-version.mjs
[MP `CLAUDE.md`]: https://github.com/mattpocock/skills/blob/49dd158d1076134a641b33efb035946536778336/CLAUDE.md
[MP `SCOPE.md`]: https://github.com/mattpocock/skills/blob/49dd158d1076134a641b33efb035946536778336/SCOPE.md
[MP `.out-of-scope/new-skills.md`]: https://github.com/mattpocock/skills/blob/49dd158d1076134a641b33efb035946536778336/.out-of-scope/new-skills.md
[MP `skills/productivity/writing-for-agents/SKILL-MECHANICS.md`]: https://github.com/mattpocock/skills/blob/49dd158d1076134a641b33efb035946536778336/skills/productivity/writing-for-agents/SKILL-MECHANICS.md
[MP `skills/productivity/writing-for-agents/SKILL.md`]: https://github.com/mattpocock/skills/blob/49dd158d1076134a641b33efb035946536778336/skills/productivity/writing-for-agents/SKILL.md
[MP `skills/engineering/implement-spec/SKILL.md`]: https://github.com/mattpocock/skills/blob/49dd158d1076134a641b33efb035946536778336/skills/engineering/implement-spec/SKILL.md
[MP `skills/productivity/grilling/SKILL.md`]: https://github.com/mattpocock/skills/blob/49dd158d1076134a641b33efb035946536778336/skills/productivity/grilling/SKILL.md
[MP `README.md`]: https://github.com/mattpocock/skills/blob/49dd158d1076134a641b33efb035946536778336/README.md
[MP `docs/productivity/writing-for-agents.md`]: https://github.com/mattpocock/skills/blob/49dd158d1076134a641b33efb035946536778336/docs/productivity/writing-for-agents.md
[MP `LICENSE`]: https://github.com/mattpocock/skills/blob/49dd158d1076134a641b33efb035946536778336/LICENSE
[MP `skills/engineering/pr/CREDITS.md`]: https://github.com/mattpocock/skills/blob/49dd158d1076134a641b33efb035946536778336/skills/engineering/pr/CREDITS.md
[PS `pstack/skills/poteto-mode/SKILL.md`]: https://github.com/cursor/plugins/blob/d457567332aa6989e753ea289f30ea2bfbc80bc0/pstack/skills/poteto-mode/SKILL.md
[PS `pstack/skills/poteto-mode/SKILL.md`, "Subagents"]: https://github.com/cursor/plugins/blob/d457567332aa6989e753ea289f30ea2bfbc80bc0/pstack/skills/poteto-mode/SKILL.md#subagents
[PS `pstack/docs/guide/README.md`]: https://github.com/cursor/plugins/blob/d457567332aa6989e753ea289f30ea2bfbc80bc0/pstack/docs/guide/README.md
[PS `pstack/skills/poteto-mode/playbooks/opening-a-pr.md`]: https://github.com/cursor/plugins/blob/d457567332aa6989e753ea289f30ea2bfbc80bc0/pstack/skills/poteto-mode/playbooks/opening-a-pr.md
[PS `pstack/skills/poteto-mode/playbooks/feature.md`]: https://github.com/cursor/plugins/blob/d457567332aa6989e753ea289f30ea2bfbc80bc0/pstack/skills/poteto-mode/playbooks/feature.md
[PS `pstack/skills/`]: https://github.com/cursor/plugins/tree/d457567332aa6989e753ea289f30ea2bfbc80bc0/pstack/skills
[PS `pstack/agents/`]: https://github.com/cursor/plugins/tree/d457567332aa6989e753ea289f30ea2bfbc80bc0/pstack/agents
[PS `pstack/docs/harness/claude-code.md`]: https://github.com/cursor/plugins/blob/d457567332aa6989e753ea289f30ea2bfbc80bc0/pstack/docs/harness/claude-code.md
[PS `harness/pstack/README.md`]: https://github.com/cursor/plugins/blob/d457567332aa6989e753ea289f30ea2bfbc80bc0/harness/pstack/README.md
[PS `pstack/skills/poteto-mode/playbooks/eval.md`]: https://github.com/cursor/plugins/blob/d457567332aa6989e753ea289f30ea2bfbc80bc0/pstack/skills/poteto-mode/playbooks/eval.md
[PT `README.md`]: https://github.com/DietrichGebert/ponytail/blob/09c434ab73ca43761c5d8b0c5d341926c2fba710/README.md
[PT `skills/ponytail/SKILL.md`]: https://github.com/DietrichGebert/ponytail/blob/09c434ab73ca43761c5d8b0c5d341926c2fba710/skills/ponytail/SKILL.md
[PT `skills/`]: https://github.com/DietrichGebert/ponytail/tree/09c434ab73ca43761c5d8b0c5d341926c2fba710/skills
[PT `.claude-plugin/plugin.json`]: https://github.com/DietrichGebert/ponytail/blob/09c434ab73ca43761c5d8b0c5d341926c2fba710/.claude-plugin/plugin.json
[PT `hooks/ponytail-activate.js`]: https://github.com/DietrichGebert/ponytail/blob/09c434ab73ca43761c5d8b0c5d341926c2fba710/hooks/ponytail-activate.js
[PT `hooks/ponytail-map.js`]: https://github.com/DietrichGebert/ponytail/blob/09c434ab73ca43761c5d8b0c5d341926c2fba710/hooks/ponytail-map.js
[PT `hooks/ponytail-subagent.js`]: https://github.com/DietrichGebert/ponytail/blob/09c434ab73ca43761c5d8b0c5d341926c2fba710/hooks/ponytail-subagent.js
[PT `hooks/ponytail-mode-tracker.js`]: https://github.com/DietrichGebert/ponytail/blob/09c434ab73ca43761c5d8b0c5d341926c2fba710/hooks/ponytail-mode-tracker.js
[PT `hooks/band/register.tsx`]: https://github.com/DietrichGebert/ponytail/blob/09c434ab73ca43761c5d8b0c5d341926c2fba710/hooks/band/register.tsx
[PT `docs/agent-portability.md`]: https://github.com/DietrichGebert/ponytail/blob/09c434ab73ca43761c5d8b0c5d341926c2fba710/docs/agent-portability.md
[PT `benchmarks/README.md`]: https://github.com/DietrichGebert/ponytail/blob/09c434ab73ca43761c5d8b0c5d341926c2fba710/benchmarks/README.md
[PT `benchmarks/results/2026-10-07-agentic.md`]: https://github.com/DietrichGebert/ponytail/blob/09c434ab73ca43761c5d8b0c5d341926c2fba710/benchmarks/results/2026-10-07-agentic.md
[PT `hooks/ponytail-instructions.js`]: https://github.com/DietrichGebert/ponytail/blob/09c434ab73ca43761c5d8b0c5d341926c2fba710/hooks/ponytail-instructions.js
[PT `benchmarks/agentic/README.md`]: https://github.com/DietrichGebert/ponytail/blob/09c434ab73ca43761c5d8b0c5d341926c2fba710/benchmarks/agentic/README.md
[PT `tests/behavior.test.js`]: https://github.com/DietrichGebert/ponytail/blob/09c434ab73ca43761c5d8b0c5d341926c2fba710/tests/behavior.test.js
[PT `LICENSE`]: https://github.com/DietrichGebert/ponytail/blob/09c434ab73ca43761c5d8b0c5d341926c2fba710/LICENSE
