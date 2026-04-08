# Sourceful Legal Documents

Public repository for Sourceful Energy's legal documents — Terms and Conditions, Privacy Policy, and any future documents that users formally consent to.

This repo is the **source of truth** for the documents themselves. Acceptance records (who agreed to what version, when) live in NovaCore's `identity_consents` table — see `srcful-novacore/services/core-api/internal/consent/`.

## Structure

```
.
├── terms-and-conditions/
│   ├── manifest.json              ← points to the current version
│   └── v1/
│       ├── meta.yml               ← version metadata (date, supersedes, ...)
│       ├── en.md                  ← English source
│       └── sv.md                  ← Swedish translation
│
├── privacy-policy/
│   ├── manifest.json
│   └── v1/
│       ├── meta.yml
│       ├── en.md
│       └── sv.md
│
├── VERSIONS.md                    ← human-readable changelog of all versions
└── README.md                      ← this file
```

### Per-document layout

Each document (terms-and-conditions, privacy-policy, ...) is a top-level folder containing:

- **`manifest.json`** — small machine-readable file naming the *current* version and which locales are available. The app fetches this on startup to know which version to show.
- **`v{N}/`** — one folder per published version. Each version is immutable once shipped.
  - **`meta.yml`** — that version's metadata (effective date, what it supersedes, available locales).
  - **`en.md`** — English source text. This is the canonical version legal owns.
  - **`sv.md`, `de.md`, ...** — translations, one file per locale. Committed directly into the same folder.

## URLs (consumed by the app)

The Flutter app fetches documents via GitHub raw URLs:

```
https://raw.githubusercontent.com/srcfl/legal/main/terms-and-conditions/manifest.json
https://raw.githubusercontent.com/srcfl/legal/main/terms-and-conditions/v1/en.md
https://raw.githubusercontent.com/srcfl/legal/main/terms-and-conditions/v1/sv.md
```

These URLs are stable. Once a version folder is published, its files never change — new versions go in new folders.

## Translations

Translations are committed directly into the same version folder as the English source. No tooling, no Crowdin, no auto-sync — translations are reviewed in PRs the same way the English text is.

**To add a new translation to an existing version:**

1. Copy `en.md` to `<locale>.md` (e.g. `sv.md`, `de.md`).
2. Translate the text. **Keep the markdown structure identical** to the English source so versions are diff-comparable.
3. Add the locale to `available_locales` in the document's `manifest.json`.
4. Add the locale to `languages:` in that version's `meta.yml`.
5. Open a PR. Legal review of the translation happens in the PR.

A locale should only appear in `manifest.json` / `meta.yml` once the translation is complete and reviewed. Placeholder files in the repo (with TODO content) are fine, but they should not be advertised as available until the text is real.

## Publishing a new version

When legal updates a document:

1. **Copy the current version folder** to the next number:
   ```bash
   cp -r terms-and-conditions/v1 terms-and-conditions/v2
   ```
2. **Edit `terms-and-conditions/v2/en.md`** with the new text.
3. **Update every `<locale>.md` in `v2/`** with the corresponding translation (or leave as a TODO placeholder and remove that locale from `manifest.json` / `meta.yml` until ready).
4. **Update `terms-and-conditions/v2/meta.yml`**: bump `version`, set `effective_at`, set `supersedes: v1`, list available `languages`.
5. **Update `terms-and-conditions/manifest.json`**: change `current_version` to `v2`, update `available_locales`.
6. **Append an entry to `VERSIONS.md`** describing what changed.
7. **Open a PR** for legal review.
8. **After merge, bump the app constant.** The Flutter app has a hardcoded `currentTosVersion` / `currentPrivacyVersion` constant — update it to `v2` and ship a release. Until then the app keeps showing v1.

The previous version (`v1/`) stays in the repo forever — users who accepted it have a permanent reference to the exact text they agreed to.

## How NovaCore stores consent records

When a user taps "Accept" in the app, NovaCore records a row in `identity_consents` with:
- `consent_type` — `platform_tos` or `privacy_policy` (backend constant — see note below)
- `document_version` — the version string from `meta.yml`, e.g. `v1`
- `document_hash` — SHA-256 of the exact document text the user was shown (computed by the app)
- `document_url` — the raw GitHub URL the document was fetched from

The hash is the integrity binding: it proves the user accepted *this specific text*, not just the version label.

### Naming note

The legal repo uses public-friendly folder names (`terms-and-conditions`, `privacy-policy`). The backend uses programmatic constants (`platform_tos`, `privacy_policy`). The app maps between them:

| Public name (this repo)  | Backend `consent_type` |
|--------------------------|------------------------|
| terms-and-conditions     | platform_tos           |
| privacy-policy           | privacy_policy         |
