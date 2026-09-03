# TenderDash — Project Understanding

## Overview

TenderDash (`dashpay/tenderdash`) is Dash's fork of Tendermint /
CometBFT. It is the BFT consensus engine that drives Dash Platform
(L2): it orders state transitions into blocks via PBFT-style voting
and exposes the **ABCI** boundary to the application server
(`drive-abci`, in the Platform repo).

Key Dash-specific extensions on top of upstream Tendermint:

- **LLMQ-based validator sets** — validator quorums are derived from
  Dash Core's Long-Living Masternode Quorums; quorum rotation and
  membership come from L1.
- **BLS threshold signatures** — votes, commits, and proposals use
  Dash's BLS scheme (via the `bls-signatures` native dependency)
  instead of Ed25519 per-validator signatures. Aggregation produces
  a single recovered threshold signature per commit.
- **Quorum-aware domain separation** — sign-bytes for votes /
  commits bind the chain-id, quorum hash, and quorum type to
  prevent cross-quorum signature reuse.

## Authoritative Guidance

Two files in the repo root are the source of truth — agents must
follow them before this skill:

- `AGENTS.md` — minimum rules: code-quality and security
  expectations, repo layout, build/test/lint commands, branching,
  secrets handling, common pitfalls.
- `STYLE_GUIDE.md` — Go style conventions (Effective Go + Uber
  guide as a starting point), naming, imports, testing with
  `testify`, error wrapping with `%w`.

`CLAUDE.md` simply delegates to `AGENTS.md`.

## Repo Structure (Key Directories)

```
abci/        Application BlockChain Interface (app ↔ consensus boundary)
cmd/         CLI / binary entrypoints (tenderdash node, debug tools)
config/      Config structs, defaults, and config tests
crypto/      Cryptographic primitives and key handling (incl. BLS)
dash/        Dash-specific components (quorums, validators, core RPC)
docs/        Documentation
internal/    Internal (unexported) packages — most consensus logic lives here
  consensus/   Round-based BFT state machine, voting, locking
  blocksync/   Fast-sync via block download
  statesync/   State-sync from snapshots
  evidence/    Evidence pool, gossip, verification
  mempool/     Tx mempool and reactor
  p2p/         Peer-to-peer transport, switch, reactors
  proxy/       Local ABCI client multiplexer
  rpc/         JSON-RPC server internals
  state/       Block execution, state store
  store/       Block / commit on-disk storage
  evidence/    Evidence types and pool
libs/        Shared utility libraries
light/       Light client (skipping + sequential verification)
node/        Node setup, lifecycle, reactor wiring
privval/     Private validator (key storage, signing service)
proto/       Protobuf definitions (source of truth for wire types)
rpc/         Public JSON-RPC server, client, OpenAPI
spec/        Protocol specification
types/       Domain types (Block, Vote, Commit, Header, Evidence, etc.)
test/        Integration tests and test support
tools/       Helper tools and automation scripts
scripts/     Build / release / dev scripts
```

**Do not edit generated files** (`*.pb.go`). Edit the matching
`.proto` source under `proto/` and regenerate with `make proto-gen`.

## Key Types and Boundaries

- `types.Vote` / `types.Commit` / `types.CommitSig` — the
  consensus signature carriers. `Vote.SignBytes(chainID)` and
  `Commit` canonical encodings are **consensus-critical**: every
  validator and every light client must produce identical bytes.
- `types.Header` / `types.Block` — block headers carry app hash,
  results hash, validators hash, last commit info, and a
  `CoreChainLockedHeight` field tying Platform blocks to Dash Core
  chainlocks (`core_chainlock.go`).
- `types.Evidence` / `types.DuplicateVoteEvidence` — equivocation
  evidence; verification rules in `verify.go` / `evidence.go`.
- `types.ValidatorSet` — augmented with Dash quorum information
  (quorum hash, quorum type, threshold public key).
- `dash/llmq/` — LLMQ membership and quorum types, mirrored from
  Dash Core via the `dash/core/` RPC client.
- `privval/` — local and remote signers; sign requests carry
  chain-id, quorum hash, and quorum type to bind the signature.
- `abci/` — ABCI protocol types and client/server. The trust
  boundary: the app must never be trusted for consensus decisions,
  and the consensus engine must never be trusted to send the app
  invalid bytes.

## Build / Test / Lint (from AGENTS.md)

```bash
make build         # build (also builds BLS native dependency)
make test_race     # full unit tests with race detector (matches CI: CGO, BLS, -tags=deadlock, -p 1)
make test          # full unit tests without race detector (matches CI)
make lint          # golangci-lint
make format        # gofmt / goimports
```

Protobuf tooling:

```bash
make proto-lint            # buf lint
make proto-check-breaking  # buf breaking-change check
make proto-gen             # regenerate *.pb.go
make proto-format          # clang-format (required)
```

Dependencies:

- Go modules; only update modules you actually need.
- `go list -u -m all` to survey upgradable modules.
- `go mod tidy` if the build complains.

## BLS Native Dependency (Critical Caveat)

TenderDash links the `bls-signatures` C++ library. **Standalone
`go build ./...` may fail if the native library has not been built
first** — prefer Makefile-driven tests/builds so CGO flags point at
`third_party/bls-signatures`.

Local macOS caveats observed on the review machine:

- With newer CMake, vendored `relic` may reject its old
  `cmake_minimum_required`; build BLS with
  `-D CMAKE_POLICY_VERSION_MINIMUM=3.5` if plain `make build-bls`
  fails during CMake configuration.
- `make build-binary` uses static link flags and can fail on macOS
  with `ld: library 'crt0.o' not found`. For local review smoke,
  package tests are usually more useful; if a binary is needed, use
  the repo CGO flags without the static `LD_FLAGS`.
- For focused package validation, use the Makefile package override,
  e.g. `make test PACKAGES=./types` or
  `make test PACKAGES=./internal/consensus`, not plain `go test`,
  so BLS include/library paths are applied.

This also affects:

- CI parity: tests must run with CGO enabled and with the BLS
  library available, hence `make test` / `make test_race`.
- Cross-compilation: building for another OS/arch requires the
  matching BLS native library for that target.
- IDE setups: `gopls` may show unresolved symbols until BLS has
  been built once.

## Security and Secrets (from AGENTS.md)

Private keys and key-bearing config files are secret — never log
them, never commit them. Particularly sensitive:

- `config/priv_validator_key.json`
- `config/node_key.json`
- TLS private key (`tls-key-file` in config)

## Branching and PR Workflow

- Main development branch is the **highest-versioned `vMAJOR.MINOR-dev`
  branch**, not `main` / `master`. Find it with:

  ```bash
  git branch -r --list 'origin/v[0-9]*-dev' --sort=-version:refname | head -1
  ```

- Feature branches start from that dev branch and PR back into it.
- Conventional Commits for commit and PR titles.
- PR descriptions: fill in every section of
  `.github/PULL_REQUEST_TEMPLATE.md`, based on the full diff
  against the target branch.

## Common Pitfalls

- BLS native dep must be built first (`make build` handles it;
  `go build` alone often fails).
- `*.pb.go` is generated — never edit by hand; edit `.proto` and
  regenerate.
- `gogoproto` extensions (`nullable`, `customtype`) can produce
  surprising Go types — always inspect generated code after
  changes.
- Some `internal/` packages import `dash/` types; watch for
  import-cycle hazards when moving code.
- Tendermint upstream may have changed; Dash-specific signing /
  quorum logic is **not** something to "just port" — it must be
  understood.
