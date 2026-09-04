# ShallGuard glossary

This page defines the terms that the ShallGuard documents use in every
repository. Each term has one meaning. The documents use the same word for
the same thing. A language implementation adds its own terms, such as
"crate" for Rust, in its own glossary.

## General terms

| Term | Meaning |
|---|---|
| Repository | A Git directory that holds the source code and the documents of one project. |
| Merge request (MR) | A proposed change to a repository. A reviewer examines the change before it goes into the main branch. GitHub calls this a pull request. |
| Continuous integration (CI) | An automated pipeline that builds and tests each proposed change. |
| Gate | A CI step that must pass before a change can merge. |
| Coding agent | A program that uses a large language model to write or change code. Examples are Claude Code, Codex, and Copilot. |
| Large language model (LLM) | A program that produces text from a prompt. ShallGuard uses an LLM only for advisory review, and every feature that needs an LLM is experimental. |
| Provider | The command-line program that gives ShallGuard access to an LLM. |

## ShallGuard terms

| Term | Meaning |
|---|---|
| Requirement | One numbered statement of what the software must do. It has the form `REQ-<AREA>-<NNN>` and uses the word SHALL. It lives in a Markdown document that the configuration selects. |
| Requirement document | A Markdown file that holds requirements. |
| Area | A group of requirements with one capability. The area name is the middle part of the requirement ID, for example `CLI` in `REQ-CLI-001`. |
| RFC 2119 | An internet standard that defines the words SHALL, SHALL NOT, and MAY for requirement statements. |
| Anchor | A mark in source code that names a requirement ID. An adapter finds anchors in the syntax of the code, or in a comment grammar that this specification defines for a language without attributes. |
| Enforcement anchor | An anchor on the code that makes a requirement true. In Rust it is the attribute `#[shallguard::enforces]` or the macro `shallguard::enforces_here!`. |
| Enforcement site | The code item or block that carries an enforcement anchor. |
| Verification anchor | An anchor on a test. It marks a test that gives evidence for a requirement. In Rust it is the attribute `#[shallguard::verifies]`. |
| Verification test | A test that carries a verification anchor. |
| Evidence | The proof that a requirement is true. Each requirement names its evidence class on its *Verified:* line. |
| Evidence class | One of four classes: `[test]` an anchored automated test, `[e2e]` an end-to-end or production validation, `[review]` a code review only, `[pending]` pending. |
| Evidence mark | The keyword that names an evidence class on a *Verified:* line, for example `[test]`. The emoji ✅, 🔬, 👁, and ⏳ are optional aliases of the four keywords. |
| Citation | The file and the symbol that a requirement names on its *Enforced:* or *Verified:* line. |
| Traceability | The link between a requirement, its enforcement anchors, and its verification anchors. |
| Core | The language-independent part of ShallGuard. It reads the requirement documents and the inventories, applies the check rules, and writes the findings. |
| Language adapter | The part of ShallGuard that reads the source code of one language and writes an inventory. |
| Inventory | The machine-readable list of anchors and tests that an adapter found in one source tree. |
| Check | The command that examines the traceability of every requirement and fails when a link is broken. |
| Finding | One result of the check. It has a location, a kind, and a message. |
| Gap | A requirement without an enforcement anchor, or a requirement without automated evidence. |
| Baseline | A committed file that lists the gaps that existed when a repository adopted ShallGuard. The baseline can only become smaller. |
| Ratchet | The rule that the number of gaps can only go down. The check rejects each new gap. |
| Hard area | An area with a policy that does not accept any gap in the baseline. |
| Oracle class | The judgment of an adapter about whether a test can fail. It is a planned extension. |
| Impact | The relation between a Git change and a requirement. The impact can be direct, transitive, or structural. |
| Coverage | Execution evidence from the coverage tools of a language. It shows if a verification test reached an enforcement site. Coverage does not prove that the code is correct. |
| Capsule | A bounded, reproducible bundle that holds one requirement, its code, its changes, and its evidence for a review. |
| Semantic review | An advisory judgment about a capsule. A person or an LLM gives the judgment. It is not part of the check. |
| Experimental feature | A feature that needs an LLM. It can change or go away in any release, also in a patch release. Its result is advisory, and the check never depends on it. |
| Verdict | The result of a semantic review for one requirement. A verdict is advisory. |
| Artifact | A machine-readable file that a ShallGuard command writes. Each artifact has a schema version. |
| Conformance fixture | A test input with its expected output. An implementation must reproduce the expected output to claim a specification version. |
