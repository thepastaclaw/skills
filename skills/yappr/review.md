# Yappr — Review Skill

## Review Posture

- Review only the PR under consideration. Keep findings scoped to behavior introduced or exposed by the PR.
- This is a public repo. GitHub comments are public; avoid private operational context.
- Favor concrete correctness, data-loss, security/privacy, Platform contract/query, token/payment, and user-facing workflow issues over style.

## Expected Local Gates

```bash
npm install
npm run lint
npm run build
```

If the PR changes generated/static export behavior, also inspect `next.config.js` and any build script used for the deployed target.

## Dash Platform Rules

- Contract JSON, constants, and service query code must agree on contract IDs, document type names, indexes, and token IDs.
- Count-tree reads must use indexes declared with count support in the target contract; do not assume a query can be counted just because it can be fetched.
- Data-contract cutovers should not silently strand old data without an intentional migration or user-visible legacy path.
- SDK version bumps should be checked for API and wire-format changes, especially around documents, tokens, direct purchase, identity credits, and protocol-version expectations.
- Do not coerce token/credit values through `Number` when precision can matter; prefer string/BigInt-safe handling.

## Security / Privacy Rules

- Never log, persist, or display user private keys, WIFs, mnemonics, seed material, encryption keys, backup passwords, upload-provider credentials, or decrypted private content.
- Encrypted private-feed, DM, key-backup, and checkout/order paths need explicit review of who can read plaintext and where decrypted data lives.
- Browser-only signing flows must not send private key material to external APIs beyond intended SDK signing calls.
- Moderation/owner actions such as freeze, unfreeze, slash, or destroy must be gated by the intended authority and fail clearly for normal users.

## UI / Workflow Rules

- Token-buying, posting, replying, liking, reposting, following, checkout, and moderation flows need loading, error, retry, and insufficient-balance states.
- Modal and dropdown additions should preserve keyboard accessibility and avoid trapping users in unactionable states.
- Mobile layout matters; Yappr is mobile-first.
- Legacy/cutover links should be visible when old content is intentionally absent.

## Test Expectations

- Bug fixes and new service-layer logic should have focused regression coverage when practical.
- For contract/query changes, review whether each app query has been audited against the new contract indexes.
- For payment/token changes, prefer tests or clearly documented manual validation of debit, max-cost guard, insufficient-balance, buy minimum, and authority failure paths.

## Things Not To Flag

- Lack of a backend database; Yappr intentionally uses Dash Platform as its state layer.
- Static export / GitHub Pages deployment by itself.
- Testnet-only IDs or keys when they are clearly documented as public test fixtures, unless a PR newly risks mainnet use or private leakage.
