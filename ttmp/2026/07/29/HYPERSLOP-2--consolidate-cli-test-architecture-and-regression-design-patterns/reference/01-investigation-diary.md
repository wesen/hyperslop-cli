---
Title: Investigation diary
Ticket: HYPERSLOP-2
Status: active
Topics:
    - testing
    - cli
    - architecture
DocType: reference
Intent: long-term
Owners: []
RelatedFiles:
    - Path: repo://cmd/hyperslop/smoke_test.go
      Note: Observed binary acceptance evidence
    - Path: repo://pkg/cli/dataset/get.go
      Note: Observed staged transaction evidence
ExternalSources: []
Summary: Chronological evidence for the test-consolidation analysis and intern guide.
LastUpdated: 2026-07-29T21:40:00-04:00
WhatFor: Explain how the guide was grounded in the current suite and why no implementation changes were made.
WhenToUse: Read before implementing the HYPERSLOP-2 test-only consolidation plan.
---


# Diary

## Goal

Create an evidence-backed HYPERSLOP-2 analysis/design/implementation guide for consolidating review-driven tests without weakening the CLI’s data-integrity, filesystem, HTTP, output, exit-code, or acceptance contracts. This ticket is intentionally documentation-only: it plans a later test refactor and does not alter production or test behavior.

## Step 1: Inventory the Review-Driven Test Surface

The first step measured the current suite and the review-fix commits rather than assuming that more tests are automatically bad. The repository has 18 `_test.go` files and 152 named tests; six review remediation commits added 58 named tests and 1,484 test lines. The data shows a real consolidation opportunity, especially in dataset, client, and tabular tests, while also showing that most added coverage maps to concrete public failure modes.

I inspected the acceptance harness, exit tests, client tests, dataset transaction code, CI targets, and test file inventory. The guide separates test layers so a future intern can reduce duplicated setup without replacing strong regression coverage with a smaller but weaker suite.

### Prompt Context

**User prompt (verbatim):** "Ok, create a ticket to consolidate and streamline tests, with a guide that explains to an intern all these testing design patterns. Create  a detailed analysis / design / implementation guide that is for a new intern, explaining all the parts of the system needed to understand what it is, with prose paragraphs and bullet point sand pseudocode and diagrams and api references and file references. It should be very clear and technical. Store in the ticket and the nupload to remarkable."

**Assistant interpretation:** Create a new docmgr ticket containing an exhaustive intern-focused technical guide for future test consolidation, maintain ticket bookkeeping, and deliver the documentation as a reMarkable PDF bundle.

**Inferred user intent:** Turn repeated review-driven regression additions into an understandable future maintenance plan without immediately changing a currently green test suite.

**Commit (code):** N/A — documentation-only ticket; no implementation changes authorized.

### What I did
- Created `HYPERSLOP-2`, the primary design doc, and this diary.
- Counted test inventory: 18 files, 152 named tests.
- Measured remediation commits `1871472`, `a6c755a`, `2114ac6`, `0e60966`, `8f230e1`, `4abebf3`, and `c72f6e6`: 58 named test functions and 1,484 added test lines.
- Read the acceptance harness, test inventory, dataset staging transaction, client request tests, exit tests, and Makefile/CI commands.
- Wrote a layered architecture, pattern catalogue, decision records, pseudocode, diagrams, phased plan, risks, and onboarding checklist.

### Why
- The tests were added in response to valid defects, including silent number corruption and partial dataset publication; deleting them blindly would repeat the original failures.
- A ledger-first, package-local fixture approach can reduce repeated setup while preserving one named regression per external contract.

### What worked
- Test inventory clearly identified bounded consolidation targets: `pkg/tabular/tabular_test.go` (824 lines), `cmd/hyperslop/smoke_test.go` (551), and `pkg/cli/dataset/get_test.go` (507).
- Existing code already demonstrates the intended layers: pure exit mapping, `httptest` client contracts, staged filesystem transaction behavior, and a narrow real binary acceptance path.
- The guide was completed without modifying implementation code or test behavior.
- `remarquee upload bundle --dry-run ...` confirmed the two-document bundle and destination; the real upload returned `OK: uploaded HYPERSLOP 2 Test Architecture Guide.pdf -> /ai/2026/07/29/HYPERSLOP-2`.

### What didn't work
- `gh pr view --json baseRefOid,headRefOid` did not accept `baseRefOid`; exact error: `Unknown JSON field: "baseRefOid"`. The inventory did not need the PR base SHA, so commit-local statistics and the current head were used instead.
- A prior remote CI failure from the previous ticket exposed a generated logcopter artifact missing for `pkg/jsondoc`; it was fixed in HYPERSLOP-1 before this documentation-only ticket. No HYPERSLOP-2 code change was made.

### What I learned
- Test count is an incomplete health metric. The relevant question is whether each test owns a distinct observable contract at the lowest trustworthy layer.
- The most valuable consolidation is setup consolidation, not assertion consolidation: hiding assertions makes a suite shorter but less explanatory.
- The standalone module boundary (`GOWORK=off`) is a release property and must remain visible in test design.

### What was tricky to build
- The guide must recommend consolidation without accidentally authorizing removal of safety regressions. The solution was a behavior ledger that maps each test to an external contract and requires a named replacement before deletion.
- The real-server smoke test is intentionally unusual: it references the server only from the test process to avoid an import cycle. The guide treats this as a narrow acceptance layer rather than a model for ordinary tests.

### What warrants a second pair of eyes
- Review the proposed threshold for promoting package-local test helpers into shared test infrastructure.
- Review whether the future implementation should measure suite runtime and flake rate before and after each phase, in addition to named-test/line counts.
- Confirm that temporary disk amplification during dataset download staging is documented in the broader dataset user documentation, not only in test design.

### What should be done in the future
- Implement the guide only in a separate approved code change, beginning with the behavior ledger and no semantic changes.
- Keep PR #1 review remediation tests intact until their ledger replacement is validated.
- Consider a follow-up schema-generation design separately; protobuf is not a lossless replacement for arbitrary JSON documents.

### Code review instructions
- Read the primary guide’s sections 2–5 for the current architecture and proposed layers.
- Start any implementation with Phase 0 and use the test selection algorithm in section 5.3.
- Validate a future implementation with the matrix in section 11; no behavior change is acceptable merely because a test count falls.

### Technical details

Commands used:

```text
find cmd pkg -name '*_test.go'
rg -n '^func Test' --glob '*_test.go'
git show --numstat <review-fix-commit>
GOWORK=off go test ./...
make ci-check
```

Observed test additions by review pass: 16, 7, 8, 10, 10, and 7 named test functions (58 total); `0e60966` added no test file lines.
