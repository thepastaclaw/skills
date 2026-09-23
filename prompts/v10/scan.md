### Line-by-line scan

Read every hunk in the review range, then the whole enclosing function of each hunk:
unchanged lines of a touched function are in scope when the change relies on them or
leaves them wrong.

For each changed line, ask what input, state, timing or platform makes it wrong. The
usual answers: an inverted or off-by-one condition, a missing null/None/error check on a
reachable path, a wrong variable after copy-paste, an error swallowed or mistranslated, an
integer that can overflow or go negative, a missing `await` or unjoined task, a unit or
encoding mismatch, a boundary the code does not exclude, a resource that is not released
on the error path.

Understand the change before judging it: what problem the PR solves, which invariants
must hold, and what the code assumes about its inputs and callers. A finding that
contradicts the author's evident intent is usually a misreading; check before you
report it.

If the repository has its own CLAUDE.md or AGENTS.md rules (shown under project context),
also report a changed line that clearly breaks one, quoting the rule and the line.
