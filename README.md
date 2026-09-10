# ShallGuard specification

This repository holds the documents that all ShallGuard implementations
share. ShallGuard connects written requirements to the code that makes them
true and to the tests that verify them. Each language has its own
implementation in its own repository under the
[ShallGuard organization](https://github.com/shallguard). The documents in
this repository define what every implementation must do in the same way,
and how people work together in every repository.

## Documents

| Document | Content |
|---|---|
| [WHY.md](WHY.md) | Why a team writes requirements when a coding agent writes the code. |
| [SPEC.md](SPEC.md) | The interchange format between the core and the language adapters, the adapter protocol, and the conformance rules. Draft 0.1. |
| [GLOSSARY.md](GLOSSARY.md) | The terms that every ShallGuard document uses. |
| [WRITING_STYLE.md](WRITING_STYLE.md) | The mandatory writing rules, Simplified Technical English, for every document in every repository. |
| [CONTRIBUTING.md](CONTRIBUTING.md) | How to contribute to any ShallGuard repository. Each implementation adds its own rules for its language. |
| [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md) | How people treat each other in the project. |
| [skills/shallguard/SKILL.md](skills/shallguard/SKILL.md) | The skill for a coding agent. It is the same for every language. One file per language, for example [rust.md](skills/shallguard/rust.md), gives the command and the anchor syntax. |

The implementation repositories link to these documents instead of a copy,
so that one change reaches every repository.

## The skill for a coding agent

The directory `skills/shallguard/` holds the ShallGuard skill. It teaches a
coding agent the requirements-first workflow, the evidence rules, and the
check as the gate. The file `SKILL.md` is the same for every language. The
agent reads the language file next to it, for example `rust.md`, for the
command and the anchor syntax. An implementation adds its language file
through a pull request to this repository.

There are two ways to install the skill:

- **The command of the implementation.** In Rust, `cargo shallguard
  install-skill` writes the skill of the installed release. The
  implementation embeds a copy of this directory.
- **The plugin.** This repository is a plugin for Claude Code and for
  Codex. It contains only the skill. In Claude Code, run
  `/plugin marketplace add shallguard/spec` and then
  `/plugin install shallguard@shallguard`. In Codex, add this repository
  as a marketplace and install the plugin `shallguard` from the `/plugins`
  browser. The plugin follows the version of this repository, not the
  version of one implementation.

The files `plugin.json`, `.claude-plugin/plugin.json`,
`.claude-plugin/marketplace.json`, and `.agents/plugins/marketplace.json`
describe the plugin. Keep the `version` in the two `plugin.json` files and
`metadata.version` in `SKILL.md` the same.

## Status

The specification is a draft for discussion. Section 11 of
[SPEC.md](SPEC.md) lists the open decisions. Each decision has a
[discussion](https://github.com/shallguard/spec/discussions) where you can
reply. The `schemas/` and `fixtures/` directories of the conformance suite
do not exist yet.

## Links

- Website: <https://shallguard.com>
- Rust implementation: <https://github.com/shallguard/shallguard-rs>
