# Debug 侧载安装失败：`ALLOW_WRITABLE_CODE_MEMORY` 无法通过 AGC debug profile 授予

> Status: Open · 环境：HarmonyOS 6.1.1 (API 24, 2in1/MateBook) · 工具链：命令行 hvigor 5.0 + hap-sign-tool（无 DevEco Studio，headless）

## 现象

按 README「构建与运行」把 HAP 构建、签名后，`hdc/bm install` 到真机（2in1）一律失败：

```
code:9568289  error: install failed due to grant request permissions failed.
PermissionName: ohos.permission.kernel.ALLOW_WRITABLE_CODE_MEMORY
```

安装器按权限检查顺序逐个卡。逐一从 `web_engine/src/main/module.json5` 移除受限权限后，会依次暴露下一个：

```
READ_PASTEBOARD → READ_WRITE_DESKTOP_DIRECTORY → SYSTEM_FLOAT_WINDOW → … → kernel.ALLOW_WRITABLE_CODE_MEMORY
```

**决定性测试**：把 `requestPermissions` 砍到只剩 `INTERNET` + `kernel.ALLOW_WRITABLE_CODE_MEMORY` 两个，安装**就卡在 `ALLOW_WRITABLE_CODE_MEMORY`**。而这个权限是 Electron/V8 的 JIT 命根子（可写可执行内存），移除即无法运行 → **无解的必需权限**。

## 根因

`ALLOW_WRITABLE_CODE_MEMORY` 是**内核级受限权限**。对普通应用（provisioning profile 的 `bundle-info.app-feature = "hos_normal_app"`），**AGC 的 debug provisioning 不会把它授进 profile**：

- 签发出的 debug profile 里 `permissions == {}`、`acls.allowed-acls == []` 始终为空。
- 命令行 provision 工具明确报：把这些权限归类为 *"normal-level permission(s) AGC does not accept in an ACL"* 而全部省略，包括 `SYSTEM_FLOAT_WINDOW` / `WINDOW_TOPMOST` / `kernel.ALLOW_WRITABLE_CODE_MEMORY` 等。
- 多次全新 on-device 浏览器登录 + provision，结果一致。

即：**headless 命令行工具链拿不到一个「授予了这些特权权限」的 debug profile**。

## 排除项（都不是原因）

- **不是签名机制问题**：`hap-sign-tool` / 命令行签名能签出合法 HAP（未出现 `9568329 verify signature failed`）；错误是 `9568289 grant permissions failed`，纯权限授予问题。
- **不是设备兼容问题**：`module.json5` 已声明 `deviceTypes: ["2in1","tablet"]`，设备 UDID 也在 debug profile 覆盖列表内。
- **不是 SDK 版本问题**：`compatibleSdkVersion/targetSdkVersion` 对齐设备 `6.1.1(24)`，构建 `BUILD SUCCESSFUL`。
- **README 的 build-profile.json5 `signingConfigs` 手动签名是 DevEco IDE 专用**：`storePassword`/`keyPassword` 需 IDE 加密串，填明文会被 hvigor 拒（`The length of the storePassword or keyPassword field ...`）。headless 只能用外部 `hap-sign-tool` 签名，绕不开上面的 profile 问题。

## 复现步骤（headless）

1. `git clone` + `git lfs pull`（`.asar`/`.so`/`.pak` 等都是 LFS，必须拉实体）。
2. `ohpm install`（hmcore/inversify 等）。
3. `build-profile.json5` 设 `compatibleSdkVersion/targetSdkVersion = "6.1.1(24)"`。
4. `hvigor assembleHap` → `electron/build/default/outputs/default/electron-default-unsigned.hap`（约 212 MB）。
5. 外部工具签名 → 真机 `install` → 报 `9568289 ... ALLOW_WRITABLE_CODE_MEMORY`。

## 可能的解法（待验证）

1. **DevEco Studio「自动签名」**：IDE 的 AGC provisioning 路径可能能把特权权限申请进 profile（比裸命令行工具强）。若可行，建议 README 在「签名配置」明确：**Electron 版必须用 DevEco 自动签名，命令行 debug 侧载不可行**。
2. **Release 签名 + 华为审批**：`ALLOW_WRITABLE_CODE_MEMORY` 作为受限权限走 AGC 申请审批，再用 release profile 签发（面向 AppGallery/企业分发）。
3. 若作者已有 release 签名包，README 补一个「直接安装签名 HAP」的分发渠道，能省掉自签流程。

## 建议

README「构建与运行」建议补一句醒目提示：**本项目依赖 `ALLOW_WRITABLE_CODE_MEMORY` 等内核级特权权限，命令行 debug 侧载会在 install 阶段失败（9568289）；请用 DevEco Studio 自动签名或 release 签名安装。**

---

*记录者：外部构建者（headless aarch64 openEuler VM + box64 工具链，为 MateBook 侧载 OHSidian 时踩到本坑）。构建全程成功，仅安装阶段因权限模型受阻。*

---

## 2026-07-03 研究结论（见 [docs/research/2026-07-03-writable-code-memory-sideload.md](../research/2026-07-03-writable-code-memory-sideload.md)）

- **Status: root-caused, solution identified** —— 空 ACL 不是 CLI 工具缺陷：AGC 只会把**已审核通过**的受限 ACL 权限回填进 profile（控制台勾选框 / Provisioning API `aclPermissionList` 同理），本 App ID 从未申请过 → `acls: []`。
- `kernel.ALLOW_WRITABLE_CODE_MEMORY` = 受限开放权限（system_basic、ACL✓、仅平板/PC/2in1、审核约 3 个工作日、禁止热更新、需适配坚盾模式）。**debug profile 审批后可以携带它**。
- 「可能的解法」核实结果：
  1. ✅ **DevEco 自动签名**确实可行（官方文档明示：自动签名过程中由 DevEco 代向 AGC 申请受限权限）——但需 Windows/Mac 机器。
  2. ✅ **AGC 受限 ACL 权限申请**（推荐主路径）：申请通过后重签 debug profile，headless 流程原样解锁。批量申请 `kernel.ALLOW_WRITABLE_CODE_MEMORY` + `READ_PASTEBOARD` + `READ_WRITE_DESKTOP_DIRECTORY`；`SYSTEM_FLOAT_WINDOW` 是普通 ACL 随 profile 勾选；`WINDOW_TOPMOST` API 13 起已是 normal。
  3. ❌ 装上游 release HAP 不是捷径：PC 侧载 = 用户账号重签名，同样需要 ACL 审批。
  4. ❌ `--jitless` 仅配作实验：wasm 在 jitless 下不可用，Obsidian wasm 插件会挂。

## Exit
- [ ] AGC 受限 ACL 权限申请提交并获批（App ID: com.mikannqaq.obsidian）
- [ ] 重新签发的 debug profile 中 `allowed-acls` 含 `kernel.ALLOW_WRITABLE_CODE_MEMORY`
- [ ] headless 重签 HAP 在 MateBook `oh app install` 成功、应用可启动
