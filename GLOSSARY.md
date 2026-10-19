# Skills

A personal software-development workflow for Claude Code: a set of skills packaged as a plugin, plus the wider Claude Code customisation that a plugin can't carry.

## Language

**Plugin**:
The installable Claude Code package this repo publishes, holding everything Claude Code lets a plugin ship.
_Avoid_: package, extension, bundle

**Repo setup**:
The part of the customisation a plugin can't carry (user-level settings, external tools, global instructions), applied through scripts or manual steps kept in this repo.
_Avoid_: dotfiles, config, install

**Workflow**:
The end-to-end development loop the skills chain together, from a loose idea to shipped code.
_Avoid_: process, pipeline, flow

**Inspiration**:
An external skill collection this effort borrows ideas from, credited but not depended on.
_Avoid_: upstream, source, dependency

**Predecessor**:
The owner's earlier skills repo (ivanhxlabs/skills), studied for lessons on what made it messy rather than for ideas to borrow.
_Avoid_: old version, v1, legacy

**Token-saving tool**:
A third-party tool that cuts the tokens a session spends, adopted into the Plugin or Repo setup after evaluation.
_Avoid_: optimizer, compressor
