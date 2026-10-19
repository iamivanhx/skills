# Skills chain downward through the Skill tool; the human routes between Stages

A Plugin with one skill per Stage needs a rule for how skills reach each other. Two Inspirations offered opposite patterns: mattpocock/skills chains through the Skill tool and keeps user-invoked skills unreachable from other skills, while pstack makes nearly every skill user-invoked and has a router skill read the others by file path. We chose the first: every Stage skill is user-invoked (`disable-model-invocation: true`) and every Discipline is model-invoked; a Stage skill reaches a Discipline only with "Call the Skill tool with X", never by reading a `SKILL.md` by path; no skill reaches a Stage skill; and there is no router skill. Between Stages, the human routes: each Stage skill ends with a closing handoff line naming its artifact and the exact next skill to type, and `/clear` between Stages is part of the Workflow, so a Stage skill never runs its successor itself.

## Considered Options

- **A router skill** (pstack's `/poteto-mode`, the Predecessor's 17 KB router): rejected because the Predecessor showed it becomes a second copy of the Workflow that drifts from the skills, and because the Skill tool refuses user-invoked skills, so a router can only read them by path, which the Claude Code docs do not support.
- **Read-by-path chaining**: rejected for skills (undocumented, and it sidesteps the invocation flag) but kept for shared reference files under `reference/` at the Plugin root, which is the one home for text several skills need.
- **A handoff file as the carrier of Workflow state**: rejected. The tracker carries Workflow state (ticket, holder, PR, Verdict); the Handoff carries only conversation context across a `/clear`.

## Consequences

- Fresh-context work (review and Verify by a non-author, the per-ticket implementer) runs in Plugin subagents that preload Disciplines through `skills:`, never through `context: fork`, since a user-invoked skill cannot be preloaded anyway.
- Shared text that two Stage skills need cannot live in either skill; it lives in `reference/`.
- Disciplines pay the per-turn description cost and need a Trigger pair; Stage skills pay nothing and need none.
