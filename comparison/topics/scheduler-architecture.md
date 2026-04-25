---
type: comparison
project: cross
status: draft
confidence: high
verified_against: 2026-04-19
sources:
  - d:\design\sglang\python\sglang\srt\managers\scheduler.py
  - d:\design\sglang\python\sglang\srt\managers\scheduler_output_processor_mixin.py
  - d:\design\sglang\python\sglang\srt\managers\scheduler_update_weights_mixin.py
  - d:\design\sglang\python\sglang\srt\managers\scheduler_profiler_mixin.py
  - d:\design\sglang\python\sglang\srt\managers\scheduler_runtime_checker_mixin.py
  - d:\design\sglang\python\sglang\srt\managers\scheduler_pp_mixin.py
  - d:\design\sglang\python\sglang\srt\managers\scheduler_dp_attn_mixin.py
  - d:\design\sglang\python\sglang\srt\observability\scheduler_metrics_mixin.py
  - d:\design\sglang\python\sglang\srt\disaggregation\decode.py
  - d:\design\sglang\python\sglang\srt\disaggregation\prefill.py
  - d:\design\sglang\python\sglang\srt\multiplex\multiplexing_mixin.py
  - d:\design\sglang\python\sglang\srt\dllm\mixin\scheduler.py
  - d:\design\vllm\vllm\v1\engine\core.py
  - d:\design\vllm\vllm\v1\engine\core_client.py
  - d:\design\vllm\vllm\v1\engine\async_llm.py
  - d:\design\vllm\vllm\v1\engine\llm_engine.py
  - d:\design\vllm\vllm\v1\core\sched\scheduler.py
  - d:\design\vllm\vllm\v1\core\sched\async_scheduler.py
  - d:\design\vllm\vllm\v1\core\sched\interface.py
  - d:\design\vllm\vllm\v1\worker\gpu_model_runner.py
  - d:\design\MindIE-LLM\src\engine\llm_engine.h
  - d:\design\MindIE-LLM\src\engine\llm_engine.cpp
  - d:\design\MindIE-LLM\src\include\engine\illm_engine.h
  - d:\design\MindIE-LLM\src\include\scheduler\ischeduler.h
  - d:\design\MindIE-LLM\src\scheduler\scheduler.h
  - d:\design\MindIE-LLM\src\scheduler\policy
  - d:\design\MindIE-LLM\src\engine\model_exec_output_handler.cpp
  - d:\design\MindIE-LLM\src\llm_manager_v2\impl\llm_manager_impl.cpp
  - d:\design\MindIE-LLM\mindie_llm\text_generator\generator.py
  - d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py
related:
  - comparison/index.md
  - comparison/dimensions.md
  - comparison/topics/scheduler.md
  - comparison/topics/engine-architecture.md
  - comparison/topics/executor-worker.md
  - comparison/topics/sync-schedule.md
  - comparison/topics/async-schedule.md
  - sglang/topics/scheduler-mixins.md
  - sglang/entities/Scheduler.md
  - vllm/entities/Scheduler.md
  - vllm/entities/EngineCore.md
  - vllm/entities/EngineCoreClient.md
  - mindie/entities/BatchScheduler.md
  - mindie/entities/Generator.md
  - mindie/entities/LlmEngine.md
  - mindie/entities/PluginManager.md
---

# Cross-project Comparison: Scheduler 架构（架构分解视角）

## Summary

三家 Scheduler 的**架构分解形态**正交于调度策略本身：

- **MindIE** — **C++ `BatchScheduler` (`mindie_llm::Scheduler`) + Python `Generator` + `PluginManager`** 跨语言双层结构。C++ 侧 `LlmEngine::SchedulerThreadEntry` 是单一 8 步主循环（[llm_engine.cpp:457-696](d:\design\MindIE-LLM\src\engine\llm_engine.cpp)），Python 侧由 `responseHandler` callback 反向触发；模块化通过 **C++ Policy 类层次（15 .h 策略文件）+ Python plugin_list（splitfuse / prefix_cache / mtp / la / memory_decoding / structured_output 等）** 双轨实现。
- **vLLM** — **`EngineCore` (单类) + `Scheduler` (单类，含 `AsyncScheduler` 子类) + 6 客户端工厂分支**。Scheduler 自身**完全没有 mixin**——所有逻辑写进 `Scheduler.schedule()` 一个 ~600 行函数 + `update_from_output` 等少数大方法（[scheduler.py:67-296, 348-955](d:\design\vllm\vllm\v1\core\sched\scheduler.py)）；可选 feature 通过 `__init__` 内 `if` 分支与策略字段（`KVConnectorBase_V1`、`StructuredOutputManager`、`KVCacheManager` 等）注入。
- **SGLang** — **`Scheduler` 单类 + 11 mixin 多继承拼装**（[scheduler.py:317-329](d:\design\sglang\python\sglang\srt\managers\scheduler.py)），其中 6 个 internal（`srt/managers/`）+ 5 个 external（`disaggregation/`/`multiplex/`/`dllm/`/`observability/`）；主循环不是单一 step，而是 [`dispatch_event_loop`](d:\design\sglang\python\sglang\srt\managers\scheduler.py)（[L3628-3654](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）按 `disaggregation_mode × pp_size × enable_pdmux × enable_overlap` 分派到 **8 个独立 `event_loop_*` 方法**。

`synthesis:` 三家把"调度器复杂度"放在不同维度：**MindIE 用语言边界 + Policy 类层次**；**vLLM 用单类巨函数 + 配置字段开关**；**SGLang 用 mixin 多继承 + 多 event-loop 分派**。选择反映项目对"模块化粒度 vs 跳转友好 vs 跨语言迭代速度"的优先级。

## Sources

见 frontmatter `sources:`（覆盖 30 个核心源文件 + 17 个相关 wiki 页）。所有具体论断在正文都带 `[file:line]` 锚点。

## §0 区分本页 vs `topics/scheduler.md`（必读 prelude）

本页与 [`comparison/topics/scheduler.md`](scheduler.md) 互补，**不重复**：

| 视角 | 页面 | 范围 |
|---|---|---|
| **架构分解**（本页） | `topics/scheduler-architecture.md` | 单类 vs mixin 拼装 vs Manager+Generator 跨语言切分；模块化机制；可选 feature 加载方式；新功能添加成本 |
| **调度策略**（兄弟页） | [`topics/scheduler.md`](scheduler.md) | FCFS / Priority / Stage policy；waiting/running/swapped 队列；抢占（SWAP/RECOMPUTE）；CPU/GPU overlap 机制；Pause 状态机 |

**简单判定**：想问"为什么 SGLang 把 PD prefill 写在 `disaggregation/prefill.py:355` 而不是 `scheduler.py`？" → **本页**；想问"SGLang 抢占是 FCFS 还是 PRIORITY？" → [scheduler.md](scheduler.md)。`synthesis:` 两页之所以分开，是因为"架构分解"对 PD 二次开发的影响大于"调度策略"——前者决定**改动半径**与**回归测试粒度**，后者决定**线上 latency/throughput 形态**。

## §1 顶层架构分解（top-level decomposition）

| 维度 | MindIE | vLLM | SGLang |
|---|---|---|---|
| **Scheduler 主体类** | C++ `mindie_llm::Scheduler` ([scheduler.h:61-306](d:\design\MindIE-LLM\src\scheduler\scheduler.h)) | Python `Scheduler` ([scheduler.py:67-296](d:\design\vllm\vllm\v1\core\sched\scheduler.py), ~2270 行) | Python `Scheduler` ([scheduler.py:317-329](d:\design\sglang\python\sglang\srt\managers\scheduler.py), ~3700 行 + 11 mixin) |
| **承载主循环的对象** | C++ `LlmEngine::SchedulerThreadEntry` ([llm_engine.cpp:457-696](d:\design\MindIE-LLM\src\engine\llm_engine.cpp), 每 DP rank 一 `std::thread`) | `EngineCore.run_busy_loop` ([core.py:1160-1168](d:\design\vllm\vllm\v1\engine\core.py)) + `step_fn = step` or `step_with_batch_queue` ([core.py:212-214, 404-433, 445-561](d:\design\vllm\vllm\v1\engine\core.py)) | `Scheduler.event_loop_*` 8 个方法，由 `dispatch_event_loop` ([scheduler.py:3628-3654](d:\design\sglang\python\sglang\srt\managers\scheduler.py)) 路由 |
| **Engine ↔ Scheduler 关系** | **跨语言双层**：C++ `LlmEngine` 直接 `Schedule(needSync)` ([llm_engine.cpp:536](d:\design\MindIE-LLM\src\engine\llm_engine.cpp))；Python `Generator` 被 C++ 反向 callback | **同进程同对象层**：`EngineCore.scheduler` ([core.py:130-152](d:\design\vllm\vllm\v1\engine\core.py))；`EngineCore.step()` 直接 `self.scheduler.schedule()` | **Engine 与 Scheduler 异进程**：`Engine` 是 launcher，fork 后退出 ([engine.py:627-678](d:\design\sglang\python\sglang\srt\entrypoints\engine.py))；Scheduler 子进程**自启** event loop，主进程只 ZMQ 投递（详 [engine-architecture.md §1](engine-architecture.md)） |
| **Scheduler 文件总行数** | C++ scheduler.h ~320 + scheduler.cpp 数千 + 15 policy .h | scheduler.py ~2270 + async_scheduler.py 61 + interface.py 244 + request_queue.py 209 | scheduler.py ~3700 + 11 mixin (合计 ~10K) |

`synthesis:` 三家都把"主循环"与"调度算法"拆开 — MindIE 沿语言边界拆（C++ engine 包 C++ scheduler）；vLLM 沿对象层级拆（`EngineCore` 包 `Scheduler`）；SGLang 沿进程边界拆（Engine 进程不持 scheduler，scheduler 子进程内自闭环）。**没有任何一家把 engine 主循环和调度算法写在同一个类**——这是普遍设计共识。

---

## §2 Mixin / 模块化形态

| 维度 | MindIE | vLLM | SGLang |
|---|---|---|---|
| **Scheduler 上的 mixin 数量** | **0**（C++ 用 Policy 多态，非 mixin） | **0**（`Scheduler` 直接继承 `SchedulerInterface`；详 [§7 Anchor 1](#§7-anchor-driven-cross-check)） | **11**（5 external + 6 internal，[scheduler.py:317-329](d:\design\sglang\python\sglang\srt\managers\scheduler.py)） |
| **模块化机制** | C++ Policy 多态：`prefillPolicy_` / `decodePolicy_` / `stagePolicy_` / `dynamicBatchSize_` / `transferPolicy_` 5 字段（[scheduler.h:249-259](d:\design\MindIE-LLM\src\scheduler\scheduler.h)）+ Python `plugin_list` 遍历（[plugin_manager.py:74-85](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py)） | 单类 + 字段注入：`kv_cache_manager` / `encoder_cache_manager` / `structured_output_manager` / `connector` / `ec_connector` 作为 `__init__` 字段（[scheduler.py:78-152](d:\design\vllm\vllm\v1\core\sched\scheduler.py)） | Python 多继承 mixin：每 mixin 提供一组方法（`process_batch_result_decode` / `event_loop_overlap_disagg_decode`），全成 `self.xxx()` |
| **Mixin 文件物理位置** | N/A（C++ Policy 在 [src/scheduler/policy/](d:\design\MindIE-LLM\src\scheduler\policy)，15 .h） | N/A（Scheduler 层 0；Worker 层有 3 mixin 给 `GPUModelRunner`，[gpu_model_runner.py:394-396](d:\design\vllm\vllm\v1\worker\gpu_model_runner.py)） | **Internal 6**（`srt/managers/scheduler_*_mixin.py` 6 文件）+ **External 5**（`observability/`、`disaggregation/decode.py:1171`、`disaggregation/prefill.py:355`、`multiplex/multiplexing_mixin.py`、`dllm/mixin/scheduler.py`），全表见 [scheduler-mixins.md §Sources](../../sglang/topics/scheduler-mixins.md) |
| **可选 feature 加载方式** | C++ Policy 工厂 [`policy_factory.h`](d:\design\MindIE-LLM\src\scheduler\policy\policy_factory.h) 按 `policyType` 创建（`SchedulerConfig` [config_info.h:399-517](d:\design\MindIE-LLM\src\include\config\config_info.h)） | **`__init__` if 字段**：`if vllm_config.kv_transfer_config: self.connector = ...`（[scheduler.py:120 上下](d:\design\vllm\vllm\v1\core\sched\scheduler.py)）；`if observability_config.kv_cache_metrics: ...`（[scheduler.py:87-91](d:\design\vllm\vllm\v1\core\sched\scheduler.py)） | mixin 永存；4 `init_*`（`init_metrics` / `init_pdmux` / `init_diffusion_llm` / `init_profiler`）在 `__init__` 按 flag 显式调（[scheduler.py:406, 411-413, 440, 449](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）；event_loop 由 `dispatch_event_loop` 决定 |

`synthesis:` 三家"模块化"语义不同：**MindIE = OOP Strategy + plugin 列表**；**vLLM = `Scheduler` 是上下文对象，feature 通过 `Optional[X] | None` 字段挂载（None 即关）**；**SGLang = mixin 多继承（方法常存，靠显式 init 与 dispatch 激活）**。vLLM 最简单但单类膨胀；SGLang 最重构友好（外部子系统"自带 scheduler 钩子"，详 [scheduler-mixins.md §为什么这样切](../../sglang/topics/scheduler-mixins.md)）；MindIE C++ 层最严格、Python plugin 层最灵活。

---

## §3 跨语言绑定（C++ ↔ Python boundary）

| 维度 | MindIE | vLLM | SGLang |
|---|---|---|---|
| **Scheduler 实现语言** | C++ (`mindie_llm::Scheduler`) | Python | Python |
| **C++ ↔ Python 绑定方向** | **双向**：Python 经 LlmManager 启动 C++ `LlmEngine` ([llm_manager_impl.cpp:1219-1250](d:\design\MindIE-LLM\src\llm_manager_v2\impl\llm_manager_impl.cpp))；C++ scheduler 通过 `responseHandler` lambda + `modelExecutor->AsyncExecuteModel` 反向触发 Python ([llm_engine.cpp:575-583, 613-620](d:\design\MindIE-LLM\src\engine\llm_engine.cpp)) | 无 C++ scheduler（CUDA 仅 kernel 层） | 无 C++ scheduler（CUDA 仅 kernel 层） |
| **纯 Python 调试 schedule** | ❌ 改动需 rebuild + pybind | ✅ 可热打补丁 / pdb | ✅ 可热打补丁 / pdb |
| **GIL 与 callback** | C++ scheduler 主线程不持 GIL；`responseHandler` 触发 Python forward 取 GIL，`PluginManager.forward_thread` 异步消费解耦 ([plugin_manager.py:107-115](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py)) | 全 Python，单进程 GIL 串行；`AsyncScheduler` 用 placeholder + `non_block=True` 缓解 ([scheduler.py:415-422](d:\design\vllm\vllm\v1\engine\core.py)) | 全 Python；scheduler 与 worker 同进程，与 detokenizer/tokenizer 异进程靠 ZMQ 解耦 |
| **跨语言绑定 grep 验证** | `mindie_llm/` Python 树 grep `class Scheduler` / `from .* import Scheduler`：**0 命中**（仅 example/test 引用）；Python 永远不直接持 C++ Scheduler 对象，只持 `LlmManager` / `Generator` 接口 | csrc/ 全树 grep `class Scheduler`：**0 命中** | sgl-kernel/ 全树 grep `class Scheduler`：**0 命中** |

`synthesis:` MindIE 是三家中**唯一在 scheduler 层有 C++/Python 紧耦合**的——好处是 C++ scheduler 主循环避免 GIL；代价是 schedule 算法改动需 C++ rebuild + pybind 维护，迭代速度上限低于 vLLM/SGLang（详 [engine-architecture.md §13.3](engine-architecture.md) "trade-off 而非单边优劣"）。

---

## §4 主循环 / 多 mode dispatch

| 维度 | MindIE | vLLM | SGLang |
|---|---|---|---|
| **主循环入口** | C++ [`SchedulerThreadEntry`](d:\design\MindIE-LLM\src\engine\llm_engine.cpp)（[L457-696](d:\design\MindIE-LLM\src\engine\llm_engine.cpp)）单一 8 步 while 循环（详 [LlmEngine.md §主循环全貌](../../mindie/entities/LlmEngine.md)） | Python [`EngineCore.run_busy_loop`](d:\design\vllm\vllm\v1\engine\core.py)（[L1160-1168](d:\design\vllm\vllm\v1\engine\core.py)）调 `step_fn` | Python [`dispatch_event_loop`](d:\design\sglang\python\sglang\srt\managers\scheduler.py)（[L3628-3654](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）按 mode 分派 |
| **Mode 数量（运行时分支）** | **1 主循环 + 内部分支**：`role_ ∈ {P,D,Flex,PnD}` × `layerwiseDisaggregated` × `distributedEnable` × dummy/real；分支散落 8 步内（如 `ScheduleExecTransfer` 仅 P/D 跑，[llm_engine.cpp:485-488, 698-737](d:\design\MindIE-LLM\src\engine\llm_engine.cpp)） | **2 种 step_fn**：`step` (普通) vs `step_with_batch_queue` (PP 流水)，由 `max_concurrent_batches > 1` 决定（[core.py:187-193, 212-214](d:\design\vllm\vllm\v1\engine\core.py)） | **8 路独立 event_loop**：`event_loop_normal` / `_overlap` / `_pdmux` / `_pp` / `_normal_disagg_prefill` / `_overlap_disagg_prefill` / `_normal_disagg_decode` / `_overlap_disagg_decode` / `_pp_disagg_prefill` / `_pp_disagg_decode`（共 10 方法，dispatch 表 8 路命中）|
| **分派语义在哪定义** | 散落 [llm_engine.cpp](d:\design\MindIE-LLM\src\engine\llm_engine.cpp) 多处 if/else（[L219, 232, 254, 262, 307, 461, 627-629](d:\design\MindIE-LLM\src\engine\llm_engine.cpp)） | `EngineCore.__init__` 末尾一次性绑定 `step_fn` ([core.py:212-214](d:\design\vllm\vllm\v1\engine\core.py))，运行时不切 | 集中在 `dispatch_event_loop`，按 `disaggregation_mode × pp_size × enable_pdmux × enable_overlap` 4 维 truth table 路由 |
| **重复代码问题** | C++ 8 步内部 if/else **不重复** 但单函数 240 行 | `step` 与 `step_with_batch_queue` 部分重复（schedule + grammar_bitmask） | **重复显著**：`event_loop_pp` 复刻 `event_loop_normal` 全 step（[scheduler_pp_mixin.py:79-145](d:\design\sglang\python\sglang\srt\managers\scheduler_pp_mixin.py)）；`event_loop_pdmux` 同样复刻（[multiplex/multiplexing_mixin.py:96-119](d:\design\sglang\python\sglang\srt\multiplex\multiplexing_mixin.py)）；详 [scheduler-mixins.md Chain 5](../../sglang/topics/scheduler-mixins.md) |

`synthesis:` 三家三种风格 — **MindIE 巨函数 + 内部分支**（240 行单点断点全可见）；**vLLM 二选一 step_fn**（一次性绑定，无运行时分支开销）；**SGLang 8 独立 event_loop**（独立性高但重复显著）。SGLang 的代价是 PP/PDMux event_loop 与 normal event_loop 同步演进的维护负担。

---

## §5 冗余处理（process_batch_result）：单点 vs 多点 dispatch

| 维度 | MindIE | vLLM | SGLang |
|---|---|---|---|
| **结果回写入口** | C++ `responseHandler` lambda ([llm_engine.cpp:575-583](d:\design\MindIE-LLM\src\engine\llm_engine.cpp)) → `ModelExecOutputHandler.Entry4Executor` ([model_exec_output_handler.cpp:80+](d:\design\MindIE-LLM\src\engine\model_exec_output_handler.cpp)) | `Scheduler.update_from_output(scheduler_output, model_runner_output)` 单一入口 ([scheduler.py:1299-1562](d:\design\vllm\vllm\v1\core\sched\scheduler.py)) | `Scheduler.process_batch_result(batch, result)` 单一入口但**内部 6-way 分派** ([scheduler.py:2913-2930](d:\design\sglang\python\sglang\srt\managers\scheduler.py)) |
| **多类型分派点数量** | C++ `Entry4Executor` 内按 `outputs_size()==0` (dummy) vs 正常 + `LwdProcessResponse` 分支 ([model_exec_output_handler.cpp:86-88, 160](d:\design\MindIE-LLM\src\engine\model_exec_output_handler.cpp))；Python 端 plugin chain 二次处理 | **1 path**：`update_from_output` 顺序处理（accum tokens → finished → connector → ec_connector），spec decode / chunked prefill 都在内部 if/else | **6-way mixin 分派**：`is_decode → process_batch_result_decode` (Output mixin) / `is_extend & is_dllm → process_batch_result_dllm` (Dllm mixin) / `is_extend & PREFILL → process_batch_result_disagg_prefill` (DisaggPrefill mixin) / `is_extend & else → process_batch_result_prefill` (Output) / `is_prebuilt` (Output) / `is_idle` (Output) — 共 **3 个 mixin 共享分派点** |
| **本质区别** | "**结果分发**"：plugin 链顺序处理，每 plugin 自决 | "**单方法多 if/else**"：所有 result 类型共享一函数 | "**显式多态 dispatch**"：3 mixin 共享同一分派表（Output 占 4/6 槽位，Dllm + DisaggPrefill 各 1）— 详 [scheduler-mixins.md Chain 2](../../sglang/topics/scheduler-mixins.md) |
| **添加新 forward_mode 成本** | C++ 加 enum + Entry4Executor 分支 + 可能 rebuild | 加 if/else 到 `update_from_output`（单文件） | 加 mixin 方法 + `process_batch_result` 加 elif（双文件） |

`synthesis:` SGLang 6-way dispatch 是 mixin 模式**最关键的协同点** — 3 mixin 通过共享分派表"互不感知地共存"。vLLM 单方法路径反映"PD 分离用外挂 connector 而非 mixin"（详 [engine-architecture.md §10](engine-architecture.md)）；MindIE 把该职责放 Python plugin 链，灵活但调试需追 plugin_list 顺序。

---

## §6 可选 feature 加载机制 / 跨子系统循环依赖回避

| 维度 | MindIE | vLLM | SGLang |
|---|---|---|---|
| **可选 feature 典型清单** | PD / layerwiseDisaggregated / activateAsyncInference / dynamicBatchSize / Lora / DistributedEnable | KVConnector (PD) / structured_output / kv_cache_metrics / encoder_cache / KV events | PD prefill / PD decode / PDMux / DLLM / Metrics / PP / DP-attn |
| **加载机制** | C++ Policy factory 按 `SchedulerConfig` 字段 + Python `plugin_list` 字符串（`Generator.__init__` 解析 [generator.py:285-301](d:\design\MindIE-LLM\mindie_llm\text_generator\generator.py) → [`get_plugin(name)`](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\__init__.py) 查表） | `Scheduler.__init__` 内 `if config.X: self.x_manager = XManager(...)`，关闭时字段 `None` | Mixin 方法**永存**；`init_*` 按 flag 显式调；event_loop 由 `dispatch_event_loop` 路由 |
| **禁用 feature 的 dead code** | ❌（按需 instantiate） | ❌（字段 None，引用前判） | ⚠️ **是**（11 mixin 方法绑到 Scheduler MRO，未触发但占内存——代价微小，主要是认知负担） |
| **跨子系统循环依赖回避** | C++ 头文件依赖图 + namespace 隔离；Python plugin 字符串解耦 | 单类 `Scheduler` 直接 import backend 类，依赖单向 | **靠 `TYPE_CHECKING` + 单向 import**：`scheduler.py` import mixin（[scheduler.py:44-67, 195, 201](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）；mixin 用 `if TYPE_CHECKING: from .scheduler import Scheduler` 注解避免反向 import（例：[multiplex/multiplexing_mixin.py:25-27](d:\design\sglang\python\sglang\srt\multiplex\multiplexing_mixin.py)、[dllm/mixin/scheduler.py:16-17](d:\design\sglang\python\sglang\srt\dllm\mixin\scheduler.py)） |

`synthesis:` SGLang "external mixin 跟随子系统目录"让子系统作者改一处即可——`disaggregation/decode.py` 同含 `DecodePreallocQueue` / `DecodeTransferQueue` / `SchedulerDisaggregationDecodeMixin` ([disaggregation/decode.py:108-1334](d:\design\sglang\python\sglang\srt\disaggregation\decode.py))，新增 PD decode 不需碰 `srt/managers/scheduler.py`。vLLM "manager 字段"反向：添加 feature 必须改 `scheduler.py` 主类，但调用图扁平。MindIE plugin_list 最 runtime-dynamic（按字符串决定顺序），代价是静态分析失效。

---

## §7 Anchor-driven cross-check（按 [AGENTS.md §8 规则 6](../../AGENTS.md)）

### Anchor 1：SGLang `dispatch_event_loop` 8 mode（[scheduler.py:3628-3654](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）

> 锚点反映"调度主循环按 mode 多路分派"。另两家全仓库扫等价语义：

- **vLLM**：grep `def step` / `def event_loop` / `dispatch_event_loop` 在 [vllm/v1/engine/](d:\design\vllm\vllm\v1\engine) 全树：仅命中 `EngineCore.step`/`step_with_batch_queue` ([core.py:404, 445](d:\design\vllm\vllm\v1\engine\core.py))、`run_busy_loop` ([core.py:1160, 1727](d:\design\vllm\vllm\v1\engine\core.py))、`LLMEngine.step` ([llm_engine.py:288](d:\design\vllm\vllm\v1\engine\llm_engine.py))。等价物 = **2 step_fn × 1 run_busy_loop**（PP 与非 PP 二分）。grep `dispatch_event_loop`：**0 命中**（**verified 2026-04-19**）。`synthesis:` vLLM 启动时一次性绑定 step_fn ([core.py:212-214](d:\design\vllm\vllm\v1\engine\core.py))，mode 复杂度推入 `Scheduler.schedule()` if/else。
- **MindIE**：grep `SchedulerThreadEntry` / `event_loop` 在 [MindIE-LLM/src/engine/](d:\design\MindIE-LLM\src\engine) 全树：仅命中 `LlmEngine::SchedulerThreadEntry` ([llm_engine.cpp:457](d:\design\MindIE-LLM\src\engine\llm_engine.cpp))。**1 主循环**，mode 复杂度通过内部分支（`role_ != Role::P` / `layerwiseDisaggregated` / `isPauseScheduling_` / `DistDecodeAcquireDummyQuota`，散落 [llm_engine.cpp:219, 232, 254, 461, 473-478, 612-637](d:\design\MindIE-LLM\src\engine\llm_engine.cpp)）。grep `dispatch` 作为 mode router：**0 命中**（**verified 2026-04-19**）。

→ **结果**：SGLang 是三家中**唯一**把主循环按 mode 拆成多独立函数的；vLLM 仅 2 step_fn，MindIE 0 拆分。**显式补入 [§4 表格](#§4-主循环-多-mode-dispatch)**。

### Anchor 2：vLLM `EngineCore.step()` 单点（[core.py:404-433](d:\design\vllm\vllm\v1\engine\core.py)）

> 锚点反映"engine 主循环每帧只调一次 scheduler.schedule + executor.execute_model"。另两家全仓库扫等价语义：

- **MindIE**：`SchedulerThreadEntry` ([L457-696](d:\design\MindIE-LLM\src\engine\llm_engine.cpp)) 每轮 `Schedule(needSync)` ([L536](d:\design\MindIE-LLM\src\engine\llm_engine.cpp)) + `AsyncExecuteModel` ([L613-614](d:\design\MindIE-LLM\src\engine\llm_engine.cpp))。等价；但有 `GetAsyncBatchNum() >= asyncScheduleRound` 门控（[L494-501](d:\design\MindIE-LLM\src\engine\llm_engine.cpp)），相当于 vLLM `step_with_batch_queue` 的 batch_queue 满判断。**verified 2026-04-19**。
- **SGLang**：等价物**在 scheduler 子进程内**：每个 `event_loop_*` while 循环体调 `recv_requests` / `process_input_requests` / `get_next_batch_to_run` / `run_batch` / `process_batch_result` / `on_idle`（[scheduler.py:1384-1468](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）。**verified 2026-04-19**。`synthesis:` SGLang 没有"engine.step"等价物——主进程 `Engine` 与 scheduler 异进程，所有"engine driver"职责由 scheduler 子进程内 event_loop 自驱。

→ **结果**：三家都有"每帧一次 schedule + execute"语义，但触发位置不同（MindIE C++ engine 线程 / vLLM EngineCore 进程主线程 / SGLang scheduler 子进程主线程）。无 N/A。

---

## §8 测试粒度 / 热更新

| 维度 | MindIE | vLLM | SGLang |
|---|---|---|---|
| **scheduler 单元测试** | C++ [tests/dlt/ut/scheduler/test_scheduler.cpp](d:\design\MindIE-LLM\tests\dlt\ut\scheduler\test_scheduler.cpp)；集成 [tests/dlt/it/test_llm_engine.cpp](d:\design\MindIE-LLM\tests\dlt\it\test_llm_engine.cpp)（含 `AsyncBatchNumTest`） | Python `tests/v1/core/test_scheduler.py` 等 | Python `test/srt/test_*` 散落 |
| **mixin/policy 独立可测** | ✅ C++ Policy（gtest，构造仅需 `SchedulerConfig`） | N/A（无 mixin） | ⚠️ 理论可，实需完整 Scheduler fixture（mixin 方法签名 `def foo(self: Scheduler, ...)` 隐含该依赖） |
| **跨语言集成测试成本** | 高（C++ + Python 双 fixture） | 低（纯 pytest） | 低（纯 pytest） |
| **热替换 schedule 算法** | ❌ rebuild + 重启 | ⚠️ monkey-patch（`self.scheduler` 对象需 swap） | ⚠️ monkey-patch（mixin MRO 类创建时已绑） |
| **运行时 PD 角色切换** | ✅ `SwitchRole()` 一等公民 ([scheduler.h:112](d:\design\MindIE-LLM\src\scheduler\scheduler.h)、[generator.py:150-152](d:\design\MindIE-LLM\mindie_llm\text_generator\generator.py)) | ❌ `kv_role` 启动定 | ❌ `disaggregation_mode` 启动定 |
| **Live weight update** | ❌（待 ingest） | ✅ `Scheduler.reset_prefix_cache` + `EngineCore.profile/reset_*` ([core.py:580-624](d:\design\vllm\vllm\v1\engine\core.py)) | ✅ `SchedulerUpdateWeightsMixin.update_weights_from_{disk,distributed,tensor,ipc}` ([scheduler_update_weights_mixin.py:46-106](d:\design\sglang\python\sglang\srt\managers\scheduler_update_weights_mixin.py)) |

`synthesis:` MindIE C++ Policy 是真独立单元，测试粒度最细；SGLang mixin "独立性是名义上的"，源于 `def foo(self: Scheduler, ...)` 签名隐含完整 Scheduler 依赖（详 [scheduler-mixins.md "为什么用 mixin"](../../sglang/topics/scheduler-mixins.md)）。MindIE 在 **PD 角色切换 + Policy 替换**最强；vLLM/SGLang 在 **weight update** 接口最清晰。三家都**不**支持"运行时换 schedule 算法"——核心调度逻辑改动都要重启。

---

## §9 综合 cheat sheet

| 维度 | MindIE | vLLM | SGLang |
|---|---|---|---|
| Scheduler 实现语言 | C++ | Python | Python |
| 顶层 engine driver | C++ `LlmEngine::SchedulerThreadEntry` | Python `EngineCore.run_busy_loop` | scheduler 子进程内 `event_loop_*` |
| Mixin 数量 | 0 | 0 | **11** (5 ext + 6 int) |
| Policy / Strategy 类数量 | **15** policy .h | 1 enum (FCFS/PRIORITY) | N/A（mixin 替代） |
| 主循环数量 | 1 (8 步内部分支) | 2 (`step` / `step_with_batch_queue`) | **8** (dispatch_event_loop) |
| `process_batch_result` 分派 | C++ Entry4Executor + Python plugin chain | 1 path (`update_from_output`) | **6-way mixin dispatch**（3 mixin 共享） |
| 跨语言绑定 | C++ ↔ Python pybind 紧耦合 | 纯 Python | 纯 Python |
| 可选 feature 加载 | C++ Policy factory + Python plugin_list | `__init__` if 字段注入 | mixin 永存 + `init_*` 显式调 |
| PD 角色运行时切换 | ✅ `SwitchRole` | ❌ 启动定 | ❌ 启动定 |
| 新增 mode 改动文件数 | C++ scheduler + plugin (≥2) | scheduler.py 单文件 | 新 mixin + scheduler.py + dispatch_event_loop |
| 代码热更新 | ❌（C++ rebuild） | ⚠️ monkey-patch | ⚠️ mixin MRO 已绑 |
| scheduler 自身代码量 | ~320 .h + 数千 .cpp + 15 policy .h | ~2270 .py + 244 iface + 209 queue | ~3700 .py + 11 mixin (~10K 合计) |
| 测试粒度 | C++ Policy 独立可测 | scheduler.py 整体 fixture | mixin "理论独立、实需完整 fixture" |

---

## §10 与 PD 优化的关联（synthesis）

1. **MindIE `SwitchRole` 一等公民可借鉴**：MindIE 在 `IScheduler` ([ischeduler.h:63](d:\design\MindIE-LLM\src\include\scheduler\ischeduler.h)) 与 `Generator` ([generator.py:150-152](d:\design\MindIE-LLM\mindie_llm\text_generator\generator.py)) 同时暴露 `SwitchRole`/`switch_role`，运行时切换无需重启。**vLLM/SGLang 把 `kv_role` / `disaggregation_mode` 钉在启动配置**（详 [engine-architecture.md §10](engine-architecture.md)），任何 P/D rebalance 需 launcher 协作。如要在 vLLM/SGLang 实现弹性 PD（K8s autoscaler 触发 P→D 转换），需先给 scheduler 接口加这一抽象。
2. **从 SGLang 借鉴 mixin 拆分降低 PD 改动半径**：`disaggregation/decode.py:1171` 的 mixin 与 `DecodePreallocQueue` / `DecodeTransferQueue` / `event_loop_normal_disagg_decode` 同文件，新人添加 PD decode feature 不必碰 `srt/managers/scheduler.py`。MindIE 上做 PD 改动通常需同时修 `Generator.PDInterface` + C++ scheduler；vLLM 改动落在 `KVConnector` 子系统但前提是 scheduler 已有 `_try_promote_blocked_waiting_request` 等钩子（[scheduler.py:2070-2102](d:\design\vllm\vllm\v1\core\sched\scheduler.py)）。**架构分解决定改动半径**——这是 PD 二次开发最被低估的成本项。
3. **vLLM 单类 Scheduler 利于 PD profiling**：`Scheduler.schedule()` 单一 ~600 行函数，cProfile / py-spy 一次看到全部热点；SGLang 11 mixin 调用图分散，需 grep `class.*Mixin` 才能枚举入口；MindIE 跨语言路径需 perf + py-spy 串联。**PD 性能调优：单类 > mixin > 跨语言**。
4. **C++ scheduler 主循环避免 GIL 但承担 callback 开销**：MindIE `LlmEngine::SchedulerThreadEntry` 不持 GIL；但 `responseHandler` 反向触发 Python forward 仍需 GIL（`PluginManager.forward_thread` 异步消费 input_queue 解耦，[plugin_manager.py:107-115](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py)）。**Trade-off**：PD 优化时需测量 callback 切换 vs Python 调度本身的开销。
5. **SGLang 8 event_loop 模式带来 PP+PDMux 组合爆炸**：`dispatch_event_loop` 当前 8 路覆盖 `disaggregation_mode × pp_size × enable_pdmux × enable_overlap` 大部分组合，但 `event_loop_pp_pdmux` / `event_loop_pp_overlap_disagg_*` 等细粒度组合**未实现**——疑似 PP 与 PDMux 互斥（详 [scheduler-mixins.md `[!todo] VERIFY: event_loop_pp + enable_pdmux`](../../sglang/topics/scheduler-mixins.md)）。**PD + PP 联合优化要先评估三家在该组合上的成熟度**。

---

## Notes / Caveats

> [!todo] VERIFY: SGLang `event_loop_pp + enable_pdmux` / `event_loop_pp + enable_overlap` 组合未在 `dispatch_event_loop`（[scheduler.py:3628-3654](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）出现——疑似互斥但源码无注释明示。如要支持 PP + PDMux 联合 mode，分支表会从 8 路增至 12+ 路，dispatch 需重构。

> [!todo] VERIFY: vLLM `Scheduler` 是否真"0 mixin"——本页基于 grep `class .*Scheduler.*Mixin` 在 vllm/v1/ 全树 0 命中（**verified 2026-04-19**）。vLLM Worker 端**有** 3 mixin 给 `GPUModelRunner`（[gpu_model_runner.py:394-396](d:\design\vllm\vllm\v1\worker\gpu_model_runner.py)）。**Scheduler 层 0 mixin，Worker 层 3 mixin** — 与 SGLang "Scheduler 层 11 mixin" 偏好正交。

> [!todo] VERIFY: MindIE C++ scheduler 测试覆盖率与 Python 集成测试断言强度——本页仅标 `tests/dlt/ut/scheduler/test_scheduler.cpp` 与 `tests/dlt/it/test_llm_engine.cpp` 存在，未读测试内容验证"Policy 独立可测"是否真等价 SGLang "mixin 独立可测"。

> [!warning] CONTRADICTION: §1 "MindIE Engine ↔ Scheduler：跨语言双层" 与 [engine-architecture.md §4](engine-architecture.md) "Engine 调用 Scheduler 方式：C++ `LlmEngine` → `Scheduler::Schedule(needSync)`" 不冲突；本页强调"语言边界即 engine/scheduler 边界"，前者强调"调用方向"。但读者需注意**"engine"在 MindIE 同时指 C++ `LlmEngine` 与 Python `Generator` 装配栈**——详 [engine-architecture.md §2](engine-architecture.md)。

> [!warning] CONTRADICTION: §6 "SGLang 5 external + 6 internal mixin" 数字 — 与 [scheduler-mixins.md §External vs Internal](../../sglang/topics/scheduler-mixins.md) 对齐（5+6=11）。但物理文件名只有 7 个 `scheduler*_mixin.py`：另 4 个 mixin 类定义在非 `*_mixin.py` 命名的文件内（`disaggregation/decode.py:1171`、`disaggregation/prefill.py:355`、`multiplex/multiplexing_mixin.py:32`、`dllm/mixin/scheduler.py:20`）。**11 是类数量，7 是文件名带 `_mixin` 的文件数** — 易误数。

---

## See also

### 同 comparison 范畴

- [comparison/index.md](../index.md)
- [comparison/dimensions.md §dim-scheduler §dim-engine](../dimensions.md)
- [comparison/topics/scheduler.md](scheduler.md)（**姊妹页 — 调度策略 / 队列 / overlap 机制**）
- [comparison/topics/engine-architecture.md](engine-architecture.md)（**engine 顶层架构 13 子维度**）
- [comparison/topics/executor-worker.md](executor-worker.md)（**worker 抽象 11 子维度**）
- [comparison/topics/sync-schedule.md](sync-schedule.md) / [comparison/topics/async-schedule.md](async-schedule.md)（**调度同步性细节**）
- [comparison/topics/pd-disaggregation.md](pd-disaggregation.md)（**PD 14 子维度**）

### MindIE 端

- [mindie/entities/BatchScheduler.md](../../mindie/entities/BatchScheduler.md)（C++ `Scheduler` + 15 Policy）
- [mindie/entities/Generator.md](../../mindie/entities/Generator.md)（Python 装配栈）
- [mindie/entities/LlmEngine.md](../../mindie/entities/LlmEngine.md)（C++ engine + 8 步主循环）
- [mindie/entities/PluginManager.md](../../mindie/entities/PluginManager.md)（Python plugin chain + forward_thread）
- [mindie/modules/text_generator.md](../../mindie/modules/text_generator.md)

### vLLM 端

- [vllm/entities/Scheduler.md](../../vllm/entities/Scheduler.md)（含 `AsyncScheduler` 子类）
- [vllm/entities/EngineCore.md](../../vllm/entities/EngineCore.md)（含 `EngineCoreProc` / `DPEngineCoreProc` / `EngineCoreActor`）
- [vllm/entities/EngineCoreClient.md](../../vllm/entities/EngineCoreClient.md)（6 客户端工厂）

### SGLang 端

- [sglang/topics/scheduler-mixins.md](../../sglang/topics/scheduler-mixins.md)（**11 mixin 完整拆解 + Chain 协作**）
- [sglang/entities/Scheduler.md](../../sglang/entities/Scheduler.md)
- [sglang/topics/manager-pipeline.md](../../sglang/topics/manager-pipeline.md)
- [sglang/modules/managers.md](../../sglang/modules/managers.md)
- [sglang/modules/disaggregation.md](../../sglang/modules/disaggregation.md)
- [sglang/modules/multiplex.md](../../sglang/modules/multiplex.md)
- [sglang/modules/dllm.md](../../sglang/modules/dllm.md)
- [sglang/modules/observability.md](../../sglang/modules/observability.md)
