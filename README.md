# PaperBell

> [中文文档](./README-ZH.md)

PaperBell is the **host plugin** of the PaperBell plugin family for Obsidian.

It does two things, and deliberately not much else:

1. **Talks to the PaperBell server** — licence activation, OAuth sign-in, the
   authenticated request proxy, protected download tickets, and host-brokered
   LLM calls.
2. **Owns configuration centrally** — one account, one licence, one AI provider,
   one CIMPO folder layout for the whole vault. Companion plugins read that
   configuration from PaperBell instead of each asking the user again.

Feature surfaces (card management, paper search, writing tools, …) live in
separate companion plugins. They register with PaperBell over the IPC contract
documented below, and inherit its account, licence and AI configuration.

---

## Install

Install via [BRAT](https://github.com/TfTHacker/obsidian42-brat) with this
repository, or download `main.js`, `styles.css` and `manifest.json` from a
[release](../../releases) into `<vault>/.obsidian/plugins/paperbell/`.

## What users get

| Area | What PaperBell provides |
|---|---|
| Account | Sign in via `paperbell.cn` (OAuth) or paste an activation code |
| Licence | Activation state, expiry, plan; offline grace period |
| AI | One place to configure provider, base URL, model and API key |
| CIMPO folders | The five top-level vault folders every companion plugin agrees on |
| Profile | Name, title, email, institution, avatar — filled once, shared |
| Permissions | Per-plugin, per-scope consent list you can revoke at any time |

Everything is configured in one place: Obsidian **Settings → PaperBell → "打开设置" (Open settings)**.

---

# Developer reference

Everything below is the stable contract companion plugins may depend on.
The complete TypeScript declaration is reproduced in [Appendix A](#appendix-a--the-contract-in-full)
at the end of this file — it has **zero imports**, so you can paste it straight
into your own plugin and compile against it. (The canonical definition lives in
the plugin's private source tree; this repository ships only release artifacts,
so the appendix is what you vendor.)

**Current contract version: `PPB_SCHEMA_VERSION = 2`.**

## 1. Entry points

| Symbol | Where | Type |
|---|---|---|
| `window.registerPPBplugin(source)` | global | `(source: PPBRequestSource) => PPBClient` |
| `app.plugins.plugins["paperbell"].api` | Obsidian plugin instance | `PPBHostApi` |
| `window.paperbell` | global | `PaperbellAPI` (legacy institution helpers) |
| `"paperbell:ready"` | `app.workspace` event | payload `PPBHostApi` |
| `"paperbell:config-changed"` | `app.workspace` event | payload `PaperBellPublicConfig` |

Load order between PaperBell and your plugin is not guaranteed, so always
handshake defensively — probe first, then fall back to the ready event:

```ts
import type { PPBHostApi, PPBClient } from "./paperbell-shared-config";

const ready = (host: PPBHostApi) => {
    const ppb: PPBClient = host.registerPPBplugin({
        id: "my-plugin",
        name: "My Plugin",
        description: "Shown on PaperBell's settings entry page",
        icon: "puzzle",                       // any lucide icon id
        onOpen: () => this.openMySettings(),  // card click handler
    });
    // ... use ppb
};

const host = (this.app as any).plugins.plugins["paperbell"]?.api;
if (host) ready(host);
else this.registerEvent(this.app.workspace.on("paperbell:ready" as any, ready));
```

`registerEvent` cleans the listener up when your plugin unloads. Call
`ppb.unregister()` in `onunload()` so PaperBell drops your settings card.

## 2. Two data layers

| Layer | Delivery | Handshake required | Contains |
|---|---|---|---|
| **Public** | `paperbell:config-changed` broadcast on `app.workspace` | no | what the user typed into their profile, plus UI language |
| **Restricted** | `PPBClient.requestSharedConfig()` / `PPBClient.onConfigChange()` | yes, plus per-scope consent | account, licence, LLM config, CIMPO folders |

The split exists because a bare `workspace.on(...)` listener is unauthenticated —
any plugin in the vault receives it. So only data the user actively entered and
expects to be reused travels that bus. PaperBell's own records (who they are to
us, what they paid for, which model they configured) are pushed point-to-point
to handshaked clients.

**API keys never appear in either layer.** They are returned only by
`requestLLMCredentials()`, behind its own scope and a licence check.

## 3. Scopes

Each scope is granted separately. The first call touching a scope opens a
consent dialog; approval is persisted, so later calls to the same scope pass
silently. Denial returns `null` — always handle it.

| Scope | Method | Returns | Extra conditions |
|---|---|---|---|
| `account` | `requestAccountInfo()` | `PaperBellAccountInfo` | — |
| `config` | `requestSharedConfig()` | `PaperBellRestrictedConfig` | — |
| `plugin-info` | `requestPluginInfo()` | `PaperBellPluginInfo` | — |
| `activation` | `requestActivationInfo()` | `PaperBellActivationInfo` | — |
| `llm-invoke` | `requestCompletion(params)` | `PPBCompletionResult` | free tier is metered, see §6 |
| `llm-credentials` | `requestLLMCredentials()` | `PaperBellLLMCredentials` | **licensed users only** — returns `null` before the consent dialog is even shown |
| `download-ticket` | `requestProtectedDownloadTicket(params?)` | `PPBProtectedDownloadTicket` | requires an activation code and an active licence |

Users review and revoke grants from PaperBell's settings; the host also exposes
`listGrants()` / `revokeGrant(sourceId)` on `PPBHostApi`.

## 4. Field reference

### `PPBRequestSource` — who you are

| Field | Type | Required | Meaning |
|---|---|---|---|
| `id` | `string` | ✔ | Stable id; use your manifest id |
| `name` | `string` | ✔ | Display name in the consent dialog and settings card |
| `description` | `string` | | Card subtitle |
| `icon` | `string` | | lucide icon id, default `"puzzle"` |
| `onOpen` | `() => void` | | Card click handler; omit and the card is inert |

### `PaperBellPublicConfig` — broadcast payload

| Field | Type | Meaning |
|---|---|---|
| `schemaVersion` | `number` | Contract version, currently `2` |
| `language` | `"en" \| "zh"` | UI language; explicit setting, else derived from the Obsidian locale (falls back to `zh`) |
| `profile` | `PaperBellUserProfile \| undefined` | Absent when the user filled nothing in |

### `PaperBellUserProfile`

| Field | Type | Meaning |
|---|---|---|
| `name` | `string?` | Display / byline name |
| `title` | `string?` | Job title or academic rank |
| `email` | `string?` | Contact email as entered by the user |
| `institution` | `string?` | Affiliation |
| `avatar` | `string?` | Avatar URL |

Empty strings are dropped, not sent as `""`. If every field is empty the whole
`profile` key is omitted.

### `PaperBellRestrictedConfig` — handshake payload

A structural superset of `PaperBellPublicConfig`.

| Field | Type | Meaning |
|---|---|---|
| `schemaVersion` | `number` | Contract version |
| `language` | `"en" \| "zh"` | Same as public layer |
| `profile` | `PaperBellUserProfile?` | Same as public layer |
| `llm` | `PaperBellLLMConfigPublic` | AI configuration **without** the key |
| `account` | `PaperBellAccountInfo?` | Licence-derived account record |
| `cimpoFolders` | `PaperBellCimpoFolders?` | Added after schema v1 — treat as possibly `undefined` and fall back to your own defaults |

### `PaperBellLLMConfigPublic`

| Field | Type | Meaning |
|---|---|---|
| `providerId` | `string?` | Active provider instance id, e.g. `openai`, `anthropic`, or a custom slug |
| `providerName` | `string?` | Display name of that provider |
| `api` | `"anthropic" \| "openai"` | Wire format — decides request and response shape |
| `baseUrl` | `string` | Gateway base URL (no trailing slash) |
| `model` | `string` | Default model id |
| `models` | `{ extract?: string; query?: string }?` | Optional per-task model routing |
| `hasApiKey` | `boolean` | Whether a key exists in Obsidian's secret storage. **The key itself is never in this object** |

### `PaperBellLLMCredentials` — only via `llm-credentials`

Same fields as above except `hasApiKey` is replaced by the real `apiKey: string`.
Throws if the host has not finished configuring provider, model, base URL or key.

### `PaperBellAccountInfo`

| Field | Type | Meaning |
|---|---|---|
| `userId` | `string?` | User id on the licence |
| `plan` | `string?` | `free`, `pro`, … |
| `displayName` | `string?` | Name on the licence — **not** the profile name |
| `email` | `string?` | Email on the licence — **not** the profile email |
| `isActive` | `boolean` | Whether the licence is currently active |

### `PaperBellActivationInfo`

| Field | Type | Meaning |
|---|---|---|
| `isActive` | `boolean` | Active licence |
| `expiresAt` | `number?` | Expiry, epoch milliseconds |
| `plan` | `string?` | Plan name |
| `userId` | `string?` | User id |
| `email` | `string?` | Email on the licence |

When PaperBell cannot reach the server it does **not** report `isActive: false` —
an offline paying user must not be demoted to the free tier. Treat
"could not check" as "still active".

### `PaperBellCimpoFolders`

The five top-level vault folders. Read them from the host instead of hardcoding
your own; the user may have renamed them.

| Field | Default path | Holds |
|---|---|---|
| `concepts` | `10 - Cards` | Self-growing wiki entries and the bounded core vocabulary |
| `inputs` | `20 - Inputs` | Papers, books, web clippings |
| `metadata` | `30 - Metadata` | People, places, dates |
| `projects` | `40 - Projects` | Long-running research projects |
| `outputs` | `50 - Outputs` | Drafts through to deliverables |

`00 - Obsidian` holds config and scripts only, which is why it is not one of the
five letters.

### `PaperBellPluginInfo`

| Field | Type | Meaning |
|---|---|---|
| `id` | `string` | `"paperbell"` |
| `name` | `string` | Manifest name |
| `version` | `string` | Host plugin version |
| `schemaVersion` | `number` | Contract version |
| `isActivated` | `boolean` | Licence active |
| `capabilities` | `PPBScope[]` | Scopes this host build can serve |

Check `capabilities` before calling a method if you want to stay compatible with
older hosts.

### `PPBProtectedDownloadParams` — argument of `requestProtectedDownloadTicket`

| Field | Type | Meaning |
|---|---|---|
| `product` | `string?` | Product slug, e.g. `paperbell-core`. Defaults to `paperbell-core` |
| `baseUrl` | `string?` | Defaults to `https://paperbell.cn`; override for test or staging |

### `PPBProtectedDownloadTicket` — its return value

| Field | Type | Meaning |
|---|---|---|
| `url` | `string` | The signed download URL. Always present — the host throws if the server returns a ticket without it |
| `filename` | `string?` | Suggested file name |
| `expires_in` | `number?` | Ticket lifetime in seconds |
| `version` | `string?` | Version of the artifact behind the ticket |
| `sha256` | `string?` | Checksum for verifying the download |
| *(any other key)* | `unknown` | The type carries an index signature, so server-added fields pass through untouched |

Unlike the other methods, this one **throws** rather than returning `null` when
the host has no activation code, the licence is inactive, or the server rejects
the request. Denial of the `download-ticket` scope still returns `null`. Wrap it:

```ts
let ticket;
try {
    ticket = await ppb.requestProtectedDownloadTicket({ product: "paperbell-core" });
} catch (e) {
    return showError(e instanceof Error ? e.message : String(e));
}
if (!ticket) return;   // consent denied
download(ticket.url);
```

`requestLLMCredentials()` behaves the same way — it throws when the host's AI
configuration is incomplete, and returns `null` when consent is denied or the
user is unlicensed.

### `PPBHostApi` — the object on `app.plugins.plugins["paperbell"].api`

| Member | Type | Meaning |
|---|---|---|
| `registerPPBplugin` | `(source: PPBRequestSource) => PPBClient` | The handshake. Same function as the `window.registerPPBplugin` global |
| `getPluginInfo` | `() => PaperBellPluginInfo` | Synchronous, **no scope required** — use it to probe the host version and capabilities before handshaking |
| `listGrants` | `() => PPBGrant[]` | The current consent list |
| `revokeGrant` | `(sourceId: string) => void` | Revoke every scope granted to one source |

### `PPBGrant` — one consent record

| Field | Type | Meaning |
|---|---|---|
| `sourceId` | `string` | The `id` from `PPBRequestSource` |
| `sourceName` | `string` | The `name` from `PPBRequestSource` |
| `scopes` | `PPBScope[]` | Scopes granted to this source |
| `grantedAt` | `number` | Epoch milliseconds |

### `PPBClient` — what the handshake returns

| Member | Signature |
|---|---|
| `requestAccountInfo` | `() => Promise<PaperBellAccountInfo \| null>` |
| `requestSharedConfig` | `() => Promise<PaperBellRestrictedConfig \| null>` |
| `requestPluginInfo` | `() => Promise<PaperBellPluginInfo \| null>` |
| `requestLLMCredentials` | `() => Promise<PaperBellLLMCredentials \| null>` |
| `requestActivationInfo` | `() => Promise<PaperBellActivationInfo \| null>` |
| `requestProtectedDownloadTicket` | `(params?: PPBProtectedDownloadParams) => Promise<PPBProtectedDownloadTicket \| null>` |
| `requestCompletion` | `(params: PPBCompletionParams) => Promise<PPBCompletionResult \| null>` |
| `onConfigChange` | `(cb: (config: PaperBellRestrictedConfig) => void) => () => void` — returns an unsubscribe function |
| `unregister` | `() => void` — drops the client and its subscriptions; does **not** revoke grants |

> **Naming trap.** `requestSharedConfig()` is declared as returning
> `PaperBellSharedConfigPublic`, which is a **deprecated alias** of
> `PaperBellRestrictedConfig` — the two are the same type. The `Public` in the
> old name meant "key-stripped", not "safe to broadcast", which is exactly the
> confusion the rename fixed. New code should write `PaperBellRestrictedConfig`.
>
> There is also an exported `PaperBellSharedConfig` — the host's **internal**
> full shape, the only one whose `llm` still carries `apiKey`. You never receive
> it over IPC; `PaperBellLLMCredentials` is defined as its `llm` member and is
> the only way that key reaches you.

## 5. Host-brokered completions

`requestCompletion()` sends one **non-streaming** completion through the host,
using the host's provider and key. The key never leaves PaperBell.

### `PPBCompletionParams`

| Field | Type | Meaning |
|---|---|---|
| `messages` | `Array<{ role: "user" \| "assistant"; content: string }>` | Required, non-empty |
| `system` | `string?` | System prompt |
| `model` | `string?` | Defaults to the host's configured model |
| `maxTokens` | `number?` | Anthropic requires it upstream; host defaults to `1024` |
| `temperature` | `number?` | Passed through when set |

### `PPBCompletionResult`

| Field | Type | Meaning |
|---|---|---|
| `ok` | `boolean` | Success |
| `text` | `string` | Model output when `ok` |
| `model` | `string` | Model actually used |
| `error` | `string?` | Failure description, never contains secrets |
| `errorCode` | `"quota-exhausted"?` | Machine-readable failure class — currently only the free-tier case |
| `quota` | `{ limit: number; remaining: number; resetsAt: number }?` | Present with `errorCode === "quota-exhausted"` |

Branch on `errorCode`, not on the text of `error` — the message is localised
Chinese prose and will change.

```ts
const res = await ppb.requestCompletion({
    messages: [{ role: "user", content: "Summarise this abstract: ..." }],
    system: "You are a terse academic assistant.",
    maxTokens: 512,
});
if (!res) return;                                  // consent denied
if (!res.ok && res.errorCode === "quota-exhausted") return promptUpgrade(res.quota);
if (!res.ok) return retryLater(res.error);
use(res.text);
```

## 6. Free tier metering

Unlicensed users get **5 host-brokered AI calls per local day**, resetting at
local midnight. Licensed users are unmetered and the counter is never touched.

The quota is charged after parameter validation but *before* the upstream
request. Two consequences you should code against: a malformed call costs
nothing, and an upstream 5xx **does** consume one call — do not assume failed
calls are free to retry.

`requestLLMCredentials()` is licence-gated, so unlicensed users cannot reach the
raw key path at all.

## 7. Reacting to configuration changes

```ts
// Public layer — no handshake, no scope, any plugin may listen.
this.registerEvent(
    this.app.workspace.on("paperbell:config-changed" as any, (cfg) => {
        applyLanguage(cfg.language);
        showByline(cfg.profile?.name);
    }),
);

// Restricted layer — handshaked clients only.
const stop = ppb.onConfigChange((cfg) => {
    setModel(cfg.llm.model);
    setFolders(cfg.cimpoFolders);
});
```

Both are dirty-checked: PaperBell only emits when the payload actually changed,
so typing into a settings field does not broadcast every keystroke.

`onConfigChange` is a direct host push, **not** the workspace bus — it does not
survive a PaperBell reload. Re-handshake on `paperbell:ready` and resubscribe.

## 8. Versioning rules

- `PPB_SCHEMA_VERSION` bumps only on **breaking** changes — a payload narrowing,
  a removed field, a changed meaning. v1 → v2 narrowed the broadcast payload
  from the full config down to the public layer.
- Appending an **optional** field does not bump it (`cimpoFolders` was added this
  way), so never assert `schemaVersion === 2` for equality. Compare with `>=`,
  and treat unknown optional fields as absent.

## 9. Legacy institution API

`window.paperbell` predates the IPC contract and remains for QuickAdd users.

```ts
interface PaperbellAPI {
    searchInstitution(name: string): Promise<InstitutionNote | null>;
    createInstitutionNote(institution: InstitutionNote): Promise<void>;
}

interface InstitutionNote {
    abbr: string;
    aliases: string[];
    website: string;
    location: [number, number];   // [lat, lon]
    logo: string;
    name: string;
    tags: string[];
}
```

Institution note templates support these placeholders:

| Variable | Meaning | Example |
|---|---|---|
| `{{ppb.institute.name}}` | Full name | Harvard University |
| `{{ppb.institute.abbr}}` | Abbreviation | HU |
| `{{ppb.institute.aliases}}` | Alternative names, rendered as a YAML list | `- Harvard` |
| `{{ppb.institute.website}}` | Website URL | https://harvard.edu |
| `{{ppb.institute.lat}}` / `{{ppb.institute.lon}}` | Coordinates | 42.3744, -71.1169 |
| `{{ppb.institute.logo}}` | Logo URL | https://example.com/logo.png |
| `{{ppb.institute.tags}}` | Tags, rendered as a YAML list | `- university` |

Array values expand to one `- item` per line, so put them where a YAML list is
expected.

> ⚠️ The institution settings tab is **commented out** in current builds, so the
> template path cannot be set from the UI right now. The commands
> (`搜索机构` / `创建机构笔记`), the ribbon icon and `window.paperbell` all still
> work, and an existing `institutionNoteTemplate` value in `data.json` is still
> honoured — but a fresh install has no way to point at a template.

## Appendix A — the contract in full

Paste this into your plugin (e.g. `src/paperbell-shared-config.ts`) and compile
against it. It has no imports and no runtime dependencies. Keep it in sync with
the `PPB_SCHEMA_VERSION` you see at the top of this file.

```ts
export const PPB_SCHEMA_VERSION = 2;
export const PPB_READY_EVENT = "paperbell:ready";
export const PPB_CONFIG_CHANGED_EVENT = "paperbell:config-changed";

export type PPBScope =
    | "account" | "config" | "plugin-info"
    | "llm-invoke" | "llm-credentials" | "activation" | "download-ticket";

export interface PaperBellUserProfile {
    name?: string; title?: string; email?: string;
    institution?: string; avatar?: string;
}

export interface PaperBellPublicConfig {
    schemaVersion: number;
    language: "en" | "zh";
    profile?: PaperBellUserProfile;
}

export interface PaperBellCimpoFolders {
    concepts: string; inputs: string; metadata: string;
    projects: string; outputs: string;
}

export interface PaperBellSharedConfig {
    schemaVersion: number;
    language: "en" | "zh";
    llm: {
        providerId?: string;
        providerName?: string;
        api: "anthropic" | "openai";
        baseUrl: string;
        apiKey: string;
        model: string;
        models?: { extract?: string; query?: string };
    };
    account?: { userId?: string; plan?: string; displayName?: string };
    cimpoFolders?: PaperBellCimpoFolders;
}

export type PaperBellLLMConfigPublic =
    Omit<PaperBellSharedConfig["llm"], "apiKey"> & { hasApiKey: boolean };

export type PaperBellLLMCredentials = PaperBellSharedConfig["llm"];

export interface PaperBellAccountInfo {
    userId?: string; plan?: string; displayName?: string;
    email?: string; isActive: boolean;
}

export interface PaperBellRestrictedConfig {
    schemaVersion: number;
    language: "en" | "zh";
    llm: PaperBellLLMConfigPublic;
    account?: PaperBellAccountInfo;
    profile?: PaperBellUserProfile;
    cimpoFolders?: PaperBellCimpoFolders;
}

/** @deprecated renamed to PaperBellRestrictedConfig */
export type PaperBellSharedConfigPublic = PaperBellRestrictedConfig;

export interface PaperBellPluginInfo {
    id: string; name: string; version: string;
    schemaVersion: number; isActivated: boolean;
    capabilities: PPBScope[];
}

export interface PaperBellActivationInfo {
    isActive: boolean; expiresAt?: number;
    plan?: string; userId?: string; email?: string;
}

export interface PPBProtectedDownloadParams {
    product?: string;
    baseUrl?: string;
}

export interface PPBProtectedDownloadTicket {
    url: string;
    filename?: string;
    expires_in?: number;
    version?: string;
    sha256?: string;
    [key: string]: unknown;
}

export interface PPBCompletionParams {
    messages: Array<{ role: "user" | "assistant"; content: string }>;
    system?: string;
    model?: string;
    maxTokens?: number;
    temperature?: number;
}

export interface PPBCompletionResult {
    ok: boolean;
    text: string;
    model: string;
    error?: string;
    errorCode?: "quota-exhausted";
    quota?: { limit: number; remaining: number; resetsAt: number };
}

export interface PPBRequestSource {
    id: string;
    name: string;
    description?: string;
    icon?: string;
    onOpen?: () => void;
}

export interface PPBGrant {
    sourceId: string;
    sourceName: string;
    scopes: PPBScope[];
    grantedAt: number;
}

export interface PPBClient {
    requestAccountInfo(): Promise<PaperBellAccountInfo | null>;
    requestSharedConfig(): Promise<PaperBellSharedConfigPublic | null>;
    requestPluginInfo(): Promise<PaperBellPluginInfo | null>;
    requestLLMCredentials(): Promise<PaperBellLLMCredentials | null>;
    requestActivationInfo(): Promise<PaperBellActivationInfo | null>;
    requestProtectedDownloadTicket(
        params?: PPBProtectedDownloadParams,
    ): Promise<PPBProtectedDownloadTicket | null>;
    requestCompletion(
        params: PPBCompletionParams,
    ): Promise<PPBCompletionResult | null>;
    onConfigChange(cb: (config: PaperBellRestrictedConfig) => void): () => void;
    unregister(): void;
}

export type RegisterPPBPlugin = (source: PPBRequestSource) => PPBClient;

export interface PPBHostApi {
    registerPPBplugin: RegisterPPBPlugin;
    getPluginInfo(): PaperBellPluginInfo;
    listGrants(): PPBGrant[];
    revokeGrant(sourceId: string): void;
}

declare global {
    interface Window {
        registerPPBplugin?: RegisterPPBPlugin;
    }
}
```

This is the contract in full, with one deliberate omission: the host also
exports `PPB_PLUGINS_CHANGED_EVENT` (`"paperbell:plugins-changed"`), fired when
the set of registered companion plugins changes. It exists so PaperBell's own
settings page can refresh its plugin cards — it is not part of the companion
contract and carries no payload you can use.

## Support

- Website: <https://paperbell.cn>
- Email: <support@paperbell.cn>
