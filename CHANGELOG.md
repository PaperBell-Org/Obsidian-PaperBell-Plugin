# Unreleased

<!-- Add entries here as you merge. `release.mjs` renames this heading to the
     version being released, inserts a fresh empty one above it, and uses the
     body as the GitHub release notes. A release with an empty Unreleased
     section is refused. -->

# 0.4.6

- Feat: meter host-mediated AI calls for unlicensed users — 5 free calls per
  local day, unlimited once activated
- Feat: split the config broadcast into a public tier (profile and language,
  any plugin may listen) and a restricted tier (account, licence, AI config and
  CIMPO folders, handshaked companion plugins only)
- Feat: broadcast the five CIMPO folder paths to companion plugins, and add a
  settings tab to configure them
- Fix: stop reporting a licence as invalid when verification simply could not
  run (offline, rate-limited, timed out) — an offline paying user is no longer
  demoted to the free tier
- Fix: route the proxy service's licence checks through the verification worker
  so both paths agree on one device id and one verdict
- Refactor: merge the two path suggesters and drop the dead folder selector
- Chore: PR gates in CI (tests, a type-error baseline, build and artifact
  checks), pnpm pinned via `packageManager`

# 0.4.5

- Refactor: PaperBell becomes the host of the companion-plugin family. Section
  View is split out into its own plugin, and the settings page gains entry
  cards for registered companion plugins
- Requires Obsidian 1.11.4 or later, for the secret storage API that now holds
  activation codes and AI keys

# 0.3.0 ~ 0.4.4

Not recorded. This changelog went unmaintained between 0.2.7 (2025-07-10) and
0.4.4 (2026-06-07), and the source history for that span is not present in the
current repository, so the entries cannot be reconstructed honestly. See the
[Releases page](https://github.com/PaperBell-Org/Obsidian-PaperBell-Plugin/releases)
for the artifacts and dates of those versions.

# 0.2.7

- fix: updater not working correctly

# 0.2.5~0.2.6

- Fix bug that prevented the proxy service from working

# 0.2.4

- Feat: support web verification of paperbell plugin, you can buy one from https://paperbell.cn
- Revert: disable network updates, you can still update manually using local files through the status bar or command palette.


# 0.2.1~0.2.3

- Fix bug that prevented the updater from working
- Fix regex pattern for glob patterns that contain `*`

# 0.2.0

- feat: support update and backup for vault
- feat: add updater setting
- feat: add updater status bar item
- feat: add updater conflict view

# 0.1.6

- Fix bug that prevented the verification process from completing


# 0.1.3

- Refactor to use `ppb.institute` instead of `paperbell` for future compatibility
- Add `lat` and `lon` to the template variables
- Add `logo` to the template variables
- Support opening the newly created institution note after creation

# 0.1.2

- Support custom templates for institution notes
