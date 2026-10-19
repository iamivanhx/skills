# Token-saving tools: candidates for the Workflow

Research for [#5](https://github.com/iamivanhx/skills/issues/5), a child of the map [#1](https://github.com/iamivanhx/skills/issues/1). Researched 2026-10-19.

**Question.** Which Token-saving tools and techniques fit this Workflow? RTK is the anchor. The other candidates are alternatives found in the wild and the levers built into Claude Code. For each one: does it install as part of the Plugin or the Repo setup, what does it save, what does it cost or risk, and should we adopt, try or skip it?

**Short answer.** The levers built into Claude Code beat every third-party tool on evidence, risk and fit. Adopt them, and ship whatever a Plugin can carry (skills, subagents with `model:`, hooks). Settings and env vars go in the Repo setup. Skip RTK for now. Its own docs say it only shrinks Bash output. Independent measurements in its issue tracker find no bill reduction in Claude Code sessions, and its command rewriting interacts badly with auto mode. Skip the API proxies (Headroom, rtk-proxy, the caveman proxy): pointing `ANTHROPIC_BASE_URL` at one silently turns off MCP tool search and can break auto mode's server-side review. The one idea worth borrowing from the third-party tools is "compress verbose output but keep the original recoverable". Claude Code already does this natively through `bashOutputMaxChars`.

Source key: **[CC]** is an official Claude Code doc page on code.claude.com. **[repo]** is the tool's own repository or docs (first-party for that tool, but self-reported). **[issue]** is a GitHub issue on the tool's tracker. **(marketing)** marks a claim that only the vendor's own README or site makes and that no source here verified independently.

---

## 1. Candidate table

| # | Candidate | What it does | Installs via | Expected saving | Cost / risk | Verdict |
|---|---|---|---|---|---|---|
| B1 | `/clear` between tasks, focused `/compact`, "Compact instructions" in CLAUDE.md | Drops stale context, steers what compaction keeps | Workflow habit, encoded in Plugin skills (handoff points) | Largest single lever on long sessions. "Stale context wastes tokens on every subsequent message" [CC costs] | `/compact` is itself a large request. `/clear` costs nothing [CC costs] | **Adopt** |
| B2 | Model choice: Sonnet by default, Opus for hard reasoning, `opusplan`, `model: haiku` on cheap subagents | Matches model price to the job | Plugin (subagent `model:` frontmatter). Repo setup (default model, `CLAUDE_CODE_SUBAGENT_MODEL`) | Large, set by the price ratio between models. Docs call it one of the "highest-impact habits" [CC costs] | Quality drop if under-modelled. A switch to Opus also moves the subagents that inherit the model [CC costs] | **Adopt** |
| B3 | Effort level (`/effort`, `CLAUDE_CODE_EFFORT_LEVEL`) | Controls thinking depth. Thinking tokens bill as output | Repo setup (or per session) | Medium. Thinking "can be tens of thousands of tokens per request" [CC costs]. Opus/Sonnet/Haiku 5.5 already default to `medium` [CC model-config] | Lower effort means weaker plans on hard tasks | **Adopt** (keep the 5.5 default of `medium`; raise it per task, not globally) |
| B4 | Delegate verbose work to subagents (Explore, custom agents) | Verbose output stays in the subagent's context. Only a summary comes back | Plugin (`agents/`) | Large on main-context growth. Explore and Plan also skip CLAUDE.md and git status at startup [CC sub-agents] | Subagent requests still count toward usage. Use a smaller model for them [CC costs] | **Adopt** |
| B5 | Skills instead of CLAUDE.md bulk (progressive disclosure) | Skill bodies load only when invoked | Plugin (skills) | Medium and permanent. Keep CLAUDE.md under 200 lines [CC costs]. Skill listing text is capped at 1,536 chars per skill. `disable-model-invocation: true` removes the description from context entirely [CC skills] | A poorly written description fails to trigger | **Adopt** (shapes skill authoring; see `writing-for-agents`) |
| B6 | MCP tool search (deferral) + prefer CLIs (`gh`) over MCP + disable unused servers | Only tool names enter context until a tool is used | Already the default. Repo setup must not break it | Medium. CLIs "don't add any per-tool listing" [CC costs] | **Turned off automatically when `ANTHROPIC_BASE_URL` points at a non-first-party host** [CC mcp]. Any proxy in candidates T3–T5 does that | **Adopt** (and guard it: no proxies) |
| B7 | `bashOutputMaxChars` | Bash output above the limit goes to a file. Claude gets a preview plus the path | Repo setup (`~/.claude/settings.json`) | Small to medium, only on very verbose commands. Default is 30,000 chars inline, clamped 4,000–128,000 [CC settings-reference] | Lossless: the full output stays on disk for Claude to read on demand [CC tools-reference]. Failures are handled separately (about 10k-char head+tail excerpt) | **Try** (lower it modestly, then measure with `/usage`) |
| B8 | Self-written output-filter hook (PostToolUse `updatedToolOutput`, or the doc's PreToolUse test filter) | Trims known-noisy output (test runners, logs) before Claude reads it | Plugin (`hooks/hooks.json` + script in the plugin) | Only for the specific noisy commands it targets. Docs give a log example: "tens of thousands of tokens to hundreds" [CC costs] | Lossy unless the original is kept. Maintenance burden. PreToolUse **input rewrites** carry the auto-mode risk below (§3.3). PostToolUse output replacement avoids it (inference, see §3.3) | **Try**, but only if `/usage` shows a real hotspot. Prefer PostToolUse and keep the original recoverable |
| B9 | Code-intelligence (LSP) plugins | "Go to definition" instead of grep followed by reading candidate files. Type errors reported after edits | Repo setup (official plugin per language) or project-level | Medium on unfamiliar code [CC costs] | Per stack, which conflicts with stack-agnostic skills. So it belongs at the project or machine level | **Try** (per stack, outside the Plugin) |
| B10 | Built-in **Concise** output style | Responses lead with the result, with no preamble or recap | Repo setup (`outputStyle`) or `/output-style` | Small. Output is a minor share of agent-session cost (see T2's evidence) | A custom style drops the built-in coding instructions unless `keep-coding-instructions: true` [CC output-styles] | **Try** (cheap and reversible) |
| B11 | Measurement: `/usage` (attribution by skill, subagent, plugin and MCP server; prompt-cache line), `/context`, status line `prompt_cache` fields, `/insights` | Shows where tokens actually go before anything is changed | Built in. A status-line config is Repo setup | None directly. It is what lets you judge every other row | None | **Adopt** (a prerequisite for any Try) |
| B12 | Plan mode, specific prompts, early course-correction, `/rewind` | Avoids expensive rework | Already part of the Workflow | Large on avoided re-work [CC costs] | None | **Adopt** (the Workflow already does this) |
| T1 | **RTK** (`rtk-ai/rtk`) | Rewrites Bash commands to `rtk <cmd>` and filters their output | Repo setup: binary via brew/curl + `rtk init -g` writes a PreToolUse hook into `~/.claude/settings.json`. The Plugin could carry the hook entry, but not the binary | Self-reported "60-90%" **of Bash output bytes** (marketing). Session-level measurements: about 0%, +2%, +0.11% input (§2.4) | Auto-mode re-review cost (§2.5). Read/Grep/Glob bypass it. Output-loss bugs. 879 open issues. Mixed Apache/BUSL licence | **Skip** (revisit if an independent benchmark shows a bill reduction) |
| T2 | **caveman** (`JuliusBrussee/caveman`) | (a) a terse "voice" skill, (b) a proxy that compresses reads, (c) "Auto" model routing through their cloud account, plus more | (a) Plugin-style skill (Apache-2.0, so the idea can be borrowed with credit). (b)/(c) Repo setup: proxy + account | Voice: `/caveman` is 3% fewer output tokens than a plain "Answer concisely." at the median [repo]. JetBrains A/B reportedly found 8.5% fewer output tokens and no quality loss (quoted by the repo; the JetBrains page returned 403, so unverified) | The proxy hits the §3.1 issues. Auto sends asks to a third-party cloud. A very broad surface (six "rocks") | **Skip** the tool. Take the lesson: ask for concision in our own words (B10) |
| T3 | **Headroom** (`headroomlabs-ai/headroom`) | Local proxy / `headroom wrap claude` that compresses tool outputs, with reversible retrieval (CCR) | Repo setup (Python CLI + proxy) | Says "20% fewer tokens for coding agents" (marketing). A competitor's benchmark found 6.7% fewer tokens and 15/18 correct [caveman repo] | `ANTHROPIC_BASE_URL` proxy (§3.1). Also installs Serena into wrapped projects [repo] | **Skip** |
| T4 | **context-mode** (`mksglu/context-mode`) | Claude Code plugin: an MCP sandbox (`ctx_execute` etc.) + 6 hooks that route tool calls through it, plus SQLite session memory | Separate plugin (marketplace) | "98% reduction" on sandboxed output (marketing). No session-level bill data found | Elastic License 2.0 (not OSI). Hooks on Bash, Read, Grep, WebFetch and Task change how every tool behaves. Its customer-logo badges link to `#` (marketing) | **Skip** (overlaps B4 and B7 and is heavy). Its "think in code" idea is already Claude's default when it writes scripts |
| T5 | rtk-proxy, lean-ctx, squeez, snip, ctk, SuperCompress, edgee | Same class as RTK or Headroom (shell-output hooks or API proxies) | Repo setup | Unverified (marketing). One comparison says squeez showed no measurable saving | Young or small projects (ctk has 1 star). Proxies hit §3.1 | **Skip** |
| T6 | claude-code-router / OmniRoute / LiteLLM | Routes Claude Code traffic to other providers or models | Repo setup (proxy) | Depends on the cheaper model used | Proxy issues (§3.1). Off-plan models. Not Claude-first | **Skip** |
| T7 | ccusage | Reads local session logs and reports usage/cost | Repo setup (`npx ccusage`) | None directly (measurement) | Mostly superseded by `/usage` attribution and the status-line fields (B11) | **Skip** (optional) |
| T8 | Serena (MCP, LSP-backed semantic retrieval) | Symbol-level navigation through MCP | Repo setup (MCP server) | Similar to B9 | Extra MCP surface. Official LSP plugins cover the same need natively | **Skip** in favour of B9 |

---

## 2. RTK in detail (the anchor)

### 2.1 What it is

- Official repo: [rtk-ai/rtk](https://github.com/rtk-ai/rtk). Docs: [rtk-ai.app/guide](https://www.rtk-ai.app/guide). Rust, created 2026-01-22. Latest stable is v1.0.1 (2026-10-14), plus near-daily `dev-1.0.2-rc.*` pre-releases. About 84k stars, 879 open issues at time of writing [gh API].
- "rtk filters and compresses command outputs before they reach your LLM context. Single Rust binary, 100+ supported commands, <10ms overhead." [repo README]
- Four strategies: smart filtering, grouping, truncation, deduplication [repo README].
- Coverage: git, gh, cargo, npm/pnpm/vitest/jest/tsc/Next/Playwright/Prisma, ruff/pytest/mypy/pip, go test/golangci-lint, Ruby, .NET, Gradle/Maven, PHP, Docker, Kubernetes, AWS, Terraform, plus `ls`/`tree`/`cat`/`grep`/`find` [repo README].
- Name collision: a different `rtk` ("Rust Type Kit") exists on crates.io. `rtk gain` tells them apart [repo troubleshooting].

### 2.2 How it hooks into Claude Code

- `rtk init -g` registers a **PreToolUse hook** (`rtk hook claude`, a native binary) in `~/.claude/settings.json`. The hook rewrites `git status` to `rtk git status` and so on, and returns `permissionDecision: "allow"` with `updatedInput` [repo README, troubleshooting; CC hooks for the `updatedInput` contract].
- **"The hook only fires on Bash tool calls. Built-in tools like `Read`, `Grep` and `Glob` do not pass through it."** [repo README]. Claude Code prefers those built-in tools for reading and searching, so a large share of the context they produce is out of RTK's reach.
- Open hook gaps include compound commands with pipes and `&&` ([#2425](https://github.com/rtk-ai/rtk/issues/2425)) and heredoc commit messages ([#4483](https://github.com/rtk-ai/rtk/issues/4483)) [issue].
- Optional pieces: `rtk-proxy` (an API-traffic daemon, BUSL-1.1) launched via `rtk claude`, and a `pipeline.toml` engine (BUSL-1.1) that can also add PostToolUse compression on Claude Code [repo README, agent-coverage].

### 2.3 Licence

The CLI is Apache-2.0, but the default `rtk` download links `rtk-engine`, which is **BUSL-1.1**. An Apache-only `rtk-oss` edition exists. `rtk-proxy` is BUSL-1.1. Production use is free; offering it to others commercially is not [repo README, DISCLAIMER]. Fine for personal use, but the licence has moved away from plain Apache within 2026.

### 2.4 Measured savings: claims vs evidence

- **The project's own careful wording:** "RTK reduces the bash output your agent reads… it is not the same as cutting your bill by the same amount." Token counts are `bytes / 4` estimates. "A command showing 90% fewer output bytes does not make your session 90% cheaper." [repo savings-explained]. The headline "60-90%" (repo description) and "up to 90%" are about **Bash output bytes only** (marketing for anything wider).
- **Independent and semi-independent measurements, all in the RTK tracker:**
  - [#4640](https://github.com/rtk-ai/rtk/issues/4640) (2026-10-18, open). skilllab harness, Claude Code headless with Haiku 5.5, rtk 1.0.1, 12 tasks × 3 runs, with a no-op control arm. The hook fired (about 380K tokens "saved" per `rtk gain`), yet **total cost was +2% (90% range −1% to +5%)**. Median input tokens +10%, output +3%. Labelled preliminary.
  - [#2001](https://github.com/rtk-ai/rtk/issues/2001) (open). 41 commands across 5 repos, Sonnet/Opus 4.6. Mean cost savings of +0.2% / +2.7%, and 34 of 41 commands within ±2%. A maintainer replied it "seems to be more of an architectural oversight than a bug" and floated a bypass threshold.
  - [#582](https://github.com/rtk-ai/rtk/issues/582) (closed). The reporter measured +18% cost (n=1). A maintainer re-ran it and could not reproduce, but the maintainers' **own** 20-iteration, 10-minute session benchmark showed "input tokens quasi-identical (+0.11%)", with output −2.43%.
  - [#3175](https://github.com/rtk-ai/rtk/issues/3175) (open). Anecdotal reports of hitting usage limits faster with the hook installed.
- **Why the saving dilutes:** Bash output is a small slice of each request next to the system prompt, tool definitions and history. Most of that slice is read from the prompt cache at a fraction of the input price, so compressing it barely moves the bill [repo savings-explained; CC costs "Why usage climbs"; CC prompt-caching].

**Conclusion:** no source shows RTK reducing a Claude Code bill. Several show no change.

### 2.5 Risks

- **Auto mode (this machine's default).** Auto mode is the built-in starting mode from Claude Code v2.1.283 [CC permission-modes]. [#4627](https://github.com/rtk-ai/rtk/issues/4627) traces that when a PreToolUse hook rewrites a Bash call's input, Claude Code drops the server-side safety review for that call and sends its own classifier request instead, on Sonnet 5. On one API-billed day that came to **14.7% of the bill**. The Claude Code docs confirm the two review paths and that local classifier calls are billed on API and Enterprise accounts [CC permission-modes]. They do **not** document the "rewrite forces re-review" behaviour, so treat it as a third-party trace, not a documented fact.
- **Lost or misleading output.** `rtk golangci-lint` maps exit 1 to 0 and drops stderr, reporting "No issues found" ([#3870](https://github.com/rtk-ai/rtk/issues/3870)). The hook drops stdin, so piped `wrangler secret put` stores empty values ([#2431](https://github.com/rtk-ai/rtk/issues/2431)). `grep | head` returns the wrong lines ([#4207](https://github.com/rtk-ai/rtk/issues/4207)). `ssh` hangs ([#3748](https://github.com/rtk-ai/rtk/issues/3748)) [issue]. Mitigations exist: a `rtk recall <hash>` store for failed or trimmed output, `rtk bypass`, and `[hooks] exclude_commands` [repo configuration]. But the agent sees a different command and different output from what a human would see, which confuses debugging.
- **Churn.** Over 500 release candidates in the run-up to 1.0.2, a large open-issue backlog, and a growing scope (proxy, pipeline engine) [gh API, repo].

### 2.6 Fit verdict

**Skip.** It fails the map's bar ("evaluate real Token-saving tools for fit"). The evidence for a real saving is absent, the main risk falls on the auto-mode Workflow, and it reaches none of the built-in read tools. If revisited, the setup would be Repo setup only: `rtk-oss` via brew, `rtk init -g --hook-only`, a time-boxed A/B run measured with `/usage`. Revisit triggers: RTK ships the "bypass threshold" floated in #2001, or an independent benchmark shows a bill reduction.

---

## 3. Cross-cutting findings that affect other decisions

### 3.1 Any API proxy silently changes Claude Code behaviour

Pointing `ANTHROPIC_BASE_URL` at a non-first-party host (Headroom, rtk-proxy, the caveman proxy, claude-code-router, LiteLLM) has two documented effects:

1. **MCP tool search turns off** and every MCP tool loads upfront, "since most proxies don't forward `tool_reference` blocks". Setting `ENABLE_TOOL_SEARCH` explicitly overrides this [CC mcp, env-vars].
2. **Auto mode's server-side review** depends on the proxy passing the `safeguards` field and the `safeguard_results` response key through unchanged. If the proxy drops them, Claude Code falls back to its own classifier requests, which are billed on API and Enterprise accounts. If it rewrites the results, actions get denied for lack of a verdict [CC permission-modes, auto-mode-classifier-billing].

A compression proxy can therefore *increase* token use. The Repo setup should not set `ANTHROPIC_BASE_URL`.

### 3.2 What a Plugin can and cannot carry for token saving

- **Can:** skills (B5), subagents with a `model:` field (B2, B4), hooks in `hooks/hooks.json` with scripts referenced via `${CLAUDE_PLUGIN_ROOT}` (B8), and output styles (B10) [CC plugins-reference, hooks].
- **Cannot:** a plugin's `settings.json` honours only `agent` and `subagentStatusLine`, and drops every other key at load [CC plugins-reference]. So `bashOutputMaxChars`, `CLAUDE_CODE_SUBAGENT_MODEL`, the effort level, the default model and `outputStyle` all belong in the **Repo setup**.
- **Caveat:** a plugin that ships a `bin/` directory (executables on the Bash `PATH`) **is not installed on claude.ai or Cowork** [CC plugins-reference]. That clashes with the standing preference that skills keep working in desktop, web and cloud. Bundling a filter binary (for example `rtk-oss`) inside the Plugin would cost that portability.

### 3.3 If we ever write our own filter hook

- PreToolUse `updatedInput` rewrites the command. In auto mode that likely triggers the extra classifier request (§2.5, third-party trace).
- PostToolUse `updatedToolOutput` replaces a tool's result after it ran, for any tool including Read, Grep and MCP tools [CC hooks]. The auto-mode classifier sees tool calls, not tool results ("Tool results are stripped from those requests") [CC permission-modes]. So replacing output *should* not disturb the review. That is an inference, not a documented guarantee, and it needs a test.
- Keep the original recoverable, the way Claude Code itself does for long Bash output (file plus path) [CC tools-reference].

### 3.4 Output tokens are the smaller lever

Every source that measured whole agent sessions (caveman's own table, JetBrains as quoted, #4640) finds input and cached context dominate, and terse-output tricks move output by single-digit to low-double-digit percentages. That is consistent with the map's "no artificial budgets" stance: favour structural levers (B1, B2, B4, B5) over prose-style rules.

---

## 4. Suggested split for the spec

- **Plugin:** skills written for progressive disclosure (B5). Workflow skills that hand off at natural `/clear` points and delegate verbose work to subagents (B1, B4). Custom subagents with an explicit `model:` (B2). Optionally an output style (B10). No filter binaries.
- **Repo setup (`~/.claude/settings.json` + checklist):** default model and effort (B2, B3). `bashOutputMaxChars` trial (B7). A status line showing context and `prompt_cache` (B11). Per-language LSP plugins (B9). No `ANTHROPIC_BASE_URL` proxy (§3.1). No RTK (§2.6).
- **Evaluation hook-in:** the skill-eval ticket could reuse a with/without A/B harness like skilllab's (#4640) to measure any future Token-saving tool on real tasks before adopting it.

---

## Sources

Claude Code docs (primary):
- Manage costs effectively: https://code.claude.com/docs/en/costs
- Permission modes (auto mode, server-side classifier review, classifier billing): https://code.claude.com/docs/en/permission-modes
- Auto mode classifier billing: https://code.claude.com/docs/en/auto-mode-classifier-billing
- Hooks reference (`updatedInput`, `updatedToolOutput`, plugin hooks): https://code.claude.com/docs/en/hooks
- Tools reference, Bash output limits: https://code.claude.com/docs/en/tools-reference
- Settings reference, `bashOutputMaxChars`: https://code.claude.com/docs/en/settings-reference
- Environment variables: https://code.claude.com/docs/en/env-vars
- MCP, tool search: https://code.claude.com/docs/en/mcp
- Model config (effort, `opusplan`): https://code.claude.com/docs/en/model-config
- Subagents: https://code.claude.com/docs/en/sub-agents
- Skills: https://code.claude.com/docs/en/skills
- Output styles: https://code.claude.com/docs/en/output-styles
- Prompt caching: https://code.claude.com/docs/en/prompt-caching
- Plugins reference: https://code.claude.com/docs/en/plugins-reference

RTK (first-party for the tool):
- Repo and README: https://github.com/rtk-ai/rtk
- How savings work: https://github.com/rtk-ai/rtk/blob/master/docs/guide/resources/savings-explained.md
- Agent coverage: https://github.com/rtk-ai/rtk/blob/master/docs/guide/resources/agent-coverage.md
- Configuration (recall, exclusions): https://github.com/rtk-ai/rtk/blob/master/docs/guide/getting-started/configuration.md
- Troubleshooting: https://github.com/rtk-ai/rtk/blob/master/docs/guide/resources/troubleshooting.md
- Issues cited: #582, #2001, #2425, #2431, #3175, #3748, #3870, #4207, #4483, #4627, #4640 on https://github.com/rtk-ai/rtk/issues

Alternatives (READMEs, self-reported):
- caveman: https://github.com/JuliusBrussee/caveman
- Headroom: https://github.com/headroomlabs-ai/headroom
- context-mode: https://github.com/mksglu/context-mode (LICENSE: Elastic License 2.0)
- claude-code-router: https://github.com/musistudio/claude-code-router
- ccusage: https://github.com/ccusage/ccusage
- Serena: https://github.com/oraios/serena
- squeez: https://github.com/claudioemmanuel/squeez · lean-ctx: https://github.com/yvgude/lean-ctx · ctk: https://github.com/AbhayShalghar/ctk
- Discovery only (secondary, not relied on for claims): https://tokenade.net/en/comparisons/alternatives/rtk-alternatives
