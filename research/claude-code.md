# What Claude Code lets a Plugin ship, and its best practices

Research for [issue #4](https://github.com/iamivanhx/skills/issues/4), child of the map [#1](https://github.com/iamivanhx/skills/issues/1).

- **Primary source**: the official Claude Code docs at <https://code.claude.com/docs/en/>, read as raw Markdown via the [llms.txt index](https://code.claude.com/docs/llms.txt) on 2026-10-19. One cross-product page from platform.claude.com (skill authoring best practices) is used where the Claude Code skills page links to it.
- **Version caveat**: the docs track Claude Code v2.1.2xx. Several features below carry a minimum version; the claims are true of the docs on the date above.
- Vocabulary follows `GLOSSARY.md`: **Plugin**, **Repo setup**, **Workflow**, **Inspiration**, **Predecessor**, **Token-saving tool**.

## Answer in brief

- A Plugin can ship far more than skills: skills, legacy commands, subagents, hooks (settings-style JSON or a JavaScript "hooks module", which makes it a mod), MCP servers, LSP servers, output styles (one can be forced on), themes, monitors, dynamic workflows, `bin/` executables, `userConfig` prompts, channels, plugin dependencies, an eval suite, and a `settings.json` limited to `agent` and `subagentStatusLine`.
- A Plugin **cannot** ship: a `CLAUDE.md` (ignored), permission rules, `env`, the main `statusLine`, model/effort defaults, `enabledPlugins`/marketplace registration, skill-listing budgets, or anything else in `settings.json`. Those belong in the Repo setup.
- Skills cost their **name + description on every turn** (unless `disable-model-invocation: true`), and their **body stays in context once invoked**. Triggering depends on the description: put the key use case first; the combined text is capped at 1,536 chars, and the whole listing is budgeted at ~1% of the context window.
- For a personal marketplace, the simplest update model is **a local-path marketplace (a clone of this repo)**: plugins load in place, edits apply on the next session or `/reload-plugins`, and `version` is ignored. A GitHub-hosted marketplace instead copies to a cache and only updates when the computed version changes; auto-update is **off by default** for third-party marketplaces.
- Built-ins already cover several things the Inspirations build by hand: `/code-review`, `/simplify`, `/verify` + `/run`, `/batch`, `/deep-research`, `/goal`, plan mode, `/init`, `/doctor` (incl. `prompt-audit`), `/fewer-permission-prompts`, `/loop`, `/schedule`, auto memory, `claude plugin eval`.

---

## 1. What a Plugin can ship

A Plugin is "a directory of components, usually with a manifest" at `.claude-plugin/plugin.json`; the manifest is optional, and inside it only `name` is required ([plugins overview](https://code.claude.com/docs/en/plugins/overview), [components](https://code.claude.com/docs/en/plugins/components)). Every component is namespaced under the plugin `name` ([manifest reference](https://code.claude.com/docs/en/plugins/manifest-reference#name)).

| Component | Default location | Manifest key (behaviour vs default) | Notes |
| :- | :- | :- | :- |
| Skills | `skills/<name>/SKILL.md` | `skills` (adds to default scan) | Run as `/<plugin>:<dir>`; frontmatter `name` replaces only the last segment. A lone `SKILL.md` at the plugin root loads as one skill. [components#skills](https://code.claude.com/docs/en/plugins/components#skills) |
| Commands | `commands/*.md` | `commands` (replaces) | "The older format"; skills supersede them. The manifest can define a command inline with `content`. [components#commands](https://code.claude.com/docs/en/plugins/components#commands) |
| Subagents | `agents/**/*.md` | `agents` (replaces) | Named `<plugin>:<subfolders>:<name>`. Supported frontmatter: `name`, `description`, `model`, `effort`, `maxTurns`, `autoCompactWindow`, `tools`, `disallowedTools`, `skills`, `memory`, `background`, `omitClaudeMd`, `isolation` (`worktree` only), `color`. **Ignored**: `permissionMode`, `hooks`, `mcpServers`, `initialPrompt`. [components#agents](https://code.claude.com/docs/en/plugins/components#agents), [sub-agents](https://code.claude.com/docs/en/sub-agents#choose-the-subagent-scope) |
| Hooks | `hooks/hooks.json` | `hooks` (merges) | Same shape as settings `hooks`. Types: command, HTTP, MCP tool, prompt, agent. Registered when the session loads the plugin, not when a skill runs; narrow with `matcher`. [components#hooks](https://code.claude.com/docs/en/plugins/components#hooks), [hooks](https://code.claude.com/docs/en/hooks) |
| Hooks module (mod) | listed under `modules` in `hooks/hooks.json` | `types` for a `.d.ts` | JS/TS handlers inside Claude Code: draw panes or "a band above the prompt", add `/commands` that run without a Claude turn, rewrite tool calls. Hooks run everywhere; drawing only in terminal, Desktop, VS Code. Tested with `claude plugin test`. [mods overview](https://code.claude.com/docs/en/plugins/mods/overview) |
| MCP servers | `.mcp.json` | `mcpServers` (merges; also `.mcpb` bundles) | Server shows as `plugin:<plugin>:<server>`; tools as `mcp__plugin_<plugin>_<server>__<tool>`. [components#mcp-servers](https://code.claude.com/docs/en/plugins/components#mcp-servers) |
| LSP servers | `.lsp.json` | `lspServers` (merges) | Configures the connection only; the binary must be on the user's `PATH`. [components#lsp-servers](https://code.claude.com/docs/en/plugins/components#lsp-servers) |
| Output styles | `output-styles/<name>.md` | `outputStyles` (replaces) | Appear as `<plugin>:<name>`. `force-for-plugin: true` applies the style whenever the plugin is enabled and overrides the user's `outputStyle`. [components#themes-and-output-styles](https://code.claude.com/docs/en/plugins/components#themes-and-output-styles), [output-styles](https://code.claude.com/docs/en/output-styles) |
| Themes | `themes/<slug>.json` | `experimental.themes` (replaces) | Read-only; edits save as a user copy. [components](https://code.claude.com/docs/en/plugins/components#themes-and-output-styles) |
| Monitors | `monitors/monitors.json` | `experimental.monitors` (replaces) | Background shell command whose output reaches Claude as notifications. Interactive sessions only; no `${user_config.*}`. [components#monitors](https://code.claude.com/docs/en/plugins/components#monitors) |
| Dynamic workflows | `workflows/*.js` | `workflows` (replaces) | Scripts that orchestrate many subagents; run as `/<plugin>:<meta.name>`. [workflows#distribute-a-workflow-in-a-plugin](https://code.claude.com/docs/en/workflows#distribute-a-workflow-in-a-plugin) |
| Executables | `bin/` | none | On the Bash tool's `PATH` while enabled, after the user's own entries (can't shadow `git`). claude.ai/Cowork refuse plugins with a top-level `bin/`. [components#executables](https://code.claude.com/docs/en/plugins/components#executables) |
| Default settings | `settings.json` | `settings` | **Only `agent` and `subagentStatusLine` take effect**; every other key is dropped. Plugin defaults are the lowest settings layer. [components#default-settings](https://code.claude.com/docs/en/plugins/components#default-settings) |
| User configuration | `userConfig` in manifest | | Prompted in `/plugin`; `sensitive: true` goes to secure storage. Exposed as `${user_config.*}` and `CLAUDE_PLUGIN_OPTION_<KEY>`. [components#user-configuration](https://code.claude.com/docs/en/plugins/components#user-configuration) |
| Channels | `channels` in manifest | | Bind an MCP server so outside systems push messages into a session. [components#channels](https://code.claude.com/docs/en/plugins/components#channels) |
| Dependencies | `dependencies` in manifest | | Other plugins, with semver ranges resolved against `<plugin>--v<version>` tags. [dependencies](https://code.claude.com/docs/en/plugins/dependencies) |
| Eval suite | `evals/` | `experimental.evals` | Read by `claude plugin eval`. [plugin-evals](https://code.claude.com/docs/en/plugin-evals) |

Path rules: every manifest path is relative, starts with `./`, must exist and stay inside the plugin root (no `..`, no symlinks out, forward slashes only) ([manifest reference#path-rules](https://code.claude.com/docs/en/plugins/manifest-reference#path-rules)). Reference your own files through `${CLAUDE_PLUGIN_ROOT}` (changes every version, don't write state there) and keep durable state in `${CLAUDE_PLUGIN_DATA}` (`~/.claude/plugins/data/<id>/`, survives updates) ([components#path-variables-and-persistent-data](https://code.claude.com/docs/en/plugins/components#path-variables-and-persistent-data)).

## 2. What a Plugin can't set (Repo setup territory)

- **Instructions files**: "Claude Code doesn't load a `CLAUDE.md` at the plugin root"; `claude plugin validate` warns about it. "To include instructions in a plugin, write them as a skill." ([components#skills](https://code.claude.com/docs/en/plugins/components#skills), [manifest reference#standard-layout](https://code.claude.com/docs/en/plugins/manifest-reference#standard-layout)). Two routes do put always-on text in context from a Plugin: a `force-for-plugin` output style ([output-styles](https://code.claude.com/docs/en/output-styles)) and a `SessionStart` hook returning `additionalContext` ([hooks#add-context-for-claude](https://code.claude.com/docs/en/hooks#add-context-for-claude)). The docs steer static instructions to CLAUDE.md instead, and warn that hook text framed as system commands "can trigger Claude's prompt-injection defenses".
- **Settings**: only `agent` and `subagentStatusLine` survive in plugin settings ([manifest reference#settings](https://code.claude.com/docs/en/plugins/manifest-reference#settings)). So permissions/allowlists, `env`, the main `statusLine`, `model`/`effort`, `outputStyle` (except via `force-for-plugin`), `skillListingBudgetFraction`, `skillOverrides`, `autoMode`, sandbox, `includeGitInstructions` and marketplace/plugin enablement all live in user (`~/.claude/settings.json`) or project settings ([settings](https://code.claude.com/docs/en/settings)).
- **Permissions**: a skill's `allowed-tools` only pre-approves tools "during the turn that invokes" it; the grant clears on the next message. Session-wide allow rules must go in permission settings ([skills#pre-approve-tools-for-a-skill](https://code.claude.com/docs/en/skills#pre-approve-tools-for-a-skill)).
- **Plugin subagents** can't carry `permissionMode`, `hooks` or `mcpServers`; ship hooks and MCP servers at plugin level instead ([sub-agents](https://code.claude.com/docs/en/sub-agents#choose-the-subagent-scope)).
- **Marketplace auto-update**: `marketplace.json` has no field to turn it on; the user toggles it or sets `autoUpdate` on an `extraKnownMarketplaces` entry in settings ([host-marketplace#turn-on-auto-update](https://code.claude.com/docs/en/plugins/host-marketplace#turn-on-auto-update)).
- **External binaries**: an LSP plugin "doesn't install the server binary" ([components#lsp-servers](https://code.claude.com/docs/en/plugins/components#lsp-servers)). A Token-saving tool that isn't a script in `bin/` has to be installed by the Repo setup.
- **Skill visibility overrides**: "Plugin skills are not affected by `skillOverrides`. Manage those through `/plugin` instead." ([skills#override-skill-visibility-from-settings](https://code.claude.com/docs/en/skills#override-skill-visibility-from-settings)).

## 3. How skills load and trigger

**Loading** ([skills#control-who-invokes-a-skill](https://code.claude.com/docs/en/skills#control-who-invokes-a-skill), [features-overview#understand-context-costs](https://code.claude.com/docs/en/features-overview#understand-context-costs)):

| Frontmatter | User can invoke | Claude can invoke | In context |
| :- | :- | :- | :- |
| default | yes | yes | description every turn; body when invoked |
| `disable-model-invocation: true` | yes | no | nothing until invoked |
| `user-invocable: false` | no | yes | description every turn; body when invoked |

- **Enabled plugins cost every session**: for each model-invocable skill, agent and command, "the name and description are in Claude's context on every turn", even if nothing from the Plugin runs ([plugins overview](https://code.claude.com/docs/en/plugins/overview#what-an-enabled-plugin-adds-to-your-sessions)). `claude plugin details <name>` prints the "Always-on" token figure per component ([measure](https://code.claude.com/docs/en/plugins/measure#measure-what-a-plugin-costs)).
- **Body lifecycle**: once invoked, the rendered `SKILL.md` "stays there across later turns" and is not re-read. After compaction, each skill's most recent invocation is re-attached, **keeping the first 5,000 tokens**, within a **25,000-token** combined budget filled from the most recent skill. Older skills can drop entirely ([skills#skill-content-lifecycle](https://code.claude.com/docs/en/skills#skill-content-lifecycle)).
- **Size**: "Keep `SKILL.md` under 500 lines. Move detailed reference material to separate files" ([skills#add-supporting-files](https://code.claude.com/docs/en/skills#add-supporting-files)). Keep references one level deep from SKILL.md, because Claude may only partially read nested references ([skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)).
- **Listing budget**: descriptions share a listing budget of ~1% of the model's context window. On overflow, Claude Code drops descriptions "starting with the skills you invoke least". Each entry's `description` + `when_to_use` is capped at 1,536 characters ([skills#skill-descriptions-are-cut-short](https://code.claude.com/docs/en/skills#skill-descriptions-are-cut-short)). Users can raise it with `skillListingBudgetFraction` (a Repo setup setting).
- **Subagents**: skills named in a subagent's `skills:` field are fully preloaded (first 32 names); other skills stay discoverable through the Skill tool ([sub-agents#preload-skills-into-subagents](https://code.claude.com/docs/en/sub-agents#preload-skills-into-subagents)).
- **`context: fork`** runs the skill body as the task of a fresh subagent with no conversation history. It runs in the **background by default** (`background: false` to wait) and "only makes sense for skills with explicit instructions" ([skills#run-skills-in-a-subagent](https://code.claude.com/docs/en/skills#run-skills-in-a-subagent)).
- **Other useful frontmatter**: `arguments`/`$ARGUMENTS`/`$0`, `argument-hint`, `model`, `effort`, `paths` (auto-load only for matching files), `hooks` (registered on invoke, stay for the session), `` !`cmd` `` dynamic context injection, `${CLAUDE_SKILL_DIR}` ([skills#frontmatter-reference](https://code.claude.com/docs/en/skills#frontmatter-reference)). Unknown frontmatter keys are ignored silently. Malformed YAML loads the skill with no metadata, so `/name` works but Claude can't match the description.
- **Portability**: claude.ai uploads and the Skills API accept only `name`, `description`, `license`, `compatibility`, `metadata` and `allowed-tools` ([skills#using-skill-frontmatter-outside-claude-code](https://code.claude.com/docs/en/skills#using-skill-frontmatter-outside-claude-code)).

**What makes a skill trigger reliably**:

1. Description says **what it does and when to use it**, with the keywords users naturally say; key use case first ([skills#skill-not-triggering](https://code.claude.com/docs/en/skills#skill-not-triggering), [frontmatter reference](https://code.claude.com/docs/en/skills#frontmatter-reference)).
2. Write it in **third person** ("Processes X…"), because it is injected into the system prompt ([skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices#writing-effective-descriptions)).
3. Avoid vague or overlapping descriptions: "If descriptions are vague or overlap, Claude may load the wrong skill or miss one" ([features-overview](https://code.claude.com/docs/en/features-overview#understand-how-features-load)).
4. Over-triggering: make it more specific, or set `disable-model-invocation: true` ([skills#skill-triggers-too-often](https://code.claude.com/docs/en/skills#skill-triggers-too-often)).
5. Rules that must always hold go in a hook, not the skill. Guidance meant for a whole task should be worded as standing instructions. Put the most important instructions at the **top** of `SKILL.md` so they survive compaction ([skills#claude-stops-following-a-skill](https://code.claude.com/docs/en/skills#claude-stops-following-a-skill)).
6. Measure it: `claude plugin eval` with a `tool_used: Skill` grader, run after every description change ([skills#skill-not-triggering](https://code.claude.com/docs/en/skills#skill-not-triggering)).

**Evaluating skills** ([plugin-evals](https://code.claude.com/docs/en/plugin-evals)): `claude plugin eval` (v2.1.269+) runs each case in an isolated `-p` session **with and without the plugin** (three runs per arm by default) and reports `WITH`, `W/OUT` and `Δ`. Cases live in `evals/<case>/prompt.md` + `graders/*.md`. Grader types: `regex`, `tool_used`, `tool_order`, `file_exists`, `llm` (2-of-3 judge votes), `baseline`. It exits non-zero below `--threshold` (default 1.0), so it can gate CI; pin `--model`. `claude plugin eval init` drafts a suite. Every run is a real model call billed to the account. The skill-creator plugin has a separate, incompatible `evals/evals.json` format ([skills#evaluate-and-iterate-on-a-skill](https://code.claude.com/docs/en/skills#evaluate-and-iterate-on-a-skill)).

## 4. Best practices that shape the Workflow

All from [best practices](https://code.claude.com/docs/en/best-practices) unless noted.

- **Context is the constraint**: "performance degrades as it fills"; manage it as "the most important resource".
- **Verification first**: give Claude a check that returns pass/fail. Escalate from the prompt, to a [`/goal`](https://code.claude.com/docs/en/goal) condition, to a **Stop hook** that blocks the turn until the check passes, to a verification subagent. Ask for evidence (test output), not assertions.
- **Explore → plan → implement → commit**, with [plan mode](https://code.claude.com/docs/en/permission-modes#analyze-before-you-edit-with-plan-mode) (`Ctrl+G` edits the plan). Skip planning "if you could describe the diff in one sentence".
- **Interview, then spec, then a fresh session**: have Claude interview you with `AskUserQuestion`, write a self-contained spec (files, interfaces, out-of-scope, an end-to-end verification step), then start a new session to implement it.
- **CLAUDE.md sizing**: target **under 200 lines per file**; for each line ask "Would removing this cause Claude to make mistakes?". `@imports` don't reduce cost. Move procedures to skills and partial-codebase rules to path-scoped `.claude/rules/` ([memory#write-effective-instructions](https://code.claude.com/docs/en/memory#write-effective-instructions)). Use "IMPORTANT" on one line only. User-level `~/.claude/CLAUDE.md` and `~/.claude/rules/` apply to every project ([memory](https://code.claude.com/docs/en/memory#choose-where-to-put-claude-md-files)).
- **`/clear` and compaction**: `/clear` between unrelated tasks; after two failed corrections, `/clear` and re-prompt. `/compact <focus>` and a "Compact instructions" section in CLAUDE.md steer summaries; `/rewind` can summarize part of a conversation; `/btw` asks without growing context. The project-root CLAUDE.md is re-injected after `/compact` ([memory#instructions-seem-lost-after-compact](https://code.claude.com/docs/en/memory#instructions-seem-lost-after-compact)).
- **Subagents for investigation and review**: delegate reading-heavy research; add an adversarial review subagent in a fresh context. Tell it to flag only correctness or requirement gaps, to avoid over-engineering. Built-in Explore/Plan skip CLAUDE.md, so restate rules they need in the delegation prompt ([sub-agents#what-loads-at-startup](https://code.claude.com/docs/en/sub-agents#what-loads-at-startup)).
- **Hooks for guarantees**: "Use hooks for actions that must happen every time with zero exceptions"; CLAUDE.md and skills are advisory ([features-overview, Hook vs Skill](https://code.claude.com/docs/en/features-overview#compare-similar-features)). Hooks cost zero context unless they return output.
- **Permissions**: auto mode is the default starting mode from v2.1.283; allowlists and `/sandbox` cut prompts in either mode.
- **CLI over MCP**: CLI tools such as `gh` are "the most context-efficient way to interact with external services"; MCP tool schemas are deferred by default ([costs#reduce-mcp-server-overhead](https://code.claude.com/docs/en/costs#reduce-mcp-server-overhead)).
- **Token saving the docs endorse** ([costs#reduce-token-usage](https://code.claude.com/docs/en/costs#reduce-token-usage)): hooks that filter output before Claude sees it, e.g. a `PreToolUse` hook that rewrites test commands to show only failures. Also: move CLAUDE.md content to skills, `model: haiku` for simple subagents, lower `/effort`, code-intelligence (LSP) plugins.
- **Headless**: `claude -p` for scripts/CI; `--bare` skips hooks, skills, plugins, MCP, CLAUDE.md and auto memory, and will become the `-p` default. Use `--plugin-dir` to load a plugin explicitly ([headless](https://code.claude.com/docs/en/headless)).
- **Grow the setup by trigger**: CLAUDE.md after a repeated mistake; skill after the third pasted playbook; hook when something must always happen; plugin when a second repository needs the setup ([features-overview#build-your-setup-over-time](https://code.claude.com/docs/en/features-overview#build-your-setup-over-time)).

## 5. Versioning and updates for a personal marketplace

**How the version is computed** ([loading#versions-and-updates](https://code.claude.com/docs/en/plugins/loading#versions-and-updates)): `plugin.json` `version` first, then the marketplace entry's `version`, else the source: a git commit SHA (12 chars) for `github`/`url`/`git-subdir` or a relative path in a git-hosted marketplace. Updates install only when this computed version changes. A pinned `"version": "1.0.0"` keeps everyone on the cached copy "however many commits they push". Don't set it in both places; `plugin.json` wins silently ([host-marketplace#release-a-new-version](https://code.claude.com/docs/en/plugins/host-marketplace#release-a-new-version)).

**Two choices**: bump `version` on every release, or **omit it** so users track commits ([publish#prepare-your-plugin-for-release](https://code.claude.com/docs/en/plugins/publish#prepare-your-plugin-for-release)). With `version` omitted, drop `--strict` from `claude plugin validate`, because a missing version is a strict-mode warning.

**Install modes that matter for a personal setup**:

| Mode | How | Update behaviour |
| :- | :- | :- |
| Local-path marketplace | `claude plugin marketplace add ~/path/to/repo` with relative-path entries | Loads **in place**; edits apply at next session start or `/reload-plugins`; `version` ignored ([loading#in-place-and-copied-plugins](https://code.claude.com/docs/en/plugins/loading#in-place-and-copied-plugins), [create-marketplace#test-an-edit-to-a-plugin](https://code.claude.com/docs/en/plugins/create-marketplace#test-an-edit-to-a-plugin)) |
| GitHub marketplace | `claude plugin marketplace add owner/repo` | Copied to `~/.claude/plugins/cache/<mkt>/<plugin>/<version>/`. Updates via `claude plugin update`, `/plugin marketplace update`, or auto-update, which is **off by default for third-party marketplaces** ([install#keep-plugins-updated](https://code.claude.com/docs/en/plugins/install#keep-plugins-updated)) |
| Skills-directory plugin | a folder with `.claude-plugin/plugin.json` under `~/.claude/skills/` (`claude plugin init` scaffolds one) | Loads every session as `<name>@skills-dir`, no install ([create#scaffold-a-plugin-that-loads-every-session](https://code.claude.com/docs/en/plugins/create#scaffold-a-plugin-that-loads-every-session)) |
| Session-only | `--plugin-dir`, `--plugin-url`, `CLAUDE_CODE_PLUGIN_DIRS` | For development; overrides a same-named installed plugin silently ([loading#name-conflicts](https://code.claude.com/docs/en/plugins/loading#name-conflicts)) |

**Auto-update mechanics**: after the first message of an interactive session, after a random delay of up to 10 minutes, it refreshes marketplaces and updates on disk. The running session keeps the old version until `/reload-plugins`; monitors need a restart ([loading#when-auto-update-runs](https://code.claude.com/docs/en/plugins/loading#when-auto-update-runs)). Old version directories are deleted 14 days later ([loading#cleanup-of-previous-versions](https://code.claude.com/docs/en/plugins/loading#cleanup-of-previous-versions)).

**Other release tools**: `claude plugin tag` creates `{name}--v{version}` tags, needed only when other plugins depend on a version range. Release channels mean two marketplaces with different `name`s pointing at different refs. `ref`/`sha` on an entry pins a version. **Never rename** a published plugin; use `displayName` or a `renames` map ([publish#ship-updates-renames-and-removals](https://code.claude.com/docs/en/plugins/publish#ship-updates-renames-and-removals), [host-marketplace#run-release-channels](https://code.claude.com/docs/en/plugins/host-marketplace#run-release-channels)). Keep the marketplace entry `name` equal to the `plugin.json` `name` ([create-marketplace#keep-the-entry-name-and-the-manifest-name-the-same](https://code.claude.com/docs/en/plugins/create-marketplace#keep-the-entry-name-and-the-manifest-name-the-same)).

**Reserved names**: plugin names starting `claude-`, `anthropic-`, `cc-plugin-`, or equal to `claude`/`claude-code` are errors; `claude` as a whole word elsewhere warns ([manifest reference#name](https://code.claude.com/docs/en/plugins/manifest-reference#name)).

## 6. Built-ins that overlap with what Inspirations build by hand

Bundled skills and commands, from the [commands reference](https://code.claude.com/docs/en/commands) and [skills#bundled-skills](https://code.claude.com/docs/en/skills#bundled-skills):

| Built-in | Overlaps with |
| :- | :- |
| `/code-review [effort] [--fix] [--comment] [pr]` (alias `/review`), `/code-review ultra`, `/security-review` | hand-built code-review skills (e.g. `mattpocock-skills:code-review`) |
| `/simplify` (four parallel review agents, applies fixes) | cleanup/refactor-review skills |
| `/verify`, `/run`, `/run-skill-generator` (writes a per-project `run-<name>` skill) | "run the app to verify" steps |
| `/goal <condition>` plus a Stop hook | "keep going until tests pass" loops |
| plan mode, `/plan`, `Ctrl+G` | planning/spec skills (keep only what differs, e.g. writing issues) |
| `AskUserQuestion` interview → SPEC.md (a documented pattern, not a skill) | grilling-style skills |
| `/deep-research` (bundled workflow) and dynamic workflows | research and fan-out skills |
| `/batch` (5-30 subagents, one worktree each) | parallel-migration skills |
| `/init` (`CLAUDE_CODE_NEW_INIT=1` also proposes skills/hooks) | per-project bootstrap of CLAUDE.md |
| `/doctor`, `/doctor prompt-audit`, `/skill-doctor` | CLAUDE.md/skill audits, finding unused skills |
| `/fewer-permission-prompts`, `/update-config` | permission/settings setup scripts |
| auto memory, `/memory`, `/recap`, `/rewind`, `/branch`, `/fork`, `/subtask` | hand-rolled memory/handoff/session notes |
| `/loop`, `/schedule` (cloud routines) | polling/recurring-task skills |
| `--worktree`, subagent `isolation: worktree` | worktree helper scripts |
| `/statusline` | hand-written statusline scripts |
| built-in output styles: Proactive, Concise, Explanatory, Learning ([output-styles](https://code.claude.com/docs/en/output-styles#built-in-output-styles)) | terse/voice prompt files |
| `claude plugin eval`, `claude plugin validate`, `claude plugin details` | custom skill-eval or lint harnesses |

Notes:

- A skill whose name matches a bundled skill replaces it, but not its aliases. Plugin skills are namespaced, so they never replace a bundled skill ([skills#resolve-skills-that-share-a-name](https://code.claude.com/docs/en/skills#resolve-skills-that-share-a-name)).
- `disableBundledSkills` turns the bundled set off ([skills#bundled-skills](https://code.claude.com/docs/en/skills#bundled-skills)).

## 7. Surprises worth carrying to other tickets

1. **Cloud sessions don't load the Plugin.** A cloud session (claude.ai/code, routines) loads neither user-enabled plugins nor plugins a repo enables in `.claude/settings.json`, and ignores `~/.claude/` entirely ([cloud-environments#what-carries-over-from-your-setup](https://code.claude.com/docs/en/cloud-environments#what-carries-over-from-your-setup)). Routes that do reach cloud or Cowork: skills enabled on the claude.ai account, plugins turned on for the claude.ai account (load as `<name>@synced`, including "your own claude.ai uploads"), or a repo's committed `.claude/skills/` ([loading#synced-plugins](https://code.claude.com/docs/en/plugins/loading#synced-plugins), [install#add-from-claude-ai](https://code.claude.com/docs/en/plugins/install#add-from-claude-ai)). This bears on the "skills should still work in desktop/web/cloud" preference.
2. **A plugin can live inside a repo.** A folder with `.claude-plugin/plugin.json` under `<project>/.claude/skills/<name>/` loads as a plugin for collaborators (after workspace trust; no monitors, MCP needs approval). Claude Code "doesn't scan a project's `.claude/plugins/` directory" ([loading#plugins-shared-through-a-repository](https://code.claude.com/docs/en/plugins/loading#plugins-shared-through-a-repository)). Relevant to the per-project bootstrap.
3. **A Plugin can force always-on instructions** through a `force-for-plugin` output style. But a custom style drops Claude Code's built-in software-engineering instructions unless `keep-coding-instructions: true` ([output-styles](https://code.claude.com/docs/en/output-styles)). It is the only Plugin-side analogue of a global CLAUDE.md; the other is a `SessionStart` `additionalContext` hook. Relevant to **Global instructions**.
4. **The main statusline can't ship in a Plugin** (only `subagentStatusLine`). A mod can draw "a band above the prompt" or a pane instead, CLI/Desktop/VS Code only ([mods overview](https://code.claude.com/docs/en/plugins/mods/overview)). Otherwise the statusline belongs in the Repo setup, which can still point `statusLine` at a script the Plugin ships in `bin/` or `${CLAUDE_PLUGIN_ROOT}`. Relevant to **Plugin components**.
5. **Plugin `verify`/`simplify` skills don't get the pre-commit hook-in.** Claude Code tells Claude to run a skill named `verify` or `simplify` before each commit only when it comes from personal/project/enterprise locations, not from plugins ([skills#run-your-checks-before-each-commit](https://code.claude.com/docs/en/skills#run-your-checks-before-each-commit)).
6. **Plugin agents can't set `permissionMode`, hooks or MCP servers**, so a "read-only reviewer" agent must restrict via `tools`/`disallowedTools` ([components#frontmatter-fields-in-plugin-agents](https://code.claude.com/docs/en/plugins/components#frontmatter-fields-in-plugin-agents)).
7. **Local-path marketplace = zero-friction updates** on the machine holding the clone. Other machines on a GitHub marketplace need auto-update toggled on (a Repo setup step: `extraKnownMarketplaces` with `autoUpdate: true`), plus either a version bump per release or no `version` field. Relevant to **Versioning and updates**.
8. **Plugin hooks fire in every session where the Plugin is enabled**, and also inside subagents ([hooks#hook-locations](https://code.claude.com/docs/en/hooks#hook-locations)). Hooks in skill frontmatter, by contrast, only register once the skill is invoked. That is a cheaper way to scope a guardrail to one Workflow phase.
9. **Every model-invocable skill is a per-turn tax** for as long as the Plugin is enabled. With many Workflow skills, user-invoked steps should use `disable-model-invocation: true`, which removes them from context entirely ([features-overview](https://code.claude.com/docs/en/features-overview#context-cost-by-feature)).

## 8. Checklist for designing the Plugin

- [ ] Kebab-case permanent `name`, not starting `claude-`; same name in `marketplace.json` entry and `plugin.json` ([publish](https://code.claude.com/docs/en/plugins/publish#prepare-your-plugin-for-release))
- [ ] Choose one versioning rule: omit `version` (track commits) or bump every release, never both places ([host-marketplace](https://code.claude.com/docs/en/plugins/host-marketplace#release-a-new-version))
- [ ] Dev loop: local-path marketplace or `--plugin-dir` + `/reload-plugins` ([create](https://code.claude.com/docs/en/plugins/create#develop-without-a-marketplace))
- [ ] Write skills, not commands; no plugin-root `CLAUDE.md` ([components](https://code.claude.com/docs/en/plugins/components#skills))
- [ ] Every description: third person, what + when, trigger keywords first, well under 1,536 chars, no overlap with sibling skills ([skills](https://code.claude.com/docs/en/skills#frontmatter-reference), [authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices))
- [ ] `disable-model-invocation: true` on side-effecting or user-driven steps ([skills](https://code.claude.com/docs/en/skills#control-who-invokes-a-skill))
- [ ] `SKILL.md` < 500 lines, concise, key instructions at the top, references one level deep ([skills](https://code.claude.com/docs/en/skills#add-supporting-files))
- [ ] Standing instructions, not one-time steps, for guidance that spans a task ([skills](https://code.claude.com/docs/en/skills#claude-stops-following-a-skill))
- [ ] Rules that must always hold → hooks (plugin `hooks/hooks.json`, or skill-frontmatter hooks to scope them) ([features-overview](https://code.claude.com/docs/en/features-overview#compare-similar-features))
- [ ] Reading-heavy or review steps → subagents or `context: fork`; restate needed rules in the delegation since Explore/Plan skip CLAUDE.md ([sub-agents](https://code.claude.com/docs/en/sub-agents#what-loads-at-startup))
- [ ] Build in a verification gate (tests, Stop hook or `/goal`) and an adversarial review step ([best-practices](https://code.claude.com/docs/en/best-practices#give-claude-a-way-to-verify-its-work))
- [ ] Prefer `gh`/CLI over MCP; ship helpers in `bin/` only if claude.ai/Cowork reach isn't needed ([costs](https://code.claude.com/docs/en/costs#reduce-mcp-server-overhead), [components](https://code.claude.com/docs/en/plugins/components#executables))
- [ ] Reference files via `${CLAUDE_PLUGIN_ROOT}`; state in `${CLAUDE_PLUGIN_DATA}` ([components](https://code.claude.com/docs/en/plugins/components#path-variables-and-persistent-data))
- [ ] Don't rebuild built-ins (section 6); wrap or chain them instead
- [ ] `claude plugin validate` on every change; `claude plugin details` to watch the always-on token figure ([measure](https://code.claude.com/docs/en/plugins/measure))
- [ ] Eval suite in `evals/` with `tool_used: Skill` triggering cases and outcome graders; compare against the no-plugin baseline; pin `--model` ([plugin-evals](https://code.claude.com/docs/en/plugin-evals))
- [ ] Repo setup covers what the Plugin can't: user CLAUDE.md/rules, permissions, `env`, `statusLine`, marketplace registration + `autoUpdate`, external binaries, `skillListingBudgetFraction` ([manifest reference#settings](https://code.claude.com/docs/en/plugins/manifest-reference#settings))
