---
type: entity
project: sglang
status: verified
confidence: high
verified_against: 2026-08-18
sources:
  - d:\design\sglang\python\sglang\srt\managers\scheduler.py:L383-L5085
  - d:\design\sglang\python\sglang\srt\managers\scheduler_components\
  - d:\design\sglang\python\sglang\srt\managers\scheduler_pp_mixin.py
  - d:\design\sglang\python\sglang\srt\disaggregation\decode.py:L2137
  - d:\design\sglang\python\sglang\srt\disaggregation\prefill.py:L485
  - d:\design\sglang\python\sglang\srt\multiplex\multiplexing_mixin.py:L33
  - d:\design\sglang\python\sglang\srt\dllm\mixin\scheduler.py:L22
  - d:\design\sglang\python\sglang\srt\hardware_backend\mlx\scheduler_mixin.py
  - d:\design\sglang\python\sglang\srt\session\session_controller.py:L354
  - d:\design\sglang\python\sglang\srt\entrypoints\engine.py:L856-L946
  - d:\design\sglang\python\sglang\srt\managers\data_parallel_controller.py
  - d:\design\sglang\python\sglang\srt\ray\scheduler_actor.py
related:
  - sglang/modules/managers.md
  - sglang/modules/disaggregation.md
  - sglang/modules/distributed.md
  - sglang/modules/model_executor.md
  - sglang/entities/TokenizerManager.md
  - sglang/entities/TpModelWorker.md
  - sglang/entities/Engine.md
  - sglang/entities/DataParallelController.md
  - sglang/topics/manager-pipeline.md
  - sglang/topics/scheduler-mixins.md
  - sglang/topics/request-lifecycle.md
---

# `Scheduler`

## Summary

`Scheduler` 是 SGLang 的核心调度器，跑在子进程（或 Ray actor）里，每 PP×TP 组合一个实例；直接持有 `TpModelWorker` / draft worker，无独立 Executor 层。HEAD `f7101b0a` 上类定义在 [scheduler.py:383-390](d:\design\sglang\python\sglang\srt\managers\scheduler.py)，**MRO 剩 6 个职责 mixin + 条件性 `SchedulerMlxOverlapMixin`**；原先 OutputProcessor / UpdateWeights / Profiler / Metrics / RuntimeChecker / DPAttn 等已迁到 [scheduler_components/](d:\design\sglang\python\sglang\srt\managers\scheduler_components) **组合对象**。主循环经 `run_event_loop` → `dispatch_event_loop` 分发到 normal / overlap / PP / PD-disagg / PDMux / MLX 变体。

## Sources

- [d:\design\sglang\python\sglang\srt\managers\scheduler.py:383-5085](d:\design\sglang\python\sglang\srt\managers\scheduler.py)（整文件 ~5085 行；`class Scheduler` L383；`dispatch_event_loop` L4902；`configure_scheduler_process` L4932；`run_scheduler_process` L4997）
- [d:\design\sglang\python\sglang\srt\managers\scheduler_components\](d:\design\sglang\python\sglang\srt\managers\scheduler_components)（19 个组件模块 + `__init__.py`）
- [d:\design\sglang\python\sglang\srt\managers\scheduler_pp_mixin.py](d:\design\sglang\python\sglang\srt\managers\scheduler_pp_mixin.py)（`SchedulerPPMixin`）
- [d:\design\sglang\python\sglang\srt\disaggregation\decode.py:2137](d:\design\sglang\python\sglang\srt\disaggregation\decode.py)（`SchedulerDisaggregationDecodeMixin`）
- [d:\design\sglang\python\sglang\srt\disaggregation\prefill.py:485](d:\design\sglang\python\sglang\srt\disaggregation\prefill.py)（`SchedulerDisaggregationPrefillMixin`）
- [d:\design\sglang\python\sglang\srt\multiplex\multiplexing_mixin.py:33](d:\design\sglang\python\sglang\srt\multiplex\multiplexing_mixin.py)（`SchedulerMultiplexMixin`）
- [d:\design\sglang\python\sglang\srt\dllm\mixin\scheduler.py:22](d:\design\sglang\python\sglang\srt\dllm\mixin\scheduler.py)（`SchedulerDllmMixin`）
- [d:\design\sglang\python\sglang\srt\hardware_backend\mlx\scheduler_mixin.py](d:\design\sglang\python\sglang\srt\hardware_backend\mlx\scheduler_mixin.py)（`SchedulerMlxOverlapMixin`；非 MPS 时 scheduler.py 内 stub）
- [d:\design\sglang\python\sglang\srt\session\session_controller.py:354](d:\design\sglang\python\sglang\srt\session\session_controller.py)（`SessionController`）
- [d:\design\sglang\python\sglang\srt\entrypoints\engine.py:856-946](d:\design\sglang\python\sglang\srt\entrypoints\engine.py)（`_launch_scheduler_processes` → `mp.Process(target=run_scheduler_process_func)`）
- [d:\design\sglang\python\sglang\srt\managers\tp_worker.py](d:\design\sglang\python\sglang\srt\managers\tp_worker.py)（`TpModelWorker`；独立实体页）
- [d:\design\sglang\python\sglang\srt\managers\schedule_batch.py:3407](d:\design\sglang\python\sglang\srt\managers\schedule_batch.py)（`NextBatchPlan`）
- [d:\design\sglang\python\sglang\srt\managers\schedule_policy.py](d:\design\sglang\python\sglang\srt\managers\schedule_policy.py)（`SchedulePolicy`）

## Architecture / Data flow

### 类层次：mixin（少）+ composition（多）

```mermaid
classDiagram
    class Scheduler {
        +ipc_channels : SchedulerIpcChannels
        +request_receiver : SchedulerRequestReceiver
        +batch_result_processor : SchedulerBatchResultProcessor
        +output_streamer : SchedulerOutputStreamer
        +weight_updater : SchedulerWeightUpdaterManager
        +metrics_reporter : SchedulerMetricsReporter
        +profiler_manager : SchedulerProfilerManager
        +dp_attn_adapter : SchedulerDPAttnAdapter
        +invariant_checker : SchedulerInvariantChecker
        +idle_sleeper
        +session_controller : SessionController
        +tp_worker / model_worker
        +run_event_loop()
        +event_loop_normal()
        +event_loop_overlap()
    }
    class SchedulerDisaggregationDecodeMixin
    class SchedulerDisaggregationPrefillMixin
    class SchedulerMultiplexMixin
    class SchedulerPPMixin
    class SchedulerDllmMixin
    class SchedulerMlxOverlapMixin
    Scheduler --|> SchedulerDisaggregationDecodeMixin
    Scheduler --|> SchedulerDisaggregationPrefillMixin
    Scheduler --|> SchedulerMultiplexMixin
    Scheduler --|> SchedulerPPMixin
    Scheduler --|> SchedulerDllmMixin
    Scheduler --|> SchedulerMlxOverlapMixin
    Scheduler o-- SchedulerIpcChannels
    Scheduler o-- SchedulerRequestReceiver
    Scheduler o-- SchedulerBatchResultProcessor
    Scheduler o-- SchedulerWeightUpdaterManager
    Scheduler o-- SchedulerMetricsReporter
    Scheduler o-- TpModelWorker
```

精确 MRO 声明见 [scheduler.py:383-390](d:\design\sglang\python\sglang\srt\managers\scheduler.py)：

```python
class Scheduler(
    SchedulerDisaggregationDecodeMixin,
    SchedulerDisaggregationPrefillMixin,
    SchedulerMultiplexMixin,
    SchedulerPPMixin,
    SchedulerDllmMixin,
    SchedulerMlxOverlapMixin,
):
```

`SchedulerMlxOverlapMixin`：`is_mps()` 时从 [hardware_backend/mlx/scheduler_mixin.py](d:\design\sglang\python\sglang\srt\hardware_backend\mlx\scheduler_mixin.py) 导入；否则在 [scheduler.py:338-339](d:\design\sglang\python\sglang\srt\managers\scheduler.py) 定义为空 stub。

### 前 mixin → 现 composition 映射

| 旧 mixin（已删除源文件） | 现组件 / 归属 | 初始化锚点 |
|---|---|---|
| `SchedulerOutputProcessorMixin` | `SchedulerBatchResultProcessor` + `SchedulerOutputStreamer` + `SchedulerLogprobResultProcessor` | [scheduler.py:2139-2175](d:\design\sglang\python\sglang\srt\managers\scheduler.py) |
| `SchedulerUpdateWeightsMixin` | `SchedulerWeightUpdaterManager` | [scheduler.py:1940-1950](d:\design\sglang\python\sglang\srt\managers\scheduler.py) |
| `SchedulerProfilerMixin` | `SchedulerProfilerManager` | [scheduler.py:1933-1938](d:\design\sglang\python\sglang\srt\managers\scheduler.py) |
| `SchedulerMetricsMixin` | `SchedulerMetricsReporter` + `SchedulerMetricsCollector` + `SchedulerKvEventsPublisher` + `SchedulerLoadInquirer` | [scheduler.py:723-737](d:\design\sglang\python\sglang\srt\managers\scheduler.py)、[L1198-1210](d:\design\sglang\python\sglang\srt\managers\scheduler.py)、[L2093-2130](d:\design\sglang\python\sglang\srt\managers\scheduler.py) |
| `SchedulerRuntimeCheckerMixin` | `SchedulerInvariantChecker` + `SchedulerPoolStatsObserver` + `create_scheduler_watchdog` | [scheduler.py:2058-2091](d:\design\sglang\python\sglang\srt\managers\scheduler.py)、[L1257-1266](d:\design\sglang\python\sglang\srt\managers\scheduler.py) |
| `SchedulerDPAttnMixin` | `SchedulerDPAttnAdapter` | [scheduler.py:2035-2056](d:\design\sglang\python\sglang\srt\managers\scheduler.py) |
| （同文件辅助类）`IdleSleeper` | [idle_sleeper.py:15](d:\design\sglang\python\sglang\srt\managers\scheduler_components\idle_sleeper.py) | [scheduler.py:777-791](d:\design\sglang\python\sglang\srt\managers\scheduler.py) |
| （同文件辅助类）`SenderWrapper` | [output_sender.py:8](d:\design\sglang\python\sglang\srt\managers\scheduler_components\output_sender.py)；经 `SchedulerIpcChannels` | [ipc_channels.py:67-68](d:\design\sglang\python\sglang\srt\managers\scheduler_components\ipc_channels.py) |

> synthesis: HEAD 把“正交横切职责”从多重继承拆成可测试的组合对象；仍保留的 5+1 mixin 主要是 **事件循环变体 / PD / PP / DLLM / MLX** 这类需要改写主循环或大量覆盖方法的路径。

### `scheduler_components/` 关键类清单

| 文件 | 关键类 | 角色 |
|---|---|---|
| [ipc_channels.py](d:\design\sglang\python\sglang\srt\managers\scheduler_components\ipc_channels.py) | `SchedulerIpcChannels` | ZMQ PULL/DEALER/PUSH + `SenderWrapper` 封装 |
| [output_sender.py](d:\design\sglang\python\sglang\srt\managers\scheduler_components\output_sender.py) | `SenderWrapper` | `send_output`；`socket is None` 时 no-op |
| [idle_sleeper.py](d:\design\sglang\python\sglang\srt\managers\scheduler_components\idle_sleeper.py) | `IdleSleeper`, `RustServerIdleSleeper` | idle 时 poll/sleep；Rust 环用 `wait_ingress` |
| [request_receiver.py](d:\design\sglang\python\sglang\srt\managers\scheduler_components\request_receiver.py) | `SchedulerRequestReceiver` | `recv_requests` + TP/CP broadcast |
| [batch_result_processor.py](d:\design\sglang\python\sglang\srt\managers\scheduler_components\batch_result_processor.py) | `SchedulerBatchResultProcessor` | prefill/decode/idle/prebuilt 结果处理 |
| [output_streamer.py](d:\design\sglang\python\sglang\srt\managers\scheduler_components\output_streamer.py) | `SchedulerOutputStreamer` | 向 detokenizer / Rust egress 推流 |
| [logprob_result_processor.py](d:\design\sglang\python\sglang\srt\managers\scheduler_components\logprob_result_processor.py) | `SchedulerLogprobResultProcessor` | logprob 后处理 |
| [weight_updater.py](d:\design\sglang\python\sglang\srt\managers\scheduler_components\weight_updater.py) | `SchedulerWeightUpdaterManager` | 在线权重 / IPC / memory occupation |
| [profiler_manager.py](d:\design\sglang\python\sglang\srt\managers\scheduler_components\profiler_manager.py) | `SchedulerProfilerManager` | torch/CUDA profiler RPC |
| [metrics_reporter.py](d:\design\sglang\python\sglang\srt\managers\scheduler_components\metrics_reporter.py) | `SchedulerMetricsReporter` | 步级 metrics + FPM |
| [dp_attn.py](d:\design\sglang\python\sglang\srt\managers\scheduler_components\dp_attn.py) | `SchedulerDPAttnAdapter` | DP-attn MLP sync batch |
| [invariant_checker.py](d:\design\sglang\python\sglang\srt\managers\scheduler_components\invariant_checker.py) | `SchedulerInvariantChecker`, `create_scheduler_watchdog` | pool/tree 不变量 + watchdog 工厂 |
| [pool_stats_observer.py](d:\design\sglang\python\sglang\srt\managers\scheduler_components\pool_stats_observer.py) | `SchedulerPoolStatsObserver` | KV/req pool 用量观察 |
| [load_inquirer.py](d:\design\sglang\python\sglang\srt\managers\scheduler_components\load_inquirer.py) | `SchedulerLoadInquirer` | `/v1/loads` 等负载查询 |
| [kv_events_publisher.py](d:\design\sglang\python\sglang\srt\managers\scheduler_components\kv_events_publisher.py) | `SchedulerKvEventsPublisher` | KV events 外发 |
| [flush_wrapper.py](d:\design\sglang\python\sglang\srt\managers\scheduler_components\flush_wrapper.py) | `SchedulerFlushWrapper` | `FlushCacheReqInput` 包装 |
| [recv_skipper.py](d:\design\sglang\python\sglang\srt\managers\scheduler_components\recv_skipper.py) | `SchedulerRecvSkipper` | 慢消费跳过 recv |
| [new_token_ratio_tracker.py](d:\design\sglang\python\sglang\srt\managers\scheduler_components\new_token_ratio_tracker.py) | `NewTokenRatioTracker` | 新 token 比例估计 |
| [memory_usage.py](d:\design\sglang\python\sglang\srt\managers\scheduler_components\memory_usage.py) | `build_memory_usage` 等 | 内存用量汇总 helper |

### 入口：`run_scheduler_process`

子进程由 `Engine._launch_scheduler_processes` 以 `mp.Process(target=run_scheduler_process_func, ...)` 拉起（[engine.py:905-919](d:\design\sglang\python\sglang\srt\entrypoints\engine.py)）；DP>1 时改走 `run_data_parallel_controller_process`（[engine.py:928-946](d:\design\sglang\python\sglang\srt\entrypoints\engine.py)）。

[run_scheduler_process](d:\design\sglang\python\sglang\srt\managers\scheduler.py)（[L4997-5085](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）：

1. `load_plugins()`（[L5013](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）
2. `publish(server_args, role="scheduler")`（[L5015](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）——**顺序已调整**：现在在进程内**任何配置读取之前**（含 `configure_scheduler_process`）先 publish（config bags 重构，见 §Increment 2026-08-18）
3. `configure_scheduler_process`（[L4932-4994](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）：`kill_itself_when_parent_died`、`setproctitle`、`faulthandler.enable`、`configure_logger`、可选 CPU affinity / NUMA bind
4. 构造 `Scheduler(...)` → `pipe_writer.send(scheduler.get_init_info())` → `scheduler.run_event_loop()`（[L5046-5064](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）
5. 异常：`parent_process.send_signal(SIGQUIT)`；可选 `SGLANG_KILLPG_ON_SCHEDULER_EXCEPTION`（[L5066-5076](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）
6. `finally`：`metrics_reporter._shutdown_fpm()`；graceful 时 `release_host_resources()`（[L5077-5085](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）

Ray 路径不走 `run_scheduler_process`，由 [ray/scheduler_actor.py](d:\design\sglang\python\sglang\srt\ray\scheduler_actor.py) 直接构造 `Scheduler`（注释 [L106-107](d:\design\sglang\python\sglang\srt\ray\scheduler_actor.py)）。

### `Scheduler.__init__` 编排

构造签名 [scheduler.py:393-404](d:\design\sglang\python\sglang\srt\managers\scheduler.py)。`__init__` 明确约定为 orchestrator：只顺序调 `init_*` / `maybe_init_*`（[L405-408](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）。主序列（[L409-659](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）：

| 步骤 | 锚点 | 角色 |
|---|---|---|
| `init_soft_watchdog` | [L414](d:\design\sglang\python\sglang\srt\managers\scheduler.py) / [L1257](d:\design\sglang\python\sglang\srt\managers\scheduler.py) | 可选 soft daemon watchdog |
| `init_model_config` | [L488](d:\design\sglang\python\sglang\srt\managers\scheduler.py) / [L710](d:\design\sglang\python\sglang\srt\managers\scheduler.py) | `ModelConfig` |
| `init_metrics_collector` | [L491](d:\design\sglang\python\sglang\srt\managers\scheduler.py) | Prometheus collector 上下文 |
| `init_ipc_channels` + `init_idle_sleeper` | [L494-495](d:\design\sglang\python\sglang\srt\managers\scheduler.py) | ZMQ + IdleSleeper |
| `init_tokenizer` / MoE / mamba | [L505-511](d:\design\sglang\python\sglang\srt\managers\scheduler.py) | tokenizer + gemm/mamba |
| `init_model_worker` | [L518](d:\design\sglang\python\sglang\srt\managers\scheduler.py) / [L993](d:\design\sglang\python\sglang\srt\managers\scheduler.py) | `TpModelWorker` + draft + pools + graphs |
| `kv_cache_builder.build_kv_cache` | [L524-559](d:\design\sglang\python\sglang\srt\managers\scheduler.py) | tree_cache / token pools |
| `init_running_status` | [L586](d:\design\sglang\python\sglang\srt\managers\scheduler.py) / [L1138](d:\design\sglang\python\sglang\srt\managers\scheduler.py) | queues + `SessionController` + `FlushWrapper` |
| `init_chunked_prefill` / `init_diffusion_llm` / `init_schedule_policy` | [L589-597](d:\design\sglang\python\sglang\srt\managers\scheduler.py) | 调度策略 |
| `init_watch_dog_memory_saver_input_blocker` | [L600](d:\design\sglang\python\sglang\srt\managers\scheduler.py) | hard watchdog + recv skipper + input blocker |
| `init_profiler` / `maybe_init_rust_server` / `init_disaggregation` | [L603-615](d:\design\sglang\python\sglang\srt\managers\scheduler.py) | profiler / Rust FE / PD queues |
| `init_overlap` | [L618](d:\design\sglang\python\sglang\srt\managers\scheduler.py) / [L1446](d:\design\sglang\python\sglang\srt\managers\scheduler.py) | `future_map` + CUDA streams |
| `init_weight_updater` / `init_request_dispatcher` | [L626-629](d:\design\sglang\python\sglang\srt\managers\scheduler.py) | RPC `TypeBasedDispatcher` |
| `init_request_receiver` … `init_batch_result_processor` | [L642-656](d:\design\sglang\python\sglang\srt\managers\scheduler.py) | 组合对象收尾 |

`ParallelState` 在 [L465-485](d:\design\sglang\python\sglang\srt\managers\scheduler.py) 固化 tp/pp/dp/attn_cp/moe 等 rank（`dp_size` / `ep_size` 现读自 `get_parallel()` bag）。`enable_overlap` = `not disable_overlap_schedule and not use_mlx()`（[L433](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）。

### IPC：`SchedulerIpcChannels`

[init_ipc_channels](d:\design\sglang\python\sglang\srt\managers\scheduler.py)（[L738-775](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）委托 [SchedulerIpcChannels.create](d:\design\sglang\python\sglang\srt\managers\scheduler_components\ipc_channels.py)（[L25-88](d:\design\sglang\python\sglang\srt\managers\scheduler_components\ipc_channels.py)）：

- leader（`pp_rank==0 and attn_tp_rank==0 and attn_cp_rank==0`）：`recv_from_tokenizer` = ZMQ PULL（`scheduler_input_ipc_name`）；`recv_from_rpc` = DEALER；`send_to_tokenizer` / `send_to_detokenizer` = PUSH（`skip_tokenizer_init` 时 detokenizer 也指向 tokenizer IPC）
- 非 leader：recv 为 `None`，send 为 `SenderWrapper(None)`（[ipc_channels.py:69-73](d:\design\sglang\python\sglang\srt\managers\scheduler_components\ipc_channels.py)）
- metrics 开启时额外 `send_metrics_from_scheduler` PUSH（[L75-80](d:\design\sglang\python\sglang\srt\managers\scheduler_components\ipc_channels.py)）

`IdleSleeper` 在 leader + `sleep_on_idle` 时注册 tokenizer/rpc 两 socket（[scheduler.py:777-791](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）；Rust server 模式用 `RustServerIdleSleeper` 覆盖（[L2010](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）。

### 事件循环

```mermaid
flowchart TB
    Run["run_event_loop<br/>scheduler.py:1667"] --> MLX{"use_mlx()?"}
    MLX -- yes --> Disp1["dispatch_event_loop"]
    MLX -- no --> Stream["schedule_stream = Stream(priority=0)<br/>+ StreamContext"]
    Stream --> Disp2["dispatch_event_loop<br/>scheduler.py:4902"]
    Disp2 --> Null{"disaggregation_mode"}
    Null -- NULL --> Pdmux{"enable_pdmux?"}
    Pdmux -- yes --> EPdmux["event_loop_pdmux"]
    Pdmux -- no --> PP{"pp_size > 1?"}
    PP -- yes --> EPP["event_loop_pp"]
    PP -- no --> Ov{"enable_overlap / mlx?"}
    Ov --> EN["event_loop_normal / overlap / overlap_mlx"]
    Null -- PREFILL --> Pref["normal/overlap/pp_disagg_prefill"]
    Null -- DECODE --> Dec["normal/overlap/pp_disagg_decode"]
```

`dispatch_event_loop` 完整分支：[scheduler.py:4902-4930](d:\design\sglang\python\sglang\srt\managers\scheduler.py)（PP/attn_cp/moe_dp 分支条件现读 `configured_pp_size()` 等 runtime_context helper，不再读 `server_args.pp_size`）。

#### `event_loop_normal`（[L1719-1751](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）

每轮：`request_receiver.recv_requests()` → `process_input_requests` → `get_next_batch_to_run`（返回 `NextBatchPlan`）→ 有 batch 则 `run_batch` + `process_batch_result`，否则 `on_idle`。

#### `event_loop_overlap`（[L1754-1826](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）

`result_queue: Deque` 让上一 batch 的 `process_batch_result` 与本 batch `run_batch` overlap；`run_batch` 后 `_apply_war_barrier()`（[L1803](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）；generation 路径再 `launch_batch_sample_if_needed`（[L1820](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）。

`is_disable_overlap_for_batch`（[L1828-1864](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）：

| 条件 | 说明 |
|---|---|
| `SGLANG_DISABLE_CONSECUTIVE_PREFILL_OVERLAP` + 连续 extend | 改善首 batch TTFT |
| spec + `grammar_needs_sync()` + decode + `result_queue` 非空 | grammar FSM 需在下一 bitmask 前同步（host-draft 路径） |

> synthesis: overlap 的真并发来自 **CUDA `forward_stream` / `copy_stream` / `schedule_stream` + `result_queue` 延迟 CPU 后处理**，不是额外 Python worker 线程跑 forward。

### 关键 step 函数

| 函数 | 锚点 | 作用 |
|---|---|---|
| `SchedulerRequestReceiver.recv_requests` | [request_receiver.py:76-102](d:\design\sglang\python\sglang\srt\managers\scheduler_components\request_receiver.py) | ZMQ/Rust 取请求并 broadcast |
| `process_input_requests` | [scheduler.py:1877-1909](d:\design\sglang\python\sglang\srt\managers\scheduler.py) | `session_controller.maybe_reap` + `_request_dispatcher` |
| `handle_generate_request` | [scheduler.py:2368](d:\design\sglang\python\sglang\srt\managers\scheduler.py) | 主 generate 入队 |
| `get_next_batch_to_run` | [scheduler.py:3015-3155](d:\design\sglang\python\sglang\srt\managers\scheduler.py) | 返回 `NextBatchPlan(batch_to_run, running_batch)` |
| `get_new_batch_prefill` | [scheduler.py:3157](d:\design\sglang\python\sglang\srt\managers\scheduler.py) | 新 prefill |
| `update_running_batch` | [scheduler.py:3481](d:\design\sglang\python\sglang\srt\managers\scheduler.py) | decode 侧更新 |
| `run_batch` | [scheduler.py:3626](d:\design\sglang\python\sglang\srt\managers\scheduler.py) | `model_worker.forward_batch_generation` |
| `launch_batch_sample_if_needed` | [scheduler.py:3886](d:\design\sglang\python\sglang\srt\managers\scheduler.py) | overlap 延迟 sample |
| `process_batch_result` | [scheduler.py:3922-3956](d:\design\sglang\python\sglang\srt\managers\scheduler.py) | 委托 `batch_result_processor` / disagg / dllm |
| `on_idle` | [scheduler.py:4041](d:\design\sglang\python\sglang\srt\managers\scheduler.py) | leak check / metrics / kv events / `maybe_sleep_on_idle` |
| `abort_request` | [scheduler.py:4442](d:\design\sglang\python\sglang\srt\managers\scheduler.py) | abort |
| `open_session` / `close_session` | [scheduler.py:4836-4851](d:\design\sglang\python\sglang\srt\managers\scheduler.py) | 经 `SessionController` |

`get_next_batch_to_run` 决策要点（[L3015-3155](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）：timeout abort → stash chunked/dllm →（非 hisparse）merge 上一 extend batch 进 `running_batch` → `get_new_batch_prefill`（或 dllm）→ **有新 prefill 优先**，否则 `update_running_batch` 走 decode → `dp_attn_adapter.maybe_prepare_mlp_sync_batch` → ngram prepare → `NextBatchPlan`。

`run_batch` overlap 路径在 `forward_stream_ctx` 内 `forward_stream.wait_stream(schedule_stream)`，再 `model_worker.forward_batch_generation`；结果 D2H 可走 `copy_stream`（[L3665-3740](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）。`forward_stream` 来自 `tp_worker.get_worker_info()`（[L1030-1043](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）。

### `IdleSleeper` / `SenderWrapper` / `SessionController`

- **IdleSleeper**（[idle_sleeper.py:15-42](d:\design\sglang\python\sglang\srt\managers\scheduler_components\idle_sleeper.py)）：`zmq.Poller.poll(1000)`；可选 `SGLANG_EMPTY_CACHE_INTERVAL` 触发 `empty_cache`。由 `on_idle` → `maybe_sleep_on_idle` 调用（[scheduler.py:4853-4855](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）。
- **SenderWrapper**（[output_sender.py:8-29](d:\design\sglang\python\sglang\srt\managers\scheduler_components\output_sender.py)）：封装 `sock_send`；复制 `http_worker_ipc` 以支持 multi-http worker（本期改为 `getattr(recv_obj, "http_worker_ipc", None)`，`recv_obj` 放宽为任意 object，[output_sender.py:15-27](d:\design\sglang\python\sglang\srt\managers\scheduler_components\output_sender.py)）。
- **SessionController**（[session_controller.py:354](d:\design\sglang\python\sglang\srt\session\session_controller.py)）：在 `init_running_status` 创建（[scheduler.py:1156](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）；`process_input_requests` 调 `maybe_reap`；open/close session RPC 委托之。

## Key APIs / Entities

| 名称 | 位置 | 作用 |
|---|---|---|
| `Scheduler` | [scheduler.py:383](d:\design\sglang\python\sglang\srt\managers\scheduler.py) | 调度器主体 |
| `run_scheduler_process` | [scheduler.py:4997](d:\design\sglang\python\sglang\srt\managers\scheduler.py) | 子进程入口 |
| `dispatch_event_loop` | [scheduler.py:4902](d:\design\sglang\python\sglang\srt\managers\scheduler.py) | 选主循环 |
| `configure_scheduler_process` | [scheduler.py:4932](d:\design\sglang\python\sglang\srt\managers\scheduler.py) | 进程标题 / logger / affinity |
| `SchedulerIpcChannels` | [ipc_channels.py:17](d:\design\sglang\python\sglang\srt\managers\scheduler_components\ipc_channels.py) | ZMQ 通道 dataclass |
| `SenderWrapper` | [output_sender.py:8](d:\design\sglang\python\sglang\srt\managers\scheduler_components\output_sender.py) | 输出发送包装 |
| `IdleSleeper` | [idle_sleeper.py:15](d:\design\sglang\python\sglang\srt\managers\scheduler_components\idle_sleeper.py) | idle CPU 节能 |
| `SchedulerRequestReceiver` | [request_receiver.py:49](d:\design\sglang\python\sglang\srt\managers\scheduler_components\request_receiver.py) | 收包 + broadcast |
| `SchedulerBatchResultProcessor` | [batch_result_processor.py:77](d:\design\sglang\python\sglang\srt\managers\scheduler_components\batch_result_processor.py) | 结果处理 |
| `SessionController` | [session_controller.py:354](d:\design\sglang\python\sglang\srt\session\session_controller.py) | session 生命周期 |
| `NextBatchPlan` | [schedule_batch.py:3407](d:\design\sglang\python\sglang\srt\managers\schedule_batch.py) | `(batch_to_run, running_batch)` |
| `TpModelWorker` | [tp_worker.py](d:\design\sglang\python\sglang\srt\managers\tp_worker.py) | 见 [TpModelWorker.md](TpModelWorker.md) |
| `TypeBasedDispatcher` | [scheduler.py:1532-1626](d:\design\sglang\python\sglang\srt\managers\scheduler.py) | RPC/请求类型派发表 |

## Hidden state（threads / queues / streams / IPC）

| 类别 | 位置 | 说明 |
|---|---|---|
| 进程 | [engine.py:905](d:\design\sglang\python\sglang\srt\entrypoints\engine.py) `mp.Process`；DP 控制器 [data_parallel_controller.py:701](d:\design\sglang\python\sglang\srt\managers\data_parallel_controller.py) | 每个 scheduler 一个 OS 进程（Ray 则 actor） |
| Watchdog 线程 | `WatchdogRaw` [watchdog.py:121-122](d:\design\sglang\python\sglang\srt\utils\watchdog.py) via `create_scheduler_watchdog` | soft（[scheduler.py:1259](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）+ hard（[L1265](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）daemon `threading.Thread` |
| FPM 发布线程 | `_FpmPublisherThread` [forward_pass_metrics.py:135-169](d:\design\sglang\python\sglang\srt\observability\forward_pass_metrics.py)；由 metrics_reporter 创建 | `queue.Queue` + ZMQ PUB 后台线程；`run_scheduler_process` finally 调 `_shutdown_fpm`（[scheduler.py:5079-5081](d:\design\sglang\python\sglang\srt\managers\scheduler.py)） |
| overlap `result_queue` | [scheduler.py:1756-1758](d:\design\sglang\python\sglang\srt\managers\scheduler.py) `deque` | CPU 后处理与 GPU forward 解耦 |
| CUDA/设备流 | `schedule_stream`（[L1684](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）；`forward_stream`（worker info [L1039](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）；`copy_stream`（[L1492](d:\design\sglang\python\sglang\srt\managers\scheduler.py)） | overlap / WAR barrier（[L1705-1717](d:\design\sglang\python\sglang\srt\managers\scheduler.py)） |
| ZMQ IPC | [ipc_channels.py:36-80](d:\design\sglang\python\sglang\srt\managers\scheduler_components\ipc_channels.py) | PULL/DEALER/PUSH；非 leader 为空 |
| 运行队列 | `waiting_queue` / `running_batch` / `last_batch`（[L1141-1148](d:\design\sglang\python\sglang\srt\managers\scheduler.py)） | 连续 batching 状态 |
| PD 队列 | `disagg_*_queue`（[init_disaggregation L1286-1444](d:\design\sglang\python\sglang\srt\managers\scheduler.py)） | bootstrap / transfer / prealloc / inflight |

## 使用方调用清单 / 跨子系统引用

### 1) 跨语言绑定

- `Scheduler` / `IdleSleeper` / `SessionController` pybind|ctypes|cffi|Cython|capsule：在 [d:\design\sglang\python\sglang\srt\managers\scheduler.py](d:\design\sglang\python\sglang\srt\managers\scheduler.py) 全文件 grep 0 命中（纯 Python 调度进程；Rust 侧经 `RustServer` 嵌入，不是 pybind 暴露 Scheduler 类）。
- C++ `src/` 树：本 upstream 检出无独立 C++ `src/` Scheduler 绑定树可扫；`Scheduler` 类名跨语言绑定在当前 tree 视为 N/A（verified 2026-08-10）。

### 2) 协作伙伴跨子系统引用（全 `srt/` grep）

| 伙伴 | 命中范围（除定义/本页自引用外） |
|---|---|
| `Scheduler` / `run_scheduler_process` | [entrypoints/engine.py](d:\design\sglang\python\sglang\srt\entrypoints\engine.py)、[entrypoints/http_server.py](d:\design\sglang\python\sglang\srt\entrypoints\http_server.py)、[managers/data_parallel_controller.py](d:\design\sglang\python\sglang\srt\managers\data_parallel_controller.py)、[ray/engine.py](d:\design\sglang\python\sglang\srt\ray\engine.py) / [ray/scheduler_actor.py](d:\design\sglang\python\sglang\srt\ray\scheduler_actor.py) / [ray/http_server.py](d:\design\sglang\python\sglang\srt\ray\http_server.py)、[disaggregation/decode.py](d:\design\sglang\python\sglang\srt\disaggregation\decode.py)、[disaggregation/prefill.py](d:\design\sglang\python\sglang\srt\disaggregation\prefill.py)、[disaggregation/encode_receiver.py](d:\design\sglang\python\sglang\srt\disaggregation\encode_receiver.py)、[multiplex/multiplexing_mixin.py](d:\design\sglang\python\sglang\srt\multiplex\multiplexing_mixin.py)、[dllm/mixin/scheduler.py](d:\design\sglang\python\sglang\srt\dllm\mixin\scheduler.py)、[constrained/grammar_manager.py](d:\design\sglang\python\sglang\srt\constrained\grammar_manager.py)、[plugins/hook_registry.py](d:\design\sglang\python\sglang\srt\plugins\hook_registry.py)（可 hook 子类化）、[managers/rust_server.py](d:\design\sglang\python\sglang\srt\managers\rust_server.py) |
| `IdleSleeper` | 定义 [idle_sleeper.py](d:\design\sglang\python\sglang\srt\managers\scheduler_components\idle_sleeper.py)；使用仅 [scheduler.py](d:\design\sglang\python\sglang\srt\managers\scheduler.py)。在 `srt/` 其余子树 grep：无额外生产调用方 |
| `SessionController` | 定义 [session_controller.py](d:\design\sglang\python\sglang\srt\session\session_controller.py)；使用仅 [scheduler.py](d:\design\sglang\python\sglang\srt\managers\scheduler.py)（`init_running_status` / open/close / `maybe_reap` / MM offset）。在 `srt/` 其余子树 grep：无额外生产调用方 |
| `TpModelWorker` / `SchedulePolicy` / `PrefillBootstrapQueue` / `DecodePreallocQueue` | 见各模块页；scheduler 在 `init_model_worker` / `init_schedule_policy` / `init_disaggregation` 组装 |

### 3) 配置 / IPC 共享结构

- `PortArgs.scheduler_input_ipc_name` / `tokenizer_ipc_name` / `detokenizer_ipc_name` / `rpc_ipc_name` / `metrics_ipc_name`：由 [SchedulerIpcChannels.create](d:\design\sglang\python\sglang\srt\managers\scheduler_components\ipc_channels.py) 与 Engine/Tokenizer/Detokenizer 共享（见 [manager-pipeline.md](../topics/manager-pipeline.md)）。
- `ServerArgs` 字段（`disable_overlap_schedule`、`enable_pdmux`、`disaggregation_mode`、`sleep_on_idle` via device bag、`schedule_policy` 等）：在 `__init__` 与 `dispatch_event_loop` 读取（[scheduler.py:417-453](d:\design\sglang\python\sglang\srt\managers\scheduler.py)、[L4902-4930](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）；本期 config bags 重构后，`page_size` / `enable_dp_attention` / `dp_size` / `ep_size` / `pp_size` 等改经 `get_schedule()` / `get_parallel()` / `configured_*()` 读取（详见 §Increment 2026-08-18）。
- env：`SGLANG_SCHEDULER_MAX_RECV_PER_POLL`、`SGLANG_DISABLE_CONSECUTIVE_PREFILL_OVERLAP`、`SGLANG_RUST_SERVER`、`SGLANG_SET_CPU_AFFINITY`、`SGLANG_ENABLE_WAR_BARRIER`、`SGLANG_KILLPG_ON_SCHEDULER_EXCEPTION` 等（分散于 init / event loop / process entry）。

### 4) 测试覆盖反查

- `IdleSleeper` / `SessionController`：在本 upstream 检出树（`/tmp/upstream/sglang`，无顶层 `test/` / `tests/` 套件目录）全树按文件名 `*test*.py` grep 0 命中。
- `Scheduler(` / `run_scheduler_process`：生产调用见上；专用 unit test 文件在本检出中 N/A（verified 2026-08-10，grep 范围：检出树内 `*test*.py`，并排除损坏的 `mem_cache/cpp_radix_tree` 路径）。

### 5) doc / config / yaml 反查

- `IdleSleeper` / `SessionController`：在 [d:\design\sglang\docs\](d:\design\sglang\docs) 下 `*.md`/`*.rst`/`*.yml`/`*.yaml` grep 0 命中（verified 2026-08-10）。
- 顶层 README 仅有产品级 “Batch Scheduler” 表述，无类级 API 文档锚点。

## Increment 2026-08-18 (06f32bab → f7101b0a)

本期 scheduler.py +109/-70（另 scheduler_components/ 多文件小改）。MRO 不变（仍 6 mixin + `SchedulerMlxOverlapMixin`，[scheduler.py:383-390](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）。实质变化：

- **Config bags 重构（#35023/#35025/#35026 系列）**：`run_scheduler_process` 中 `publish(server_args, role="scheduler")` 从 `configure_scheduler_process` **之后**提前到**之前**（现 [L5015](d:\design\sglang\python\sglang\srt\managers\scheduler.py)，紧跟 `load_plugins()`，注释明言 "Publish before anything in this process reads configuration"）。进程内配置读取改走 bags：`self.page_size = get_schedule().page_size`（[L441](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）、`enable_dp_attention` / `dp_size` / `ep_size` 读 `get_parallel()`（[L452](d:\design\sglang\python\sglang\srt\managers\scheduler.py)、[L461-481](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）、`process_input_requests` 读 `get_mm().mm_feature_transport`（[L1880](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）、`dispatch_event_loop` / `configure_scheduler_process` 改用 `configured_pp_size()` / `configured_tp_size()` / `configured_attn_cp_size()` / `configured_moe_dp_size()`（[L4908](d:\design\sglang\python\sglang\srt\managers\scheduler.py)、[L4965-4987](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）。相应地 `SchedulerRecvSkipper.maybe_create()`（[L1275](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）与 `NewTokenRatioTracker.from_config()`（[L1255](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）不再接收 `server_args` 参数。
- **max prefill batch size 跟踪重做（#34284）**：`self.max_prefill_bs` 的 "0.998/pass 衰减 + `max()` 高水位" 机制被删除，改为 `RecentPrefillBatchSizeTracker` 滑窗（[prefill_delayer.py:22-39](d:\design\sglang\python\sglang\srt\managers\prefill_delayer.py)，scheduler 侧初始化 [L1221-1224](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）；`get_new_batch_prefill` 内 `finalize(actual_prefill_bs=...)` 后经 `observe_attempt` 更新（[L3173-3181](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）。
- **ngram 经 FutureMap 中继（#35198）**：`_relay_forward_payload`（[L3869-3884](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）不再跳过 ngram——ngram 的 `next_draft_input` 现以 `RelayPayload.from_ngram` stash 进 `future_map`（[L3873-3877](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）。旧页无此论断，无失效。
- **WAR fastpath 改名 shared-read-done（#34916）**：`_apply_war_barrier`（[L1705-1717](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）读的字段从 `model_worker.war_fastpath_runner.war_fastpath_read_done_event` 改为 `model_worker.last_shared_read_runner.shared_read_done_event`；语义（fast path event wait / coarse `wait_stream` fallback、`SGLANG_FORCE_COARSE_WAR_BARRIER`）不变。
- **delayed-sample D2H 拷贝移至 `copy_stream`（#32637）**：`launch_batch_sample_if_needed` 中 `copy_to_cpu` 现在 `copy_stream.wait_stream(forward_stream)` 后于 `copy_stream_ctx` 内执行（[L3900-3909](d:\design\sglang\python\sglang\srt\managers\scheduler.py)），可与后续 forward 重叠。
- **tracing v2 异步导出（#30023）**：`process_batch_result` 开头新增 `flush_trace_batch(batch.reqs)`（[L3925-3929](d:\design\sglang\python\sglang\srt\managers\scheduler.py)），overlap 模式下该 CPU flush 与下一 batch GPU forward 重叠。
- **启动权重加载与 CUDA graph capture 重叠（#32017）**：`init_model_worker` 在 `is_startup_weight_load_overlap` 时调 `tp_worker.start_startup_weight_load()`（[L996-997](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）/ `finalize_startup_weight_load()`（[L1014-1015](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）；worker 侧实现见 [TpModelWorker.md](TpModelWorker.md)。
- **PD / unified memory 增补（#33362、#34801）**：`init_disaggregation` 新增 `disagg_prefill_pending_chunk_rids: Set[str]`（[L1418-1419](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）与 unified memory 下 `token_to_kv_pool_allocator.set_disagg_move_gate(unified_memory_disagg_move_gate(self))`（[L1423-1430](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）；decode 侧 retraction abort 改走 `retraction_discard(...)`（[L4552-4556](d:\design\sglang\python\sglang\srt\managers\scheduler.py)），不再 `del decode_req.kv_cache_cpu`。

> synthesis: 本期无结构性变化——组合对象拆分格局、事件循环分发、IPC 拓扑均与 `06f32bab` 相同；改动集中在配置读取途径（bags）、prefill delayer 输入信号与若干 overlap 细化。

## Notes / Caveats

> ~~[!warning] CONTRADICTION: 旧 wiki 多页仍写 “Scheduler = **11 mixin**”。HEAD `06f32bab` MRO 为 **6 mixin + `SchedulerMlxOverlapMixin`**；冲突页含 [topics/scheduler-mixins.md](../topics/scheduler-mixins.md)、[modules/managers.md](../modules/managers.md) 等。~~
> **RESOLVED 2026-08-10**: [topics/scheduler-mixins.md](../topics/scheduler-mixins.md) 与 [modules/managers.md](../modules/managers.md) 已与本页对齐（6 mixin + `scheduler_components/`）。[overview.md](../overview.md) / [topics/pd-disaggregation.md](../topics/pd-disaggregation.md) / [modules/disaggregation.md](../modules/disaggregation.md) / [modules/observability.md](../modules/observability.md) / comparison 页仍可能残留旧 11-mixin 表述，需各自 verify。

> ~~[!warning] CONTRADICTION: [modules/managers.md](../modules/managers.md) 仍把 `SchedulerRecvSkipper` 指向已删除的 `scheduler_recv_skipper.py`。~~
> **RESOLVED 2026-08-10**: [modules/managers.md](../modules/managers.md) 已指向 [scheduler_components/recv_skipper.py](d:\design\sglang\python\sglang\srt\managers\scheduler_components\recv_skipper.py)。

> ~~[!todo] VERIFY: `topics/scheduler-mixins.md` 应按 “5 残留 mixin + composition 目录” 重写。~~
> **RESOLVED 2026-08-10**: [topics/scheduler-mixins.md](../topics/scheduler-mixins.md) 已 re-ingest（6 mixin + composition 全表）。

> [!todo] VERIFY: PD disagg event loop 方法体（`event_loop_*_disagg_*`）细节以 mixin 文件为准，未在本页逐行展开；见 [topics/pd-disaggregation.md](../topics/pd-disaggregation.md)（其中行号可能 stale）。

## See also

- [modules/managers.md](../modules/managers.md)
- [entities/TokenizerManager.md](TokenizerManager.md)
- [entities/TpModelWorker.md](TpModelWorker.md)
- [entities/Engine.md](Engine.md)
- [entities/DataParallelController.md](DataParallelController.md)
- [topics/manager-pipeline.md](../topics/manager-pipeline.md)
- [topics/request-lifecycle.md](../topics/request-lifecycle.md)
- [topics/scheduler-mixins.md](../topics/scheduler-mixins.md) — **DONE**（re-ingest 2026-08-10：6 mixin + `scheduler_components/`）
- [topics/pd-disaggregation.md](../topics/pd-disaggregation.md)
- [topics/kv-cache.md](../topics/kv-cache.md) — `build_kv_cache` / `registry` 工厂矩阵（re-ingest 2026-08-10）
- [modules/mem_cache.md](../modules/mem_cache.md) — mem_cache 模块全景（stale；以 kv-cache topic 为准）
- [comparison/topics/scheduler-architecture.md](../../comparison/topics/scheduler-architecture.md)
