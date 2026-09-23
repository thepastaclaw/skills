### Callers and callees

For each function, method, type or interface the diff changes, find its callers across the
repository (search for the symbol, including re-exports, bindings and generated code) and
check whether the change breaks any of them: a new precondition they do not meet, a
changed return shape or error type they do not handle, a new panic or exception, a changed
default, an ordering or lifecycle assumption (called before init, after shutdown, twice,
concurrently).

Then the other direction: for each call the changed code makes, check that the callee
still behaves the way the new code assumes, especially where another change in this same
PR modified it.

Also check where the behaviour lives: if the PR patches a symptom in one caller while the
defect is in a shared function that other callers also reach, the other callers stay
broken. Name them.
