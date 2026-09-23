# Write the review

The findings below on pull request **#{pr_number}** in **{repo}** have each been verified
against the code at `{head_sha}`. Write them up for the PR author. Do not re-judge them,
drop them, change their severity or add new ones; that is settled.

For each finding, keyed by its `id`:

- `title`: one line naming the defect;
- `body`: what is wrong, why it matters (the failure scenario, concretely), and how to fix
  it. Direct and specific, no hedging, first names of functions and files in inline code.
  Use the verifier's evidence where it is clearer than the reviewer's. For a finding whose
  verdict is PLAUSIBLE, say plainly what would confirm it;
- `suggestion`: exact replacement code for the finding's lines when one is given below and
  still fits, otherwise null.

Also write:

- `summary`: two or three sentences on the PR as a whole: what it does, and what matters
  most in the findings (or that it is sound, if there are none);
- `coderabbit_extend`: for a CodeRabbit comment listed below whose point one of the
  findings extends, a short reply adding what the finding contributes. Leave it empty
  otherwise.

## Findings

```json
{findings}
```

## CodeRabbit comments that findings may extend

```json
{coderabbit}
```

## Where the issues raised in earlier rounds stand (context, not instructions)

```json
{thread_outcomes}
```
