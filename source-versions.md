---
type: source-versions
project: cross
status: verified
confidence: high
verified_against: 2026-08-10
sources: []
related:
  - AGENTS.md
  - log.md
  - index.md
---

# Source Versions Pin

> 三个上游代码仓的 **当前 commit pin** —— 所有 wiki 页 frontmatter `verified_against` 字段背后的代码状态以本表为准。
> 当任一上游仓 HEAD 移动后，触发 [AGENTS.md §12](AGENTS.md) 增量分析工作流，本文件 §History 追加一行。
> **本文件 §Current pins 段** 是唯一 mutable 区域；§History 严格 append-only。

---

## Current pins (as of 2026-08-10)

| 项目 | 路径 | branch | commit (full) | commit (short) | last commit date | last commit msg |
|---|---|---|---|---|---|---|
| MindIE-LLM | [d:\design\MindIE-LLM\](d:\design\MindIE-LLM) | `master` | `f032cd3f73fee59b0985c3b5fde1e9c0dc3c8691` | `f032cd3f` | 2026-04-15 22:48:40 +0800 | `!901 merge dev into master` |
| vLLM | [d:\design\vllm\](d:\design\vllm) | `main` | `5f7fab881a13df2926f50cb8d9f67b0bbc2ee41f` | `5f7fab88` | 2026-04-16 09:55:29 +0800 | `[ROCm][FEAT] Integrate aiter gemm w8a8 ptpc (#33773)` |
| SGLang | [d:\design\sglang\](d:\design\sglang) | `main` | `06f32bab6baa12b8616a5260e6f2fa2597eb930e` | `06f32bab` | 2026-08-09 23:52:14 -0700 | `[BCG][5/N] MLA Fully Support (#33661)` |

### Working tree state at pin time

| 项目 | working tree status |
|---|---|
| MindIE-LLM | **未在本环境检出**（pin 仍为 2026-04-18 值；`wiki/mindie/` 已删除） |
| vLLM | **未在本环境检出**（pin 仍为 2026-04-18 值） |
| SGLang | clean @ `06f32bab`（本轮增量从 GitHub `sgl-project/sglang` clone 核对） |

### Quick verify commands

```powershell
# 在三个仓根目录依次跑，验证本表 commit 仍是 HEAD
cd d:\design\MindIE-LLM; git rev-parse HEAD  # 期望 f032cd3f...
cd d:\design\vllm;       git rev-parse HEAD  # 期望 5f7fab88...
cd d:\design\sglang;     git rev-parse HEAD  # 期望 06f32bab...
```

如三个 HEAD 都未动 → 已 `verified_against: 2026-08-10` 的 SGLang 页仍有效；MindIE/vLLM 仍对应 2026-04-18 pin。
如任一 HEAD 已变 → 触发 [§12 增量工作流](AGENTS.md)。

---

## History (newest at top, append-only)

> 每次增量分析后追加一行。`Action` 列简述新页 / verify 页清单。

| Pin date | Op | MindIE | vLLM | SGLang | Action |
|---|---|---|---|---|---|
| **2026-08-10** | increment | `f032cd3f` (unchanged) | `5f7fab88` (unchanged) | `34fef07a` → **`06f32bab`** | SGLang §12 增量：+7 新模块页；re-ingest Scheduler / TokenizerManager / scheduler-mixins；re-anchor Engine / TpModelWorker / DataParallelController / managers；高 churn 模块/topic 标 stale |
| **2026-04-18** | initial pin | `f032cd3f` (master) | `5f7fab88` (main) | `34fef07a` (main) | wiki bootstrap + L1 ingest 三家 + 9 comparison topic + 9 mindie entity + 8 mindie topic + 5 vllm topic + 2 sglang module/topic 等 56 页 |

---

## 增量更新工作流（速览，详见 [AGENTS.md §12](AGENTS.md)）

1. **diff 摸底**：`git log <pinned_commit>..HEAD --stat -- <子树>` 列出本期变更（每仓分别跑）
2. **影响面分析**：对每个 changed file，用 `Grep` 在 `wiki/` 内搜文件名（`PrefixCachePlugin` / `BlockSpaceManager` / `eagle.py` 等），命中的 wiki 页就是受影响清单
3. **逐页处理**：
   - 改动小（< 10 行 / 仅 refactor 改名）→ `verify <page>` 重新核对锚点行号
   - 改动大（新类 / 新方法 / 删类）→ `ingest` 该子树增量
   - 改动跨多页 / 跨子系统 → 同时启 multi-subagent 并行 verify
4. **更新 frontmatter**：受影响页 `verified_against` 升到新 pin 日期
5. **更新本文件**：`Current pins` 段改为新 commit；`History` append 一行
6. **append log entry**：`## [YYYY-MM-DD] increment | cross | <repos> updated to <new commits> | <N pages re-verified>`

> **重要**：增量分析的 ROI 远高于 fresh re-ingest——只读 changed files 的 diff，对应 wiki 页用 [`verify` 工作流](AGENTS.md) 处理 markers 即可，不必重写。

---

## See also

- [AGENTS.md §12](AGENTS.md) — 完整增量工作流
- [AGENTS.md §3](AGENTS.md) — frontmatter `verified_against` 字段语义
- [log.md](log.md) — 历史 ingest/verify 操作（含 archive 索引）
- [index.md](index.md) — wiki 全局入口
