# Skills

A personal founder's workflow for Claude Code: a set of skills packaged as a plugin, plus the wider Claude Code customisation that a plugin can't carry.

## Language

**Plugin**:
The installable Claude Code package this repo publishes, holding everything Claude Code lets a plugin ship.
_Avoid_: package, extension, bundle

**Repo setup**:
The part of the customisation a plugin can't carry (user-level settings, external tools, global instructions), applied through scripts or manual steps kept in this repo.
_Avoid_: dotfiles, config, install

**Workflow**:
The founder's loop the skills chain together, from a raw idea through validation, build and launch, to a product in use.
_Avoid_: process, pipeline, flow, dev loop

**Track**:
One of the four segments of the Workflow: Discover, Build, Launch, Operate. Every skill names the Track and stage it serves.
_Avoid_: phase, module, area

**Stage**:
A step inside a Track with one trigger, one artifact handed on, and a known place for the human.
_Avoid_: step, phase

**Inspiration**:
An external skill collection this effort borrows ideas from, credited but not depended on.
_Avoid_: upstream, source, dependency

**Predecessor**:
The owner's earlier skills repo (ivanhxlabs/skills), studied for lessons on what made it messy rather than for ideas to borrow.
_Avoid_: old version, v1, legacy

**Token-saving lever**:
A setting, model choice or habit that cuts the tokens a session spends, built into Claude Code or applied by the Repo setup.
_Avoid_: token-saving tool, optimizer, compressor

**Verdict**:
The outcome Verify posts on a pull request, PASS, PASS+NOTES or FAIL, from an agent that did not write the change, backed by evidence from running it on the real surface.
_Avoid_: review, approval, green CI

**Eval case**:
One prompt plus its graders, the unit the eval harness runs against the Plugin to check that a skill fires and behaves.
_Avoid_: test, spec, scenario

**Trigger pair**:
The two Eval cases every model-invoked skill carries: one prompt that should fire it, and one near-miss that should fire a sibling skill instead.
_Avoid_: routing test, positive/negative

**Hillclimb**:
A bounded tuning run of a skill against the Eval cases, one change per round, kept only when the held-out score improves.
_Avoid_: optimisation, prompt tuning, iteration

**Stage skill**:
A skill the human runs to enter a Stage, named by that Stage, that ends by naming the artifact it produced and the next skill to run.
_Avoid_: command, orchestrator, entry point

**Discipline**:
A skill that does one practice well (questioning, research, test-first building, review) and is reached by Stage skills or fired by the agent on its own, never run as a Stage.
_Avoid_: leaf, helper, sub-skill

**Handoff**:
The file that carries one session's conversation context (what was learned, tried and left half-done) to the next, pointing at the ticket for everything the tracker already holds.
_Avoid_: context dump, memory, summary
