# How to evaluate skills

Research for [#6](https://github.com/iamivanhx/skills/issues/6), a child of the map [#1](https://github.com/iamivanhx/skills/issues/1). Gathered 2026-10-19 against Claude Code 2.1.303.

**Question.** What options exist for evaluating skills and Plugins, and how do they work? What is each one good for, what does it cost, and what is the smallest useful eval setup for a personal Plugin?

**Short answer.** There are four tools, and they answer different questions:

- `claude plugin validate` checks that the files are well formed.
- `claude plugin eval` checks behaviour: does the skill fire, and does it produce the right outcome compared with no Plugin?
- `/skill-doctor` reports on real-world usage: which skills cost context but never run.
- skill-creator is a skill-at-a-time iteration loop, and the place to tune a description.

For a personal Plugin, run `validate` in CI. Keep a small `claude plugin eval` suite in `evals/` with a trigger case per skill and a few near-miss negatives, and run it locally when a skill changes. Check `/skill-doctor` now and then. Of the Inspirations, only ponytail runs a real behavioural benchmark. mattpocock tests by hand, and pstack runs blinded A/B tests by hand.

---

## 1. `claude plugin validate` (structure)

- Checks a Plugin or marketplace manifest, or the skills, agents and commands in a directory, for syntax and schema errors. It does not check behaviour. Sources: [cli-reference, plugin validate](https://code.claude.com/docs/en/plugins/cli-reference#plugin-validate), and the intro of [plugin-evals](https://code.claude.com/docs/en/plugin-evals).
- Flags: `--strict` turns warnings into failures. `--json` needs v2.1.259 or later.
- Exit codes: 0 means it passed, 1 means it failed, 2 means an unexpected error.
- The docs also suggest `claude plugin validate .claude/skills` to catch malformed skill frontmatter (v2.1.233 or later). Source: [skills, Skill not triggering](https://code.claude.com/docs/en/skills).
- **Cost:** free and instant, with no model calls and no credentials. It belongs in CI on every push.

## 2. `claude plugin eval` (behaviour)

Primary source: [Test plugins with evals](https://code.claude.com/docs/en/plugin-evals). Flags are also in [cli-reference, plugin eval](https://code.claude.com/docs/en/plugins/cli-reference).

### What it measures

- "Each case is a realistic prompt plus one or more graders. A grader is a pass/fail check on what Claude produced, such as a regex over the reply, whether a particular tool was called, or a rubric that a second model judges the reply against."
- It covers three things:
  - **Triggering:** a `tool_used` grader with `tool: Skill`.
  - **Outcome:** `regex`, `llm`, `file_exists` and `baseline` graders.
  - **Process:** `tool_used` and `tool_order` graders.
- **Tokens and time are reported but not graded.** The summary shows a list-price COST, and the JSON has `costUsd` and `durationSeconds`.
- **No-plugin baseline.** Each case runs with the Plugin (WITH) and without it (W/OUT), and reports the difference, Δ. Δ is "what the plugin contributed".
  - In a two-arm run, `tool_used: Skill` graders are left out of the score, along with plugin-declared mock graders and `arm: with-only` graders.
  - Under `--ablation none` nothing is left out.
- **Scoring:** "each case runs three times by default. A run's score is the fraction of its graders that passed, weighted if you set weights, and the case's score is the mean across its runs. A case passes when its score meets the `--threshold`, `1.0` by default."
  - The only variance handling is repeated runs. There are no confidence intervals or flakiness statistics.
- "There are no custom-code graders."
- **LLM judge:** an `llm` grader passes if the judge votes PASS in at least 2 of 3 votes. The judge defaults to the background-task model and can be changed with `--judge-model`.

### Suite format

The suite lives in `evals/` at the Plugin root. To move it, set `"experimental": { "evals": "..." }` in `plugin.json` or pass `--eval-dir`. Each case is a directory:

```
evals/
├── first-case/
│   ├── prompt.md          # frontmatter: case fields; body: the prompt
│   ├── graders/
│   │   ├── criteria.md    # frontmatter: type + options; body: rubric or pattern
│   │   └── skill-fired.md
│   └── case.yaml          # optional: only for context.* fields
└── results/               # written by each run; add to .gitignore
```

A `prompt.md` and a trigger grader, verbatim from the docs:

```markdown
---
max_turns: 10
allowed_tools: [Read, Glob, Grep, Skill]
---

Write me a commit message for this change: I renamed getUser to fetchUser and updated the three call sites.
```

```markdown
---
type: tool_used
tool: Skill
input_match: '"skill"\s*:\s*"(?:[\w-]+:)?your-skill-name"'
---
```

- **`prompt.md` frontmatter:**
  - Fields: `name`, `description`, `tags`, `plugins`, `runs` (default 3, range 1 to 50), `expected_outcome`, `model`, `max_turns` (default 10), `timeout_seconds` (default 300), `allowed_tools`, `append_system_prompt` and `env` (keys must be `EVAL_*`).
  - "An unknown key is an error."
- **`case.yaml`** is only for:
  - `context.scaffold_script`, which seeds the workspace and runs only with `--scaffold`.
  - `context.history_file`, which resumes a recorded conversation.
  - `context.add_dirs`.
- **Never-called check:** to assert that a tool was never called (for example, the skill must not fire), set `min: 0` and `max: 0` on a `tool_used` grader.
- **Mock MCP servers** go in `evals/mocks/<server>/<tool>.md`. Real servers start only with `--allow-real-servers`.
- **Writing the suite for you:** `claude plugin eval init [name]` interviews you, proposes cases and graders, tries them, and writes the files. `--bare` writes a blank template instead, which is the form for CI.

### Sandbox

The source for this section is plugin-evals, "What a run can access" and "How runs are isolated".

- **Each run is isolated.** It gets "a temporary home directory, working directory, and Claude Code configuration, and the agent under test runs there as a `claude -p` child process with only your plugin loaded."
- **Nothing of yours loads.** "Your user settings, hooks, CLAUDE.md files, MCP servers, other installed plugins, memory, and skills are absent." No project `.claude/`, `CLAUDE.md` or `.mcp.json` loads either.
- **The run can't see the suite.** "A run can't read the eval directory."
- **The Artifact tool is off.**
- **Tool permissions:**
  - Runs never stop to ask for permission.
  - The default tool set is read-only: Read, Glob, Grep, Skill, Agent, AskUserQuestion, TodoWrite and the Task\* tools.
  - Bash, Write, Edit, WebFetch and WebSearch are removed unless you grant them with `--allow-tools`. Bash runs under the OS sandbox.
- **It is not a security boundary.** The run still happens "on your machine, as you". The isolation "isn't a boundary against the plugin's own code". The first run in a directory asks you to trust it, and `--trust-plugin` answers that prompt in CI.

### Report and JSON

- **Output files:** each run writes `results/<timestamp>/aggregate-result.json` and a self-contained `report.html`. `--output-dir` changes where they go.
- **Publishing:** with a claude.ai login, the report is published as a private artifact unless you pass `--no-publish`.
- **Summary line:** `CASE  WITH  W/OUT  Δ  RUNS  COST  NOTES`.
- **JSON:** `--json [path]` writes the result document (`schemaVersion: 1`, camelCase). The fields the docs name are:
  - `partial` and `partialReason`
  - `aggregates.{overallScore, casesPassed, casesTotal, meanDelta}`
  - `cases[].aggregates.{score, delta}`
  - `cases[].arms.{with,without}[]`
  - `costUsd`, `durationSeconds` and `claudeVersion`
- The docs give no full JSON sample, so the per-grader shape is unconfirmed.
- **Exit codes:**
  - 0 means every case met the threshold.
  - 1 means a case fell below the threshold, or there was a load, start or auth problem.
  - 2 means the run is partial (the `--max-cost-usd` ceiling was hit, or a credential was rejected).
  - 130 means interrupted, and 143 means terminated.
  - Δ never changes the exit code.

### CI

This is the verbatim example from plugin-evals, "Run evals in CI":

```bash
claude plugin eval . --trust-plugin --json results.json --threshold 0.8 \
  --model claude-sonnet-5 --judge-model claude-haiku-4-5 --no-publish --max-cost-usd 20
```

- **Requirements:** the runner needs Claude Code installed and credentials in the environment, such as `ANTHROPIC_API_KEY`. The docs have no GitHub Actions workflow for eval, and they don't say whether `CLAUDE_CODE_OAUTH_TOKEN` (subscription auth) works.
- **Keeping cost down, from the docs:** give every-change suites only graders that don't call a judge, and pass `--ablation none` when you don't need Δ.
- **While iterating:** run one case with `claude plugin eval . --case <name> --runs 1 --ablation none`.

### Cost

- "Every eval run and every judge grader is a real model call on your account."
- One case at the defaults is six agent runs (3 with the Plugin and 3 without), plus 3 judge calls for each `llm` grader in each run.
- The docs' example spends $0.41 on one case.
- Requires v2.1.269 or later. If the command is switched off for your account, it reports "early access" or "unavailable" and exits 1.

## 3. `/skill-doctor` (real-world usage)

- "Show what each of your skills costs in context and how often it gets used, so you can find skills to turn off." Sources: [commands](https://code.claude.com/docs/en/commands), and [skills, Find unused skills](https://code.claude.com/docs/en/skills).
- **What the report covers:**
  - The context cost of each skill and how often it is invoked.
  - Skills that are listed but have never been invoked, and where to turn them off.
  - Plugins you haven't used recently.
  - Bundled and enterprise skills are excluded.
- **Where it shows:** in the `/plugin` Stats tab, or as text under `-p`.
- **Requirements:** v2.1.252 or later and feature-flag fetching. It is not available over Remote Control.
- **Usage data stays local.** It is recorded only on your machine: "Claude Code doesn't report a plugin's usage back to its author." Source: [plugins/measure](https://code.claude.com/docs/en/plugins/measure).
- **Related command:** `claude plugin details <name>` prints a Plugin's components and its projected token cost, split into always-on and on-invoke.
- **What it is for:** pruning, and trimming context cost. It does not test behaviour. It does tell you which skills are dead weight in practice. It costs nothing to run.

## 4. skill-creator (iterate on one skill)

Anthropic's plugin `skill-creator@claude-plugins-official` ([source](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/skill-creator)). Read locally at `~/.claude/plugins/marketplaces/claude-plugins-official/plugins/skill-creator/skills/skill-creator/`.

- **Outcome loop:**
  - Write 2 or 3 realistic prompts to `<skill>/evals/evals.json`. Assertions are added later. `references/schemas.md` names them `expectations`, while SKILL.md says `assertions`, so the field name is inconsistent.
  - For each prompt, spawn two subagents in the same turn. One runs with the skill. The other runs without it (for a new skill) or with a snapshot of the old version (for an edited skill).
  - Record `total_tokens` and `duration_ms` from the task notification into `timing.json`.
  - A grader subagent (`agents/grader.md`) writes `grading.json`, where each entry has `text`, `passed` and `evidence`.
  - `scripts/aggregate_benchmark.py` writes `benchmark.json`, with pass rate, time and tokens per configuration as mean ± stddev, plus the delta. This is the "variance analysis".
  - An analyst pass flags assertions that pass either way (non-discriminating) and high-variance evals.
  - `eval-viewer/generate_review.py` opens an HTML viewer for human review, and the feedback is saved to `feedback.json`.
- **Blind A/B:** `agents/comparator.md` judges two outputs without knowing which version produced which, and `agents/analyzer.md` explains why the winner won.
- **Description (triggering) optimisation:**
  - About 20 queries, split between should-trigger and should-not-trigger. Near-miss negatives are the valuable ones.
  - `scripts/run_loop.py` splits them 60/40 into train and test sets. It runs each query 3 times through `claude -p` and proposes better descriptions, for up to 5 iterations.
  - It picks `best_description` by test score, not train score, to avoid overfitting.
  - **Caveat** (`scripts/run_eval.py`): it does not load the real skill. It writes a temporary `.claude/commands/<name>-skill-<id>.md` holding the candidate description, and counts a trigger when Claude calls `Skill` or `Read` on that file. It tests the description in isolation, not the installed Plugin.
- **`scripts/quick_validate.py`** checks frontmatter. Its allowed keys are only `name`, `description`, `license`, `allowed-tools`, `metadata` and `compatibility`, so it would reject Claude Code fields such as `when_to_use`. That makes it a poor fit for Claude Code Plugins. Use `claude plugin validate` instead.
- **Not compatible with `claude plugin eval`.** The docs say: "neither tool reads the other's case files." For plugin skills, the docs point to `claude plugin eval`, because `skillOverrides` (the way to switch a skill off for a baseline) doesn't apply to plugin skills ([skills, Evaluate and iterate on a skill](https://code.claude.com/docs/en/skills)).
- **What it is for:** a skill whose quality needs a human to judge, and one-off tuning of a description. The workflow is heavyweight and conversational, with a browser viewer and subagents. Every run costs model calls, and nothing in it is built for CI.

## 5. How the Inspirations test

All three are MIT licensed. Commits are pinned below.

| Inspiration | Structural checks | Behavioural evals | Trigger tests |
|---|---|---|---|
| [mattpocock/skills](https://github.com/mattpocock/skills/tree/49dd158d1076134a641b33efb035946536778336) | `claude plugin validate . --strict`, by hand per CLAUDE.md. Not in CI. | None. "There is no automated eval. You check by running the document by hand" (`docs/productivity/writing-for-agents.md`). Dogfoods through `scripts/link-skills.sh`. Every skill's docs page has an "It's working if" list of checks the user can make. | None |
| [cursor/plugins pstack](https://github.com/cursor/plugins/tree/d457567332aa6989e753ea289f30ea2bfbc80bc0/pstack) | CI: Ajv schema validation of manifests, versions in sync, frontmatter `name` equals the folder name, generated Claude Code tree up to date (`.github/workflows/validate-plugins.yml`). | A manual playbook (`skills/poteto-mode/playbooks/eval.md`): a blinded A/B with sanitised directories (no "eval/test/judge" words visible), N candidates, one judge from a different model family, grading from transcripts, and a human reads every output. "A fix written from one bad session tends to overfit that session." | None |
| [DietrichGebert/ponytail](https://github.com/DietrichGebert/ponytail/tree/09c434ab73ca43761c5d8b0c5d341926c2fba710) | CI with no API key: unit tests for graders and hooks, checks that rule copies match, version checks, and `run.py --selftest` (each task's good reference must pass and its bad reference must be caught). | A local agentic benchmark (`benchmarks/agentic/run.py`): 39 tasks run through headless `claude -p` with `--plugin-dir`. Arms compared: baseline, ponytail, a candidate and rival styles. Hidden deterministic checks, plus LLM judges that each have a self-test, and paired sign tests between arms. Contributors must show 3 arms with at least 6 runs each. | None (always on through a SessionStart hook) |

Lessons from ponytail that carry over to any suite:

- Prove each grader can tell good from bad with a good and a bad reference, before spending money on runs.
- Watch for baseline contamination. One of its runs had to be thrown out because the Plugin's SessionStart hook fired in the baseline arm too. `claude plugin eval` sidesteps this by giving each run its own home directory and loading only the Plugin under test.

## 6. Comparison

| Approach | Good for | Setup cost | Run cost |
|---|---|---|---|
| `claude plugin validate --strict` | Broken manifests, malformed frontmatter | Minutes | Free; CI on every push |
| `claude plugin eval` | Triggering, outcome against a no-plugin baseline, regressions after model releases | About an hour for a first suite (`eval init` drafts it) | Model calls: 6 runs per case at defaults, cut with `--runs 1 --ablation none`, capped with `--max-cost-usd` |
| `/skill-doctor`, `claude plugin details` | Dead skills, context cost | None | Free |
| skill-creator | Deep work on one subjective skill, description tuning | None (install the plugin) | Many subagent runs plus human review; manual |
| Blinded A/B (pstack style) | "Is version B really better?" on subjective output | High; manual | High; manual |

## 7. Smallest useful eval setup for a personal Plugin

1. **CI on every push, free.** A GitHub Action runs `claude plugin validate . --strict`, plus any repo-specific checks such as frontmatter `name` equal to the folder name.
2. **`evals/` at the Plugin root, with `evals/results/` gitignored.** For each skill:
   - **Trigger case.** A realistic, natural-language prompt that doesn't name the skill. Grade it with `tool_used: Skill` and an `input_match` on the skill's name.
   - **Near-miss negative.** A prompt that shares keywords with the skill but shouldn't trigger it. Grade it with `tool_used: Skill`, `min: 0`, `max: 0`.
   - **One outcome grader**, but only where the outcome is objective. Prefer `regex`, `tool_order` or `file_exists`, and keep `llm` graders for short outputs.
   - Bootstrap the suite with `claude plugin eval init`, then trim what it writes.
3. **Run it locally, not on every push.**
   - While editing a skill: `claude plugin eval . --case <skill>* --runs 1 --ablation none`.
   - Before tagging a release or after a model change: `claude plugin eval . --max-cost-usd <n>`, with the default 3 runs and the baseline.
   - Optionally, a `workflow_dispatch` GitHub Action with `ANTHROPIC_API_KEY`, `--trust-plugin --no-publish --json`, and a cost cap.
4. **Check `/skill-doctor` monthly** to drop skills that never fire, and `claude plugin details` when token cost matters.
5. **Use skill-creator only on demand.** Bring it in when a skill under-triggers and its description needs tuning (`run_loop.py`), or when a subjective skill needs a human-reviewed A/B.

## Gaps and caveats

- **Help text not read locally.** The verbatim `claude plugin eval --help` could not be captured here, because the worktree agent's Bash guard refuses any command containing the word `eval`. The flags above come from the docs, and `claude plugin --help` was read locally. Run `claude plugin eval --help` in a normal terminal to confirm.
- **Interactive Workflow skills are hard to evaluate.** Runs never ask for permission and are headless (`claude -p`). Skills that grill the user over several turns (grilling, domain modeling) probably need `context.history_file` to resume a recorded conversation, or a grader that only checks the first turn. The docs do not show what `AskUserQuestion` does in an eval run.
- **Repo setup is invisible to evals.** Eval runs load no user `CLAUDE.md`, settings or other plugins, so a skill that depends on the Repo setup's global instructions will look worse in evals than it does in daily use.
- **Unconfirmed:** the full JSON result schema, a GitHub Actions example, and subscription (OAuth) auth for eval in CI.
- **Not read:** the external pages linked from the skills docs, [agentskills.io evaluating skills](https://agentskills.io/skill-creation/evaluating-skills) and [platform best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices).
