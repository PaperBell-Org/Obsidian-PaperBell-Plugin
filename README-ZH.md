# PaperBell

> [English](./README.md)

PaperBell 是 Obsidian 上 PaperBell 系列插件的**主插件（宿主）**。

它只负责两件事，并且刻意不做更多：

1. **与 PaperBell 服务器交互** —— 许可证激活、OAuth 登录、带鉴权的请求代理、
   受保护资源的下载票据，以及由宿主代发的 LLM 调用。
2. **集中管理配置** —— 全库只有一个账户、一份许可证、一套 AI 提供方配置、
   一份 CIMPO 文件夹布局。系列子插件从主插件读取这些配置，而不是各问用户一遍。

具体的功能界面（卡片管理、文献检索、写作工具……）都在各自的子插件里。
子插件通过下面这套 IPC 契约向主插件握手注册，从而继承它的账户、许可证与 AI 配置。

---

## 安装

用 [BRAT](https://github.com/TfTHacker/obsidian42-brat) 添加本仓库，
或从 [Release](../../releases) 下载 `main.js`、`styles.css`、`manifest.json`
放进 `<vault>/.obsidian/plugins/paperbell/`。

## 用户能得到什么

| 方面 | 主插件提供 |
|---|---|
| 账户 | 通过 `paperbell.cn` OAuth 登录，或粘贴激活码 |
| 许可证 | 激活状态、有效期、套餐；离线宽限 |
| AI | 一处配置提供方、Base URL、模型与 API 密钥 |
| CIMPO 文件夹 | 所有子插件共同认可的五个顶层目录 |
| 个人信息 | 姓名、头衔、邮箱、单位、头像 —— 填一次，全家共享 |
| 权限 | 按插件、按 scope 的授权名单，随时可撤销 |

统一在 Obsidian **设置 → PaperBell → 打开设置** 里配置。

---

# 开发者参考

以下是子插件可以依赖的稳定契约。完整的 TypeScript 声明见文末
[附录 A](#附录-a--完整契约声明) —— 零依赖零运行时，可以整段贴进你自己的插件直接编译。
（契约的规范定义在主插件的私有源码仓库里，本仓库只发布产物，所以要 vendor 的是附录那份。）

**当前契约版本：`PPB_SCHEMA_VERSION = 2`。**

## 1. 入口

| 符号 | 位置 | 类型 |
|---|---|---|
| `window.registerPPBplugin(source)` | 全局 | `(source: PPBRequestSource) => PPBClient` |
| `app.plugins.plugins["paperbell"].api` | Obsidian 插件实例 | `PPBHostApi` |
| `window.paperbell` | 全局 | `PaperbellAPI`（早期的机构笔记辅助 API） |
| `"paperbell:ready"` | `app.workspace` 事件 | 载荷 `PPBHostApi` |
| `"paperbell:config-changed"` | `app.workspace` 事件 | 载荷 `PaperBellPublicConfig` |

主插件与你的插件谁先加载是不确定的，所以务必用握手模式 —— 先主动探测，
探不到再等 ready 事件：

```ts
import type { PPBHostApi, PPBClient } from "./paperbell-shared-config";

const ready = (host: PPBHostApi) => {
    const ppb: PPBClient = host.registerPPBplugin({
        id: "my-plugin",
        name: "My Plugin",
        description: "显示在 PaperBell 设置入口页的卡片描述",
        icon: "puzzle",                       // 任意 lucide 图标 id
        onOpen: () => this.openMySettings(),  // 点击卡片的回调
    });
    // ... 使用 ppb
};

const host = (this.app as any).plugins.plugins["paperbell"]?.api;
if (host) ready(host);
else this.registerEvent(this.app.workspace.on("paperbell:ready" as any, ready));
```

`registerEvent` 会在你的插件卸载时自动摘除监听。请在 `onunload()` 里调用
`ppb.unregister()`，好让 PaperBell 及时摘掉你的入口卡片。

## 2. 两层数据

| 层 | 下发方式 | 是否需要握手 | 内容 |
|---|---|---|---|
| **公开层** | `app.workspace` 上广播 `paperbell:config-changed` | 否 | 用户自己填的 profile，以及 UI 语言 |
| **受限层** | `PPBClient.requestSharedConfig()` / `PPBClient.onConfigChange()` | 是，且需按 scope 授权 | 账户、许可证、LLM 配置、CIMPO 文件夹 |

分层的理由：裸 `workspace.on(...)` 谁都能听，库里任何插件都会收到。所以只有
「用户主动录入、也预期会被复用」的内容走那条总线；PaperBell 自己的记录
（用户在我们这儿是谁、买了什么、配了哪个模型）只点对点推给握手过的 client。

**API 密钥不出现在任何一层。** 它只由 `requestLLMCredentials()` 返回，
且该方法有独立 scope 与许可证校验。

## 3. Scope 列表

每个 scope 独立授权。首次触达某 scope 会弹同意框；用户批准后写入授权名单，
之后同 scope 免弹。拒绝一律返回 `null`，调用方必须处理。

| Scope | 方法 | 返回 | 附加条件 |
|---|---|---|---|
| `account` | `requestAccountInfo()` | `PaperBellAccountInfo` | — |
| `config` | `requestSharedConfig()` | `PaperBellRestrictedConfig` | — |
| `plugin-info` | `requestPluginInfo()` | `PaperBellPluginInfo` | — |
| `activation` | `requestActivationInfo()` | `PaperBellActivationInfo` | — |
| `llm-invoke` | `requestCompletion(params)` | `PPBCompletionResult` | 免费档有额度，见 §6 |
| `llm-credentials` | `requestLLMCredentials()` | `PaperBellLLMCredentials` | **仅已激活用户** —— 未激活时连同意框都不弹，直接返回 `null` |
| `download-ticket` | `requestProtectedDownloadTicket(params?)` | `PPBProtectedDownloadTicket` | 需要激活码且许可证处于激活态 |

用户可在 PaperBell 设置里查看与撤销授权；宿主也在 `PPBHostApi` 上暴露了
`listGrants()` / `revokeGrant(sourceId)`。

## 4. 字段清单

### `PPBRequestSource` —— 你是谁

| 字段 | 类型 | 必填 | 含义 |
|---|---|---|---|
| `id` | `string` | ✔ | 稳定 id，建议与你的 manifest id 一致 |
| `name` | `string` | ✔ | 同意框与设置卡片上的展示名 |
| `description` | `string` | | 卡片副标题 |
| `icon` | `string` | | lucide 图标 id，缺省 `"puzzle"` |
| `onOpen` | `() => void` | | 卡片点击回调；不提供则卡片不可点击 |

### `PaperBellPublicConfig` —— 广播载荷

| 字段 | 类型 | 含义 |
|---|---|---|
| `schemaVersion` | `number` | 契约版本，当前为 `2` |
| `language` | `"en" \| "zh"` | UI 语言；显式设置优先，否则按 Obsidian locale 推导（回退 `zh`） |
| `profile` | `PaperBellUserProfile \| undefined` | 用户什么都没填时整个字段缺席 |

### `PaperBellUserProfile`

| 字段 | 类型 | 含义 |
|---|---|---|
| `name` | `string?` | 展示名 / 署名 |
| `title` | `string?` | 职位或学术头衔 |
| `email` | `string?` | 用户自填的联系邮箱 |
| `institution` | `string?` | 所属机构 |
| `avatar` | `string?` | 头像链接 |

空字符串会被剔除，不会以 `""` 下发；五个字段全空时整个 `profile` 键不出现。

### `PaperBellRestrictedConfig` —— 握手后的载荷

结构上是 `PaperBellPublicConfig` 的超集。

| 字段 | 类型 | 含义 |
|---|---|---|
| `schemaVersion` | `number` | 契约版本 |
| `language` | `"en" \| "zh"` | 同公开层 |
| `profile` | `PaperBellUserProfile?` | 同公开层 |
| `llm` | `PaperBellLLMConfigPublic` | **去密钥**的 AI 配置 |
| `account` | `PaperBellAccountInfo?` | 从许可证派生的账户记录 |
| `cimpoFolders` | `PaperBellCimpoFolders?` | schema v1 之后追加的可选字段 —— 按「可能为 undefined」处理，自行回落到默认布局 |

### `PaperBellLLMConfigPublic`

| 字段 | 类型 | 含义 |
|---|---|---|
| `providerId` | `string?` | 当前激活的提供方实例 id，如 `openai`、`anthropic` 或自定义 slug |
| `providerName` | `string?` | 该提供方的展示名 |
| `api` | `"anthropic" \| "openai"` | 请求协议，决定请求与响应形态 |
| `baseUrl` | `string` | 网关基址，已规范化（无尾部斜杠）。这是**生效值** —— 用户留空时回落到内置提供方的默认地址 |
| `model` | `string` | 默认模型 id，同样是**生效值** —— 用户没选过时回落到内置提供方的默认模型 |
| `models` | `{ extract?: string; query?: string }?` | 可选：按任务路由到不同模型 |
| `hasApiKey` | `boolean` | 宿主是否已在 Obsidian secretStorage 中配好密钥。**密钥本身绝不在这个对象里** |

### `PaperBellLLMCredentials` —— 仅经 `llm-credentials` 返回

字段同上，区别是 `hasApiKey` 换成真正的 `apiKey: string`。
若宿主尚未配全提供方、模型、Base URL 或密钥，该方法会抛错。

### `PaperBellAccountInfo`

| 字段 | 类型 | 含义 |
|---|---|---|
| `userId` | `string?` | 许可证上的用户 id |
| `plan` | `string?` | `free`、`pro`…… |
| `displayName` | `string?` | 许可证上的姓名 —— **不是**用户自填的 profile 姓名 |
| `email` | `string?` | 许可证上的邮箱 —— **不是**用户自填的 profile 邮箱 |
| `isActive` | `boolean` | 许可证当前是否激活 |

### `PaperBellActivationInfo`

| 字段 | 类型 | 含义 |
|---|---|---|
| `isActive` | `boolean` | 是否激活 |
| `expiresAt` | `number?` | 过期时间戳（毫秒） |
| `plan` | `string?` | 套餐名 |
| `userId` | `string?` | 用户 id |
| `email` | `string?` | 许可证上的邮箱 |

宿主查不到服务器时**不会**报 `isActive: false` —— 离线的付费用户绝不能被推到
免费档。请把「没查成」当作「仍然有效」。

### `PaperBellCimpoFolders`

库里五个顶层文件夹。请从宿主读，不要各自硬编码 —— 用户可能改过名。

| 字段 | 默认路径 | 存放 |
|---|---|---|
| `concepts` | `10 - Cards` | 自我生长的 wiki 词条与有界核心词表 |
| `inputs` | `20 - Inputs` | 论文、图书与网页剪藏等外部输入 |
| `metadata` | `30 - Metadata` | 人、地、时的流水账 |
| `projects` | `40 - Projects` | 长周期研究项目 |
| `outputs` | `50 - Outputs` | 亲笔草稿与长文，直至交付物 |

`00 - Obsidian` 只放配置与脚本，不计入五个字母，因此不在此列。

### `PaperBellPluginInfo`

| 字段 | 类型 | 含义 |
|---|---|---|
| `id` | `string` | `"paperbell"` |
| `name` | `string` | manifest 名称 |
| `version` | `string` | 主插件版本 |
| `schemaVersion` | `number` | 契约版本 |
| `isActivated` | `boolean` | 许可证是否激活 |
| `capabilities` | `PPBScope[]` | 这个宿主版本能提供的 scope 列表 |

想兼容老版本宿主，就在调用前先查 `capabilities`。

### `PPBProtectedDownloadParams` —— `requestProtectedDownloadTicket` 的参数

| 字段 | 类型 | 含义 |
|---|---|---|
| `product` | `string?` | 产品 slug，如 `paperbell-core`。缺省 `paperbell-core` |
| `baseUrl` | `string?` | 缺省 `https://paperbell.cn`；测试 / 预发可覆盖 |

### `PPBProtectedDownloadTicket` —— 它的返回值

| 字段 | 类型 | 含义 |
|---|---|---|
| `url` | `string` | 签名后的下载链接。一定存在 —— 服务端没返回它时宿主会抛错 |
| `filename` | `string?` | 建议的文件名 |
| `expires_in` | `number?` | 票据有效期（秒） |
| `version` | `string?` | 票据背后产物的版本 |
| `sha256` | `string?` | 校验和 |
| *(其它任意键)* | `unknown` | 类型带索引签名，服务端新增的字段会原样透传 |

和其它方法不同，这个方法在「宿主没有激活码」「许可证未激活」「服务端拒绝」时
是 **throw** 而不是返回 `null`。`download-ticket` 授权被拒仍然返回 `null`。
所以要包一层：

```ts
let ticket;
try {
    ticket = await ppb.requestProtectedDownloadTicket({ product: "paperbell-core" });
} catch (e) {
    return showError(e instanceof Error ? e.message : String(e));
}
if (!ticket) return;   // 用户拒绝授权
download(ticket.url);
```

`requestLLMCredentials()` 也是同样的形态 —— 宿主 AI 配置不全时抛错，
授权被拒或用户未激活时返回 `null`。

### `PPBHostApi` —— 挂在 `app.plugins.plugins["paperbell"].api` 上的对象

| 成员 | 类型 | 含义 |
|---|---|---|
| `registerPPBplugin` | `(source: PPBRequestSource) => PPBClient` | 握手入口。与全局的 `window.registerPPBplugin` 是同一个函数 |
| `getPluginInfo` | `() => PaperBellPluginInfo` | 同步、**不需要任何 scope** —— 握手前用它探测宿主版本与能力 |
| `listGrants` | `() => PPBGrant[]` | 当前授权名单 |
| `revokeGrant` | `(sourceId: string) => void` | 撤销某来源的全部授权 |

### `PPBGrant` —— 一条授权记录

| 字段 | 类型 | 含义 |
|---|---|---|
| `sourceId` | `string` | 来自 `PPBRequestSource` 的 `id` |
| `sourceName` | `string` | 来自 `PPBRequestSource` 的 `name` |
| `scopes` | `PPBScope[]` | 该来源已获得的 scope |
| `grantedAt` | `number` | 授权时间戳（毫秒） |

### `PPBClient` —— 握手返回的客户端

| 成员 | 签名 |
|---|---|
| `requestAccountInfo` | `() => Promise<PaperBellAccountInfo \| null>` |
| `requestSharedConfig` | `() => Promise<PaperBellRestrictedConfig \| null>` |
| `requestPluginInfo` | `() => Promise<PaperBellPluginInfo \| null>` |
| `requestLLMCredentials` | `() => Promise<PaperBellLLMCredentials \| null>` |
| `requestActivationInfo` | `() => Promise<PaperBellActivationInfo \| null>` |
| `requestProtectedDownloadTicket` | `(params?: PPBProtectedDownloadParams) => Promise<PPBProtectedDownloadTicket \| null>` |
| `requestCompletion` | `(params: PPBCompletionParams) => Promise<PPBCompletionResult \| null>` |
| `onConfigChange` | `(cb: (config: PaperBellRestrictedConfig) => void) => () => void` —— 返回取消订阅函数 |
| `unregister` | `() => void` —— 注销客户端并清理订阅，**不**撤销授权 |

> **命名陷阱。** `requestSharedConfig()` 的声明返回类型是
> `PaperBellSharedConfigPublic`，它是 `PaperBellRestrictedConfig` 的
> **废弃别名**，两者是同一个类型。旧名里的 `Public` 指的是「去密钥」，
> 而不是「可公开广播」—— 这正是那次改名要消除的歧义。新代码请写
> `PaperBellRestrictedConfig`。
>
> 另外还导出了一个 `PaperBellSharedConfig` —— 那是宿主**内部**持有的完整形态，
> 也是唯一一个 `llm` 里还带 `apiKey` 的类型。你不会通过 IPC 拿到它；
> `PaperBellLLMCredentials` 就定义为它的 `llm` 成员，是那把密钥到达你手上的唯一途径。

## 5. 宿主代发补全

`requestCompletion()` 用宿主自己的提供方与密钥代发一次**非流式**补全，
密钥全程不出宿主。

### `PPBCompletionParams`

| 字段 | 类型 | 含义 |
|---|---|---|
| `messages` | `Array<{ role: "user" \| "assistant"; content: string }>` | 必填且非空 |
| `system` | `string?` | 系统提示 |
| `model` | `string?` | 缺省用宿主配置的默认模型 |
| `maxTokens` | `number?` | anthropic 形态为上游必填，宿主缺省 `1024` |
| `temperature` | `number?` | 设置了才透传 |

### `PPBCompletionResult`

| 字段 | 类型 | 含义 |
|---|---|---|
| `ok` | `boolean` | 是否成功 |
| `text` | `string` | `ok` 时的模型输出 |
| `model` | `string` | 实际使用的模型 id |
| `error` | `string?` | 失败描述，不含密钥等敏感信息 |
| `errorCode` | `"quota-exhausted"?` | 机器可判别的失败类别，目前只有「免费额度耗尽」一种 |
| `quota` | `{ limit: number; remaining: number; resetsAt: number }?` | 伴随 `errorCode === "quota-exhausted"` 出现 |

请按 `errorCode` 分支，不要去匹配 `error` 里的文案 —— 那是会变的中文提示。

```ts
const res = await ppb.requestCompletion({
    messages: [{ role: "user", content: "帮我总结这段摘要：..." }],
    system: "你是一个惜字如金的学术助手。",
    maxTokens: 512,
});
if (!res) return;                                  // 用户拒绝授权
if (!res.ok && res.errorCode === "quota-exhausted") return promptUpgrade(res.quota);
if (!res.ok) return retryLater(res.error);
use(res.text);
```

## 6. 免费额度

未激活用户每个自然日可用 **5 次**宿主代发的 AI 调用，本地零点重置；
已激活用户不限次，连计数器都不会碰。

额度在参数校验**之后**、请求上游**之前**结算。有两条后果需要你在代码里考虑：
参数写错不扣次数；而上游 5xx **会**扣一次 —— 不要假设失败的调用可以免费重试。

`requestLLMCredentials()` 卡了许可证，未激活用户根本走不到裸密钥这条路径。

## 7. 订阅配置变更

```ts
// 公开层 —— 无需握手、无需 scope，任何插件都能听。
this.registerEvent(
    this.app.workspace.on("paperbell:config-changed" as any, (cfg) => {
        applyLanguage(cfg.language);
        showByline(cfg.profile?.name);
    }),
);

// 受限层 —— 只有握手过的 client 收得到。
const stop = ppb.onConfigChange((cfg) => {
    setModel(cfg.llm.model);
    setFolders(cfg.cimpoFolders);
});
```

两条都做了脏检查：载荷没变就不下发，所以用户在设置框里逐字输入不会被逐前缀
广播出去。

`onConfigChange` 是宿主的定向推送，**不是** workspace 总线，因此 PaperBell
重载后订阅会失效 —— 请在 `paperbell:ready` 上重新握手并重新订阅。

## 8. 版本约定

- `PPB_SCHEMA_VERSION` 只在**破坏性变更**时 bump：载荷收窄、字段删除、语义改变。
  v1 → v2 就是把广播载荷从完整配置收窄到公开层。
- 追加**可选**字段不 bump（`cimpoFolders` 就是这么加的）。所以永远不要写
  `schemaVersion === 2` 这样的等值校验，请用 `>=`，并把不认识的可选字段当作缺席。

## 9. 早期的机构笔记 API

`window.paperbell` 早于 IPC 契约存在，为 QuickAdd 用户保留。

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

机构笔记模板支持以下占位符：

| 变量 | 含义 | 示例 |
|---|---|---|
| `{{ppb.institute.name}}` | 机构全称 | 哈佛大学 |
| `{{ppb.institute.abbr}}` | 机构缩写 | HU |
| `{{ppb.institute.aliases}}` | 别名，渲染成 YAML 列表 | `- Harvard` |
| `{{ppb.institute.website}}` | 机构网站 | https://harvard.edu |
| `{{ppb.institute.lat}}` / `{{ppb.institute.lon}}` | 地理坐标 | 42.3744, -71.1169 |
| `{{ppb.institute.logo}}` | Logo 链接 | https://example.com/logo.png |
| `{{ppb.institute.tags}}` | 标签，渲染成 YAML 列表 | `- university` |

数组类型会展开成每行一个 `- item`，所以要放在 YAML 列表的位置上。

> ⚠️ 机构笔记设置页在当前版本里是**注释掉的**，所以模板路径暂时无法从界面设置。
> 命令（`搜索机构` / `创建机构笔记`）、ribbon 图标和 `window.paperbell` 都仍然可用，
> `data.json` 里已有的 `institutionNoteTemplate` 也仍然生效 ——
> 但全新安装的用户没有途径指定模板。

## 附录 A —— 完整契约声明

把下面这段贴进你自己的插件（例如 `src/paperbell-shared-config.ts`）直接编译。
它没有任何 import，也没有运行时依赖。请与本文开头标注的 `PPB_SCHEMA_VERSION`
保持同步。

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

以上就是契约的全部，只有一处是刻意省略的：宿主还导出了
`PPB_PLUGINS_CHANGED_EVENT`（`"paperbell:plugins-changed"`），在已注册子插件
集合变化时触发。它的存在只是为了让 PaperBell 自己的设置入口页刷新子插件卡片，
不属于子插件契约，也不带任何你用得上的载荷。

## 支持

- 官网：<https://paperbell.cn>
- 邮箱：<support@paperbell.cn>
