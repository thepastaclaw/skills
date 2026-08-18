# Docs Platform — Review Skill

## Review Posture

- Establish the PR's intent and expected head SHA before reviewing. Independently run `git rev-parse HEAD` in the review checkout and require it to equal the head SHA supplied for the run. If it does not, stop: validation or conclusions from another commit do not satisfy the review gate.
- Inspect the actual base-to-head diff and relevant commits. Run validation on the exact head commit, not a synthetic merge commit, a nearby branch tip, or CI output from an older revision.
- Keep findings proportional and in scope: introduced by the PR, exposed by its new behavior, or necessary for its stated goal. Do not turn a focused docs or tutorial change into a general repository audit.
- Favor concrete correctness, security, browser-runtime, dependency, Read the Docs deployment, build, navigation, link, and validation failures over prose taste or code-style preferences.
- This is a public repository. Do not quote private-repository context in findings, and treat newly committed real credentials as a blocking leak.

## Expected Independent Gates

Set up both pinned dependency sets on a fresh checkout, then perform a clean site build:

```bash
pip install -r requirements.txt
make sdk-install
make clean
make html
```

`make html` must rebuild the gitignored `_static/vendor/evo-sdk.js` through the `sdk` dependency. Confirm the completed site contains the generated SDK asset and inspect Sphinx warnings relevant to the changed pages; a zero exit status does not make broken navigation, unresolved references, or a missing browser asset acceptable.

For focused SDK build investigation, use the same commands wired into Read the Docs:

```bash
npm ci
npm run build:sdk
```

For changes to synchronized tutorial code, use an independent `dashpay/platform-tutorials` checkout at the intended source revision:

```bash
python3 scripts/tutorial-sync/sync_tutorial_code.py --check --source /path/to/platform-tutorials
```

For new pages or toctree/sidebar changes, also run:

```bash
python scripts/sync_sidebar.py
```

The repository has no automated browser test suite for the interactive runner. For runner, bundle, or interactive-markup changes, serve or open the built output as the changed flow requires and exercise affected widgets in a browser. Record the exact head SHA, commands, and observed result; do not substitute assumptions or another run's CI status for independent validation. If an external dependency prevents a gate, report the concrete blocker rather than treating the gate as passed.

## Sphinx / Documentation Rules

- New pages must be reachable from the correct root or nested toctree and reflected in the custom sidebar when applicable. An orphaned file that builds but is not navigable is a correctness issue.
- Preserve valid MyST/Sphinx directive structure, local references, intersphinx targets, image/static paths, and page-relative links at the rendered nesting level.
- GitHub source links and line anchors must match the branch named by the URL. Do not validate a versioned link against an unrelated local branch.
- DAPI endpoint documentation must remain synchronized between overview and detail pages and be grounded in the referenced Platform protobuf definitions. Do not accept guessed endpoint names, request fields, or version annotations.
- Tutorial code blocks synchronized from `platform-tutorials` should not be hand-diverged without a deliberate reason. Run the sync check when the PR changes those blocks or the synchronization logic.
- Separate user-visible factual errors and broken examples from editorial preferences. Flag wording only when it is misleading, unsafe, internally contradictory, or prevents users from completing the documented flow.

## SDK Bundle / Read the Docs Rules

- Keep `package.json`, `package-lock.json`, `scripts/evo-sdk-entry.js`, the runner's imported symbols, and the generated bundle contract aligned. A version/export change that bundles successfully but fails at dynamic import or first use is still broken.
- `_static/vendor/evo-sdk.js` is intentionally gitignored. Do not require committing it; require reproducible generation by fresh local and Read the Docs builds.
- Changes to Make targets, `.readthedocs.yml`, devcontainer setup, or `make.bat` must preserve their intended environments. Pay special attention when a new Node/npm prerequisite is added to one path but not another.
- Read the Docs `pre_build` and local `make html` are distinct entry paths. Validate that neither silently publishes a site whose interactive runner points at a missing or stale bundle.
- Dependency changes expand the browser and hosted-build supply chain. Require lockfile integrity and exact package/version alignment; report concrete compromised/unpinned/executable-input risks, not generic objections to npm.
- Workflow path filters must cover the files that can break the gated behavior. A build check that never runs for package, runner, or RTD configuration changes is not meaningful coverage.

## Interactive Tutorial Rules

- Preserve the explicit operation allow-list. Never execute JavaScript assembled from Markdown, raw HTML, `data-*` attributes, displayed source, query parameters, or remote responses (`eval`, `Function`/`AsyncFunction`, script injection, and equivalent paths are blocking regressions).
- Keep displayed snippets and executed behavior tied to the same maintained operation implementation. A tutorial that shows one call but executes another is a user-facing correctness and trust issue.
- Validate `data-operation`, required `data-param` names, renderer choice, defaults, and expected response shape together. Mismatches commonly produce a polished widget that fails only after a network call.
- Current operations are read-only testnet queries through `EvoSDK.testnetTrusted()`. Scrutinize any addition of state transitions, signing, private keys, mainnet selection, or a network label that does not match the SDK client actually constructed.
- Normalize and bound user-controlled numeric/query inputs before SDK calls. Reject invalid values with useful UI errors rather than issuing misleading or unexpectedly expensive queries.
- Render remote data and errors with text-safe DOM APIs. Do not pass testnet responses or exception strings to `innerHTML` or template execution.
- Preserve lazy bundle loading, loading/error/reset states, and repeat-run behavior. Test from rendered pages at more than one Sphinx nesting depth because the SDK URL is resolved from the runner script, not the page URL.
- Check BigInt and SDK wrapper normalization before JSON display; browser rendering must not throw on otherwise valid responses.

## GitHub Actions / Public-Repo Rules

- Treat PR-controlled Markdown, raw HTML, JavaScript, build scripts, and package lifecycle code as untrusted until merge.
- A `pull_request_target` workflow has base-repository privileges. It must not check out the PR head, import PR files, run PR scripts, interpolate attacker-controlled values into shell commands, or expose secrets/tokens to untrusted code. Using event metadata to add a preview URL is a different, narrower boundary.
- Keep third-party actions pinned to immutable commits. For workflow changes, check token permissions and whether the changed path can write PR content or repository state.
- Flag any real secret newly committed to docs, examples, fixtures, generated HTML, JavaScript, logs, or workflow output. Placeholder IDs and public testnet identifiers are not secrets.

## Test Expectations

- A docs-only correction should at least pass the clean Sphinx/SDK build and relevant link/navigation inspection.
- A tutorial-sync change should pass the external-source `--check` command against the intended source revision.
- A runner or bundle change needs independent browser evidence for every affected operation/renderer class and failure state. Add automated coverage when practical, but do not invent a nonexistent project test command.
- A build/RTD/workflow change needs validation of each entry path it modifies, including failure behavior on a fresh checkout.
- Bug fixes should include a regression check at the narrowest layer that would have caught the defect.

## Things Not To Flag

- The generated `_static/vendor/evo-sdk.js` being absent from git; it is deliberately generated and ignored.
- Use of framework-free JavaScript, MyST `{raw} html` wrappers for the existing declarative widgets, or Sphinx/pydata theme conventions solely as style choices.
- Read-only calls to public testnet services or public testnet IDs used in examples, absent a concrete privacy, safety, or correctness problem.
- Existing `esm.sh` use in standalone lite examples when an unrelated PR only changes the interactive runner; review it when the PR modifies that dependency path.
- Pure formatting, wording, or generated sidebar churn unless it changes meaning, navigation, links, security, or rendered behavior.
- Missing broad unit/E2E infrastructure by itself. Require evidence appropriate to the changed behavior, not an unrelated testing-framework migration.
