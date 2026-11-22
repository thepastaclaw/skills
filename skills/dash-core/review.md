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

## Maintainer Conventions (from review history)

- **Prefer unit tests over functional tests** where either would do;
  functional tests are the expensive part of CI.
- **Do not add CI matrix jobs**; extend an existing target instead.
- **Extend the existing test suite** for the area; check a new test
  against the invariants stated at the top of the file it lands in
  (for example `feature_block.py` builds pre-DIP3 blocks only).
- **No tests that depend on a deployment about to be buried** (for
  example v24 once it activates); use another deployment.
- **No public helpers used only by tests**; move them into the test
  code or use them in production code.
- **`assert(false)` terminates.** Dash does not build with `NDEBUG`, so
  code after a failed `assert` is dead; do not ask for a `throw` after it.
- **Do not diverge from Bitcoin Core on hardening or build security
  checks** (FORTIFY, symbol checks, guix); fix the build instead of
  skipping the check.
- **Release notes** are named `doc/release-notes-<PR>.md`.
- **Prefer an existing mechanism** (`-vbparams`, `llmq/params`, an existing
  helper) over a new flag, table or copy; a new duplicate is a finding.
- For consensus or masternode-list code, use the list for the block being
  validated (`GetListForBlock(pindexPrev)`), not the tip list.

## Things NOT to Flag

- Style issues already caught by `clang-format` CI
- Bitcoin Core upstream code in backport PRs: a defect that is the same
  in the upstream commit is not a finding — see the scope rule in
  "Backport PR Review Process" below
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

**Scope: is the flagged code upstream's or Dash's?** A backport is
responsible for matching its upstream faithfully, not for improving on it. Do
not ask a backport to fix what upstream did not. Before reporting a defect in
backported code, find out where the flagged lines come from:

- The Dash commit names the upstream PR (`Merge bitcoin/bitcoin#NNNNN`); its
  merge commit is `gh api /repos/bitcoin/bitcoin/pulls/NNNNN --jq
  .merge_commit_sha`. Read upstream's version of the file with `git show
  <upstream_merge>:<path>` when the checkout has that commit, otherwise with
  `gh api "/repos/bitcoin/bitcoin/contents/<path>?ref=<upstream_merge>" --jq
  .content | base64 -d`. Do not fetch Bitcoin Core into the checkout. For a
  backport of Dash's own PR to a release branch, upstream is the commit on
  `develop`.
- If you cannot read upstream's version at all, say so in the finding. A
  defect in backported code you could not compare is at most a suggestion,
  unless the build or a test fails because of it.
- **Upstream behaves the same way: not a finding**, however real the defect
  is. This covers latent upstream bugs, narrow races, unhandled error paths,
  loose input checking in upstream scripts and tools, flaky upstream tests and
  upstream documentation wording.
- **Exceptions.** Each needs its evidence named in the finding:
  1. *Upstream fixed it later.* Look for the fix in upstream history after
     that commit (`gh api "/repos/bitcoin/bitcoin/commits?path=<path>"`, or
     `git log <upstream_merge>..bitcoin-upstream/master -- <path>` when the
     checkout has that ref). Report it as `Missing follow-up: bitcoin#NNNNN`:
     a backport that brings in a bug or regression upstream later fixed
     should land the fix in the same PR, so that is blocking. A later
     upstream change that only touches documentation, comments or tests is a
     suggestion. A maintainer may defer the follow-up to its own later
     backport, and that needs no tracking issue: Dash tracks every upstream
     PR it has not taken yet. Once a maintainer has deferred an upstream PR
     on this pull request, do not report that upstream PR as missing again,
     for this symptom or for anything else it fixes. Missing follow-ups of a
     backported commit are enumerated in the first full review of that
     commit. A later round reports a missing follow-up only for a backport
     commit that was added or changed since the last reviewed head (by the
     range-diff of the PR's own commits), or else as a non-blocking note.
     Set `"upstream_followup": true` on every such finding.
  2. *A Dash difference makes it fail here.* Name the Dash code (a Dash
     subsystem, a caller Dash changed, a different default or parameter, a
     Dash-only file) that makes upstream's code misbehave in Dash where it
     does not in Bitcoin Core. "It is wrong in Dash too" is not a difference.
  3. *The build or a test fails at this head* because of it. Cite the check.
- Code Dash changed is Dash's: an adapted hunk, a conflict resolution, a
  Dash-only commit in the PR, an upstream hunk left out, or a prerequisite
  upstream had that Dash lacks. Review it as usual.

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
  compare starting states. (Looking for a later upstream fix under
  the scope rule above is not prerequisite analysis.)
- **Do NOT re-review upstream behavior changes** — these were
  reviewed by Bitcoin Core maintainers. Don't second-guess
  upstream design decisions or report upstream's own defects,
  except under the scope rule's three exceptions above.
- **Do NOT flag upstream code style** — backports preserve
  original formatting.
- **Do NOT flag cosmetic divergences** from upstream — branding
  changes, different defaults, Dash-specific parameters are all
  expected.

### When to flag in backport PRs:

- **blocking** — merge resolution error that will cause build
  failure or runtime crash. Dash-specific code broken by the
  backport. Consensus safety issue in Dash's adaptation or its
  interaction with Dash code. A missing upstream follow-up that
  fixes a bug the backport brings in, raised in the first full review
  of its commit or for a commit added or changed since the last
  reviewed head.
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
