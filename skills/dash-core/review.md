# Dash Core — Review Skill

## Code Style

- Follows Bitcoin Core style: `clang-format` enforced in CI
- C++17, some C++20 features where supported
- Python PEP 8 for test framework (`test/functional/`)
- Naming: `CClassName`, `m_member`, `g_global`, `nLocalVar`
  (Bitcoin Core conventions)
- Headers use `#ifndef` guards, not `#pragma once`

## Commit Conventions

- One logical change per commit
- Format: `area: short description` (e.g., `evo: fix DMN list update`)
- Bitcoin backport commits preserve original Bitcoin Core authorship
  and commit messages

## Test Expectations

- **Unit tests** (`src/test/`): for new logic, data structures,
  serialization changes
- **Functional tests** (`test/functional/`): for RPC changes, P2P
  behavior, consensus rule changes
- **Consensus changes MUST have tests** — no exceptions
- Run tests: `make check` (unit), `test/functional/test_runner.py`
  (functional)
- New Dash-specific features should have both unit + functional
  tests where applicable

## Known Patterns / Accepted Exceptions

- `LOCK(cs_main)` followed by `LOCK(cs_wallet)` is the correct
  lock order — do not flag
- `Assert` vs `assert` — Dash uses both; `Assert` (capital A) is
  the Bitcoin Core fatal assert, `assert` is the C assert. Both
  are intentional in different contexts
- `CConnman` friend classes — intentional for test access
- Raw pointer usage in `CNode`, `CConnman` — legacy Bitcoin Core
  pattern, managed lifetime
- `goto` in `AppInit` / initialization code — matches Bitcoin Core
  upstream
- Intentional duplicate code between mainnet/testnet/devnet params
  — these diverge and should not be abstracted

## Common Pitfalls

- **Lock ordering:** Always `cs_main` before `cs_wallet`, always
  `cs_main` before any LLMQ lock. Violations cause deadlocks.
- **Missing `EXCLUSIVE_LOCKS_REQUIRED` / `LOCKS_EXCLUDED`
  annotations** on functions that acquire or require locks
- **LLMQ quorum type confusion:** Different quorum types have
  different sizes, thresholds, and purposes. Using the wrong type
  is consensus-critical.
- **Forgetting `src/Makefile.am`** when adding new source files
- **Not updating `src/rpc/client.cpp`** `vRPCConvertParams` when
  adding RPC commands with non-string parameters
- **Serialization versioning:** Changes to serialized types need
  version handling for backwards compatibility
- **Bitcoin backport conflicts:** When reviewing backports, the
  merge resolution is what matters, not the upstream code itself
- **DevNet vs TestNet vs MainNet params:** Changes to chain params
  must be consistent across all three (or intentionally different
  with justification)

## Things NOT to Flag

- Style issues already caught by `clang-format` CI
- Bitcoin Core upstream code in backport PRs — review the merge
  commit's conflict resolution, not the upstream changes
- `TODO` / `FIXME` comments that are part of upstream Bitcoin Core
- Minor variable naming differences from Bitcoin Core convention
  in Dash-specific code (some divergence is historical)
- Use of `boost::` where `std::` equivalent exists — migration is
  in progress but not complete

## Per-Reviewer Preferences

(This section evolves from feedback. Initially empty.)

## Consensus-Critical Code Areas

Extra scrutiny required for changes in:
- `src/validation.cpp` — block/transaction validation
- `src/consensus/` — consensus parameters and validation
- `src/evo/` — deterministic masternode lists, special transactions
- `src/llmq/` — quorum formation, signing, InstantSend, ChainLocks
- `src/governance/` — governance object validation
- `src/coinjoin/` — CoinJoin transaction construction/validation

Changes here should have:
1. Clear reasoning in PR description
2. Comprehensive tests
3. Consideration of upgrade/activation path
