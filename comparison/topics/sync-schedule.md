---
type: comparison
project: cross
status: verified
confidence: high
verified_against: 2026-04-18
sources:
  - d:\design\MindIE-LLM\src\scheduler\scheduler.h
  - d:\design\MindIE-LLM\src\scheduler\scheduler.cpp
  - d:\design\MindIE-LLM\src\engine\llm_engine.cpp
  - d:\design\MindIE-LLM\src\include\config\config_info.h
  - d:\design\MindIE-LLM\mindie_llm\text_generator\generator.py
  - d:\design\vllm\vllm\v1\engine\core.py
  - d:\design\vllm\vllm\v1\core\sched\scheduler.py
  - d:\design\vllm\vllm\v1\core\sched\async_scheduler.py
  - d:\design\vllm\vllm\config\scheduler.py
  - d:\design\vllm\vllm\config\vllm.py
  - d:\design\sglang\python\sglang\srt\managers\scheduler.py
  - d:\design\sglang\python\sglang\srt\server_args.py
related:
  - comparison/dimensions.md
  - comparison/topics/scheduler.md
  - vllm/entities/Scheduler.md
  - sglang/entities/Scheduler.md
---

# Cross-project Comparison: Sync-Schedule（同步调度路径）

> 三项目"**同步调度**（sync schedule）路径"对比，是 [comparison/topics/scheduler.md §5 CPU/GPU Overlap](scheduler.md) 的反向视角：每一项的 *默认/退化* 路径——**同一时刻最多 1 个 batch 在 GPU 上 in-flight，CPU 调度决策与 GPU forward 严格串行**。本页明确每家如何配置进入这条路径、退化后跑哪段代码、以及哪些条件下系统会**强制**退到这条路径。

> 命名澄清：本页 "sync schedule" 指 "**no overlap / single in-flight batch**"，与 MindIE 源码里 `Schedule(needSync)` 的 `needSync` 参数**不是**一个概念——后者是 DP 跨 rank 同步标志（[llm_engine.cpp:462](d:\design\MindIE-LLM\src\engine\llm_engine.cpp)），而非"是否同步执行"。详见 §6。

---

## TL;DR

| 维度 | MindIE | vLLM | SGLang |
|---|---|---|---|
| **是否默认** | ✅ 默认（`activateAsyncInference=false`） | ❌ 默认 async（auto-disable 才退到 sync） | ❌ 默认 overlap（`disable_overlap_schedule=False`） |
| **配置开关** | `activateAsyncInference` (engine config) → `maxScheduledBatch_` | `scheduler_config.async_scheduling` (`bool \| None`，None=auto) | `--disable-overlap-schedule` server arg |
| **在 flight batch 上限** | `maxScheduledBatch_=1`（同步）vs 2（异步单发） | 1（`step` 内 `future.result()` 等 worker） | 1（`event_loop_normal` run_batch 后立即 process_batch_result） |
| **类切换/函数切换** | **同一个 `Scheduler` 类**，行为差异在 `maxScheduledBatch_` 数值 | **不同子类**：`Scheduler` (sync) vs `AsyncScheduler` (async) | **不同 event loop**：`event_loop_normal` vs `event_loop_overlap` |
| **EngineCore/Engine 层入口** | `LlmEngine` C++ 主循环（[llm_engine.cpp:462](d:\design\MindIE-LLM\src\engine\llm_engine.cpp)） + `step_fn = self.step` | `EngineCore.run_busy_loop` → `_process_engine_step` → `step_fn = self.step`（[core.py:212-214](d:\design\vllm\vllm\v1\engine\core.py)） | `dispatch_event_loop` → `event_loop_normal`（[scheduler.py:3637-3640](d:\design\sglang\python\sglang\srt\managers\scheduler.py)） |
| **forward 调用方式** | C++ 端把 batch 发给 modelExec，Python `Generator.generate_token` 同步返回 | `model_executor.execute_model(scheduler_output, non_block=True)` 返 future，`future.result()` 立即 await（[core.py:416-422](d:\design\vllm\vllm\v1\engine\core.py)） | `run_batch(batch)` → `process_batch_result(batch, result)` 同帧串行（[scheduler.py:1399-1401](d:\design\sglang\python\sglang\srt\managers\scheduler.py)） |
| **被强制退到 sync 的场景数** | 用户显式设置；无文档化的"自动 fallback" | 4 类自动 fallback（pooling、特定 spec、不支持 async 的 executor 等） | 10+ 类自动 fallback（设备、attention backend、PP、特定 spec、稀疏 head 等） |
| **PD 分离时仍走 sync？** | 是（同 `Schedule(needSync)` 路径，PD-aware 在 policy 层） | 是（同 `step` + KVConnector 钩子） | **独立 sync 函数**：`event_loop_normal_disagg_prefill` / `event_loop_normal_disagg_decode`（[scheduler.py:3641-3654](d:\design\sglang\python\sglang\srt\managers\scheduler.py)） |

---

## 1. 配置开关与默认值

### MindIE
- 字段：`activateAsyncInference`（[config_info.h:171, 388, 429](d:\design\MindIE-LLM\src\include\config\config_info.h)），三处 struct 都有，从 `LlmManagerConfig` → `EngineConfig` → `SchedulerConfig` 透传。
- 默认值：`false`（[config_info.h:171, 388](d:\design\MindIE-LLM\src\include\config\config_info.h)）。
- Engine 端读取：[llm_manager_impl.cpp:1240](d:\design\MindIE-LLM\src\llm_manager_v2\impl\llm_manager_impl.cpp) `schedulerConfig.activateAsyncInference = modelConfigs_[0]["asyncBatchscheduler"] == "true";` —— 由模型配置 `asyncBatchscheduler` 字段控制。
- Python 端镜像：[generator.py:319-323](d:\design\MindIE-LLM\mindie_llm\text_generator\generator.py) 把 `ENV.async_inference` 写到 `model_config["async_inference"]`，并在 init 时打印 "Async inference is activated"。
- 退化效果：[scheduler.cpp:122-128](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)
  ```cpp
  uint32_t asyncScheduleRound = schedulerConfig_->layerwiseDisaggregated ?
                              schedulerConfig_->maxDispatchBatchNum : MAX_ASYNC_SCHEDULE_TIMES;
  if (schedulerConfig_->activateAsyncInference) {
      maxScheduledBatch_ = asyncScheduleRound + 1;
      ...
  }
  // 否则保持 maxScheduledBatch_{1}; 见 scheduler.h:292-293 默认值
  ```

### vLLM
- 字段：`SchedulerConfig.async_scheduling: bool | None = None`（[config/scheduler.py:146-149](d:\design\vllm\vllm\config\scheduler.py)）。
- 注释（原文）："If set to False, disable async scheduling. Async scheduling helps to avoid gaps in GPU utilization, leading to better latency and throughput."
- 类工厂：[config/scheduler.py:168-174](d:\design\vllm\vllm\config\scheduler.py)
  ```python
  def get_scheduler_cls(self) -> type["SchedulerInterface"]:
      if self.scheduler_cls is None:
          if self.async_scheduling:
              from vllm.v1.core.sched.async_scheduler import AsyncScheduler
              return AsyncScheduler
          from vllm.v1.core.sched.scheduler import Scheduler
          ...
  ```
- 自动决策（[config/vllm.py:798-832](d:\design\vllm\vllm\config\vllm.py)）：在 `VllmConfig.__post_init__` 阶段，根据 4 类条件**强制**关闭 async：
  | 触发 | 行号 |
  |---|---|
  | pooling model | [vllm.py:798-800](d:\design\vllm\vllm\config\vllm.py) |
  | speculative_config 不支持 (条件 1) | [vllm.py:801-812](d:\design\vllm\vllm\config\vllm.py) |
  | speculative_config 不支持 (条件 2) | [vllm.py:813-822](d:\design\vllm\vllm\config\vllm.py) |
  | executor 不支持 async sched | [vllm.py:823-830](d:\design\vllm\vllm\config\vllm.py) |
  否则 [vllm.py:831-832](d:\design\vllm\vllm\config\vllm.py) `self.scheduler_config.async_scheduling = True` —— **默认 async**。
- EngineCore 内额外保存：[core.py:215](d:\design\vllm\vllm\v1\engine\core.py) `self.async_scheduling = vllm_config.scheduler_config.async_scheduling`，用于 `post_step` 决定要不要回拉 draft token id（详 §5）。

### SGLang
- 字段：`ServerArgs.disable_overlap_schedule: bool = False`（[server_args.py:639](d:\design\sglang\python\sglang\srt\server_args.py)）。**默认 overlap 开**。
- Scheduler 内镜像：[scheduler.py:376](d:\design\sglang\python\sglang\srt\managers\scheduler.py) `self.enable_overlap = not server_args.disable_overlap_schedule`。
- TpModelWorker 内镜像：[tp_worker.py:318](d:\design\sglang\python\sglang\srt\managers\tp_worker.py) `self.enable_overlap = not server_args.disable_overlap_schedule`。
- 自动 fallback 条件（按 `server_args.py` 行号）：
  | 触发 | 行号 |
  |---|---|
  | `device == "mps"` | [server_args.py:1132-1134](d:\design\sglang\python\sglang\srt\server_args.py) |
  | `SGLANG_EMBEDDINGS_SPARSE_HEAD` env 设了 | [server_args.py:2185-2187](d:\design\sglang\python\sglang\srt\server_args.py) |
  | 部分 attention backend + radix cache 组合 | [server_args.py:2300-2309](d:\design\sglang\python\sglang\srt\server_args.py) |
  | `pp_size > 1`（PP 与 overlap 不兼容） | [server_args.py:3001-3005](d:\design\sglang\python\sglang\srt\server_args.py) |
  | DFLASH speculative decoding | [server_args.py:3249-3253](d:\design\sglang\python\sglang\srt\server_args.py) |
  | spec v2 off / 不支持的 spec 算法 | [server_args.py:3279-3293](d:\design\sglang\python\sglang\srt\server_args.py) |
  | ngram speculative + 特定参数组合 | [server_args.py:3378-3382](d:\design\sglang\python\sglang\srt\server_args.py) |
  | flashinfer + 特殊 attention 组合（强制 overlap=true） | [server_args.py:3855-3861](d:\design\sglang\python\sglang\srt\server_args.py) |
- > [!todo] VERIFY: 上面 `server_args.py:6468`、`server_args.py:6522` 两处 `self.disable_overlap_schedule` 被引用是 print/log 还是逻辑——本轮未读全。

> synthesis: **三家的"sync 默认性"截然相反**——MindIE 默认 sync（async 是开关 opt-in）；vLLM 与 SGLang 默认 overlap，sync 是 *不得已* 的退化。这反映了三家对"何时 overlap 是安全的"的判断不同：MindIE 倾向保守，vLLM/SGLang 倾向激进 + 显式列黑名单。

---

## 2. Sync 模式下的"in-flight batch 上限"

| 项目 | 上限 | 锚点 | 说明 |
|---|---|---|---|
| MindIE | `maxScheduledBatch_ = 1` | [scheduler.h:292-293](d:\design\MindIE-LLM\src\scheduler\scheduler.h)（"调度实际下发的批次，当前实现，同步时是1，异步（只支持异步单发）时是2"） | 这个数值同时控制 placeholder token 预留：[scheduler.cpp:769](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp) `size_t maxPlaceHolderNum = maxScheduledBatch_ * tokenNumPerIter + tokenNumPerIter;` |
| vLLM | 1 | [core.py:404-433](d:\design\vllm\vllm\v1\engine\core.py) | `step()` 内 `model_output = future.result()` 立即等 worker，所以即使 executor 提供 future，sync 路径下永远只有 1 个 batch 在跑 |
| SGLang | 1 | [scheduler.py:1399-1401](d:\design\sglang\python\sglang\srt\managers\scheduler.py) | `event_loop_normal` 内 `result = self.run_batch(batch); self.process_batch_result(batch, result)` 串行 |

注意：**vLLM PP 流水线**用 `step_with_batch_queue`（[core.py:445-559](d:\design\vllm\vllm\v1\engine\core.py)）跑 `batch_queue_size = max_concurrent_batches > 1`，但这与 `async_scheduling` 是**正交**的两个轴：
- `step_with_batch_queue` 由 `batch_queue_size > 1` 触发（[core.py:191-193, 212-214](d:\design\vllm\vllm\v1\engine\core.py)）
- `AsyncScheduler` 由 `async_scheduling=True` 触发

> [!warning] CONTRADICTION: vLLM 在 PP=2+ 但 `async_scheduling=False` 的场景，会用 sync `Scheduler` + `step_with_batch_queue`——这是 "scheduler-sync, executor-overlap" 的组合，不算本页定义的"严格同步"。这条混合模式独立于本对比，详见 [vllm/entities/EngineCore.md](../../vllm/entities/EngineCore.md)。

---

## 3. Sync 模式下的主循环代码

### MindIE — `LlmEngine` C++ 主循环（[llm_engine.cpp:462-540](d:\design\MindIE-LLM\src\engine\llm_engine.cpp)，详细 entity 页见 `mindie/entities/LlmEngine.md`（已删））

```cpp
bool needSync = isDistributedPNodeProcessCCReady_ || isCentralizedThreadCCReady_;  // L462
while (...) {
    ScheduleExecTransfer(enginePerDP);                       // 1. PD 分离 KV 调度
    ...
    uint32_t asyncScheduleRound = schedulerConfig_->layerwiseDisaggregated ?
                             schedulerConfig_->maxDispatchBatchNum : MAX_ASYNC_SCHEDULE_TIMES;  // L494
    if (enginePerDP->modelExecOutputHandler->GetAsyncBatchNum() >= asyncScheduleRound ||
        enginePerDP->lastScheduleEmpty) {                    // L497-501 异步轮次控制
        std::this_thread::sleep_for(milliseconds(DEFAULT_SLEEP_TIME_BETWEEN_TWO_ITER));
        continue;
    }
    ...
    enginePerDP->scheduler->FetchSeqGeneratedTokens(...);    // 3. 拉上一步结果
    ...
    auto [seqGroupMetadata, scheduleOut] = enginePerDP->scheduler->Schedule(needSync);  // 4. 调度
    ...
    auto [allDpMetas, allDpOuts] = PostScheduleSyncUp(needSync, ...);                   // 5. DP 同步
    ...
    // 6. batch 下发给 modelExec → Python Generator.generate_token
}
```

要点：
- **同步路径在 sync 模式下退化为**：`asyncScheduleRound` 用作 in-flight 阈值；`maxScheduledBatch_=1` 时 `GetAsyncBatchNum() >= 1` 立刻触发 sleep + continue，强制等上一次 forward 收尾。
- 同一帧 6 步严格串行：`Schedule → PostScheduleSync → 下发 → forward → 拉结果`。

### vLLM — `EngineCore.run_busy_loop`（[core.py:1160-1219](d:\design\vllm\vllm\v1\engine\core.py)）

```python
def run_busy_loop(self):
    while self._handle_shutdown():
        self._process_input_queue()       # 1. 收 client 请求
        self._process_engine_step()       # 2. step()

def _process_engine_step(self) -> bool:
    outputs, model_executed = self.step_fn()      # sync 模式下 step_fn = self.step
    for output in outputs.items() if outputs else ():
        self.output_queue.put_nowait(output)
    self.post_step(model_executed)
    if not model_executed and self.scheduler.has_unfinished_requests():
        time.sleep(0.001)                         # WAITING_FOR_REMOTE_KVS 时让出 GIL
    return model_executed
```

`step()` 体（[core.py:404-433](d:\design\vllm\vllm\v1\engine\core.py)）：

```python
def step(self) -> tuple[dict[int, EngineCoreOutputs], bool]:
    if not self.scheduler.has_requests():
        return {}, False
    scheduler_output = self.scheduler.schedule()
    future = self.model_executor.execute_model(scheduler_output, non_block=True)
    grammar_output = self.scheduler.get_grammar_bitmask(scheduler_output)
    with (self.log_error_detail(scheduler_output), self.log_iteration_details(scheduler_output)):
        model_output = future.result()                       # 立即等 worker
        if model_output is None:
            model_output = self.model_executor.sample_tokens(grammar_output)
    self._process_aborts_queue()
    engine_core_outputs = self.scheduler.update_from_output(scheduler_output, model_output)
    return engine_core_outputs, scheduler_output.total_num_scheduled_tokens > 0
```

要点：
- `step_fn` 在 init 末尾绑定一次性切换：[core.py:212-214](d:\design\vllm\vllm\v1\engine\core.py) `self.step_fn = self.step if self.batch_queue is None else self.step_with_batch_queue`
- "sync" 仅指**调度层**串行：scheduler 决策完 → 立即 await worker future → 立即 update。`AsyncScheduler` 把 update 推迟。

### SGLang — `event_loop_normal`（[scheduler.py:1383-1409](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）

```python
@DynamicGradMode()
def event_loop_normal(self):
    """A normal scheduler loop."""
    while True:
        recv_reqs = self.recv_requests()
        self.process_input_requests(recv_reqs)
        if self._engine_paused:
            self.cancel_bubble_timer()
            continue

        batch = self.get_next_batch_to_run()
        self.cur_batch = batch

        if batch:
            result = self.run_batch(batch)
            self.process_batch_result(batch, result)
        else:
            self.on_idle()

        self.last_batch = batch
        if envs.SGLANG_ENABLE_STRICT_MEM_CHECK_DURING_BUSY.get():
            self.self_check_during_busy()
```

dispatch（[scheduler.py:3628-3654](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）：

```python
def dispatch_event_loop(scheduler: Scheduler):
    server_args = scheduler.server_args
    disaggregation_mode: DisaggregationMode = scheduler.disaggregation_mode
    if disaggregation_mode == DisaggregationMode.NULL:
        if scheduler.enable_pdmux:           scheduler.event_loop_pdmux()
        elif server_args.pp_size > 1:        scheduler.event_loop_pp()
        elif scheduler.enable_overlap:       scheduler.event_loop_overlap()
        else:                                scheduler.event_loop_normal()    # ← sync
    elif disaggregation_mode == DisaggregationMode.PREFILL:
        if server_args.pp_size > 1:          scheduler.event_loop_pp_disagg_prefill()
        elif scheduler.enable_overlap:       scheduler.event_loop_overlap_disagg_prefill()
        else:                                scheduler.event_loop_normal_disagg_prefill()   # ← sync (P)
    elif disaggregation_mode == DisaggregationMode.DECODE:
        if server_args.pp_size > 1:          scheduler.event_loop_pp_disagg_decode()
        elif scheduler.enable_overlap:       scheduler.event_loop_overlap_disagg_decode()
        else:                                scheduler.event_loop_normal_disagg_decode()    # ← sync (D)
```

要点：
- **6 个独立函数**：`event_loop_normal{,_disagg_prefill,_disagg_decode}` 与对应 overlap 版本。`pp_size>1` 与 `pdmux` 独立分支。
- sync 模式下 `run_batch → process_batch_result` 严格串行，无 `result_queue`。

---

## 4. 关键 sync vs async 行为对比表

| 行为 | MindIE sync | vLLM sync | SGLang sync |
|---|---|---|---|
| In-flight batch 数 | 1（[scheduler.h:292-293](d:\design\MindIE-LLM\src\scheduler\scheduler.h)） | 1（[core.py:422](d:\design\vllm\vllm\v1\engine\core.py)） | 1（[scheduler.py:1399-1401](d:\design\sglang\python\sglang\srt\managers\scheduler.py)） |
| Placeholder token 预留 | `1 * tokenNumPerIter + tokenNumPerIter`（[scheduler.cpp:769](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)） | sync 不预留；async 路径在 `_update_after_schedule` 加 `num_output_placeholders`（[async_scheduler.py:32](d:\design\vllm\vllm\v1\core\sched\async_scheduler.py)） | sync 不用 result_queue；async 路径用 `result_queue: deque`（[scheduler.py:1414](d:\design\sglang\python\sglang\srt\managers\scheduler.py)） |
| spec decode draft token 回拉时机 | C++ 端在 `FetchSeqGeneratedTokens` 一并拉（[llm_engine.cpp:520-521](d:\design\MindIE-LLM\src\engine\llm_engine.cpp)） | sync: `post_step` 立即 `take_draft_token_ids` + `update_draft_token_ids`（[core.py:439-443](d:\design\vllm\vllm\v1\engine\core.py)）<br>async: 跳过，由 worker 进程 in-place 更新（同 `if not self.async_scheduling`） | sync: `process_batch_result` 内同步处理；async: `launch_batch_sample_if_needed`（[scheduler.py:1462-1463](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）+ `pop_and_process` 错位 |
| 上一步 batch 引用 | `lastScheduleEmpty` flag + `modelExecOutputHandler->GetAsyncBatchNum()`（[llm_engine.cpp:497](d:\design\MindIE-LLM\src\engine\llm_engine.cpp)） | 单帧内完成，无 last_batch 状态 | `self.last_batch`（[scheduler.py:1407](d:\design\sglang\python\sglang\srt\managers\scheduler.py)），sync 路径只用于 `is_extend` 判断 |
| 类/函数切换粒度 | **数值差异**（同一个 `Scheduler` 类） | **类层切换**（`Scheduler` vs `AsyncScheduler`，[config/scheduler.py:168-174](d:\design\vllm\vllm\config\scheduler.py)） | **函数层切换**（`event_loop_normal*` vs `event_loop_overlap*`） |

> synthesis: 类切换粒度反映工程取舍——
> - **MindIE 数值化**：`maxScheduledBatch_` 一个数同时控制调度上限、placeholder 预留、in-flight 阈值，简单但未来加更多并行度（如 4）会牵动多处 assert。
> - **vLLM 子类化**：`AsyncScheduler` 仅 61 行（[async_scheduler.py](d:\design\vllm\vllm\v1\core\sched\async_scheduler.py)），覆盖 2 个方法。代价是后续要在所有 scheduler 子方法处考虑"我是哪个子类"。
> - **SGLang 函数化**：6 个 event loop 函数 + dispatch table。代价是逻辑可能在多个 loop 间漂移（已经看到 `event_loop_normal` 与 `event_loop_overlap` 的 `if self._engine_paused` 处理差异）。

---

## 5. Sync 路径下与其它特性的兼容性

| 特性 | MindIE sync | vLLM sync | SGLang sync |
|---|---|---|---|
| **chunked prefill** | ✅（policy 层处理） | ✅（统一 `num_computed_tokens` 抽象，[scheduler.py:349-358 注释](d:\design\vllm\vllm\v1\core\sched\scheduler.py)） | ✅（`get_next_batch_to_run` 内决定） |
| **prefix caching** | ✅ | ✅ | ✅ |
| **speculative decoding** | ✅，`maxScheduledBatch_*tokenNumPerIter` 预留（[scheduler.cpp:881-883](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)） | ✅，sync 模式下 `post_step` 走 `update_draft_token_ids` 同步路径（[core.py:439-443](d:\design\vllm\vllm\v1\engine\core.py)） | ✅，且**部分 spec 算法只能在 sync**（[server_args.py:3249-3293](d:\design\sglang\python\sglang\srt\server_args.py)） |
| **structured output / grammar** | ✅ | ✅，sync `update_from_output` 直接处理 | ✅，**spec v2 + grammar + decode 强制 sync**（[scheduler.py:1486-1497](d:\design\sglang\python\sglang\srt\managers\scheduler.py)） |
| **PD 分离** | ✅，同 `Schedule(needSync)` 路径 + `ScheduleTransfer()`（[scheduler.h:70-72](d:\design\MindIE-LLM\src\scheduler\scheduler.h)） | ✅，同 `step` 路径 + `KVConnector` 钩子（[scheduler.py:120, 2070-2102](d:\design\vllm\vllm\v1\core\sched\scheduler.py)） | ✅，**专门函数**：`event_loop_normal_disagg_{prefill,decode}`（[scheduler.py:3641-3654](d:\design\sglang\python\sglang\srt\managers\scheduler.py)） |
| **PP（pipeline parallel）** | sync 是"非 PP"或 "PP=1" 的默认模型，PP 由 aclgraph_pp 设计文档另开（`mindie/topics/aclgraph-pp.md`（已删）） | PP=2+ 用 `step_with_batch_queue`（与 sync/async 正交，[core.py:191-214](d:\design\vllm\vllm\v1\engine\core.py)） | **PP=2+ 强制 sync** + 专门 `event_loop_pp` 函数（[server_args.py:3001-3005](d:\design\sglang\python\sglang\srt\server_args.py), [scheduler.py:3635-3636](d:\design\sglang\python\sglang\srt\managers\scheduler.py)） |
| **DP attention** | DP 跨 rank 同步靠 `needSync` 参数（[scheduler.cpp:573-575](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)） | sync `Scheduler` 用 `has_finished_requests` (interface.py:169-182) | `event_loop_pp` / `event_loop_pdmux` 处理 |

> [!todo] VERIFY: SGLang 的 `event_loop_normal_disagg_prefill` / `event_loop_normal_disagg_decode` 内部具体差异；本轮只确认了 dispatch 入口，未深入函数体。

---

## 6. ⚠️ 命名陷阱：MindIE 的 `Schedule(needSync)` 不是"sync schedule"

**这是本页最容易混淆的一点，单独列出。**

`Schedule(needSync)` 接口（[scheduler.h:70](d:\design\MindIE-LLM\src\scheduler\scheduler.h), [ischeduler.h:42](d:\design\MindIE-LLM\src\include\scheduler\ischeduler.h)）：

```cpp
std::pair<SequenceGroupMetaDatas, SchedulerOutputs> Schedule(bool needSync = false) override;
```

`needSync` 的真实语义在 [llm_engine.cpp:462](d:\design\MindIE-LLM\src\engine\llm_engine.cpp)：

```cpp
isCentralizedThreadCCReady_ = enginePerDPs_.size() > 1;       // 集中式多 DP 主节点
isDistributedPNodeProcessCCReady_ = schedulerConfig_->distributedEnable && isProcessGroupInit && role_ == Role::P;
bool needSync = isDistributedPNodeProcessCCReady_ || isCentralizedThreadCCReady_;
```

`Schedule(needSync)` 内部对 `needSync=true` 的处理（[scheduler.cpp:283-289, 550-575](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)）：

```cpp
PDPriorityType pdPriorityType = DecidePDPriority(needSync);
if (role_ == Role::P) {
    WaitingAvoidDummyBatch(pdPriorityType, needSync);
}
...
// DecidePDPriority 内：
if (needSync) {
    // cross dp sync info
    SchedulerMetric metrics = CollectSchedulerMetric();
    // ... 跨 DP rank 广播 metric, 一致选择优先级
}
```

`WaitingAvoidDummyBatch` 注释（[scheduler.cpp:260-262](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)）："不需要同步 或 不做prefill时，不需要wait"。

> **结论**：MindIE 的 `Schedule(needSync)` 中 `needSync` ≈ "是否做 *跨 DP rank 一致性同步*"，与本页讨论的"sync vs async（CPU/GPU overlap）"完全是**正交**的两个轴：
>
> | 轴 | MindIE 控制点 |
> |---|---|
> | sync vs async（in-flight batch 数） | `activateAsyncInference` → `maxScheduledBatch_` |
> | 是否跨 DP rank 同步 | `needSync` 参数（由 distributed/centralized DP 状态决定） |
>
> 两者互不影响：你可以 `activateAsyncInference=true`（async）+ 单 DP（needSync=false），或 `activateAsyncInference=false`（sync）+ 多 DP（needSync=true）。

---

## 7. 与你 PD 优化的关联（synthesis）

> 注意：本节是综合性建议，不是任何一方源码原文。

如果你在 MindIE 上做 PD 优化、决定要不要切到 async：

1. **先看 sync 路径的瓶颈分布**：MindIE C++ 主循环（[llm_engine.cpp:497-501](d:\design\MindIE-LLM\src\engine\llm_engine.cpp)）已经有 `DEFAULT_SLEEP_TIME_BETWEEN_TWO_ITER` 这个 sleep，意味着 sync 模式下 schedule 决策可能比 forward 快。如果你的 schedule + KV transfer 加起来 < forward，async（`maxScheduledBatch_=2`）能直接提吞吐；如果反过来，async 帮助有限。
2. **PD 分离 + sync 是兼容的，但 ScheduleTransfer 本身不受 sync 影响**：[llm_engine.cpp:485-488](d:\design\MindIE-LLM\src\engine\llm_engine.cpp) 在每帧开头都跑 `ScheduleExecTransfer`，无论 sync/async。所以 KV transfer 的并发度由 `transferPolicy_`（[scheduler.h:259](d:\design\MindIE-LLM\src\scheduler\scheduler.h)）决定，不是 `maxScheduledBatch_`。
3. **跨项目借鉴**：
   - **vLLM 的"sync 是 fallback"思路**：[config/vllm.py:798-832](d:\design\vllm\vllm\config\vllm.py) 用 4 条 if/elif 自动决定 async on/off，避免用户手动调。MindIE 当前完全靠 `asyncBatchscheduler` 配置（[llm_manager_impl.cpp:1240](d:\design\MindIE-LLM\src\llm_manager_v2\impl\llm_manager_impl.cpp)），可以考虑加 auto-detect。
   - **SGLang 的 `is_disable_overlap_for_batch`（[scheduler.py:1466-1497](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）**：每个 batch 级别动态判断要不要 overlap（"连续两个 prefill 不 overlap 来改 TTFT"），比 MindIE 的全局开关更细粒度。

---

## 8. 三家 sync 模式默认值对照

| 项目 | sync 是否默认 | 触发 sync 的方式 | 锚点 |
|---|---|---|---|
| MindIE | ✅ **默认 sync** | 不设 `asyncBatchscheduler=true` 即 sync | [config_info.h:171, 388](d:\design\MindIE-LLM\src\include\config\config_info.h) `activateAsyncInference = false` |
| vLLM | ❌ **默认 async**（auto） | (a) 用户显式 `--async-scheduling=False`；(b) pooling/spec/特定 executor 触发 auto-disable | [config/vllm.py:798-832](d:\design\vllm\vllm\config\vllm.py) |
| SGLang | ❌ **默认 overlap** | (a) 用户显式 `--disable-overlap-schedule`；(b) 10+ 条件触发 auto-disable | [server_args.py:639](d:\design\sglang\python\sglang\srt\server_args.py), 多处 `_handle_*` |

---

## Notes / Caveats

> [!todo] VERIFY: vLLM `step_with_batch_queue` 在 `async_scheduling=True` 与 `async_scheduling=False` 下行为差异（本页仅核心 sync 路径用 `step`，未深入 batch_queue 路径）。
> [!todo] VERIFY: SGLang `event_loop_normal_disagg_prefill` / `event_loop_normal_disagg_decode` 函数体（本轮仅确认 dispatch 入口）。
> [!todo] VERIFY: MindIE Python 端 `ENV.async_inference` 与 C++ 端 `activateAsyncInference` 是否始终一致——若不一致会出现 Python 走 sync、C++ 端预留 placeholder 按 async 算的不对齐。

> [!warning] CONTRADICTION: 见 §2 末尾——vLLM 在 `PP>=2 + async_scheduling=False` 是"sync scheduler + queued executor"混合模式，与本页 sync 严格定义有差。

## See also
- [comparison/topics/scheduler.md](scheduler.md) — sync vs async 视角的姊妹页
- [comparison/dimensions.md §dim-async-schedule](../dimensions.md) — 反向维度
- [comparison/dimensions.md §dim-sync-schedule](../dimensions.md) — 本页对应维度
- [vllm/entities/Scheduler.md](../../vllm/entities/Scheduler.md)
- [vllm/entities/EngineCore.md](../../vllm/entities/EngineCore.md)
- [sglang/entities/Scheduler.md](../../sglang/entities/Scheduler.md)
