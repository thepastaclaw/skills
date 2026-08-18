# Docs Platform — On-Demand Reference

Path-specific detail for `thephez/docs-platform`. Durable invariants live in
`review-core.md`; this file does not restate them. Each `##` section begins with
an HTML-comment path marker. The orchestrator injects only sections matching the
changed files; `(manual)` sections are never selected automatically.

## Interactive Tutorial Runtime
<!-- paths: _static/js/**, docs/tutorials/**, _static/css/pydata-overrides.css, conf.py -->

At the August 2026 reference head, the shared runner resolves a fixed operation
registry, constructs a trusted testnet Evo SDK client for each operation, and
lazily imports the same-origin generated module after Run. Enumerate the current
registry from the reviewed JavaScript rather than relying on a saved operation
list.

The prior DOM-snippet `AsyncFunction` design was removed in favor of predefined
functions. For changed widgets, verify `data-operation`, required `data-param`
names, renderer, defaults, status text, displayed source, SDK argument types,
and response shape as one contract.

Specific failure modes to reproduce when relevant:

- a click bypasses native form-submit validity checks;
- `Number(value).toLocaleString()` can round `bigint`/large decimal results;
- Reset during module import can change parameters or allow a stale completion;
- rejected import or stalled connection can leave controls disabled;
- `data-network` can be only display metadata unless SDK construction consumes
  and validates it;
- directly opened `file://` pages may not support the same module/CORS behavior
  as the deployed HTTP site.

If CSP changes, test the built page with the actual same-origin module and any
SDK-created worker/WASM requests. Account for separately configured theme,
analytics, and embed hosts without adding `unsafe-eval`.

## SDK Bundle Pipeline
<!-- paths: package.json, package-lock.json, Makefile, make.bat, .readthedocs.yml, .devcontainer/**, scripts/evo-sdk-entry.js, _static/vendor/**, _static/js/interactive-tutorial.js, conf.py, .gitignore -->

The reference pipeline bundles the SDK as browser ESM into a gitignored vendor
asset before Sphinx copies static files. `package-lock.json` and `npm ci` define
the install. Local Make targets, Windows/build entry points, development setup,
Sphinx registration, and Read the Docs jobs must agree on the current entry,
output, and build order.

Do not treat the existence of a bundler binary in `node_modules` as proof that
installed dependencies match a changed lockfile. Validate from a clean install
or an equivalent lockfile-dependent stamp. For dependency/bundling changes,
measure the clean base and head assets and verify lazy loading, caching,
import/parse behavior, and low-bandwidth failure feedback. Use an established
budget or measured regression, not a historical preview size.

## Standalone Browser Demos
<!-- paths: _static/*-lite.html -->

Standalone lite demos are synchronized whole from `platform-tutorials` and have
a dependency-loading boundary separate from the shared runner. At the reference
head they use an explicitly versioned remote Evo SDK import. When that boundary
changes, verify the source mapping, immutable version, CSP/source policy, and SRI
where the loading mechanism supports it. Do not require lockfile changes for an
untouched remote-demo boundary.

## Tutorial Synchronization
<!-- paths: docs/tutorials/**, scripts/tutorial-sync/**, .github/workflows/check-tutorial-sync.yml, _static/*-lite.html -->

`scripts/tutorial-sync/tutorial-code-map.yml` identifies mapped code regions and
whole-file demos owned by `dashpay/platform-tutorials`. Raw HTML widget shells,
the shared renderer/runtime, and surrounding explanations remain local unless
the current map says otherwise. Avoid independent edits to mapped content and
run the check against a compatible source checkout when practical.

## Navigation and Sidebar
<!-- paths: index.md, docs/index.md, docs/**/index.md, _templates/sidebar-main.html, scripts/sync_sidebar.py, conf.py -->

The root and nested MyST indexes own page discoverability. The custom sidebar is
derived from current built HTML, so build before synchronization and inspect the
result for omitted pages or accidental rewrites. A Sphinx warning and a stale
sidebar detect different failures.

## DAPI and Protocol Reference
<!-- paths: docs/reference/**, docs/protocol-ref/**, RELEASE.md -->

Keep DAPI overview rows and detail pages synchronized. Validate changed endpoint
names, request bounds/defaults, response fields, version annotations, and source
links against the platform protobuf/source for the documented branch. Preserve
source-link markup and branch-correct line anchors.

## Network-Sensitive Examples
<!-- paths: docs/tutorials/**, docs/sdk-js/**, docs/dapi-client-js/** -->

Where testnet is the intended safety boundary, changed examples must configure it
explicitly because SDK defaults can differ by version. Put wallet/payment or
credential warnings next to the risky step, use disposable sample values, and
keep constructors, links, prose, and expected output on the same network.

## GitHub Actions and Public-Repo Safety
<!-- paths: .github/workflows/** -->

For `pull_request_target`, never check out or execute PR-head code, scripts, or
package installation under base-repository credentials. Keep third-party actions
immutably pinned. Verify path filters include new executable, generated, and
synchronized sources so required checks cannot silently stop running.
