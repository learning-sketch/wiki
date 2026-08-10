# Wiki Schema for d:\design

> 这份文件是 wiki 的 **施工手册**。任何 LLM agent 在本目录工作时必须先读完这份。
> 它定义了：三层模型、命名约定、页面模板、ingest / query / lint 工作流、跨项目对比规则。

---

## 0. 5 条硬约束（违反任意一条都视为该次产出无效）

1. **Source-anchored**：wiki 页里每个具体论断必须给出代码引用（绝对路径 + 行号），用 markdown 链接指向真实文件。禁止"看起来合理"的描述但没有源码 anchor。
2. **派生物明示**：wiki 页是原始代码 / 原始文档的派生物，不是事实来源。需要综合的结论用 `synthesis:` 前缀，区别于直接观察。
3. **置信度标注**：每页 frontmatter 必须有 `confidence: high|medium|low` 和 `verified_against`。
4. **矛盾 / 未验证显式标记**：用 `> [!warning] CONTRADICTION` / `> [!todo] VERIFY` 块，不静默吞掉。
5. **链接非有限**：每页保留 `## See also` 段，允许跨页 / 跨项目链接。`comparison\topics\*` 必须指回每个项目的具体实现页。

---

## 1. 三层模型

| 层 | 位置 | 谁拥有 | 是否可变 |
|---|---|---|---|
| **Raw 资料** | `d:\design\wiki\raw\` | 用户 | 不可变 |
| **项目代码** | `d:\design\MindIE-LLM\`, `d:\design\vllm\`, `d:\design\sglang\` | 上游 / 用户 | 不可变（从 wiki 视角看） |
| **Wiki 派生物** | `d:\design\wiki\` 下除 `raw\` 外的所有 `.md` | LLM agent | 可变，但需溯源 |

**重要**：wiki 不是真理。任何被引用的 wiki 论断都必须能回溯到原始代码 / 原始文档。

---

## 2. 命名约定

| 类型 | 命名 | 例子 |
|---|---|---|
| 模块页 | `snake_case`，与代码模块名对齐 | `executor.md`, `text_generator.md` |
| 实体页 | `PascalCase` 类名 / `snake_case` 函数名 | `EngineCore.md`, `make_async_executor.md` |
| 主题页 | `kebab-case`，描述性 | `request-lifecycle.md`, `pd-disaggregation.md` |
| 对比页 | `kebab-case` 主题名 | `comparison/topics/scheduler.md` |

目录布局（每个项目下相同）：

```
<project>\
├── index.md         # 项目内所有页面的目录
├── overview.md      # 架构总览
├── modules\         # 一级代码模块对应的概念页
├── entities\        # 关键 class / 关键函数对应的实体页
└── topics\          # 跨模块主题（PD分离 / KV cache / scheduler 等）
```

---

## 3. 页面 Frontmatter 模板（必填）

```yaml
---
type: module | entity | topic | comparison | overview | index
project: mindie | vllm | sglang | cross
status: draft | verified | stale
confidence: high | medium | low
verified_against: <git commit short hash 或 ISO 日期>
sources:
  - d:\design\<...>:L<begin>-L<end>
  - d:\design\<...>
related:
  - <project>\<dir>\<page>.md
---
```

字段说明：

- `status`：
  - `draft` — 已写但未交叉验证
  - `verified` — 与源码至少对照过一遍
  - `stale` — 源码已变，本页需要重新校对
- `confidence`：
  - `high` — 论断都直接来自源码 / 文档原文
  - `medium` — 含部分综合 / 推断，但有强证据
  - `low` — 含猜测，必须配合 `> [!todo] VERIFY` 块
- `verified_against`：写一个能回溯的锚（commit hash 或日期），方便后续 lint 判断 stale。

---

## 4. 标准页面模板

```markdown
---
<frontmatter，见 §3>
---

# <标题>

## Summary
<1-3 句话的 TL;DR>

## Sources
<本页用到的所有源文件，必须是绝对路径 + 行号区间（如适用）>
- [d:\design\vllm\vllm\v1\executor\multiproc_executor.py](d:\design\vllm\vllm\v1\executor\multiproc_executor.py) (entire file)
- [d:\design\vllm\vllm\v1\engine\core.py:120-340](d:\design\vllm\vllm\v1\engine\core.py)

## Architecture / Data flow
<可选的 mermaid 图 + 文字说明，每条具体论断带 [file:line] 引用>

## Key APIs / Entities
| 名称 | 位置 | 作用 |
|---|---|---|
| `MultiprocExecutor` | [path](path) | ... |

## Notes / Caveats
> [!todo] VERIFY: <未验证点>
> [!warning] CONTRADICTION: <跨页冲突>

## See also
- [<project>\<dir>\<page>.md](<相对路径>)
- [comparison\topics\<topic>.md](comparison\topics\<topic>.md)
```

> **关于代码引用**（务必遵守）：用户的 user rule 要求"链接到真实文件，不是 context 生成文件"。所以：
>
> - 引用整个文件：`[d:\design\<...>\foo.py](d:\design\<...>\foo.py)`
> - 引用区段：`[d:\design\<...>\foo.py:120-340](d:\design\<...>\foo.py)`（链接还是指向真实文件，行号写在显示文本里）
> - 引用源码片段时使用 Cursor 的 startLine:endLine:filepath 块引用（这一条只在 chat 中适用，wiki markdown 不用）

---

## 5. Ingest 工作流

**触发**：用户说 `ingest <模块/文件/主题>`，或 `ingest <project> <topic>`。

**步骤**（单次 ingest 应当原子提交，能 rollback）：

1. **范围确认**：
   - 列出本次要读的源文件清单（用 Glob / SemanticSearch 找，但读用 Read）。
   - 与用户确认范围（如清单 > 30 文件需重新切分）。
2. **读源**：
   - 用 Read 读完整文件，禁止只读片段后下结论。
   - 用 Grep 验证关键符号在哪定义、被谁引用。
3. **Hidden cross-reference 全仓库 grep（必跑）**：

   主源文件读完后**强制**执行以下 5 类全仓库 grep（含跨语言、跨子系统），把命中的纳入正文 `§使用方调用清单` / `§跨子系统引用` 章节，把 **N/A 必须明示已 grep 范围**（不能只写 "未发现"）。这些是 entity 自身源文件**看不到**的协作关系——是 [§9 hidden state](#9-代码探索的最佳实践写给-llm-agent-自己看) 的姊妹规则。

   | # | 类别 | grep 模板 | 防漏 |
   |---|---|---|---|
   | 1 | **跨语言绑定** | C++ entity → grep 类名在 `mindie_llm/` Python 树；Python entity → grep 类名在 `src/` C++ 树。关注 `pybind` / `ctypes` / `cffi` / `Cython` / `capsule` / 命名管道 / 文件 IPC | C++↔Python 双向调用入口（如 LlmEngine 是否经 Manager 暴露给 Python） |
   | 2 | **协作伙伴跨子系统引用** | 列出 entity 已知的 3-5 个协作类名（如 `BlockSpaceManager` 的协作 = `Scheduler`/`PluginManager`/`Mooncake`/`LLMDataDist`），每个名字在**全仓库**做 grep | 协作伙伴在 connector / server / examples 等远端模块的意外引用 |
   | 3 | **配置 / IPC 共享数据结构** | grep entity 读的配置字段名（如 `BlockManagerConfig` 字段名），看是否在远端模块也被读 | 通过共享 protobuf / yaml / json 隐式协作的远端代码 |
   | 4 | **测试覆盖反查** | grep `tests/` 下对该 entity 的 unit + integration 测试 | 测试代码反推"这个类预期被怎么用"，常发现 prod 没有但测试覆盖的调用模式 |
   | 5 | **doc / config / yaml 反查** | grep entity 名在 `docs/` / `config/` / `*.yaml` / `*.json` | 产品文档/部署配置里提到的"未在代码里直接出现"的协作模式 |

   **N/A 输出格式**：必须写"<key>: 在 <已 grep 范围> 全树 grep 0 命中"，不能写"未发现"。例：

   ```
   - Mooncake / LLMDataDist：在 d:\design\MindIE-LLM\src\ 全 C++ 树 grep 0 命中
   - LlmEngine pybind: 在 d:\design\MindIE-LLM\mindie_llm\ 全 Python 树 grep 0 命中（仅 Manager 间接接入）
   ```

   **跳过条件**：仅当 entity 是纯叶子模块（如单个数据类 `dataclass`，无任何协作）可跳过本步，跳过时在 `[!todo] VERIFY` 里说明。
4. **起草**：
   - 按 §4 模板新建 / 更新页面。
   - 每条具体论断 inline 带 `[file:line]` 链接。
   - 综合性结论加 `synthesis:` 前缀。
5. **找候选关联页**：
   - 在已有 wiki 中 grep 关键概念名，列出可能要回链的页面。
   - 更新对应页面的 `related` 和 `## See also`。
6. **跨页校验**：
   - 与已存在的页面对比是否有矛盾，有则在矛盾双方都加 `> [!warning] CONTRADICTION`。
7. **更新 index 和 log**：
   - `<project>\index.md`：补/改条目（一行简介 + 链接 + 来源数量）。
   - 全局 `index.md`：仅当新增 topic 类页时补条目。
   - `log.md`：append 一条 `## [YYYY-MM-DD] ingest | <project> <topic> | <pages updated>`。
   - **append 后检查 `log.md` 是否超阈值**（详 §11 Log rotation）；超则触发轮转。

---

## 6. Query 工作流

**触发**：用户提问。

**步骤**：

1. **优先 grep wiki**：先用 Grep 在 `d:\design\wiki\` 内搜索关键词。
2. **若 wiki 不够**：再读源码（按 ingest 步骤 2）。
3. **答案中给链接**：所有具体论断都用 `[<project>\<dir>\<page>.md](<相对路径>)` 形式链回 wiki 页，便于追溯。
4. **回写**：高价值的综合答案（涉及 ≥ 2 个页面 / 跨项目对比）应当作为新的 topic 页 / comparison 页存回 wiki，并 append log。

### 特殊指令：`verify <page>` 与 anchor-driven cross-check

用户也可能用以下两种方式触发**已建页面的 verify pass**：

| 触发 | 含义 | 处理 |
|---|---|---|
| `verify <page>` | 系统化复查整页：源码锚点是否仍有效、`[!todo] VERIFY` / `[!warning] CONTRADICTION` 是否能解决、frontmatter 是否合规 | 按 §7 lint 检查清单跑；resolved 的 marker 改为划线 + `**RESOLVED <日期>**` 并附结论 |
| `<file>:<line> 这个其它两家有吗 / 你漏了` | 用户提供单点锚点，要求做 anchor-driven cross-check | 按 §8 规则 6 处理：用 §9 hidden-state grep 模板在另两家全仓库扫等价 pattern，命中即补入对比表；漏了的论断 fix 并在 log 中**显式写入**"重大补漏"条目 + Lessons learned |

> 价值排序：**用户提供 anchor 触发的 cross-check 是黄金反馈机制**——已知锚点提供精确 mental model，pattern matching 比 fresh search 高效得多。

---

## 7. Lint 工作流

**触发**：用户说 `lint`，或每完成一轮 ingest 后自动跑一次。

**检查项**：

| 检查 | 说明 |
|---|---|
| **死链** | wiki 内部链接（无论 markdown link 还是 source 链接）能否打开 |
| **孤立页** | wiki 内某页没有任何入链（在 index / overview / 其它页中均未被引用） |
| **stale** | `verified_against` 早于源码最近修改时间 → 标 `status: stale` |
| **contradiction** | 两个页面对同一事实给出不同描述 → 在双方加 `> [!warning] CONTRADICTION` |
| **缺 source** | 包含具体论断（行号 / 类名 / 数值 / 算法名）但没有 `## Sources` 锚点 |
| **缺 frontmatter** | 缺必填字段 |

输出格式：列表，每项 `<page>: <issue>`。修复由 LLM 给出 patch，由人确认后应用。

---

## 8. 跨项目对比规则

**comparison\topics\<topic>.md** 必须满足：

1. **三方对照表**：表头固定为 `维度 | MindIE | vLLM | SGLang`。
2. **每个 cell 必须指向具体页面**：`[实现](vllm\topics\xxx.md)` 而不是泛泛而谈。若某项目 wiki 子树已删除，该 cell 写 `N/A (wiki removed <日期>)` 并尽量保留源码绝对路径锚点。
3. **缺失项明示**：未实现 / 未验证写 `N/A (verified <日期>)`，禁止臆测"应该有"。
4. **维度参考** `comparison\dimensions.md`，新增维度先去 dimensions.md 注册。
5. **不做主观判断**：不写"X 比 Y 好"，只写"X 用 A 方案，Y 用 B 方案"，价值判断留给读者。
6. **Anchor-driven cross-check（写完后必跑的自检 pattern）**：每写完 / 修订完一个 comparison 页后，执行"**anchor → 三方反向扫**"：
   - 任选 1-2 个本页已确认的具体实现锚点（如 MindIE `PluginManager.forward_thread`），用 Grep 在**另两家全仓库**扫该锚点的"等价语义" pattern（见 §9 hidden-state grep 模板）。
   - 任何"一家有、另两家未提及"的命中都应**显式补入对比表**——要么是真等价物（补充论断），要么是 N/A（明示并写 verified 日期）。
   - 这种 pattern matching 比 fresh search 高效得多，因为已知 anchor 提供了精确的 mental model（pattern matching 找等价物 ≫ fresh exploration）。
   - **触发方式**：(a) 自驱——agent 写完页面前主动跑；(b) 用户驱动——用户说 `verify <page>` 或贴一个 file:line 锚点说"另两家应该也有"。两种都按本规则处理。

---

## 9. 代码探索的最佳实践（写给 LLM agent 自己看）

- 用 Glob 找文件，用 Grep 找符号，用 Read 读文件，**不要用** `cat / head / tail / sed / awk / find / grep` 这些 shell 命令。
- 大文件（> 1k 行）先用 Grep 找关键符号定位，再 Read 局部，避免无脑全读。
- SemanticSearch 用于"我不知道实现在哪里"的探索性问题；已知具体符号 / 文件名时直接用 Grep / Read。
- 探索三个项目时优先并行 Task subagent。
- 写 ingest 草稿前先列待读文件清单，避免漫游。
- **跨项目对比时，只看入口 / 调用链不够，要扫"hidden state initialization"**——下面这些通常在 `__init__` 里 spawn，从 `forward()` / `step()` 调用链上看不到，但实际承担一半的并行：

  | 类别 | grep 模板（跨三家全仓库扫） | 典型 hidden state |
  |---|---|---|
  | 后台线程 | `Thread\(target` / `threading.Thread` / `CoreThread\(target` / `mp.Process` | forward thread / output copy thread / monitor thread |
  | 进程级队列 | `queue.Queue` / `asyncio.Queue` / `mp.Queue` / `deque\(` | input_queue / output_queue / result_queue / async_output_queue |
  | CUDA / NPU stream | `cuda\.Stream\(` / `npu\.Stream\(` / `cuda\.Event\(` / `npu\.Event\(` / `record_stream` | async_output_copy_stream / forward_stream / schedule_stream |
  | 跨进程 IPC | `zmq\.PULL` / `zmq\.PUSH` / `shm` / `MessageQueue` / `SHM` | input/output shm queues / ZMQ DEALER/ROUTER |
  | callback 注册 | `callback` / `responseHandler` / `register_hook` | C++ async callback / Python event listeners |

  **判断哪个写进对比页**：扫到的命中只要在三家**至少 1 家是关键并行来源**，就值得在对比页里建一行——即使另两家是 N/A。N/A 本身就是有价值的对比信息（区分"没做" vs "用别的方式做了"）。

- **真异步 vs 逻辑异步的精确判定**：调用链表面上"异步"（如 `non_block=True` / `submit()` / `future`）不代表底层真有并发，需要追到：
  - 是不是真起了独立线程 / 进程？（grep `Thread\(target` 等）
  - 还是只是用 future 做语法糖、`.result()` 立即 await？（grep `future\.result\(\)` / `await ... future`）
  - 还是靠 CUDA stream 隐式异步？（grep `record_stream` / `wait_stream` / `Stream\(`）
  - 三家可能用完全不同的范式实现"看起来一样的"async 行为——必须各家分别判定，**不能拿一家结论套另一家**。

- **Hidden cross-reference grep 模板**（与上方 hidden state 表是姊妹规则；在 ingest entity 时**必跑**，详 [§5 ingest 工作流第 3 步](#5-ingest-工作流)）——hidden state 找"实体自己藏在 init 里的并发"，hidden cross-reference 找"实体在另一个完全不相关的模块/语言里的协作伙伴"：

  | 类别 | grep 模板（**全仓库**扫，跨语言、跨子系统） | 防漏的典型 case |
  |---|---|---|
  | 跨语言绑定 | C++ entity → grep 类名在对应 Python 树（`pybind` / `ctypes` / `cffi` / `Cython`）；反之亦然 | C++ `LlmEngine` 是否经 Manager 间接暴露给 Python；Python `MooncakeMempool` 是否被 C++ 通过 capsule 调 |
  | 协作伙伴跨子系统引用 | 列出 entity 已知 3-5 个协作类名，**每个名字** 在全仓库 grep | `BlockSpaceManager` 与 `Mooncake`/`LLMDataDist` 之间是否在 connector / server / examples 模块有间接引用 |
  | 配置 / IPC 共享数据结构 | grep entity 读的配置字段名（如 `BlockManagerConfig` 字段） | 远端模块通过共享 protobuf / yaml 隐式协作 |
  | 测试覆盖反查 | grep `tests/` 下对该 entity 的 unit + integration 测试 | 测试代码反推 prod 没有但实际存在的调用模式 |
  | doc / config 反查 | grep entity 名在 `docs/` / `config/` / `*.yaml` / `*.json` | 部署配置/产品文档里"未在代码里直接出现"的协作 |

  **N/A 输出格式**：必须写"`<key>`: 在 `<已 grep 范围>` 全树 grep 0 命中"，不能只写"未发现"——区分"真没有"vs"我没找"。

---

## 10. 已确立的目录映射（speedy reference）

| 项目 | 主代码根 | 顶层模块清单（用作 index 分组） |
|---|---|---|
| MindIE-LLM | [d:\design\MindIE-LLM\mindie_llm\](d:\design\MindIE-LLM\mindie_llm) | connector / distributed / examples / modeling / model_wrapper / runtime / server / text_generator / tokenizer / utils（**注意**：`wiki/mindie/` 子树已于 2026-08-10 删除；ingest MindIE 前需先恢复目录骨架） |
| vLLM | [d:\design\vllm\vllm\](d:\design\vllm\vllm) | 30 个，重点 v1 子树（attention/core/engine/executor/kv_offload/metrics/pool/sample/spec_decode/structured_output/worker） |
| SGLang | [d:\design\sglang\python\sglang\srt\](d:\design\sglang\python\sglang\srt) | managers / disaggregation / mem_cache / model_executor / distributed / dllm / elastic_ep / eplb / multiplex / speculative 等 36 个 |

> 这张表用于让 ingest 时不要重复 ls，但 **最终判断仍以实地读到的代码为准**。

> **代码版本 pin**：所有 wiki 页 `verified_against` 字段背后的精确 commit / branch 见 [source-versions.md](source-versions.md)。三仓 HEAD 移动后必须先看该文件再决定 ingest 还是 §12 增量。

---

## 11. Log rotation（log.md 滚动归档）

`log.md` 是 append-only 操作日志。当文件过大时，LLM agent 每次读 log 都要付出大量 token；grep 也变慢。本节定义滚动归档机制。

### 触发阈值

任一满足即触发：

- `log.md` **行数 > 1500**
- `log.md` **字节 > 110 KB**

agent 在以下时机检查阈值并触发轮转：

- `lint` 命令时（必查）
- `ingest` / `verify` / `compare` 后 append log entry 时（按 §5 step 7）
- 用户主动说 `archive log` 或 `rotate log`

### 归档目录约定

- **位置**：`d:\design\wiki\log-archive\`
- **命名**：`log-NNN.md`（从 001 起，3 位数字 zero-padded，按时间顺序递增）
- **不可重写**：每个 `log-NNN.md` 一旦写入即 append-only / 锁死，**不允许后续修改**（除非校对错别字）。新 entries 只能 append 到主 `log.md`。

### 保留策略（主 log.md 在轮转后保留什么）

主 `log.md` 在轮转后保留：

1. **schema header**（前 ~6 行：标题 + Append-only 提示）
2. **`## Archive Index` 段**：表格列出所有 `log-NNN.md` 的期间 / entries 数 / 大小 / 主要内容摘要
3. **最近 ~10 条 entries**（涵盖 ingest/verify/compare/lint，足够新 agent 快速建立"最近做了什么"的 context）
4. **本次 archive 操作 entry**（即 `## [YYYY-MM-DD] archive | wiki | log rotation NNN/...`，包括轮转前后行数 / 字节 / entries 净变化）

### 链接修复

归档文件的相对路径需从 `log-archive/` 出发（即 `../vllm/...` / `../comparison/...`）。绝对源码路径（`d:\design\<...>`）不变。历史归档中指向已删 `../mindie/...` 的链接保留不改（archive 不可重写）。

### 检索归档内容

未来 agent 检索归档：

- **Grep**：直接对 `wiki/log-archive/log-NNN.md` 跑 grep。注意 entries 跨文件分布，可能需要遍历多个 archive。
- **Read**：从 Archive Index 表的 "主要内容" 列定位 → Read 对应文件。

### 与 §5 step 7 的协同

§5 ingest 工作流第 7 步 append log 后 **必须** 检查阈值；超阈值即触发本节定义的轮转流程。轮转操作本身也作为一条 entry 写到主 log.md（见上 "保留策略" 第 4 条）。

### 归档示例（首次轮转 2026-04-18）

- 触发时：`log.md` = 1971 行 / 143 KB / 44 entries
- 归档 → `log-001.md`（21 entries / ~70 KB，2026-04-17 全部）+ `log-002.md`（17 entries / ~110 KB，2026-04-18 早段）
- 轮转后：主 `log.md` = ~600 行 / ~85 KB / 6 + 1 archive entry = 7 entries

详见 [log.md](log.md) 中 archive entry 与 [log-archive/](log-archive) 目录。

---

## 12. Incremental update workflow（上游代码增量分析）

wiki 是上游三仓代码的派生物。当上游 HEAD 移动后，**不要 fresh re-ingest** —— ROI 太低；改用增量工作流。

### 触发

- 用户说 `increment` / `update` / `三仓 HEAD 动了` / 或主动询问"代码更新了，wiki 还准吗"
- agent 启动时若发现 `git rev-parse HEAD` 与 [source-versions.md §Current pins](source-versions.md) 不一致

### 单一真相源

**[source-versions.md](source-versions.md)** —— 记录三仓当前 `branch + commit + last commit date + last commit msg` 的 pin 表。所有 wiki 页 frontmatter `verified_against: YYYY-MM-DD` 字段语义指向 **该日期下 source-versions.md 中记录的 commit**。

### 工作流（5 步）

1. **diff 摸底（每仓分别跑）**

   ```powershell
   cd d:\design\<repo>
   git fetch  # 如远端有新提交
   git log --oneline <pinned_commit>..HEAD  # 查看新 commit 列表
   git log --stat <pinned_commit>..HEAD -- <相关子树>  # 查看具体文件 diff stat
   git diff <pinned_commit>..HEAD --stat | head -50  # 全局 diff stat 摸底
   ```

   关注的"相关子树" = 该仓在 wiki 中已 ingest 的目录（详 §10 表）。

2. **影响面分析**

   对 step 1 列出的每个 changed file（例如 `vllm/vllm/v1/spec_decode/eagle.py`）：

   ```
   Grep 在 d:\design\wiki\ 内搜：
     - 文件名（不含路径）：`eagle.py`
     - 主类名（已知的）：`EagleProposer` / `EagleSpeculator`
     - 关键方法名（如有）
   ```

   命中文件 = 受影响 wiki 页清单。注意 `sources:` frontmatter 字段也要 grep（绝对路径或相对路径）。

3. **逐页处理**

   按改动量分层：

   | 改动类型 | 处理 |
   |---|---|
   | 仅行号偏移（refactor 移动 / format） | `verify <page>` 重新核对所有 `[file:line]` 锚点；不改正文 |
   | 内部细节变化（新参数 / 新分支 / 错误处理） | `verify <page>` + 对应小节 inline 增补 anchor |
   | 大改（新类 / 新文件 / 删类） | 该子树重新跑一次 `ingest` 增量段（不必整页重写，新增章节追加在 §Notes 之前；旧章节如已失效划线 + RESOLVED） |
   | 跨多页 / 跨子系统 | 启 multi-subagent 并行 verify（参考 entity P1 / topic P1 batch pattern） |

4. **更新 frontmatter**

   - 受影响页：`verified_against: <new_pin_date>` + 可选追加 `(verify pass: <YYYY-MM-DD>, increment from <old commit short>)` 注释
   - 未受影响页：**保持原 verified_against 不变**（说明该子树未变，旧 pin 仍准确）

5. **更新 source-versions.md + log**

   - **`source-versions.md` §Current pins**：直接覆盖（mutable 区域）
   - **`source-versions.md` §History**：append 一行 `| YYYY-MM-DD | increment | <new MindIE> | <new vLLM> | <new SGLang> | <action 摘要> |`
   - **`log.md`**：append 一条 `## [YYYY-MM-DD] increment | cross | <repos> updated to <new commits> | <N pages re-verified, M pages re-ingested>`

### 增量分析的"重大变更"启发式

下面这类 commit message / 路径变化在 step 1 时要 **特别关注**（往往触发 large rewrite 而非小修）：

| 信号 | 通常意味着 |
|---|---|
| `refactor` / `move` / `rename` 大批文件 | 整个 wiki 子区域 anchor 全失效，可能要重新 ingest |
| 新增大目录 / 大文件（> 500 行） | 新功能点，可能要建新 entity 页 / topic 页 |
| `BREAKING` / `remove` / `deprecate` | 已 ingest 的类 / 函数可能消失，wiki 需删除或标 deprecated |
| `merge dev into master` 类聚合 commit | 摸底时要 `git log master..dev` 看真实 changeset，单条 commit 信息无信息量 |

### 反向使用：用 wiki 帮上游 review

**对 PD 优化等长期主题特别有用**：每次拉新代码后跑 step 1-2，看哪些 changed file 落在 `comparison/topics/pd-disaggregation.md` 等关键页的 sources 内 —— 这些 commit 就是要重点 review 的"对你的优化方向有影响"的上游变更。

### 与其它工作流的关系

| 工作流 | 关系 |
|---|---|
| §5 ingest | 增量首次发现新文件 / 新模块时回退到 ingest 流程 |
| §6 verify | 增量的主要执行方式：每个受影响页跑一次 verify |
| §7 lint | 增量完成后跑一次 lint，确认 frontmatter 同步、死链清零 |
| §8 anchor-driven cross-check | 大变更影响 comparison 页时必跑 |
| §11 log rotation | 增量也是 log entries 的来源，正常计入轮转阈值 |
