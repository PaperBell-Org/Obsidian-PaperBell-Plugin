# Unreleased

<!-- Add entries here as you merge. `release.mjs` renames this heading to the
     version being released, inserts a fresh empty one above it, and uses the
     body as the GitHub release notes. A release with an empty Unreleased
     section is refused. -->

# 0.4.8

- **Removed: institution notes.** `window.paperbell`
  (`searchInstitution` / `createInstitutionNote`), the `搜索机构` /
  `创建机构笔记` commands, the graduation-cap ribbon icon, the institution
  settings tab, the `{{ppb.institute.*}}` template placeholders and the
  `Persons/Scholars` create-watcher that auto-generated institution notes are
  all gone. **This is a breaking change if you drove the old API from QuickAdd**
  — but the feature had already stopped working: it called ROR's **v1**
  endpoint, which ROR retired and now answers with `410 Gone`, so every lookup
  failed regardless.

  Institution lookup lives in the **Paper In Bell** sub-plugin now, as its
  built-in script 「关联机构（ROR）」. That one uses ROR **v2**'s `affiliation`
  endpoint, so it accepts names carrying a department prefix, dedupes on
  `ror_id`, and records `ror_id` / `country` / `city` / `established` that the
  host version never had. The only field it cannot fill is `logo`, which ROR v2
  no longer publishes.

  The five settings keys behind the feature (`noteLocation`,
  `institutionNoteTemplate`, `autoGenerateInstitutionNotes`,
  `openInstitutionAfterCreation`, `listenToFileCreatedInPath`) are stripped from
  `data.json` on the first load after upgrading, the same way `registrationId`
  and `pluginGrants` were in 0.4.7. Rolling back to 0.4.7 therefore loses those
  five values — they configured a feature that could not succeed anyway.

# 0.4.7

- **Security: the plugin instance no longer exposes its internals.** Anything a
  companion plugin could reach through `app.plugins.plugins["paperbell"]` was an
  API in practice, and the scope/consent system sat beside it rather than in
  front of it. The consent list was forgeable (`settings.pluginGrants`), the
  activation code was readable in memory, and `verificationWorker` /
  `proxyService` / `agentExecutor` / `updater` were directly callable — also via
  Obsidian's `_children` array. Implementation objects and credentials now live
  in a module-private registry; the instance exposes only `api` and `settings`.
- **Security: `PaperbellSettings` no longer carries secrets or the consent list.**
  `registrationId`, `oauthLoginState`, `pluginGrants` and `agentAutoExec` moved
  out. Existing `data.json` files are read and rewritten unchanged, so nothing is
  lost on upgrade or on a rollback to 0.4.6.
- Fix: companion plugins that were denied every scope no longer appear in
  PaperBell's "connected plugins" list, and the list now refreshes the moment a
  scope is granted.
- Fix: `llm.model` falls back to the built-in provider's default the same way
  `llm.baseUrl` already did — a user who set a key but never picked a model got
  a green "connection OK" and a failing AI call. The error now names the missing
  item instead of listing all three, and the connection test checks the model.
- Fix: a companion plugin holding a client from before a PaperBell reload now
  gets a clear null/`ok: false` instead of an opaque throw. Previously the
  companion swallowed it and called the provider with no key, surfacing an
  unrelated upstream 401.
- Fix: reading the activation code no longer hits the OS keychain on every
  settings render — the settings panel is responsive again.
- Fix: the onboarding wizard's status pills are no longer rendered as dead
  buttons, the complete step has one closing action instead of two identical
  ones, and a failed tutorial image shows an explanation rather than a broken
  icon.
- Chore: 58 unused runtime dependencies removed (a fresh install drops from
  318 MB to 154 MB), type errors down to zero, ESLint added to CI.

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
