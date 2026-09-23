# Group review candidates and match them to issues already raised

Several independent reviewers examined pull request **#{pr_number}** in **{repo}** at
`{head_sha}`. Before anything is verified or published, their candidates need two things
done:

1. **Group** candidates that describe the same defect: the same root cause at the same
   place, even when the wording, line range or severity differs. Different defects stay
   separate, even on the same line. Pick the member with the most concrete failure
   scenario as the group's `representative`.
2. **Match** each group against the PR's issue ledger (issues raised in earlier rounds)
   and CodeRabbit's open comments:
   - `open:<hash>`: the group is the same issue as an open ledger entry. It will be treated
     as fresh evidence on that issue's existing thread, not as a new comment.
   - `closed:<hash>`: the same issue as a ledger entry that was fixed, withdrawn,
     deferred or outdated. The harness decides what happens next (a fixed issue that
     reappears is a regression and gets verified; a withdrawn one only comes back if its
     code changed since).
   - `coderabbit:<comment_id>`: the same point CodeRabbit already made.
   - `new`: none of the above.

Match on substance: same defect, same code. Similar wording about different code is not a
match; different wording about the same defect is. When unsure between `new` and a match,
choose `new`; the verifier and publication still guard against duplicates.

Put every candidate id in exactly one group. You may read the code in the worktree when a
decision needs it, but most do not.

## Candidates

```json
{candidates}
```

## Issue ledger for this PR

```json
{ledger}
```

## Open CodeRabbit comments

```json
{coderabbit}
```

Return `groups`: for each, `members` (candidate ids), `representative` (one of them),
`match` (as above), and a short `reason`.
