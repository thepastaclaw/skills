# Docs Platform — Core Review Context

Always-loaded context for reviewing `thephez/docs-platform`, a public fork of the
Dash Platform Sphinx/MyST documentation site. Documentation is product behavior:
copy-paste commands, rendered markup, generated navigation, browser JavaScript,
and network-backed examples can introduce correctness and security defects.

Evidence was checked against PR #12 head
`7ce44404c82b7d173c9c1295f8f3c61e4a50abef` on its `4.1.0` base in August
2026. This identifies the reference snapshot; verify mutable implementation facts
against the exact head under review.

## Architecture and Build Contract

The site is built by Sphinx with MyST Markdown. `conf.py` owns Sphinx extensions,
theme/static registration, intersphinx, and site-wide JavaScript. `docs/` contains
tutorial, explanation, DAPI/protocol reference, and SDK content. Custom browser
code and standalone demos live under `_static/`.

The interactive Evo SDK asset is generated from pinned npm inputs and copied by
Sphinx; it is intentionally gitignored. Treat the manifest, lockfile, bundle
entry and output, local Make targets, Sphinx registration, development setup,
and Read the Docs jobs as one reproducibility contract. Read their current
versions and paths from the reviewed tree. A partial update can pass Markdown
review while publishing a missing or stale browser asset.

Tutorial synchronization and sidebar generation are separate derived-content
boundaries. A passing tutorial-sync check does not cover every raw-HTML widget or
the shared runner, and a Sphinx toctree does not by itself update the custom
sidebar.

## Interactive Tutorial Invariants

Tutorial widgets are declarative page data. The shared runner resolves an
explicit operation registry, loads a same-origin generated SDK module, and
performs network queries. Preserve these invariants:

- Never execute Markdown, displayed snippets, DOM text, or attribute values with
  `eval`, `Function`, `AsyncFunction`, dynamic script text, or equivalent hooks.
- Page markup may select only closed, predefined operations and validated
  parameters—not arbitrary SDK methods, URLs, networks, renderers, or code.
- Interactive operations remain read-only and testnet-only: no secrets, wallet
  or key-manager loading, signing, broadcast, or state transition submission.
- Displayed code must remain semantically coupled to the operation that executes;
  a safe-looking example must not mask different runtime behavior.
- Treat user input, SDK responses, and errors as untrusted at DOM and URL sinks.
  Build nodes and assign text; never insert untrusted markup or handlers.
- Enforce required, integer, finite, and range constraints in JavaScript before
  network work; HTML attributes do not enforce a non-submit button flow.
- Preserve exact platform integers without converting values beyond
  `Number.MAX_SAFE_INTEGER` through `Number`.
- Repeated Run, Reset, import failures, and slow completions must not display stale
  results, sample reset values after work starts, or leave controls stuck.
- The displayed network, links, and effective SDK construction must agree.

## Remote Code and Testnet Safety

Keep three resource boundaries distinct: the lockfile-backed same-origin runner
bundle, separately synchronized standalone demos that may use versioned remote
imports, and remote script examples shown as copy-paste documentation. Review
only the boundary changed or newly relied on by the PR.

When executable remote code changes, require an explicit immutable version and
integrity where the loading mechanism supports it; do not weaken CSP or permit
`unsafe-eval` merely to accommodate a runner. The repository does not currently
establish a universal CSP, so missing CSP/SRI is not a finding without a changed
executable-resource or policy boundary.

The JavaScript SDK may default to mainnet. Network-sensitive instructions must
align prose, constructors, credentials, faucet/explorer links, and expected
output. Any sample secret must be obviously disposable and must never look safe
to reuse.

## Public Repository and Workflow Boundary

Treat every committed key, token, mnemonic, or credential as public. GitHub
Actions using `pull_request_target` execute with base-repository privileges: they
must not check out or execute untrusted PR-head code, scripts, or dependencies.
Keep action references immutable and ensure path filters cannot silently skip a
required build or synchronization check.

## Documentation Correctness

Validate changed commands, API names, request bounds/defaults, response shapes,
release annotations, and source links against the versioned implementation they
claim to document. Preserve source-link markup and branch-correct line anchors.
New or moved pages need the correct MyST toctree and synchronized sidebar.
