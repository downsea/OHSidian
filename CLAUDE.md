# CLAUDE.md

<!-- docs-check: tier=2 -->

> **Size budget: 65536 bytes for `CLAUDE.md` + `AGENTS.md` combined.** Past it the
> host truncates the tail silently. Keep only entry index / hard rules / criteria here;
> detail → `docs/`, state → the ledger. Pointers, not prose.

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.
Template: `~/code/claude-template.md` — this file follows its conventions.

> **Fork setup** — `origin` = our fork `git@github.com:downsea/OHSidian.git`
> (push target for local additions like CLAUDE.md and `docs/`); `upstream` =
> `git@github.com:HanversionOvO/OHSidian.git` (the original — **never push
> there**; contributions go through issues/PRs). We are an external
> builder/forker, not the maintainer.

## What OHSidian is

OHSidian runs the **unmodified official Obsidian 1.12.7 (`obsidian.asar`) on
HarmonyOS** by emulating the Electron API surface with native HarmonyOS
capabilities — it is a compatibility layer, **not** a reimplementation of
Obsidian. Do not patch `obsidian.asar` or reimplement Obsidian features in
ArkTS; the fix always belongs in the adapter/bridge layers.

## Architecture (big picture)

Call flow, top to bottom:

```
obsidian.asar (untouched)                    ← web_engine/src/main/resources/resfile/resources/app/
  → Electron compat (main.js, @electron/remote, Node runtime)
  → JSBind bridge (web_engine/src/main/ets/jsbindings/ — JsBindingUtils.bindFunction("X.y", …),
    all activated at startup via JsBindingMethod.ets)
  → C++ libadapter.so (LFS blob; type decls in web_engine/src/main/cpp/types/libadapter/)
  → ArkTS adapter layer (~50 adapters in web_engine/src/main/ets/adapter/, one system
    capability each: CloudSync, FilePicker, IMF, NativeTheme, Trash, …)
  → HarmonyOS native (Ability / ArkUI / Account Kit / Cloud Foundation Kit)
```

Two-module hvigor build:

- **`web_engine/`** (HAR static library) — the whole engine: adapters, JSBind
  bindings, WebWindow/WebSubWindow components, ChildProcess. Adapters extend
  `BaseAdapter` and are registered as InversifyJS singletons in
  `src/main/ets/common/` (CommonModule / AdapterModule). A new adapter needs:
  adapter class + DI registration + JSBind binding class — all three.
- **`electron/`** (HAP entry) — Abilities (EntryAbility, BrowserAbility,
  StatusBarEntryAbility, …), UI pages (`pages/`), AGC init. Depends on
  `web_engine`.

Huawei-ecosystem specifics: Account Kit login is polling-based (Obsidian side
writes `.hcs-login-pending`, ArkTS side polls every 3s, writes back
`hcs-user.json`); Cloud sync goes through `adapter/CloudSyncAdapter.ets` → Cloud
Foundation Kit bucket `{userId}/vaults/{vaultName}/…`. Auto-update verifies
`obsidian-{version}.asar` with RSA-SHA256 signature + SHA256 hash before
hot-swapping.

## Design principles — at-hand summary

Distilled from the architecture above and upstream's design; there is no
separate philosophy doc.

1. **Obsidian runs unaware.** The official `obsidian.asar` is shipped untouched;
   OHSidian only emulates the Electron API surface underneath it.
2. **The fix lives in the adapter/bridge layers** — Electron compat, JSBind,
   `libadapter.so`, ArkTS adapters — never in the asar, never as an ArkTS
   reimplementation of an Obsidian feature.
3. **One adapter per system capability**, and a new one is always the triad:
   adapter class + DI registration + JSBind binding class.
4. **Stay even with upstream.** No code divergence on the fork unless a fix
   genuinely belongs here; local additions are docs.

## Stable contracts

What the untouched asar-side JS depends on — change one and Obsidian (or an
existing user's data) breaks without an error on our side:

- **JSBind names** (`JsBindingUtils.bindFunction("X.y", …)`, activated in
  `web_engine/src/main/ets/jsbindings/JsBindingMethod.ets`) are the call surface
  the JS side invokes by string — renaming one is a silent break.
- **Login handshake files** in the app's `filesDir`: JS writes
  `.hcs-login-pending`, ArkTS (`CloudSyncAdapter.checkLoginTrigger`) answers with
  `hcs-user.json`.
- **Cloud bucket layout** `{userId}/vaults/{vaultName}/{relativePath}`
  (`CloudSyncAdapter.ets`) — changing it orphans every synced vault.

## Build & Test (headless, on this VM)

This machine is **oh-vm** (aarch64 openEuler; x86_64 toolchain under box64).
**Read the `vm-openeuler` skill before building** and use the `oh` CLI wrappers
(`oh build` / `oh pkg` / `oh app` skills), never raw `hvigorw`/`hdc`/`bm`. The
device side (MOR-M1 HarmonyOS PC) is covered by the `host-hmos` skill.

```bash
git lfs pull        # MANDATORY before first build — *.asar/*.so/*.pak/*.bin are LFS pointers
ohpm install        # hmcore, inversify, @electron/remote, …
# then: oh build (assembleHap) → electron/build/default/outputs/default/electron-default-unsigned.hap (~212 MB)
```

- `build-profile.json5` is **gitignored** (holds signing config; commit
  `b949ff9`-era decision). Bootstrap it from `build-profile.example.json5`. On
  this host it's already set to SDK `6.1.1(24)` to match the target device;
  the README's `6.0.2(22)` is upstream's baseline.
- `web_engine/BuildProfile.ets` is **regenerated by hvigor per build mode**
  (debug/release flips `DEBUG`/`BUILD_MODE_NAME`). A dirty diff there is build
  churn — don't commit it.
- Tests: only `electron/src/ohosTest/` (hypium, on-device); there is no
  meaningful headless test gate. Verification = successful `assembleHap` +
  behavior on the real device.
- Docs gate (stdlib-only, runs anywhere): `python3 scripts/docs-gate.py` —
  budget + `docs/issues/README.md` index + in-repo dead links.
- **Repo-specific disk footprint**: a HAP is ~212 MB, plus `*/build/`,
  `oh_modules/` and the LFS object store. Wrap-up purge here is `rm -rf */build`
  (and `oh_modules` if no rebuild is planned — `ohpm install` restores it).

Build slot, disk hygiene (incl. the real water level on this VM), the global
cargo profile and OOM triage are **host rules** — see the host-level
`~/.claude/CLAUDE.md` ("Builds on a shared machine"). New Rust repo conventions:
nan-fedora `docs/rust-build.md`.

## Host (HarmonyOS PC) workspace — one-way sync, VM is the source of truth

A copy of this repo lives on the host at `~/code/OHSidian` (VM view:
`/mnt/linux_share/storage/Users/currentUser/code/OHSidian`) as a **DevEco Studio sign/build/run station**
(the DevEco auto-signing path for restricted permissions). Rules:

- `watchexec` + `~/.bin/code-sync` mirror VM `~/code` → host **one-way**,
  gitignore-filtered. **Edit tracked source only on the VM** — host-side edits
  to tracked files never flow back and get clobbered on the next VM change.
- Gitignored files are invisible to the sync, so they are **intentionally
  divergent per side**: root `build-profile.json5` is `6.1.1(24)` on the VM
  (headless toolchain, matches device) but `6.1.0(23)` on the host (DevEco's
  max supported SDK). DevEco-written signingConfigs, `oh_modules/`, `build/`,
  `agconnect-services.json` on the host all survive syncs for the same reason.

## ⚠ Known blocker: debug sideload is impossible from the CLI

Read [`docs/issues/debug-sideload-blocked-by-writable-code-memory.md`](docs/issues/debug-sideload-blocked-by-writable-code-memory.md)
**before** attempting to install on a device. Summary: the app requires the
kernel-restricted permission `ohos.permission.kernel.ALLOW_WRITABLE_CODE_MEMORY`
(V8 JIT — non-removable); AGC's debug provisioning never grants it to a
headless-issued profile, so `install` always fails with `9568289 grant request
permissions failed`. Signing itself works fine — do not re-debug the signature
path. Root cause + verified solution (AGC restricted-ACL application, or DevEco
auto-signing) in
[`docs/research/2026-07-03-writable-code-memory-sideload.md`](docs/research/2026-07-03-writable-code-memory-sideload.md).

## Current state (snapshot — 2026-08-13; update, don't re-derive from git)

- Fork is **even with `upstream/main`** (0 behind); our only commits on top are
  the two local docs commits (research + host-workspace convention). No code
  divergence from upstream — keep it that way unless a fix genuinely belongs here.
- **Blocked** on device install by the restricted-permission issue above; the
  build itself (`assembleHap` → ~212 MB HAP) and signing are known-good.
- Not yet done: AGC restricted-ACL application, or a DevEco auto-signing run on
  the host station — either unblocks install. Nothing else is in flight.

## Hard rules

- **Never modify `obsidian.asar`** or anything under
  `resfile/resources/app/node_modules/` beyond what upstream ships — the
  compat-layer contract is "Obsidian runs unaware".
- **Never commit or print** `agconnect-services.json`, `build-profile.json5`,
  or signing materials (`*.p12/*.p7b/*.cer`) — all gitignored for a reason.
- LFS-tracked binaries (`.asar`, `.so`, `.pak`, `.bin`, `.dat`) must go through
  `git lfs`; a "modified" 130-byte pointer file means LFS wasn't pulled.

## Research → dev

Non-trivial investigation is **research-first, and the findings are durable** —
never let a root-cause hunt dissolve into chat history.
`docs/research/YYYY-MM-DD-<slug>.md` holds the point-in-time investigation:
what was tried, what was ruled out, the mechanism, the proposed solutions
(exemplar: `2026-07-03-writable-code-memory-sideload.md`). The date is
load-bearing: findings are true as-of that day's SDK/AGC behaviour. Findings are
host-shaped too — say which device/SDK they were verified on.

Research is mandatory before adopting a new HarmonyOS capability or a new
signing/provisioning path; not needed for build-config touch-ups or a fix an
existing research doc already explains — cite that doc instead.

## Work units

Every piece of work so far has been **task**-sized: one `docs/issues/<slug>.md`
(+ its research doc when non-trivial), a few commits. If work ever spans several
issues toward one goal (e.g. getting the app installed and running end-to-end
on the device), make it an **epic**: an umbrella issue + one research doc it
cites, per the family template (`~/code/claude-template.md`, "Work units").

## Usage-time feedback

Anything hit while building, signing, or running OHSidian on the device — a
failing adapter, a README gap, a toolchain trap — is filed at discovery time,
not left in chat history:

- `docs/issues/<slug>.md` (indexed in `docs/issues/README.md`, same commit) in
  the existing style 现象 → 根因 → 排除项 → 复现步骤 → 可能的解法, plus a
  `Status:` line. File first, then work around — and name the workaround.
- Anything **upstream** should fix (README gaps, adapter bugs) is noted here
  first, for a later GitHub issue on `HanversionOvO/OHSidian`.
- A gap in the tooling (`oh` CLI, `vm-openeuler` / `host-hmos` skills) is filed
  in the repo that owns that tool, not worked around here.

## Cross-repo development

- **Upstream owns the code.** `HanversionOvO/OHSidian` is the source of truth;
  we are an external builder. Fixes that belong upstream go as GitHub issues/PRs
  there — never push to `upstream`, and keep the fork's code even with it.
- **Tooling gaps belong to their owners**: build/sign/install behaviour of `oh`
  and the `vm-openeuler` / `host-hmos` playbooks → `~/code/oh-cli`. File there
  and back-link from the issue here.

## Session orchestration

The main session is the orchestrator: it owns decisions, the real device, the
signing materials, and the commits. Delegate a self-contained investigation or
a multi-file adapter change to a sub-agent, and give it the **required reading**
(this file, the relevant research/issue doc, the `vm-openeuler` skill), the
**environment traps** (`git lfs pull` first, gitignored `build-profile.json5`,
never commit `web_engine/BuildProfile.ets` churn), and the **boundaries**
(no commits — the main session verifies the build itself and commits). One-line
fixes and read-only checks stay in the main session.

## Commit style

Upstream uses conventional commits with Chinese descriptions
(`fix: 更新构建配置为发布模式`) — match that if asked to commit. No
`Co-Authored-By` footers.

## New work checklist

- [ ] Non-trivial investigation → findings written to `docs/research/` (or an
      existing research doc cited) before the fix.
- [ ] The fix lives in the adapter/bridge layers, not in `obsidian.asar`.
- [ ] New adapter → adapter class **+** DI registration **+** JSBind binding class
      (all three), and the architecture description above updated if the call
      flow or module split changed.
- [ ] `assembleHap` succeeds; device behaviour verified when a device path exists.
- [ ] No signing material, `agconnect-services.json`, or `build-profile.json5`
      staged; no `web_engine/BuildProfile.ets` build churn in the commit.
- [ ] `docs/issues/` updated if a tracked problem moved, and the project ledger
      updated; `docs/issues/README.md` index row moved in the same commit.
- [ ] `CLAUDE.md` + `AGENTS.md` still under the 65536-byte budget (`wc -c`); anything
      added here is entry index / hard rule / criterion, or it went to `docs/` instead.
- [ ] `python3 scripts/docs-gate.py` green.

Project ledger (cross-session progress): `~/Documents/Vault/20-Projects/Active/OHSidian.md`
— create via `kms new project "OHSidian"` on first substantial work session
(per `30-Areas/Technology/project-tracking.md` SOP); none exists yet.
