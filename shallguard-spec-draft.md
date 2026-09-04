# ShallGuard Interchange Format (SIF) — draft 0.1

Draft for the `shallguard/spec` repository. Written in the project's
documentation style (short sentences, defined terms, normative statements
only inside numbered requirements). Status: **Proposed — discussion draft.**

## 1. Purpose

ShallGuard links a requirement to the code that enforces it and to the test
that proves it. The link model is language-independent. The way a language
marks an anchor is not. This specification splits the tool into two parts
and defines the contract between them:

- A **language adapter** reads source code in one language. It finds
  anchors and tests. It emits an inventory file.
- The **core** reads requirement documents, inventory files, the baseline,
  and the configuration. It joins them, applies the check rules, and emits
  findings. The core does not read source code.

One core, many adapters. An adapter is small: it parses one language and
emits one JSON file. Everything else — document grammar, gap semantics,
the ratchet, reports — lives in the core and is defined here, once.

## 2. Terms

- **Requirement**: a normative statement with a stable ID in a Markdown
  document. Example ID: `REQ-AUTH-003`.
- **Enforcement anchor**: a mark in source code that links code to a
  requirement.
- **Verification anchor**: a mark on a test that links the test to a
  requirement.
- **Inventory**: the machine-readable list of anchors and tests that an
  adapter found in one source tree.
- **Gap**: a broken or missing link that the check reports.
- **Baseline**: the committed record of historical gaps. The baseline can
  only shrink, except through the explicit `extend` operation.
- **Finding**: one check result with a location, a kind, and a message.
- **Oracle class**: the adapter's syntactic judgment of whether a test can
  fail. See section 5.

## 3. Architecture and process contract

```
requirement docs (Markdown)          source code (any language)
        │                                     │
        │                            language adapter
        │                                     │
        │                          inventory (SIF JSON)
        └──────────────┬──────────────────────┘
                     core
        ┌──────────────┼──────────────────────┐
   findings        baseline I/O           reports
```

The core invokes each adapter as a subprocess. FFI embedding is a permitted
optimization, not the contract; the contract is the file format plus the
CLI protocol in section 8. This keeps adapters writable in their native
language and distributable through their native package manager.

Multi-language workspaces: the core accepts one or more inventories per
check run and merges them. Two adapters never scan the same file.

## 4. Inventory format (`shallguard-inventory/1`)

One JSON document per adapter run. UTF-8, LF line endings.

### 4.1 Top level

| Field | Type | Meaning |
| --- | --- | --- |
| `schema` | string | Exactly `"shallguard-inventory/1"`. |
| `generator` | object | `{ "name": string, "version": string }`. |
| `language` | string | Lowercase language identifier: `"rust"`, `"python"`, `"go"`, `"typescript"`, `"java"`, `"kotlin"`, `"csharp"`. |
| `root` | string | The scanned root, as given to the adapter. All paths in the file are relative to it, with `/` separators. |
| `capabilities` | array of string | What the adapter computed. Defined values in v1: `"anchors"`, `"tests"`, `"oracle-class"`. |
| `enforcement_anchors` | array | Section 4.2. |
| `verification_anchors` | array | Section 4.3. |
| `diagnostics` | array | Section 4.4. Adapter-side problems, not check findings. |

### 4.2 Enforcement anchor entry

| Field | Type | Meaning |
| --- | --- | --- |
| `requirement` | string | The requirement ID, verbatim. |
| `path` | string | Relative file path. |
| `line` | integer | 1-based line of the anchor mark. |
| `item` | string | Stable, human-readable path of the anchored item in the language's own idiom (`auth::session::SessionStore::create`, `auth.session.SessionStore.create`, `pkg.Session.Create`). Opaque to the core; the core uses it only for identity and display. |
| `kind` | string | How the language expresses the anchor: `"attribute"`, `"decorator"`, `"annotation"`, `"comment"`, `"macro"`. Open set; unknown values pass through. |

### 4.3 Verification anchor entry

All fields of 4.2, plus:

| Field | Type | Meaning |
| --- | --- | --- |
| `test` | object | `{ "ignored": bool, "oracle": string, "oracle_reason": string?, "suppression": string? }` |

`ignored` is true when the test exists but does not run in a normal test
invocation (Rust `#[ignore]`, pytest skip marks without condition, JUnit
`@Disabled`, Go `t.Skip` at function head where detectable). `oracle` and
`suppression` follow section 5.

### 4.4 Diagnostics entry

`{ "path": string, "line": integer?, "code": string, "message": string }`.
Defined codes in v1: `parse-error`, `ambiguous-anchor`, `unreadable-file`.
The core surfaces diagnostics in reports and treats `parse-error` on a file
that contains anchors as a check failure — a file the adapter cannot read
cannot prove anything.

### 4.5 Determinism

The same source tree and adapter version produce a byte-identical
inventory. Rules: arrays sorted by (`requirement`, `path`, `line`, `item`);
object keys in the field order of this specification; no timestamps; no
absolute paths; no environment data. Byte-identical output makes
inventories diffable in review and hashable for future capsule work.

## 5. Oracle classes

The adapter judges, syntactically, whether each verification test can fail.
The judgment is Tier 1 of the evidence-floor design: pure AST, no
execution, no data flow.

| Value | Meaning |
| --- | --- |
| `"present"` | At least one non-trivial failure path exists. |
| `"weak"` | Failure paths exist, but every one is weak (example: an expected-panic mark without an expected message). |
| `"vacuous"` | No non-trivial failure path exists. |
| `"suppressed"` | The author opted out explicitly; `suppression` holds the declared class (`"panic"`, `"compile"`, `"external"`). |
| `"unknown"` | The adapter does not implement oracle classification, or the construct is outside its rules. |

The conservatism rule is normative: **when the adapter does not fully
understand a test body, it reports `present` or `unknown`, never `vacuous`
or `weak`.** A false `vacuous` blocks honest work; a false `present` only
returns the status quo. Each adapter maps its language's native failure
idioms (assert statements, exception expectations, `t.Fatal`, xUnit
asserts) to these classes in its own conformance fixtures; the classes and
the conservatism rule are fixed here.

The core, not the adapter, decides what a class means for the check:
demotion of a sole vacuous citation, warning for redundant vacuous
citations, the `strict_oracle` area policy. Adapters report; the core
judges.

## 6. Document grammar, baseline, findings

These three formats already exist in the Rust implementation. The spec
repository extracts them from the implementation docs and makes them
normative, unchanged in substance:

- **Document grammar**: requirement ID pattern (`REQ-<AREA>-<NNN>`,
  `US-<AREA>-<NNN>`), the SHALL/SHALL NOT/MAY statement rules, evidence
  marks with the ASCII forms as canonical (`[auto]`, `[e2e]`, `[review]`,
  `[pending]`) and the emoji forms as accepted aliases, citation syntax,
  and the `fmt` normalization rules.
- **Baseline**: the committed TOML schema with its `schema` version field,
  the gap-kind registry, the ratchet rule (remove-only), and the `extend`
  operation for schema upgrades. Gap kinds registered in v1:
  `enforcement-anchor`, `verification-anchor`, `evidence-citation`;
  reserved for the evidence floor: `vacuous-evidence`, `weak-evidence`.
  A core that meets spec version N ignores no registered kind and invents
  no unregistered kind.
- **Findings**: one JSON line per finding —
  `{ "kind": string, "severity": "error"|"warning", "requirement": string?, "path": string?, "line": integer?, "message": string }` —
  sorted with the same determinism rules as the inventory. Human-readable
  report text stays implementation-defined; the JSON stream is the
  contract for CI and future services.

## 7. Requirements (normative)

Area proposals: `SIF` (inventory), `ADPT` (adapter protocol), `CONF`
(conformance). Initial set; numbering final only after review.

- **REQ-SIF-001** — An adapter SHALL emit an inventory that validates
  against the `shallguard-inventory/1` schema.
- **REQ-SIF-002** — An adapter SHALL emit byte-identical inventories for
  identical inputs (section 4.5).
- **REQ-SIF-003** — An adapter SHALL NOT read requirement documents, the
  baseline, or the configuration of the core.
- **REQ-SIF-004** — An adapter SHALL report every anchor whose requirement
  ID it can read, including IDs that no document defines; the core, not
  the adapter, decides whether an ID is dangling.
- **REQ-SIF-005** — An adapter that reports oracle classes SHALL apply the
  conservatism rule of section 5.
- **REQ-SIF-006** — An adapter SHALL emit a `parse-error` diagnostic for a
  file it cannot parse and SHALL NOT silently skip it.
- **REQ-ADPT-001** — An adapter SHALL implement the CLI protocol of
  section 8.
- **REQ-ADPT-002** — An adapter SHALL declare every schema version it
  supports through the `protocol` command.
- **REQ-ADPT-003** — The core SHALL refuse an inventory whose `schema`
  value it does not support, with an error that names both versions.
- **REQ-CONF-001** — An implementation SHALL pass every conformance
  fixture of its role (adapter or core) for the spec version it claims.
- **REQ-CONF-002** — Each conformance test in an implementation SHALL
  carry a verification anchor to the spec requirement it proves.

REQ-CONF-002 is the dogfooding clause: the spec repository is itself a
ShallGuard requirement document, and every implementation anchors its
conformance tests against these IDs with the same mechanism it implements.

## 8. Adapter CLI protocol

- `<adapter> protocol` — prints supported schema versions as JSON:
  `{ "schemas": ["shallguard-inventory/1"] }`. Exit 0.
- `<adapter> scan --root <dir> --output <file>` — writes the inventory.
  Exit 0 on success (diagnostics included in the file are still success);
  exit 2 on invocation error; exit 3 on internal error.
- Discovery: the core config names each adapter explicitly
  (`[[adapter]] command = "shallguard-adapter-python"` with an optional
  `roots` filter). No PATH magic, no auto-detection in v1.
- The adapter writes nothing except the output file and stderr logs. It
  makes no network calls.

## 9. Conformance suite layout

```
spec/
  SPEC.md                      (this document, as requirement doc)
  schemas/
    inventory.schema.json
    findings.schema.json
    baseline.schema.toml.md    (prose schema; TOML has no standard schema language)
  fixtures/
    adapter/<case>/
      input/                   (a small source tree in one language, or
                                language-neutral pseudo-fixtures where the
                                case tests protocol, not parsing)
      expected-inventory.json
    core/<case>/
      docs/                    (requirement documents)
      inventories/             (one or more SIF files)
      baseline.toml            (optional)
      expected-findings.jsonl
```

Adapter fixtures with language-specific input live in the adapter's own
repository and follow this layout; protocol-level and core fixtures live in
the spec repository. A fixture is added for every normative SHALL before
the SHALL is marked implemented anywhere — the spec repo runs the same
evidence discipline as the implementations.

## 10. Versioning and compatibility

- Schema identifiers carry a major version (`shallguard-inventory/1`).
  A breaking field change bumps the major.
- Additive optional fields do not bump the major; consumers ignore unknown
  fields (and MUST NOT fail on them) — this is the forward-compatibility
  rule that lets adapters and core release independently.
- New gap kinds and new oracle classes are additive: they enter the
  registry in a minor spec revision, and the baseline `extend` operation
  is the sanctioned upgrade path for repositories that newly detect them.
- The spec repository tags releases; an implementation states the spec
  version it conforms to in its README and in `generator.version`
  metadata.

## 11. Open questions for the maintainer

1. **Where does `fmt` live for non-Rust repositories?** The document
   grammar is core territory, so one answer is: the core binary ships
   `fmt`/`check` for documents in every ecosystem, and language packages
   (pip, npm) wrap the core binary plus their adapter. The alternative —
   reimplementing document parsing per language — violates the one-core
   principle. Recommendation: wrap the binary; treat the core as the
   single distributed engine (the ruff/biome model).
2. **Anchor syntax per language.** This draft fixes the *model*, not the
   surface syntax. Comment-based anchors (Go, C) need a exact grammar
   (`// shallguard:enforces REQ-X`?) — one shared comment grammar in the
   spec, or per-adapter freedom with only the inventory normalized?
   Recommendation: one shared comment grammar in the spec for languages
   without attributes; attributes/decorators where the language has them.
3. **Is `unknown` oracle class acceptable in hardened areas?** A strict
   reading says a hardened area demands classified evidence; a pragmatic
   reading says `unknown` equals `present` until the adapter matures.
   Recommendation: `unknown` behaves as `present` in v1; revisit when two
   adapters exist.
4. **Multi-language requirement areas.** May one requirement be enforced
   in Rust and verified in Python? The merged-inventory model allows it
   naturally; the question is whether to permit it in v1 or require
   area-per-language until the workflow is proven. Recommendation: permit
   it; the case study for it is any polyglot service, which is exactly the
   target user.
5. **Naming.** `shallguard-inventory` vs a project-wide `SIF` brand for
   all three formats. Cosmetic, but it sets the vocabulary of every future
   doc.
