# Verify one review candidate

A reviewer raised the candidate below on pull request **#{pr_number}** in **{repo}**,
checked out here at `{head_sha}`. Decide whether it is real, using the code, not the
reviewer's confidence. You are checking one claim; take the time to read what it depends
on (the enclosing function, callers, the guards around it).

## The candidate

```json
{candidate}
```

{context_note}

## Project context

{project_skill}

{review_skill}

## The PR

- Title: {pr_title}
- Review range: `git diff {coverage_from}..{head_sha}`

{pr_description}

## Verdict

Return exactly one of:

- `CONFIRMED`: you can name the inputs or state that trigger the failure and the wrong
  result it produces. Quote the line that shows it.
- `PLAUSIBLE`: the mechanism is real, but whether it triggers depends on something you
  cannot settle from the code (timing, environment, configuration, a caller outside this
  repository). Say in `confirm_by` what would confirm it.
- `REFUTED`: the code shows the claim is wrong. Quote the line that contradicts it, show
  the invariant or type that rules it out, cite the guard that already handles it, show
  that this diff already fixes it, show that it lies outside what this PR changes or
  needs, or show that it is style with no observable effect.

Some candidates look speculative but describe real failures. A race on shared state, a
null or error on a rare but reachable path (an error handler, a cold cache, a missing
optional field), zero or empty treated as missing, an off-by-one at a boundary the code
does not exclude, a retry or partial-failure path, an allowlist or pattern that lost its
anchor: these are PLAUSIBLE unless the code rules them out. "It depends on runtime state"
is not a refutation when that state is realistic.

For a backport of an upstream merge: a hunk that is missing because Dash lacks the code it
would edit is the gap the candidate describes, not a refutation. A declared omission (the
PR description, a `partial` subject, or a `not_for_dash` adaptation note) is evidence
against it. Upstream's own design choices are not findings; a backport must match
upstream.

Also return `severity`: `blocking` for a bug, security hole, data loss, consensus failure
or crash that must be fixed before merge; `suggestion` for a real improvement that is not
a defect; `nitpick` for something minor. Judge it yourself; the reviewer's severity is only
a starting point. Put your reasoning, with the quoted lines, in `evidence`.
