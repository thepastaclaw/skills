# Docs Platform — Review Skill

How to weigh findings on `thephez/docs-platform`. Review the supplied exact head,
not a synthetic merge, stale preview, nearby branch tip, or dirty checkout.
Establish the merge-base and report behavior introduced or materially worsened by
the pull request; surrounding code is context, not an expanded finding scope.

The core context preceding this file owns repository invariants. This file sets
severity, validation expectations, and where to look hardest without redefining
those rules.

## Review Posture

Assess only outputs the diff can affect:

- Markdown/MyST and copy-paste instructions
- rendered Sphinx pages, links, and navigation
- browser JavaScript, DOM state, and network behavior
- dependency and generated-static-asset production

A clean build does not prove a changed widget works, and a working widget does
not prove its displayed instructions are accurate.

## Severity Policy

**Blocking** — a core invariant is violated or the PR causes wrong behavior: a
build/publish failure, executable untrusted page content, unsafe DOM/script sink,
credential exposure, write/sign/broadcast in a read-only tutorial, promised
testnet behavior that can use mainnet, stale/missing runtime asset, rounded
platform value, or copy-paste guidance likely to cause loss or incorrect
protocol use.

**Suggestion** — a non-defect improvement or defense-in-depth opportunity, such
as useful additional coverage, clearer maintenance boundaries, or a measured
performance improvement without an established regression. Do not downgrade a
demonstrated race, validation bug, incorrect output, or reproducibility failure
to suggestion merely because it is non-destructive.

**Nitpick** — wording or naming that does not mislead readers or change runtime
behavior. Do not report formatting handled by normal tooling.

Security findings require a concrete source, sink, attacker influence, and
impact. Raw HTML, a large generated file, or absent CSP/SRI is not independently
a vulnerability.

## Validation Expectations

Run gates once per exact head in a writable validation checkout; read-only review
lanes may use trustworthy exact-head CI evidence rather than each reinstalling
and rebuilding independently.

On a fresh checkout, the repository-documented setup and primary gate are:

```sh
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
make sdk-install
make html
```

`make html` regenerates the SDK bundle, so `npm ci && npm run build:sdk` is an
alternative focused asset check, not an extra build required before `make html`.
Use a clean `_build` when stale output could hide a defect. Compare warnings with
the merge-base only when the head emits warnings that may be pre-existing.

Apply additional gates by changed surface:

- Browser runner/markup changes: syntax-check changed JavaScript and exercise an
  affected widget from an HTTP-served clean build across success, invalid input,
  failure, repeated Run, and Reset. Test multiple page depths when static/module
  URL resolution changes. `file://` is evidence only when the changed guidance
  promises direct-file support.
- Mapped tutorial/demo changes: run the repository tutorial-sync check against a
  compatible `platform-tutorials` checkout when available; otherwise state that
  this cross-repository gate was not run.
- Page/toctree changes: in a writable scratch checkout, build first, run the
  sidebar synchronization script, and inspect whether it produces the tracked
  template update the PR should contain.

Missing browser automation is not automatically a finding. Tie coverage findings
to meaningful new logic or an uncovered regression path.

## Where to Look Hardest

For interactive changes, trace changed attributes and parameters through the
operation registry, effective SDK method/network, response renderer, and every
new asynchronous state. Confirm `data-param`, renderer choice, defaults, and
expected response shape agree; operation names and prose are not proof of
read-only behavior.

For dependency/build changes, trace the current manifest and lockfile through
the generated output and deployed import path on both local and Read the Docs
builds. Measure transfer/parse cost when a browser payload materially changes;
do not infer a regression from file size alone.

For prose/reference changes, verify factual claims against the documented
version and authoritative platform source. Check network and credential safety
only in changed examples or behavior newly recommended by the PR.

For workflow changes, apply the public-repository and `pull_request_target`
trust rules from the core context.

## Things Not to Flag

- Behavior wholly outside the PR with no changed behavior depending on it
- Sphinx warnings that reproduce at the merge-base
- Intentional absence of a generated, gitignored vendor asset
- Unchanged CDN-backed demos or remote script examples
- Missing CSP/SRI without a changed executable-resource or policy boundary
- Raw HTML merely permitted by project configuration
- Unit tests for prose-only edits when build/render checks cover the change
