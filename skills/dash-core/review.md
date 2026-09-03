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
Bitcoin Core and were already reviewed there.

**A dedicated backport-reviewer specialist handles prerequisite
analysis.** General review agents should focus only on:

1. **Critical correctness** — will this compile? Are there
   obvious runtime errors introduced by the merge resolution?
2. **Dash-specific interaction** — does the backported change
   break or conflict with Dash subsystems (LLMQ, evo,
   governance, CoinJoin, etc.)? If the upstream change modifies
   validation, networking, or wallet code, verify Dash extensions
   still work correctly.
3. **Test adaptation** — were Bitcoin Core tests properly adapted
   for Dash parameters (block times, reward structure, etc.)?
4. **`non-backported.txt`** — new Dash-specific files need to be
   listed here.

### What general agents should NOT do on backport PRs:

- **Do NOT perform prerequisite analysis** — the backport-reviewer
  specialist owns this. Don't trace upstream dependency chains or
  compare starting states.
- **Do NOT re-review upstream behavior changes** — these were
  reviewed by Bitcoin Core maintainers. Don't second-guess
  upstream design decisions unless they are critical/consensus-
  affecting in the Dash context.
- **Do NOT flag upstream code style** — backports preserve
  original formatting.
- **Do NOT flag cosmetic divergences** from upstream — branding
  changes, different defaults, Dash-specific parameters are all
  expected.

### When to flag in backport PRs:

- **blocking** — merge resolution error that will cause build
  failure or runtime crash. Dash-specific code broken by the
  backport. Consensus safety issue.
- **suggestion** — potential Dash subsystem interaction concern.
  Test adaptation that might need Dash-specific adjustment.
- **nitpick** — minor observation about the adaptation quality.

Keep findings minimal and high-signal. The backport-reviewer
specialist handles the thorough upstream comparison work.

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
