# Dash Core — Review Skill

## Code Style

- Follows Bitcoin Core style: `clang-format` enforced in CI
- C++20
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
- Bitcoin Core upstream code in backport PRs — see "Backport PR
  Review Process" above for the specialized review approach
- `TODO` / `FIXME` comments that are part of upstream Bitcoin Core
- Minor variable naming differences from Bitcoin Core convention
  in Dash-specific code (some divergence is historical)
- Use of `boost::` where `std::` equivalent exists — migration is
  in progress but not complete

## Per-Reviewer Preferences

(This section evolves from feedback. Initially empty.)

## Backport PR Review Process

Backport PRs (titles starting with `backport:` or containing
`Merge bitcoin#`) require a **different review approach** than
regular feature PRs. The code changes originated upstream in
Bitcoin Core and were already reviewed there — re-reviewing the
behavior changes is mostly unnecessary.

### What to check in backport PRs:

1. **Conflict resolution:** The merge commits are the primary
   review target. Did the conflict resolution preserve both the
   upstream intent AND the Dash-specific code? Are there incorrect
   deletions of Dash-specific logic during merge?

2. **Missing prerequisites:** Does this backport depend on earlier
   Bitcoin Core changes that haven't been backported yet? Look for:
   - References to functions/types that don't exist in the Dash
     codebase
   - Changed function signatures that don't match callers
   - Missing `#include` for newly used headers

3. **Dash-specific interaction:** Does the backported change
   interact with Dash-specific subsystems (LLMQ, evo, governance,
   etc.)? If the upstream change modifies validation, networking,
   or wallet code, check that the Dash extensions still work
   correctly after the merge.

4. **`non-backported.txt` updates:** If the backport adds new
   Dash-specific files (e.g., adaptation layers), they must be
   added to `test/util/data/non-backported.txt`.

5. **Test adaptation:** Were Bitcoin Core tests properly adapted
   for Dash? Tests that reference Bitcoin-specific behavior may
   need adjustment for Dash parameters (block times, reward
   structure, etc.).

### What NOT to review in backport PRs:

- **Upstream behavior changes** — these were reviewed by Bitcoin
  Core maintainers. Don't second-guess upstream design decisions
  unless they are critical/consensus-affecting in the Dash context.
- **Upstream code style** — backports preserve original formatting.
- **Upstream commit messages** — these preserve original authorship.

### When to flag in backport PRs:

- Only flag issues that are **critical** (security, consensus) or
  specific to the **Dash adaptation** (merge conflicts, missing
  prerequisites, broken Dash-specific features).
- Use severity `blocking` only for merge errors or missing
  prerequisites that would cause build/runtime failures.
- Prefer `suggestion` severity for potential interaction concerns.

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
