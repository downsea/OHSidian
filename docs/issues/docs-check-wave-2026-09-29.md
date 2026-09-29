# 文档闸对齐家族模板 + 删除主机规则副本（2026-09-29 docs-check 波）

> Status: Done · 2026-09-29 · 发起：nan-fedora docs-check 家族波（ADR-0014 两层闸；ADR-0028 主机规则只留一份正本）

## 现象

`docs_check.py ~/code/OHSidian --strict` 基线 **E=7 W=0**：

- A2 未声明档位（推断 T2）；A6 入口无预算提醒；A12 checklist 无预算自检行
- A3 按 T2 缺节：Build & Test（原名 `Build (headless, on this VM)`）· Design principles ·
  Research → dev（原名 `Research → dev, and the issue tracker`，逗号不是闸认的分隔符）·
  Work units · Usage-time feedback；建议节 Stable contracts · Cross-repo development
- S4 `docs/issues/` 无 `README.md` 索引；G1 仓里没有文档机械闸

另：`CLAUDE.md` 的 `Build-cache / disk hygiene` 小节是**主机规则**的副本（水位、收工清理、
trim 滞后），2026-09-29 起正本只在主机级 `~/.claude/CLAUDE.md`（nan-fedora ADR-0028）。

## 根因

`CLAUDE.md` 按早期模板实例化，之后模板加了档位 / 预算 / 必有节；仓里从未有过文档闸，漂移无人报。

## 改动

- `CLAUDE.md`：声明 `tier=2` + 预算提醒；`Build` → `Build & Test`（修正测试目录为
  `electron/src/ohosTest/`、CloudSync 文件名为 `CloudSyncAdapter.ets`）；删 disk hygiene 副本，
  只留本仓特有的体积与清理对象 + 指向主机规则的指针；新增 Design principles（取自现有架构与硬规则）、
  Stable contracts（JSBind 名 / 登录握手文件 / 云桶路径，均已对照源码）、Work units、
  Usage-time feedback（issue 体例从原 Research 节移入）、Cross-repo development（upstream 与 oh-cli 的归属）；
  checklist 加预算自检与文档闸两行。
- `docs/issues/README.md`：新建索引（slug 命名，不编号）。
- `scripts/docs-gate.py`：`docs_check.py --emit-gate py` 生成的仓层最小闸（预算 · 索引双向 · 仓内死链）。
  按家族 2026-09-28 规则不接 CI（无 push/PR CI）。

## Exit

- [x] `docs_check.py ~/code/OHSidian --strict`：E 7→1，W 0→0；剩下的唯一 E 是 **G2（闸未接 CI）**，
      按家族 2026-09-28 规则（无 push/PR CI，nan-fedora issue 068）有意保留 —— 本仓的接入点是 New work checklist 里的手跑行
- [x] `python3 scripts/docs-gate.py` 当场跑过，绿
- [x] 无 `--fix` 落盘（全部手改；`--fix --dry-run` 目检后发现 A12 会把勾选行追加到 checklist 之后的散文段末尾，改为手工插入）

最终 `--strict` 输出（末尾）：

```
| `OHSidian` | T2 | 1 | 0 | 2 | 0 | ✓ 13517 | 2026-08-13 |

### ✗ E · G2 —— 闸存在但没接进 CI / 测试套件（只能手跑 = 会忘）
- `scripts/docs-gate.py`

### · I · B1 —— 指令预算 ✓ 13517 / 65536 B
### · I · G1 —— 仓层闸：scripts/docs-gate.py
```
