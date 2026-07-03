# Research: 绕过 `ALLOW_WRITABLE_CODE_MEMORY` 侧载墙的可行路径

> Date: 2026-07-03 · Motivated by: [`docs/issues/debug-sideload-blocked-by-writable-code-memory.md`](../issues/debug-sideload-blocked-by-writable-code-memory.md)
> Method: two parallel web-research passes (AGC/ACL provisioning mechanics; JIT permission semantics + ecosystem prior art) + local SDK / profile inspection.

## TL;DR

`ohos.permission.kernel.ALLOW_WRITABLE_CODE_MEMORY` 是 **ACL-enabled 的受限开放权限**（system_basic / system_grant / `provisionEnable: true` / since API 14，仅平板、PC/2in1 可申请）。**debug profile 完全可以携带它** —— 但前提是该 App ID 已通过 AGC 的「受限 ACL 权限」审核。我们 headless CLI 流程拿到的 profile `permissions:{} / acls:[]` 为空，不是 CLI 的 bug，而是 **App ID 从未申请过受限权限** → AGC 的 profile 签发（控制台勾选框 / Provisioning API `aclPermissionList`）只会回填**已审核通过**的 ACL。

两条走得通的路：
1. **AGC 受限 ACL 权限申请**（≈1–3 个工作日）→ 重新签发 debug profile → headless 流程原样跑通。**推荐**，一次申请永久解锁 CLI 路径。
2. **DevEco Studio 自动签名**（当天）：DevEco ≥4.0 在自动签名过程中**代开发者向 AGC 提交受限权限申请**并签发含 ACL 的 debug profile。需要一台能跑 DevEco 的 Windows/Mac 机器（本 VM 无法运行）。

不可行/下策：装上游 release HAP（签名/UDID 不覆盖本机，PC 侧载工具链如小白助手也是**重签名**流程，同样撞 ACL）；V8 `--jitless`（Electron 支持 `--js-flags=--jitless`，但 **WebAssembly 直接不可用**，Obsidian 的 wasm 插件会挂，且无人验证过华为 OHOS Electron 的 jitless 端到端行为）。

## 权限本体（本机 SDK 权威定义）

`command-line-tools/sdk/default/openharmony/previewer/common/resources/module.json` → `definePermissions`:

```json
{"name":"ohos.permission.kernel.ALLOW_WRITABLE_CODE_MEMORY",
 "availableLevel":"system_basic", "grantMode":"system_grant",
 "provisionEnable":true, "isKernelEffect":true, "since":14}
```

- 官方[受限开放权限列表](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/restricted-permissions)条件：仅用于自带引擎的 JIT（**禁止热更新**）；需**主动适配坚盾模式（不闪退）**；**仅平板、PC/2in1 设备应用可申请**；审核约 3 个工作日。
- 内核强制是真实且静默的：新内核对代码页 `mprotect(PROT_WRITE)` **静默忽略**（调用"成功"、首次写崩溃），Servo 的 OHOS 移植为此做了运行时探测 + 解释器回退（[servo#40130](https://github.com/servo/servo/pull/40130)）。
- ArkWeb（系统 webview）的渲染进程是系统组件，应用自身**不需要**此权限即可获得 JS 执行 —— 这就是为什么只有自带 Chromium/V8 的 Electron 移植才撞墙。

## 本项目声明的 5 个受限/ACL 类权限的归类

| 权限 | 等级 | 受限开放? | 处置 |
|---|---|---|---|
| `kernel.ALLOW_WRITABLE_CODE_MEMORY` | system_basic, ACL✓ | **是**（~3 工作日审核） | **必须申请**，V8 命根子，不可移除 |
| `READ_PASTEBOARD` | system_basic, user_grant, ACL✓ | 是 | 同批申请 |
| `READ_WRITE_DESKTOP_DIRECTORY` | system_basic, user_grant, ACL✓，2in1/平板 only | 是（申请通过后即时生效） | 同批申请 |
| `SYSTEM_FLOAT_WINDOW` | system_basic, ACL✓ | 否（普通 ACL） | 审核通过后随 profile 勾选携带 |
| `WINDOW_TOPMOST` | API 13 起 normal | 否 | 无需处理 |

## AGC profile 签发机制（为什么 CLI 拿到空 ACL）

- 官方流程「[申请调试Profile](https://developer.huawei.com/consumer/cn/doc/app/agc-help-add-debugprofile-0000001914423102)」：添加 Profile 页有可选步骤「在Profile内添加ACL权限」，弹窗列出的是「**当前应用已获取的ACL权限**」——从未申请 → 列表为空 → profile `acls` 为空。
- [Provisioning API](https://developer.huawei.com/consumer/cn/doc/App/agc-help-provision-api-apply-provision-0000002271000689) 的 `aclPermissionList` 字段同理：「仅支持受限ACL权限**已审核通过**的应用使用」。
- 社区实测：受限权限勾选框**在华为审批通过前根本不出现**（[坚果派](https://www.nutpi.net/thread?topicId=590)）。
- 本机实测 profile（`~/.config/oh/profiles/com.mikannqaq.obsidian.p7b`）：`type: debug, apl: normal, app-feature: hos_normal_app, permissions: {}, allowed-acls: []` —— 与机制完全吻合。

## DevEco 自动签名为什么是例外

官方「[申请使用受限权限](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/declare-permissions-in-acl)」原文：

> 在调试阶段……还可以通过 DevEco Studio 自动签名完成申请。**在自动签名的过程中，将由 DevEco Studio 完成向 AGC 申请受限权限的步骤**。在发布阶段，必须根据申请发布Profile完成受限权限的手动申请。

即 IDE 读取 module.json5 的 `requestPermissions`，代为向 AGC 提交受限权限申请并签发含 ACL 的 debug profile。这解释了生态现状：上游 README、[ohos_electron_hap 模板](https://github.com/ohosvscode/ohos_electron_hap)、KeeWeb 移植全都走 DevEco 自动签名，从没人提过 9568289。

## 生态 / 上游现状

- 上游 [HanversionOvO/OHSidian](https://github.com/HanversionOvO/OHSidian) 有 release `HarmonyOS-1.12.7`（2026-05-30，`OHsidian.hap` 219 MB，587 次下载），无 AppGallery 上架、无安装说明；issue #1「怎么在HarmonyOS 6.0电脑上安装HAP」**无人回答**。11 个 issue 零提及 JIT/9568289。
- HarmonyOS PC 侧载文化 = 开发者模式 + 小白调试助手，本质是**用用户自己的开发者账号重签名** —— 遇到受限权限同样需要该账号的 App ID 有 ACL 审批，装上游 release HAP 不是捷径。
- `--jitless`：V8 官方支持（[v8.dev/blog/jitless](https://v8.dev/blog/jitless)，JS 慢约 6–40%），Electron 可 `--js-flags=--jitless`；但 **wasm 无解释器、全 tier 需可执行内存 → jitless 下 WebAssembly 不可用**（[chromium-dev 讨论](https://groups.google.com/a/chromium.org/g/chromium-dev/c/HDVGsHrhWCY)）。只配作实验。

## 结论 → 建议决策

**主路径（推荐）**：在自己的 AGC 账号下为 App ID（bundle `com.mikannqaq.obsidian`）走「申请受限ACL权限」：
1. AGC 控制台（或历史流程邮件 agconnect@huawei.com）提交 `kernel.ALLOW_WRITABLE_CODE_MEMORY` + `READ_PASTEBOARD` + `READ_WRITE_DESKTOP_DIRECTORY`，理由模板：**「应用为 Electron/Chromium 自带引擎，申请可写可执行匿名内存用于 V8 即时编译；不用于热更新；已适配坚盾模式」**；设备形态填 2in1/平板。
2. ≈1–3 个工作日批复后，重新签发 debug profile（控制台勾选 ACL，或 Provisioning API 传 `aclPermissionList`，`SYSTEM_FLOAT_WINDOW` 一并勾上）。
3. headless 流程不变：`hvigor assembleHap` → hap-sign-tool 重签 → `oh app install`。

**备选（更快，需外部机器）**：任意 Windows/Mac 装 DevEco Studio ≥4.0，登录同一 AGC 账号自动签名跑一次 —— 当天即可验证应用在 MateBook 上能否真正运行，同时等主路径审批。

**上游可反哺**：README 应注明「命令行 debug 侧载需先在 AGC 申请受限 ACL 权限，否则 install 阶段报 9568289；或使用 DevEco 自动签名」——可作为 PR/issue 提给上游（顺带回答其 issue #1）。

## Sources

见各节内联链接；完整检索记录由两个 research agent 产出（AGC/ACL 机制 pass 与 JIT/生态 pass），关键源：华为官方 restricted-permissions / declare-permissions-in-acl / agc-help-add-debugprofile / Provisioning API 文档、本机 SDK definePermissions、servo#40130、v8.dev jitless、上游 repo releases/issues。
