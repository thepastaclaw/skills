## Backport prerequisite claims (verifier only)

A **prerequisite claim** is a reviewer finding that says a backport PR is
missing part of the upstream Bitcoin Core PR it says it backports: a hunk, a
symbol, a file, a test section, or a subsystem that hunk depends on. No later
stage re-checks these, so every one you receive must get an explicit answer
from you.

For each prerequisite claim:

1. **Check it yourself.** Read the upstream PR's diff, compare the upstream
   code as it was before that PR with Dash's base and head, and trace the
   missing symbol with `git log -S`. Do not decide by which reviewer raised it.
2. **Decide KEEP or DROP.**
   - When the PR presents itself as the whole upstream PR (for example
     `Merge bitcoin#NNNNN`), a hunk left out because Dash lacks the code it
     touches is a missing prerequisite, not a harmless adaptation. Keep it,
     as blocking, even if the Dash tree compiles.
   - Drop it only with evidence: the code is already there (name the commit),
     the claim is wrong (show the upstream diff), or the PR intentionally
     leaves that area out (the PR description, `non-backported.txt`, or a
     maintainer says so).
3. **Record it.** Add one entry to `prerequisite_adjudications`:
   `{"claim": "<the finding's title>", "source": [...], "verdict": "KEEP|DROP",
   "evidence": "<what you checked and found>"}`.
   - KEEP: the finding also appears in `findings`.
   - DROP: it also appears in `dropped_findings`, with a `reason` that starts
     with `OUTDATED`, `REFUTED`, `NOT_ACTIONABLE`, `FIXED` or
     `INTENTIONAL_EXCLUSION`, followed by the evidence.

If you cannot give every prerequisite claim a verdict with evidence, set
`adjudication_complete` to `false`. The run is then retried rather than
published with a guess.

Example:

```json
"prerequisite_adjudications": [
  {
    "claim": "Missing upstream validation hunk from Merge bitcoin#12345",
    "source": ["phase2"],
    "verdict": "KEEP",
    "evidence": "bitcoin#12345 adds CheckFoo() at validation.cpp L1200-1218; the PR title claims the whole merge but head omits this hunk, and git log -S'CheckFoo' shows Dash never had the symbol."
  },
  {
    "claim": "Missing wallet RPC section from Merge bitcoin#12345",
    "source": ["phase1"],
    "verdict": "DROP",
    "evidence": "The PR description says the wallet RPC changes are excluded and tracked in dashpay/dash#9999."
  }
]
```
