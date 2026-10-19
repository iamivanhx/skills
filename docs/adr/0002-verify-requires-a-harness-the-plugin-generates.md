# Verify requires a project Harness, and the Plugin ships the skill that builds it

Verify is the Build track's merge gate: a non-author proves a change on the surface a user touches and posts a Verdict. Claude Code already bundles `/run`, `/verify` and `/run-skill-generator`, which launch and drive an app and record a per-project recipe, and [Decide what Setup writes into a repo](https://github.com/iamivanhx/skills/issues/14) first chose to lean on them and ship no verification authoring. [Decide the skill inventory per Track and Stage](https://github.com/iamivanhx/skills/issues/15) reversed that: the Plugin ships `verify-harness`, a Stage skill carrying pstack's proven create and maintain passes, and `verify <PR>` stops when the repo has no Harness rather than falling back to the bundled recipes or a hand-rolled script.

## Considered Options

- **Lean on the bundled skills only**: rejected. They cover launch, drive and a recorded recipe, but not Doctor before driving, Cleanup that never eats evidence, the evidence standard, the Feature Map, or an upkeep loop that keeps the map honest as the app drifts. Those are the parts that turn "I ran it" into proof, and they are what pstack and the Work fork have shown to work.
- **A gate that degrades when no Harness exists**: rejected. A gate that drives the app from an improvised recipe is the "trust me, I ran it" the gate exists to kill. Stopping loudly costs one `verify-harness` run per repo; degrading silently costs the Verdict its meaning.
- **A separate generator skill**: folded into one Stage skill with two entry points (`verify` for a PR, `verify-harness` for the Harness), under the naming rule of ADR 0001, because create and maintain both edit only the Harness and never product code.

## Consequences

- The Harness lives in the product repo under `.claude/skills/`, named so the bundled `/run` and `/verify` also find it, so a product repo stays verifiable without the Plugin (see [Decide whether the Plugin must work in cloud sessions](https://github.com/iamivanhx/skills/issues/12)).
- Setup does not create the Harness: an empty repo has nothing to drive. The first `verify <PR>` on a repo sends the human to `verify-harness`.
- An unattended Verify on a repo with no Harness ends as FAIL with that reason, never as a pass.
- `verify` no longer reaches the built-in `run`; the Harness is the one lever.
