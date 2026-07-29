---
Title: Test Architecture Consolidation Analysis Design and Intern Implementation Guide
Ticket: HYPERSLOP-2
Status: active
Topics:
    - testing
    - cli
    - architecture
DocType: design-doc
Intent: long-term
Owners: []
RelatedFiles:
    - Path: repo://Makefile
      Note: Standalone CI and validation command composition
    - Path: repo://cmd/hyperslop/smoke_test.go
      Note: Acceptance-test architecture and standalone/workspace boundary
    - Path: repo://pkg/cli/dataset/get.go
      Note: Staged filesystem transaction model for test fixture design
    - Path: repo://pkg/cli/dataset/get_test.go
      Note: Dataset integrity and publication regression inventory
    - Path: repo://pkg/cli/exit_test.go
      Note: Pure exit and formatter lifecycle contract examples
    - Path: repo://pkg/client/datasets_test.go
      Note: HTTP client contract-test examples
    - Path: repo://pkg/tabular/tabular_test.go
      Note: Largest deterministic parser/inference test surface
ExternalSources: []
Summary: Evidence-backed plan to consolidate review-driven CLI regression tests without weakening data-integrity, filesystem, wire-contract, or binary acceptance coverage.
LastUpdated: 2026-07-29T21:40:00-04:00
WhatFor: Orient an intern to Hyperslop's test layers and provide a safe, phased plan for reducing duplicated test setup while preserving the defects each test prevents.
WhenToUse: 'Before restructuring tests added during PR #1 review remediation, adding a new CLI test, or deciding whether a finding warrants unit, integration, or acceptance coverage.'
---


# Test Architecture Consolidation: Analysis, Design, and Intern Implementation Guide

## Executive summary

`hyperslop-cli` is the customer-facing Go command-line client for the Datadrop service. It is deliberately split from the proprietary server: the CLI contains the wire types, HTTP client, tabular projection, command framework, and customer commands; `go-go-datadrop` imports the customer surface back for its administrative binary. This boundary is important for the test design because most CLI tests must run standalone with `GOWORK=off`, while a smaller acceptance layer intentionally builds and runs the real server from the split workspace.

PR #1 received six successive review waves. They identified genuine correctness defects—lossy JSON numbers, unsafe dataset publication, incorrect exit behavior, stale upload snapshots, streaming formatter misuse, and invalid-input defaults. The remediation added **58 named test functions and 1,484 test lines** across seven review-fix commits. The current repository contains **18 Go test files and 152 named test functions**. That is not evidence that the tests are useless: many protect public contracts. It is evidence that the suite needs an intentional architecture before the next feature adds another layer of one-off setup.

This ticket proposes a consolidation, not a reduction target. The goal is to make every test answer one question at the cheapest trustworthy layer:

- **Pure/domain test:** Does a deterministic parser, normalizer, projection, or error classifier obey its contract?
- **HTTP/client test:** Does an outbound request have the correct method, URL, query, header, body, and response/error interpretation?
- **Filesystem transaction test:** Does an interrupted operation preserve the old observable state and remove temporary state?
- **Command wiring test:** Does Glazed decode flags and enforce command-mode invariants before client work?
- **Binary acceptance test:** Does the compiled `hyperslop` program, with environment and process exit semantics, complete an essential user journey against a real server?

The recommended implementation creates shared fixture builders and table-driven matrices inside the existing package-local test files. It does **not** introduce a global test framework, a separate test module, mocks that duplicate the production client, or a broad rewrite. High-risk tests stay explicit. Low-risk repetitive validation tests are consolidated into named matrices. The resulting suite should be easier to navigate, faster to extend, and equally strict about externally observable behavior.

## 1. Scope, non-goals, and terminology

### 1.1 Scope

This ticket plans a future test-only refactor of the `hyperslop-cli` module. It covers:

1. package-local unit tests in `pkg/datadrop`, `pkg/tabular`, `pkg/jsondoc`, and `pkg/cli`;
2. `httptest`-backed client tests in `pkg/client`;
3. filesystem transaction tests in `pkg/cli/dataset`;
4. the real binary/server acceptance test in `cmd/hyperslop/smoke_test.go`;
5. CI and local test commands in `Makefile` and `.github/workflows/push.yml`.

### 1.2 Non-goals

This ticket intentionally does **not** authorize implementation changes now.

- Do not remove a regression merely because it was review-driven.
- Do not alter the HTTP protocol, exit codes, output row shapes, environment variables, or the split-module dependency direction.
- Do not merge the server into hyperslop-cli or make standalone tests import `go-go-datadrop`.
- Do not introduce a third-party assertion/mocking framework merely to shorten syntax.
- Do not move arbitrary JSON contract coverage into protobuf; dynamic event payloads, manifests, and JSON Schema remain raw JSON documents.
- Do not make real-server tests mandatory under `GOWORK=off`; the standalone/module boundary is a required release property.

### 1.3 Terms

| Term | Meaning in this guide |
|---|---|
| **SUT** | System under test: a function, package, command, or binary under examination. |
| **Contract** | An observable promise: request shape, exit code, destination state, row field set, or diagnostic behavior. |
| **Fixture** | Deterministic input/state used by more than one test. It must be readable and local to the behavior it supports. |
| **Fake server** | An `httptest.Server` whose handler asserts a client protocol contract and returns controlled responses. |
| **Acceptance test** | A test that executes the compiled CLI against an actual Datadrop server process. |
| **Transaction** | An operation that makes either all intended observable changes or preserves the prior observable state on failure. |

## 2. Current system map

### 2.1 Production architecture

```text
+----------------------------- hyperslop-cli -----------------------------+
| cmd/hyperslop                                                        |
|   process args + HYPERSLOP_* + exit status                            |
|       |                                                               |
| pkg/cli                                                               |
|   Glazed commands: drops, events, dataset, schema, auth, rows, exit  |
|       |                                                               |
| pkg/client                                                            |
|   HTTP request construction, API errors, SSE, uploads/downloads       |
|       |                                                               |
| pkg/datadrop + pkg/tabular + pkg/jsondoc                              |
|   wire/domain types, validation, projections, lossless JSON           |
+-------------------------------|-----------------------------------------+
                                | REST/SSE
+-------------------------------v-----------------------------------------+
| go-go-datadrop (separate module)                                      |
| server, SQLite store, blob storage, auth/OIDC, embedded PBUI shell    |
+-------------------------------------------------------------------------+
```

The one-way dependency is part of the product design: hyperslop-cli must not import `github.com/go-go-golems/go-go-datadrop`. The smoke suite makes the companion dependency explicit only at test process level; its comments and setup distinguish an absent companion in standalone mode from a broken companion in a workspace ([`cmd/hyperslop/smoke_test.go:18-39`](../../../../../../cmd/hyperslop/smoke_test.go)).

### 2.2 Test-layer map

```text
                         cheap / deterministic
+-------------------------------------------------------------------+
| Domain and codec tests                                            |
| datadrop, jsondoc, tabular, rows, exit                            |
+-------------------------------------------------------------------+
                               |
+-------------------------------------------------------------------+
| Client protocol tests                                             |
| httptest handlers inspect outbound HTTP; no real server/store     |
+-------------------------------------------------------------------+
                               |
+-------------------------------------------------------------------+
| Command/filesystem tests                                          |
| command settings, staged download, path handling, temp cleanup    |
+-------------------------------------------------------------------+
                               |
+-------------------------------------------------------------------+
| Binary acceptance tests                                           |
| compiled hyperslop + compiled datadrop server + OIDC stub + SQLite|
+-------------------------------------------------------------------+
                         expensive / broad confidence
```

The important rule is **not** “every defect gets a test at every layer.” It is “each public failure mode gets one primary regression at the lowest layer that can prove it, plus acceptance coverage only when binary wiring is part of the promise.”

### 2.3 Inventory and hot spots

Repository inspection on 2026-07-29 found 18 test files and 152 named tests. The largest files are:

| File | Lines | Named tests | Primary responsibility |
|---|---:|---:|---|
| `pkg/tabular/tabular_test.go` | 824 | 32 | CSV/JSON/NDJSON read limits, flattening, inference, schema typing |
| `cmd/hyperslop/smoke_test.go` | 551 | 6 | compiled-binary and real-server acceptance paths |
| `pkg/cli/dataset/get_test.go` | 507 | 14 | verified downloads, staging, path safety, archive behavior |
| `pkg/client/client_test.go` | 406 | 15 | HTTP/SSE client behavior |
| `pkg/cli/rows_test.go` | 349 | 21 | public output row-shape change detection |
| `pkg/datadrop/datadrop_test.go` | 263 | 14 | validation and query/domain rules |
| `pkg/cli/drops/push_test.go` | 257 | 9 | typed key/value payload construction |

This distribution identifies useful consolidation targets. It does **not** justify moving unrelated domain tests into a single mega-file.

## 3. Evidence-backed current behavior

### 3.1 CI is intentionally standalone first

`Makefile` defines `test` as `GOWORK=off go test ./...`, and `ci-check` combines format, lint, generated logger verification, tests, and build ([`Makefile:28-52`](../../../../../../Makefile)). This means helpers must preserve standalone operation and never assume the server checkout is present.

The smoke test is the exception by design. It builds the real `hyperslop` binary, then the real Datadrop server only when the companion module resolves; test setup failures after resolution are failures rather than skips ([`cmd/hyperslop/smoke_test.go:80-140`](../../../../../../cmd/hyperslop/smoke_test.go)).

### 3.2 Exit code tests protect a binary-facing contract

The CLI cannot rely only on Cobra’s default error handling because scripts branch on fixed codes. `pkg/cli/exit_test.go` checks status-to-code mapping, error wrapping, cancellation semantics, Glazed’s success signal, and failure-path formatter closure ([`pkg/cli/exit_test.go:21-110`](../../../../../../pkg/cli/exit_test.go)). These are compact pure/unit tests; replacing them with six process invocations would make the suite slower and less diagnostic.

### 3.3 Dataset download is a multi-step filesystem transaction

`downloadVersion` resolves a numeric version once, validates logical paths, streams every file into hidden stages, then publishes the batch ([`pkg/cli/dataset/get.go:204-252`](../../../../../../pkg/cli/dataset/get.go)). `stageDownloadedFile` checks symlinks, creates a temporary sibling, streams and fsyncs bytes, and verifies the digest before the destination changes ([`pkg/cli/dataset/get.go:261-335`](../../../../../../pkg/cli/dataset/get.go)). `publishStagedDownloads` backs up force targets and rolls them back if a later promotion fails ([`pkg/cli/dataset/get.go:348-413`](../../../../../../pkg/cli/dataset/get.go)).

These are separate invariants and deserve a shared transaction fixture rather than tests that duplicate an HTTP server and directory setup each time.

### 3.4 Client tests are protocol tests, not server duplicates

`pkg/client/datasets_test.go` has concise handlers that assert request behavior and return typed responses. For example, it proves invalid numeric settings make **zero** requests and that a file mutation after snapshot creation cannot change a cache-hit digest ([`pkg/client/datasets_test.go:17-105`](../../../../../../pkg/client/datasets_test.go)). This is the correct layer for request suppression and client-side snapshot behavior.

### 3.5 Output rows are a stable interface

`pkg/cli/rows_test.go` contains deliberate “change detector” tests. Its 21 test functions protect ordered field names consumed by `--output-fields` and structured formats. Consolidation must preserve explicit field-order expectations; it may share row construction helpers but must not replace exact key assertions with vague snapshots.

## 4. Problem statement

The current tests are safe but increasingly expensive to comprehend because review remediation followed this pattern:

```text
review finding
  -> localized fix
  -> localized regression with fresh setup
  -> next finding near the same boundary
  -> another localized regression with similar setup
```

This leads to four maintainability risks.

1. **Repeated fixture mechanics obscure the contract.** Multiple tests recreate temp directories, HTTP handlers, versions, digests, and assertions. An intern spends time reading setup instead of learning what behavior matters.
2. **The wrong layer may be used by accident.** A command-option test may spin up a server; a pure validator may be tested through a binary. This slows feedback and hides the reason a failure matters.
3. **Cross-cutting invariants lack named ownership.** “No invalid side-effecting request,” “preserve existing files,” and “JSON numbers are lossless” appear in several packages but are not collected as explicit patterns.
4. **Review-driven exact-string tests can become brittle.** Help text and diagnostics should be tested where they are a public contract, but incidental phrasing should not make normal documentation editing painful.

The solution is a test architecture with explicit layers, fixture factories, a behavior-to-layer decision rule, and a review process that prevents duplication before it is merged.

## 5. Proposed architecture

### 5.1 Package-local test kits, not one global framework

Create small `testkit_test.go` files only where repeated setup is already measurable. Test-only files stay in the owning package so they can use unexported symbols without exporting production seams.

```text
pkg/client/
  client_test.go
  datasets_test.go
  testkit_test.go       # new: recording HTTP handler/request assertions

pkg/cli/dataset/
  get_test.go
  push_test.go
  import_test.go
  testkit_test.go       # new: temp root, dataset version, digest/file helpers

pkg/cli/
  exit_test.go
  rows_test.go
  testkit_test.go       # optional only if repeated row/error construction proves useful
```

Do not create `pkg/testutil` until at least two **different production packages** need the same helper and it can avoid importing production internals. A global helper package otherwise becomes a hidden dependency graph and can create its own testing architecture problem.

### 5.2 Canonical test helpers

#### A. Client recording server

```go
// Test-only API sketch; do not export from production packages.
type recordedRequest struct {
    Method string
    Path string
    Query url.Values
    Header http.Header
    Body []byte
}

type recordingServer struct {
    Requests []recordedRequest
    Respond func(http.ResponseWriter, *http.Request)
}

func newRecordingClient(t *testing.T, respond func(...)) (*Client, *recordingServer)
func (s *recordingServer) RequireNoRequests(t *testing.T)
func (s *recordingServer) RequireOne(t *testing.T, want requestExpectation)
```

Use it for client guarantees such as “negative input sends no request,” correct media type, concrete version pinning, and cache-hit mount behavior. Do not use it to imitate store semantics, authorization, or schema evaluation; those are server responsibilities.

#### B. Dataset transaction fixture

```go
type datasetFixture struct {
    Root string
    Version datadrop.DatasetVersion
    Content map[string]string
}

func newDatasetFixture(t *testing.T, files map[string]string) datasetFixture
func (f datasetFixture) SeedDestination(t *testing.T, old map[string]string)
func (f datasetFixture) Server(t *testing.T, mutate func(path string) io.Reader)
func (f datasetFixture) AssertDestination(t *testing.T, want map[string]string)
func assertNoHiddenDownloadArtifacts(t *testing.T, root string)
```

This fixture should represent only data needed by `dataset get`: manifest paths, SHA-256 digest, bytes, output root, and controlled failure. It must not know CLI flags or HTTP endpoints unrelated to retrieval.

#### C. Command setting matrix

```go
type validationCase[S any] struct {
    Name string
    Settings S
    WantError string
}

for _, tc := range cases {
    t.Run(tc.Name, func(t *testing.T) {
        err := tc.Settings.Validate()
        requireErrorContains(t, err, tc.WantError)
    })
}
```

Use a local matrix for mutually exclusive selectors, follow/formatter requirements, negative bounds, and payload-source exclusivity. Keep validation methods small and deterministic. The test does not need a fake server if `Validate` runs before client construction.

#### D. Exact contract assertions

Keep narrow, explicit assertions for:

- ordered row keys;
- JSON number lexemes crossing dynamic decode/re-encode boundaries;
- exit code mapping;
- data digest and prior filesystem state;
- single authoritative help examples when a command is intended to be copied.

Avoid generic golden files for all CLI text. Golden files are suitable for large, intentional documents or stable protocol artifacts, but they make small unrelated edits noisy.

### 5.3 Test selection algorithm

Use this decision tree before writing a test.

```text
Is the failure visible only after compiling/running the binary?
  yes -> Add/extend acceptance test.
  no  -> Does it concern bytes on HTTP/SSE?
           yes -> Client httptest.
           no  -> Does it change filesystem state across several files?
                    yes -> Dataset transaction test using fixture.
                    no  -> Does it depend on Glazed decoded settings?
                             yes -> Command validation/wiring test.
                             no  -> Pure package/domain test.

Would a second test at a higher layer prove a different public contract?
  yes -> Add it and name that contract in the test.
  no  -> Do not duplicate it.
```

## 6. Pattern catalogue for interns

### Pattern 1: Table-driven domain matrices

Use for many inputs with one deterministic rule. The exit-code matrix is the model: a status code maps to a documented exit code with no network or process needed.

```go
cases := []struct { name string; input int; want int }{
    {"unauthorized", 401, ExitAuth},
    {"missing", 404, ExitNotFound},
}
for _, tc := range cases {
    t.Run(tc.name, func(t *testing.T) {
        if got := ExitCodeFor(apiError(tc.input)); got != tc.want { ... }
    })
}
```

Use when each row is independently understandable. Do not hide distinct contracts in one enormous table merely to reduce function count.

### Pattern 2: Arrange–act–assert filesystem transaction

```text
Arrange: existing downloaded version A on disk
Act:     request version B; make B/file-2 transfer or digest fail
Assert:  A/file-1 still exists unchanged
         A/file-2 still exists unchanged
         hidden stages/backups are absent
```

This tests the **observable transaction**, not implementation details such as a specific temporary filename. `TestDownloadVersionForceDoesNotMixVersionsOnLaterFailure` is the current canonical regression in `pkg/cli/dataset/get_test.go`.

### Pattern 3: Protocol test with a minimal HTTP server

A fake server should record what the client sends and respond with only enough protocol to reach the branch under test.

```go
server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    requireMethodPath(t, r, http.MethodPost, "/v1/.../import")
    // Or fail the test if this request should never occur.
}))
```

Do not test server business logic here. For a negative precondition, the strongest assertion is often `requests == 0`.

### Pattern 4: Binary acceptance slice

Acceptance tests are expensive because they compile two binaries, seed a token, open a real socket, and use a real SQLite-backed server. Maintain a small number of user journeys:

```text
authenticate -> create -> push -> query -> tail -> export
             -> schema put/show -> dataset push/get -> whoami
```

Add a binary case only when the behavior includes argument parsing, environment configuration, stdout/stderr, final formatter behavior, or process exit code. Keep unit coverage for the underlying branch.

### Pattern 5: Lossless dynamic JSON boundary

`pkg/jsondoc` exists because `encoding/json` decoding into `any` otherwise turns numbers into `float64`. Tests must preserve representative high-risk lexemes:

```text
9007199254740993                 # beyond exact IEEE-754 integer range
0.123456789012345678901          # high-precision decimal
```

A test should assert semantic preservation (the numeric lexeme remains), not map key ordering after a re-marshal.

### Pattern 6: Copyable documentation is executable surface

A command shown as a round trip has a stronger contract than prose. Test the smallest stable property that makes the example safe. For `schema show`, verify the help contains one structured formatter and explicit extraction of `.spec`; do not freeze unrelated paragraphs or wrapping.

## 7. Proposed package organization

```text
cmd/hyperslop/
  smoke_test.go
    - 2–4 acceptance journeys
    - binary-only exit/output assertions

pkg/client/
  client_test.go
  datasets_test.go
  testkit_test.go
    - recording server and request expectation helpers

pkg/cli/dataset/
  get_test.go
    - unit/transaction scenarios grouped by operation
  push_test.go
  import_test.go
  testkit_test.go
    - dataset version, digest, output-tree helpers

pkg/cli/
  exit_test.go
  rows_test.go
    - unchanged, because these are public contract change detectors

pkg/datadrop/, pkg/tabular/, pkg/jsondoc/
  *_test.go
    - pure deterministic matrices and codec properties
```

File names should communicate behavior rather than review history. Avoid names such as `review6_test.go`; defects outlive the review that discovered them.

## 8. Decision records

### Decision: Consolidate setup, retain contract-specific tests

- **Context:** 58 named regressions were added in response to reviews. Repeated setup now competes with the behavior being documented.
- **Options considered:** Delete tests; leave every test untouched; rewrite into one global test framework; extract package-local helpers.
- **Decision:** Extract package-local helpers and table matrices while retaining one explicit primary test for every external contract.
- **Rationale:** This removes duplication without treating past defects as unimportant.
- **Consequences:** Refactor commits must be behavior-preserving and continuously validated; helpers need clear scope.
- **Status:** proposed.

### Decision: Preserve a small real-server acceptance layer

- **Context:** The CLI’s process environment, exit code, Glazed formatter lifecycle, and client/server integration cannot be fully proven in process-local tests.
- **Options considered:** Test every command end-to-end; remove acceptance tests; retain a narrow journey suite.
- **Decision:** Retain a narrow real-server smoke suite and move branch-specific behavior down to unit/client tests.
- **Rationale:** It protects the integration boundary while limiting runtime and flakiness.
- **Consequences:** New acceptance tests need an explicit binary-level reason.
- **Status:** proposed.

### Decision: Use test-only package-local helpers before shared infrastructure

- **Context:** Client and dataset tests repeat setup, but production packages have different unexported details.
- **Options considered:** `internal/testkit`; external test framework; package-local `testkit_test.go` files.
- **Decision:** Begin package-local. Promote only genuinely cross-package, dependency-light helpers.
- **Rationale:** It avoids a central fixture framework becoming a second application architecture.
- **Consequences:** Some superficially similar helpers remain duplicated by design when their semantics differ.
- **Status:** proposed.

### Decision: Keep explicit row-shape and precision tests

- **Context:** These tests appear detailed, but callers use output fields and dynamic documents as API data.
- **Options considered:** Snapshot all output; use loose structural checks; retain ordered/lexeme assertions.
- **Decision:** Retain ordered key tests and exact-number tests as contract tests.
- **Rationale:** These are the smallest tests that catch breaking changes with useful diagnostics.
- **Consequences:** They must be consciously updated with changelog evidence when the public contract changes.
- **Status:** proposed.

## 9. Phased implementation plan (future work)

### Phase 0 — Establish a behavior ledger

Before moving code, create `reference/02-test-contract-ledger.md`. For every existing test, record:

- package/file/test name;
- contract it protects;
- intended test layer;
- duplicate or shared fixture candidate;
- required result after refactor.

**Exit criterion:** every test added in PR #1 review commits is classified. No deletion is allowed without a replacement ledger entry.

### Phase 1 — Consolidate pure validation matrices

Target `pkg/datadrop`, `pkg/jsondoc`, `pkg/cli/exit`, and simple command setting validators.

1. Group cases by public rule, not by review pass.
2. Convert duplicated test setup to table rows only where assertion shape is identical.
3. Preserve descriptive case names such as `negative-import-limit-sends-no-request`.
4. Run package tests and compare named test coverage ledger.

**Likely files:** `pkg/datadrop/datadrop_test.go`, `pkg/jsondoc/jsondoc_test.go`, `pkg/cli/exit_test.go`, `pkg/cli/events/tail_test.go`, `pkg/cli/dataset/*_test.go`.

### Phase 2 — Introduce client protocol test kit

1. Add `pkg/client/testkit_test.go` with request recorder and response helpers.
2. Migrate independent tests one at a time.
3. Keep each test’s own request-specific assertion near the test body.
4. Verify no helper performs server business logic.

**Exit criterion:** client tests become shorter without losing method/path/query/header/body assertions or no-request assertions.

### Phase 3 — Introduce dataset transaction fixture

1. Add local helpers for version manifests, content/digest maps, destination seeding, output tree assertions, and controlled response failure.
2. Refactor `get_test.go` around three scenarios: successful publish, failure before publish, failure during promotion/rollback.
3. Keep symlink/path escape tests distinct because they represent a security boundary rather than ordinary transaction flow.

**Exit criterion:** the mixed-version regression remains readable in fewer lines, and all temporary/backup cleanup assertions remain present.

### Phase 4 — Rebalance binary smoke tests

1. Document each existing smoke subflow’s unique binary-only reason.
2. Move any branch that only proves client/domain behavior into its lower layer.
3. Keep one full authenticated happy path and a compact exit/output contract path.
4. Keep workspace-availability gating strict: absence skips; build/seed failures fail.

**Exit criterion:** acceptance runtime does not grow; standalone `GOWORK=off` behavior remains validated.

### Phase 5 — Add test-authoring guardrails

Add a short `pkg/TESTING.md` or a section in `AGENT.md` describing the selection algorithm and required review checklist. Do not introduce a lint rule until conventions are stable.

## 10. Pseudocode for the target workflow

```text
function addRegression(finding):
    contract = state the observable failure in one sentence
    layer = chooseLayer(contract)
    existing = search contract ledger and local testkit

    if existing primary test already proves contract:
        extend its table/matrix with a named case
    else:
        write one primary regression at layer

    if layer is acceptance:
        explain why unit/client/filesystem test cannot prove it

    run targeted package test
    run standalone full suite
    if changing shared command/client behavior:
        run workspace smoke and companion suite

    record test name, contract, and command in ledger/diary
```

## 11. Test strategy and validation matrix

| Change category | Required targeted command | Required broader evidence |
|---|---|---|
| Domain/codec consolidation | `GOWORK=off go test ./pkg/datadrop ./pkg/jsondoc ./pkg/tabular -count=1` | `make ci-check` |
| Client test-kit change | `GOWORK=off go test ./pkg/client -count=1` | full standalone suite |
| Dataset transaction fixture | `GOWORK=off go test ./pkg/cli/dataset -count=1` | workspace hyperslop smoke and admin suite |
| Exit/rows test reorganization | `GOWORK=off go test ./pkg/cli -count=1` | binary exit smoke |
| Smoke suite change | `go test ./cmd/hyperslop -run TestHyperslop -count=1 -v` | standalone suite proves graceful skip behavior |
| Any new package | `make logcopter-generate && make logcopter-check` | CI generated-file check |

Always run `git diff --check`, `gofmt`, `go vet`, and `golangci-lint`. Do not claim a test refactor is safe merely because it compiles: compare the behavior ledger before and after.

## 12. Risks and safeguards

### Risk: “Consolidation” deletes meaningful edge cases

**Safeguard:** ledger-first migration; each deleted test must map to a named replacement case and the same fault model.

### Risk: shared helpers hide important setup

**Safeguard:** helpers may build state but must not contain assertions that conceal protocol details. Tests retain the meaningful assertion adjacent to their scenario.

### Risk: acceptance tests become a substitute for unit tests

**Safeguard:** require an explicit binary-only justification in the test comment or review description.

### Risk: tests become coupled to implementation details

**Safeguard:** assert public outputs, requests, filesystem state, and error class—not local temporary-name spelling, helper call count, or private struct layout.

### Risk: filesystem transaction fixture fails to model hostile paths

**Safeguard:** keep traversal/symlink tests separate and explicit; they are security tests, not data-table rows.

### Risk: refactor changes standalone behavior

**Safeguard:** preserve `GOWORK=off` suite as the primary local/CI command; the companion module is only an opt-in workspace integration.

## 13. Alternatives considered

### Delete review-driven tests

Rejected. The review findings included silent data corruption and partial publication. Removing their regressions would recreate the exact conditions that made six review waves necessary.

### Replace everything with end-to-end tests

Rejected. Real-server tests are slower, require workspace dependencies, and provide poorer diagnosis for a bad parser or request query. They are essential for a narrow set of binary contracts, not for every branch.

### Introduce a large third-party mocking/assertion framework

Rejected for now. The standard library’s `testing` and `httptest` already match the codebase. A framework would add syntax and dependency churn without addressing the core problem: selecting the correct layer and sharing only appropriate fixtures.

### Use one global `internal/testkit` immediately

Rejected. It would tempt unrelated packages to share helpers and create hidden coupling. Start local; extract only after demonstrated cross-package reuse.

### Migrate arbitrary dynamic JSON to protobuf as a testing solution

Rejected. Protobuf is a potential future schema/SDK tool for known DTOs, but `Struct`/`Value` represents numbers as floating-point values and would not preserve arbitrary JSON numeric lexemes. The current `pkg/jsondoc` boundary is the correct safety mechanism for dynamic documents.

## 14. Intern onboarding checklist

Before editing a test, an intern should be able to answer:

1. What is the public behavior being protected?
2. Which production package owns that behavior?
3. Which test layer is the cheapest layer that can observe it?
4. Does an existing fixture or test already exercise the same contract?
5. What failure would occur if this regression were removed?
6. Does the test work with `GOWORK=off`? If not, why is workspace acceptance necessary?
7. If the test creates files, what must exist after failure? What must not?
8. If the test sends HTTP, what request must be absent or present?
9. If the test checks text, is that text a copyable/public contract or incidental prose?
10. Which exact commands prove the refactor did not weaken behavior?

## References

### Primary source files

- [`cmd/hyperslop/smoke_test.go`](../../../../../../cmd/hyperslop/smoke_test.go) — real binary/server acceptance harness.
- [`pkg/cli/dataset/get.go`](../../../../../../pkg/cli/dataset/get.go) — staged verified dataset download transaction.
- [`pkg/cli/dataset/get_test.go`](../../../../../../pkg/cli/dataset/get_test.go) — filesystem integrity and transaction regressions.
- [`pkg/client/datasets_test.go`](../../../../../../pkg/client/datasets_test.go) — client request/no-request and snapshot tests.
- [`pkg/cli/exit_test.go`](../../../../../../pkg/cli/exit_test.go) — exit and formatter lifecycle contract.
- [`pkg/cli/rows_test.go`](../../../../../../pkg/cli/rows_test.go) — output row-shape contract.
- [`pkg/tabular/tabular_test.go`](../../../../../../pkg/tabular/tabular_test.go) — reader/inference/flattening behavior.
- [`pkg/jsondoc/jsondoc_test.go`](../../../../../../pkg/jsondoc/jsondoc_test.go) — lossless dynamic JSON number behavior.
- [`Makefile`](../../../../../../Makefile) — standalone test and CI composition.
- [`../HYPERSLOP-1--extract-customer-facing-cli-from-go-go-datadrop-into-hyperslop-cli/code-review/01-pr-1-takeover-review.md`](../../HYPERSLOP-1--extract-customer-facing-cli-from-go-go-datadrop-into-hyperslop-cli/code-review/01-pr-1-takeover-review.md) — finding-by-finding remediation history.

### Evidence snapshot

Review remediation commits: `1871472`, `a6c755a`, `2114ac6`, `0e60966`, `8f230e1`, `4abebf3`, and `c72f6e6`.
