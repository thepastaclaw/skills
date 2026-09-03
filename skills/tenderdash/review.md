# TenderDash — Review Skill

## Authoritative Sources

Before reviewing, defer to these files in the TenderDash repo:

- `AGENTS.md` — minimum rules for AI agents (security, quality,
  build/test/lint, branching).
- `STYLE_GUIDE.md` — Go style conventions.

This skill summarizes review-time expectations on top of those.

## Code Style (from STYLE_GUIDE.md)

- `gofmt` / `goimports` formatting enforced; `golangci-lint` is the
  source of truth for lint (`.golangci.yml`).
- Effective Go + Uber Go style guide as the starting point.
- **No dot imports**; side-effect-only imports use `_`.
- Three import blocks: stdlib, external, application.
- Names should not stutter (`pkg.PkgFoo` is wrong).
- Acronyms are ALL CAPS (`RPC`, `gRPC`, `API`, `MyID`).
- Product names capitalized in prose ("Tenderdash"); CLI flags are
  lowercase (`tenderdash --help`).
- Prefer `errors.New` over `fmt.Errorf` unless formatting is needed.
- Wrap errors with `%w`; use stdlib `errors` package.
- `TODO` is **not allowed** — file an issue. `BUG` / `FIXME` are
  used sparingly; `XXX` must be removed before merge.
- Reserve `Save` / `Load` for long-running persistence; use
  `Encode` / `Decode` for byte parsing.
- Functions returning functions take the suffix `Fn`.

Things `golangci-lint` and `gofmt` already catch should not be
review findings.

## Commit Conventions

- Conventional Commits for PR and commit titles.
- One logical change per commit.
- PR description follows `.github/PULL_REQUEST_TEMPLATE.md` — fill
  in every section.
- Target branch is the highest-versioned `vMAJOR.MINOR-dev`, not
  `main` or `master`.

## Test Expectations (from STYLE_GUIDE.md + AGENTS.md)

- **All code must have tests.**
- Table-driven tests where they fit.
- `stretchr/testify` `assert` / `require`. Mocks: testify `mock`
  with `Mockery` for generation.
- Consensus-critical changes (anything under `internal/consensus`,
  `types/`, `privval/`, `light/`, `state/`, `evidence/`, `dash/`)
  **must have tests** — no exceptions.
- Run with race detector for code that touches shared state:
  `make test_race` (CGO + BLS + `-tags=deadlock`, `-p 1`).
- New behavior reachable from RPC or P2P should have an
  integration test under `test/` or a reactor-level test where
  feasible.

## Error Handling

- Errors are concise, clear, and traceable.
- Wrap with `%w` so downstream code can `errors.Is` / `errors.As`.
- `panic` only on broken internal invariants; everything else
  returns an error.
- CLI / server entrypoints **may** panic with a stack trace on
  unrecoverable errors — that is the explicit project rule.

## Concurrency

- Race detector must stay clean (`make test_race`).
- Watch for: data races on shared maps / slices, missing
  `sync.Mutex` on shared state, lock-order inversions across
  reactors, goroutine leaks (every spawned goroutine must have a
  clear exit), `context.Context` propagation through long calls,
  channels closed by the wrong side, send-on-closed.

## Consensus-Critical Code Areas

Apply extra scrutiny to:

- `internal/consensus/` — round / step state machine, voting,
  locking (`Locked`/`Valid` block), commit formation.
- `types/` — `Vote`, `Commit`, `Header`, `Block`, `Evidence`,
  canonical encodings, `SignBytes`.
- `privval/` — anything that produces a signature; chain-id /
  quorum-hash / quorum-type binding must be preserved.
- `light/` — trust period, trust level, bisection logic,
  conflicting-header handling.
- `internal/state/` — block execution and state transitions.
- `internal/evidence/` — equivocation detection, evidence
  validation.
- `dash/` and `crypto/bls12381/` — LLMQ quorum logic and BLS
  threshold operations.
- `proto/**.proto` — any wire-format change is potentially
  consensus-affecting; check generated `*.pb.go` is regenerated
  and both serializer and deserializer are updated symmetrically.

Changes here should ship with:

1. Clear rationale in the PR description.
2. Tests, including edge cases (boundary heights, partial
   signatures, byzantine messages).
3. Migration / upgrade story when on-disk or wire formats change.

## ABCI and RPC Boundaries

- Treat ABCI app responses as untrusted: validate validator updates,
  consensus param updates, app hash, snapshot chunks, and tx
  results before consuming them.
- RPC inputs (HTTP/JSON, WebSocket) are public attack surface:
  every handler needs bounds checks, sensible timeouts (via
  `context.Context`), and no panics reachable from user input.
- P2P messages: every reactor's `Receive` path is a trust boundary
  — size-limit, type-check, reject malformed frames without
  panicking, do not block the receive loop on slow work.

## Things NOT to Flag

- Style or formatting that `gofmt` / `goimports` / `golangci-lint`
  already enforce.
- Bare upstream Tendermint patterns unless they are unsafe in the
  Dash context — be explicit about the Dash-specific reason.
- `*.pb.go` formatting / "improvements" — these files are
  generated.
- Documentation wording in godocs unless it states something
  factually wrong about consensus behavior.

## Known Patterns / Accepted Exceptions

- Applications (CLI / node binary) intentionally panic on
  unrecoverable startup errors — this is the project rule, not a
  bug.
- `Mockery`-generated mocks under `types/mocks/`, `dash/mocks/`,
  etc. are checked-in artifacts; do not flag style nits in them.
- `deadlock` build tag (`-tags=deadlock`) substitutes a
  lock-tracing mutex package in tests; release builds use stdlib
  mutexes. Do not flag this as inconsistent.

## Per-Reviewer Preferences

(Populated by feedback loop as preferences emerge.)

## Build / Test / Lint Quick Reference

```bash
make build         # build (also builds BLS native dep first; Linux/CI-oriented static link)
make test          # full unit tests (matches CI: CGO, BLS, -p 1)
make test_race     # full unit tests with race detector + -tags=deadlock
make test PACKAGES=./types              # focused package test with repo CGO/BLS flags
make test PACKAGES=./internal/consensus # another focused example
make lint          # golangci-lint
make format        # gofmt / goimports
make proto-gen     # regenerate *.pb.go from proto/ sources
```

Local review-machine caveats:

- Plain `go test` bypasses the Makefile CGO flags and may fail to
  find `dashbls/bls.hpp`; use `make test PACKAGES=...` for focused
  packages.
- Newer CMake may require `-D CMAKE_POLICY_VERSION_MINIMUM=3.5`
  when building `third_party/bls-signatures` / vendored `relic`.
- On macOS, `make build-binary` can fail at static link time with
  `ld: library 'crt0.o' not found`; that does not mean package tests
  are unusable. Prefer focused tests for review validation unless the
  PR specifically touches binary/link behavior.

If a PR adds files under `proto/`, confirm `make proto-gen` was
run and the regenerated `*.pb.go` is included.
