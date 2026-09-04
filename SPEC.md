# ShallGuard interchange format

Status: draft 0.1, proposed for discussion.

This document specifies the contract between the language-independent core
of ShallGuard and the language adapters. It is for a person who writes an
adapter for a new language, or who changes the core. The
[glossary](GLOSSARY.md) defines each technical term. Requirement statements
keep their RFC 2119 form with SHALL, SHALL NOT, and MAY.

## 1. Purpose

ShallGuard links a requirement to the code that makes it true and to the
test that proves it. The link model is the same for every language. The way
a language marks an anchor is not. This specification splits the tool into
two parts and defines the contract between them:

- A **language adapter** reads the source code of one language. It finds
  the anchors and the tests. It writes an inventory file.
- The **core** reads the requirement documents, the inventory files, the
  baseline, and the configuration. It joins them, applies the check rules,
  and writes the findings. The core does not read source code.

There is one core and there are many adapters. An adapter is small. It
parses one language and writes one JSON file. The document grammar, the gap
rules, the ratchet, and the reports live in the core. This document defines
them once.

## 2. Terms

- **Requirement**: a statement with a stable ID in a Markdown document. The
  statement uses SHALL, SHALL NOT, or MAY. An example ID is `REQ-AUTH-003`.
- **Enforcement anchor**: a mark in source code that links the code to a
  requirement.
- **Verification anchor**: a mark on a test that links the test to a
  requirement.
- **Inventory**: the machine-readable list of anchors and tests that an
  adapter found in one source tree.
- **Gap**: a broken or missing link that the check reports.
- **Baseline**: the committed record of the gaps that existed when a
  repository adopted ShallGuard. The baseline can only become smaller.
- **Finding**: one result of the check. It has a location, a kind, and a
  message.
- **Oracle class**: the judgment of an adapter about whether a test can
  fail. Section 5 defines it as a planned extension.

## 3. Architecture and process contract

The core and the adapters exchange files. This diagram shows the flow:

```
requirement documents (Markdown)     source code (any language)
        |                                     |
        |                            language adapter
        |                                     |
        |                          inventory (JSON)
        +--------------+----------------------+
                       |
                     core
        +--------------+----------------------+
        |              |                      |
    findings      baseline I/O            reports
```

The core starts each adapter as a subprocess. The contract is the file
format of section 4 and the command-line protocol of section 8. An
implementation can embed an adapter through a foreign function interface as
an optimization. That is not the contract. With the subprocess contract, a
person can write an adapter in the language of the adapter and distribute
it through the package manager of that language.

A workspace can hold several languages. The core accepts one or more
inventories in one check run and merges them. Two adapters never scan the
same file.

## 4. Inventory format `shallguard-inventory/1`

An adapter writes one JSON document per run. The file is UTF-8 with LF
line endings.

### 4.1 Top level

The top-level object has these fields:

| Field | Type | Meaning |
|---|---|---|
| `schema` | string | Exactly `"shallguard-inventory/1"`. |
| `generator` | object | `{ "name": string, "version": string }`. |
| `language` | string | The language identifier in lower case: `"rust"`, `"python"`, `"go"`, `"typescript"`, `"java"`, `"kotlin"`, or `"csharp"`. |
| `root` | string | The scanned root, as the core gave it to the adapter. All paths in the file are relative to it and use `/` as the separator. |
| `capabilities` | array of string | What the adapter computed. The defined values in version 1 are `"anchors"`, `"tests"`, and `"oracle-class"`. |
| `enforcement_anchors` | array | See section 4.2. |
| `verification_anchors` | array | See section 4.3. |
| `diagnostics` | array | See section 4.4. These are problems of the adapter, not findings of the check. |

### 4.2 Enforcement anchor entry

Each entry has these fields:

| Field | Type | Meaning |
|---|---|---|
| `requirement` | string | The requirement ID, exactly as written in the code. |
| `path` | string | The relative file path. |
| `line` | integer | The line of the anchor mark. The first line is 1. |
| `item` | string | A stable, readable path of the anchored item in the idiom of the language, for example `auth::session::SessionStore::create`, `auth.session.SessionStore.create`, or `pkg.Session.Create`. The core does not interpret it. The core uses it only for identity and display. |
| `kind` | string | How the language expresses the anchor: `"attribute"`, `"decorator"`, `"annotation"`, `"comment"`, or `"macro"`. The set is open. The core passes an unknown value through. |

### 4.3 Verification anchor entry

Each entry has all fields of section 4.2 and one more field:

| Field | Type | Meaning |
|---|---|---|
| `test` | object | `{ "ignored": bool, "oracle": string, "oracle_reason": string?, "suppression": string? }` |

The field `ignored` is true when the test exists but does not run in a
normal test invocation. Examples are `#[ignore]` in Rust, a pytest skip
mark without a condition, `@Disabled` in JUnit, and `t.Skip` at the start
of a Go test function where the adapter can detect it. The fields `oracle`
and `suppression` follow section 5. An adapter that does not compute
oracle classes writes `"unknown"` into `oracle`.

### 4.4 Diagnostics entry

Each entry has the form
`{ "path": string, "line": integer?, "code": string, "message": string }`.
The defined codes in version 1 are `parse-error`, `ambiguous-anchor`, and
`unreadable-file`. The core shows diagnostics in the reports. The core
treats a `parse-error` on a file that contains anchors as a failure of the
check, because a file that the adapter cannot read cannot prove anything.

### 4.5 Determinism

The same source tree and the same adapter version give a byte-identical
inventory. These rules make that possible:

- Arrays are sorted by `requirement`, then `path`, then `line`, then
  `item`.
- Object keys follow the field order of this specification.
- The file has no timestamps, no absolute paths, and no data from the
  environment.

A byte-identical inventory is easy to compare in a review and easy to hash
for a future review capsule.

## 5. Oracle classes (planned extension)

This section is a planned extension. It comes from the evidence-floor design
of the Rust implementation, which is not released yet. An adapter for
version 1 can write `"unknown"` for every test and stay conformant.

The adapter judges, from the syntax alone, whether each verification test
can fail. The judgment uses the syntax tree only. It does not run the test
and it does not follow data flow. The classes are:

| Value | Meaning |
|---|---|
| `"present"` | At least one non-trivial failure path exists. |
| `"weak"` | Failure paths exist, but every one is weak. An example is an expected-panic mark without an expected message. |
| `"vacuous"` | No non-trivial failure path exists. |
| `"suppressed"` | The author opted out explicitly. The field `suppression` holds the declared class: `"panic"`, `"compile"`, or `"external"`. |
| `"unknown"` | The adapter does not implement oracle classification, or the construct is outside its rules. |

The conservatism rule is normative. When the adapter does not fully
understand a test body, it reports `present` or `unknown`, never `vacuous`
or `weak`. A false `vacuous` blocks honest work. A false `present` only
keeps the current state. Each adapter maps the failure idioms of its
language, such as assert statements, exception expectations, `t.Fatal`, and
xUnit asserts, to these classes in its own conformance fixtures. The classes
and the conservatism rule are fixed here.

The core, not the adapter, decides what a class means for the check. The
core demotes a sole vacuous citation, warns about a redundant vacuous
citation, and applies a strict area policy. Adapters report. The core
judges.

## 6. Document grammar, baseline, and findings

The Rust implementation defines the document grammar and the baseline
today. This repository will take them over as normative documents without a
change of substance. The findings format is new.

- **Document grammar.** The requirement ID pattern `REQ-<AREA>-<NNN>` and
  the user story pattern `US-<AREA>-<NNN>`. The rules for SHALL, SHALL NOT,
  and MAY. The evidence marks with the keywords `[test]`, `[e2e]`,
  `[review]`, and `[pending]` as the canonical form and the emoji as
  accepted aliases. The citation syntax. The normalization rules of the
  `fmt` command.
- **Baseline.** The committed TOML file with its `schema` version field, the
  registry of gap kinds, and the ratchet rule that only removes entries. The
  gap kinds registered in version 1 are `enforcement-anchor`,
  `verification-anchor`, and `evidence-citation`. The kinds
  `vacuous-evidence` and `weak-evidence` are reserved for the evidence
  floor. A core that meets specification version N ignores no registered
  kind and invents no unregistered kind. A future `extend` operation will
  add newly detected kinds to a baseline during a schema upgrade. The Rust
  implementation does not have that operation yet.
- **Findings.** One JSON object per line:
  `{ "kind": string, "severity": "error"|"warning", "requirement": string?, "path": string?, "line": integer?, "message": string }`.
  The lines are sorted with the determinism rules of section 4.5. The
  readable report text stays implementation-defined. The JSON stream is the
  contract for CI and for future services. The Rust implementation prints
  text only today, so this format is new work there.

## 7. Requirements

The areas are `SIF` for the inventory, `ADPT` for the adapter protocol, and
`CONF` for conformance. This is the initial set. The numbering becomes
final after the review.

- **REQ-SIF-001** — An adapter SHALL write an inventory that validates
  against the `shallguard-inventory/1` schema.
- **REQ-SIF-002** — An adapter SHALL write byte-identical inventories for
  identical inputs, as section 4.5 defines.
- **REQ-SIF-003** — An adapter SHALL NOT read the requirement documents,
  the baseline, or the configuration of the core.
- **REQ-SIF-004** — An adapter SHALL report every anchor whose requirement
  ID it can read, including an ID that no document defines; the core, not
  the adapter, SHALL decide whether an ID is undefined.
- **REQ-SIF-005** — An adapter that reports oracle classes SHALL apply the
  conservatism rule of section 5.
- **REQ-SIF-006** — An adapter SHALL write a `parse-error` diagnostic for a
  file it cannot parse and SHALL NOT skip the file in silence.
- **REQ-ADPT-001** — An adapter SHALL implement the command-line protocol
  of section 8.
- **REQ-ADPT-002** — An adapter SHALL declare every schema version it
  supports through the `protocol` command.
- **REQ-ADPT-003** — The core SHALL refuse an inventory whose `schema`
  value it does not support, with an error that names both versions.
- **REQ-CONF-001** — An implementation SHALL pass every conformance
  fixture of its role, adapter or core, for the specification version it
  claims.
- **REQ-CONF-002** — Each conformance test in an implementation SHALL carry
  a verification anchor to the specification requirement it proves.

REQ-CONF-002 makes this repository a ShallGuard requirement document. Every
implementation anchors its conformance tests to these IDs with the same
mechanism that it implements.

## 8. Adapter command-line protocol

An adapter is an executable with two commands:

- `<adapter> protocol` prints the supported schema versions as JSON, for
  example `{ "schemas": ["shallguard-inventory/1"] }`, and exits with
  code 0.
- `<adapter> scan --root <dir> --output <file>` writes the inventory. The
  exit code is 0 on success. An inventory with diagnostics is still a
  success. The exit code is 2 on an invocation error and 3 on an internal
  error.

The configuration of the core names each adapter explicitly, for example
`[[adapter]] command = "shallguard-adapter-python"`, with an optional
`roots` filter. Version 1 has no search of the `PATH` and no automatic
detection.

An adapter writes only the output file and log lines on standard error. It
makes no network calls.

## 9. Conformance suite layout

The specification repository has this layout:

```
spec/
  SPEC.md                      this document, as a requirement document
  schemas/
    inventory.schema.json
    findings.schema.json
    baseline.schema.md         a prose schema, because TOML has no
                               standard schema language
  fixtures/
    adapter/<case>/
      input/                   a small source tree in one language, or a
                               language-neutral input when the case tests
                               the protocol and not the parser
      expected-inventory.json
    core/<case>/
      docs/                    requirement documents
      inventories/             one or more inventory files
      baseline.toml            optional
      expected-findings.jsonl
```

An adapter fixture with language-specific input lives in the repository of
that adapter and follows this layout. A protocol fixture and a core fixture
live in this repository. A fixture exists for every SHALL before any
implementation marks that SHALL as implemented. This repository follows the
same evidence rules as the implementations.

## 10. Versioning and compatibility

- A schema identifier carries a major version, for example
  `shallguard-inventory/1`. A change that breaks a field increases the
  major version.
- An added optional field does not increase the major version. A consumer
  ignores an unknown field and does not fail on it. This rule lets an
  adapter and the core release independently.
- A new gap kind and a new oracle class are additive. They enter the
  registry in a minor revision of the specification. The `extend`
  operation of the baseline is the upgrade path for a repository that
  detects them for the first time.
- This repository tags each release. An implementation states the
  specification version it conforms to in its README and in the
  `generator.version` field.

## 11. Open decisions

The maintainer decides these points before the numbering becomes final.

1. **Where `fmt` and `check` live for a repository without Rust.** The
   document grammar belongs to the core. One answer is that the core binary
   ships `fmt` and `check` for every ecosystem, and each language package
   wraps the core binary together with its adapter. The other answer is a
   new document parser in each language. That answer breaks the one-core
   principle. The recommendation is to wrap the binary and to treat the
   core as the one distributed engine, in the way that ruff and biome do.
2. **Anchor syntax per language.** This draft fixes the model, not the
   surface syntax. A language without attributes, such as Go or C, needs an
   exact comment grammar, for example `// shallguard:enforces REQ-X-001`.
   The choice is one shared comment grammar in this specification, or
   freedom per adapter with only the inventory normalized. The
   recommendation is one shared comment grammar for languages without
   attributes, and native attributes or decorators where the language has
   them.
3. **The `unknown` oracle class in a hard area.** A strict reading says
   that a hard area demands classified evidence. A pragmatic reading says
   that `unknown` equals `present` until the adapters mature. The
   recommendation is that `unknown` behaves as `present` in version 1.
4. **Requirement areas across languages.** Can one requirement be enforced
   in Rust and verified in Python? The merged-inventory model permits it.
   The question is whether version 1 permits it or requires one area per
   language until the workflow is proven. The recommendation is to permit
   it, because a service with several languages is the target user.
5. **Naming.** The choice is `shallguard-inventory` as the schema name, or
   one project-wide name for all three formats. The recommendation is to
   keep `shallguard-inventory` and to add no separate brand.
