### Removed and replaced behaviour

Look only at what the diff deletes or replaces. For every removed or rewritten line, name
the job it did: the guard, validation, error path, invariant, cleanup, lock, ordering or
test case it provided. Then find where the new code does that job. If nothing does, that
is a candidate: a guard that disappeared, an error path that now falls through, a
validation that got narrower, a cleanup that no longer runs on one path, a lock scope that
shrank, a test that covered a real case and was deleted rather than updated.

Code that moved counts too: when a block is extracted or moved, check that its guards,
early returns and anchors came with it.

A removal the PR's stated goal explains, and that nothing else depended on, is not a
finding.
