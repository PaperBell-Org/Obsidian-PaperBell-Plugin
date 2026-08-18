# 子插件适配指南：0.4.5 → 0.4.7

> 面向 PaperBell 系列子插件的开发者。
> 对应主插件 [0.4.7](https://github.com/PaperBell-Org/Obsidian-PaperBell-Plugin/releases/tag/0.4.7)。
> 契约字段的完整清单见 [README-ZH](./README-ZH.md)。

## 先说结论

**`PPB_SCHEMA_VERSION` 仍然是 `2`，契约的字段形状一个都没改。**

只用 `registerPPBplugin()` / `app.plugins.plugins["paperbell"].api` 的子插件，
**大概率不需要改任何代码**。这份文档要处理的是：从前能绕开契约、直接伸手去拿
主插件内部对象的写法 —— 那些路径在 0.4.7 里全部关掉了。

判断自己受不受影响，先跑这一条：

```bash
# 在你的子插件仓库里
grep -rnE 'plugins\[.paperbell.\]\.(?!api)|plugins\.paperbell\.(?!api)' src/
```

有命中就往下看；一条都没有，只看第 4 节和第 5 节即可。

---

## 1. 主插件实例上只剩 `api` 和 `settings`

### 变了什么

`app.plugins.plugins["paperbell"]` 对同一 vault 里任何插件都可达，所以它上面
每一个可读的东西都是事实上的对外 API —— 而我们的 scope / 同意流是**摆在**它旁边，
不是挡在它前面。0.4.7 把实现对象全部移进了模块私有的登记处。

### 失效的写法

```js
const p = app.plugins.plugins.paperbell;

p.verificationWorker.getLicenseData()      // → undefined
p.proxyService.proxyRequest({ url })       // → undefined
p.agentExecutor.executeAdHoc(type, prompt) // → undefined
p.updater / p.fileWatcherService / p.ppbHost / p.templateProcessor
p.setLlmApiKey(v) / p.hasLlmApiKey() / p.clearLlmApiKey()
p.handleActivationURI(params) / p.startAccountLogin()

// 这条也堵了 —— Obsidian 的 addChild 会把子组件塞进 plugin._children
p._children.find(c => c.executeAdHoc)
```

### 换成契约

| 你原来想拿的 | 现在走 |
|---|---|
| 许可证 / 账户信息 | `requestAccountInfo()`（scope `account`） |
| 激活状态、有效期 | `requestActivationInfo()`（scope `activation`） |
| AI 配置（不含密钥） | `requestSharedConfig()`（scope `config`） |
| AI 密钥 | `requestLLMCredentials()`（scope `llm-credentials`，且要求已激活） |
| 让宿主代发一次补全 | `requestCompletion()`（scope `llm-invoke`） |
| 受保护资源下载 | `requestProtectedDownloadTicket()`（scope `download-ticket`） |
| 主插件版本 / 能力探测 | `getPluginInfo()`（`PPBHostApi` 上，**无需 scope**） |

没有对应 scope 的能力（触发 CLI agent、借代理任意出网）是**有意不开放**的。
确有需要请开 issue 讨论，我们会评估要不要为它设一个 scope。

---

## 2. `plugin.settings` 里少了四个键

`settings` 本身还在（暂时），但这四个已经迁出：

| 键 | 去向 | 你该怎么办 |
|---|---|---|
| `registrationId` | secretStorage，主插件私有 | 别读。要判断是否激活用 `requestActivationInfo()` |
| `oauthLoginState` | 同上 | 内部实现，本就不该读 |
| `pluginGrants` | 主插件私有 | 用 `api.listGrants()`；**注意它已经不可伪造了**（见下） |
| `agentAutoExec` | 主插件私有 | 内部实现 |

### 这一条修的是一个真实的越权

从前 `settings.pluginGrants` 是公开可变数组，而 `GrantStore` 直接读它。于是：

```js
p.settings.pluginGrants.push({ sourceId:"me", scopes:["llm-credentials"], ... });
await client.requestLLMCredentials();   // 从前：直接拿到真实 API Key，不弹框
```

七个 scope 无一幸免。**如果你的插件（哪怕是出于省事）用过这个手法跳过同意框，
0.4.7 之后必须改成正常的授权流程** —— 首次调用会弹框，用户点允许之后就不再弹。

---

## 3. 主插件重载后，旧的 `PPBClient` 会失效

### 症状

`registerPPBplugin()` 返回的是普通对象，你的插件会一直持有它。PaperBell 更新或
被禁用再启用之后，那个 client 仍然可以调用，但它闭包里的宿主已经没了。

0.4.6 及更早：这种调用会半能用或抛出难以理解的错误。实测有子插件把异常吞掉、
当作「没拿到密钥」继续，直接不带 Authorization 头去打上游 —— 用户看到的是
OpenAI 的 401「You didn't provide an API key」，跟真实原因毫无关系。

0.4.7 起：所有 `request*` 明确失败 —— 返回 `null`，`requestCompletion` 返回
`{ ok: false, error: "PaperBell 已重新加载，请重新握手后再调用" }`，
并在控制台留一条 `[PaperBell]` 警告。**不再抛异常。**

### 正确写法：在 `paperbell:ready` 上重新握手

```ts
export default class MyPlugin extends Plugin {
    private ppb: PPBClient | null = null;

    async onload() {
        const attach = (host: PPBHostApi) => {
            this.ppb?.unregister();          // 丢掉旧的
            this.ppb = host.registerPPBplugin({
                id: this.manifest.id,
                name: this.manifest.name,
                onOpen: () => this.openMySettings(),
            });
        };

        // 谁先加载不确定：先探测，探不到再等事件
        const host = (this.app as any).plugins.plugins["paperbell"]?.api;
        if (host) attach(host);
        // ready 事件在 PaperBell **每次**装载时都会广播，所以这条监听同时
        // 承担「首次握手」和「重载后自动恢复」两个职责，必须常驻。
        this.registerEvent(this.app.workspace.on("paperbell:ready" as any, attach));
    }

    onunload() {
        this.ppb?.unregister();
    }
}
```

关键是 `registerEvent` 那条**不要**在首次握手成功后摘掉。做到这一点，
PaperBell 更新之后你的插件会自动恢复，用户什么都不用做。

同理，`onConfigChange()` 的订阅也随旧 client 失效，要在 `attach` 里重新订阅。

---

## 4. `llm.baseUrl` 与 `llm.model` 现在是「生效值」

`requestSharedConfig()` 和 `requestLLMCredentials()` 返回的这两个字段，
从前给的是用户填过的**原始字段**，现在给的是**实际会被使用的值**：

| | 0.4.6 及更早 | 0.4.7 |
|---|---|---|
| `baseUrl` | 用户填的原文，可能是空串、可能带尾部斜杠 | 已规范化（**去掉尾部斜杠**）；用户留空时回落到内置提供方的默认地址 |
| `model` | 用户填的原文，可能是空串 | 用户没选过时回落到内置默认模型（仅当 baseUrl 也用着默认值） |

**要改的地方**：如果你自己拼过 URL，检查一下别拼出双斜杠。

```diff
- const url = `${cfg.llm.baseUrl}/chat/completions`;   // 旧值带斜杠时会变成 //
+ const url = `${cfg.llm.baseUrl.replace(/\/+$/, "")}/chat/completions`;
```

（0.4.7 起 `baseUrl` 已经不带尾部斜杠了，上面这行是防御性的，两边都安全。）

**顺带修掉的一个坑**：从前「填了密钥但从没手敲过模型名」的用户，AI 设置页的
「测试连接」是绿的（那个测试只验密钥和 baseUrl），而真正发起补全时宿主会拒绝。
现在模型有了回落，报错也会**指名缺哪一样**，不再三样并列。

---

## 5. 没被授权的插件不再出现在设置入口页

从前只要调用过 `registerPPBplugin()` 就会在 PaperBell 设置页出现一张卡片 ——
哪怕每个 scope 都被用户拒了。这等于让任何插件把自己的名字、描述、图标和一个
可点击的 `onOpen` 塞进用户的设置界面。

0.4.7 起的条件是：**在线，且至少持有一个 scope**。

对你的影响：如果希望自己的卡片出现在 PaperBell 设置页，就得实际请求过至少一个
scope 并获得用户同意。只握手不请求任何东西的插件不会显示 —— 这是有意的。

---

## 6. 快速自检

装上 0.4.7 后，在 DevTools 控制台跑一遍：

```js
const p = app.plugins.plugins.paperbell;

// 期望 ["settings", "api"]（外加 Obsidian 自己的 app / manifest / _events 等）
Object.keys(p);

// 期望全部 undefined —— 有任何一个不是，说明你还在用旧写法
[p.verificationWorker, p.proxyService, p.agentExecutor, p.updater,
 p.settings.registrationId, p.settings.pluginGrants].every(x => x === undefined);

// 契约仍然可用
p.api.getPluginInfo();   // → { id, name, version, schemaVersion: 2, capabilities: [...] }
```

再把你的插件禁用→启用一次，确认 PaperBell 更新之后能自动恢复。

---

## 附：为什么要做这次收紧

PaperBell 的定位是**系列插件的宿主**，鼓励用户装配套插件 —— 这意味着
「同一 vault 里有别的插件」这个前提比一般插件更容易成立。而付费墙、API 密钥、
账户信息，正是这套 scope / 同意机制要保护的东西。

把实现对象挂在插件实例上，等于在那套机制旁边留了一扇没锁的门。收紧之后，
契约是**唯一**的入口 —— 这对按契约写的子插件没有任何损失，
反而让「用户授权了什么」变成一件说得清、也真的作数的事。

有任何适配问题，欢迎在
[主仓库](https://github.com/PaperBell-Org/Obsidian-PaperBell-Plugin/issues) 开 issue。
