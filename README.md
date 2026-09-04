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
| [SPEC.md](SPEC.md) | The interchange format between the core and the language adapters, the adapter protocol, and the conformance rules. Draft 0.1. |
| [GLOSSARY.md](GLOSSARY.md) | The terms that every ShallGuard document uses. |
| [WRITING_STYLE.md](WRITING_STYLE.md) | The mandatory writing rules, Simplified Technical English, for every document in every repository. |
| [CONTRIBUTING.md](CONTRIBUTING.md) | How to contribute to any ShallGuard repository. Each implementation adds its own rules for its language. |
| [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md) | How people treat each other in the project. |

The implementation repositories link to these documents instead of a copy,
so that one change reaches every repository.

## Status

The specification is a draft for discussion. Section 11 of
[SPEC.md](SPEC.md) lists the open decisions. Each decision has a
[discussion](https://github.com/shallguard/spec/discussions) where you can reply. The `schemas/` and `fixtures/`
directories of the conformance suite do not exist yet.

## Links

- Website: <https://shallguard.com>
- Rust implementation: <https://github.com/shallguard/shallguard-rs>
