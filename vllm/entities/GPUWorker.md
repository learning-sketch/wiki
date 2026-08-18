---
type: entity
project: vllm
status: verified
confidence: high
verified_against: 2026-08-18 (vLLM d29dc3ab, increment from 5f7fab88)
sources:
  - d:\design\vllm\vllm\v1\worker\gpu_worker.py
  - d:\design\vllm\vllm\v1\worker\worker_base.py
  - d:\design\vllm\vllm\v1\worker\gpu_model_runner.py:4294-4299,4673-4676,7766-7770,7906-7913
  - d:\design\vllm\vllm\v1\executor\abstract.py:120-280
  - d:\design\vllm\vllm\v1\executor\multiproc_executor.py:340-373,639-702,1029-1054
related:
  - vllm/entities/MultiprocExecutor.md
  - vllm/entities/GPUModelRunner.md
  - vllm/entities/OutputProcessor.md
  - vllm/entities/EngineCore.md
  - vllm/topics/multiproc-ipc.md
  - vllm/topics/request-lifecycle.md
  - vllm/modules/executor.md
---

# `Worker` (v1 GPU worker)

## Summary
[`Worker`](d:\design\vllm\vllm\v1\worker\gpu_worker.py) 继承 [`WorkerBase`](d:\design\vllm\vllm\v1\worker\worker_base.py)，在单进程内持有 [`GPUModelRunner`](d:\design\vllm\vllm\v1\worker\gpu_model_runner.py)（V1/V2 由 `VllmConfig.use_v2_model_runner` 切换，见增量小节），负责 CUDA 设备与 `init_worker_distributed_environment`、KV 显存剖析、KV 分配与 warmup/CUDAGraph 捕获，并把调度器下发的 [`SchedulerOutput`](d:\design\vllm\vllm\v1\core\sched\output.py) 交给 `GPUModelRunner.execute_model` / `sample_tokens`。多进程场景下 **不**直接操作 [`MessageQueue`](d:\design\vllm\vllm\distributed\device_communicators\shm_broadcast.py)；IPC 由 [`MultiprocExecutor`](d:\design\vllm\vllm\v1\executor\multiproc_executor.py) 侧的 `WorkerProc` 在 busy loop 中 `dequeue` 后 `getattr(self.worker, method)` 调用（见 [multiproc_executor.py:1029-1054](d:\design\vllm\vllm\v1\executor\multiproc_executor.py)）。

## Sources
- 全文：[d:\design\vllm\vllm\v1\worker\gpu_worker.py](d:\design\vllm\vllm\v1\worker\gpu_worker.py)（1439 行）
- 基类：[d:\design\vllm\vllm\v1\worker\worker_base.py](d:\design\vllm\vllm\v1\worker\worker_base.py)
- `GPUModelRunner` 关键 API：[d:\design\vllm\vllm\v1\worker\gpu_model_runner.py](d:\design\vllm\vllm\v1\worker\gpu_model_runner.py) — `execute_model` [4294-4299](d:\design\vllm\vllm\v1\worker\gpu_model_runner.py)、`sample_tokens` [4673-4676](d:\design\vllm\vllm\v1\worker\gpu_model_runner.py)、`initialize_kv_cache` [7766-7770](d:\design\vllm\vllm\v1\worker\gpu_model_runner.py)、`get_kv_cache_spec` [7906-7913](d:\design\vllm\vllm\v1\worker\gpu_model_runner.py)
- Executor RPC 抽象：[d:\design\vllm\vllm\v1\executor\abstract.py](d:\design\vllm\vllm\v1\executor\abstract.py)
- 多进程 worker 进程与 busy loop：[d:\design\vllm\vllm\v1\executor\multiproc_executor.py](d:\design\vllm\vllm\v1\executor\multiproc_executor.py)

## 类层次（mermaid classDiagram）

```mermaid
classDiagram
    class WorkerBase {
        +vllm_config
        +parallel_config
        +device_config
        +cache_config
        +model_runner
        +execute_model(scheduler_output)
        +sample_tokens(grammar_output)
    }
    class Worker {
        +elastic_ep_executor
        +worker_sentinel
        +weight_transfer_engine
        +profiler
        +_pp_send_work
        +_sleep_saved_buffers
        +model_runner : GPUModelRunner
        +init_device()
        +load_model(load_dummy_weights)
        +determine_available_memory()
        +initialize_from_config(kv_cache_config)
        +compile_or_warm_up_model()
        +execute_model(scheduler_output)
        +sample_tokens(grammar_output)
        +profile(is_start, profile_prefix)
        +shutdown()
    }
    WorkerBase <|-- Worker
```

| 字段 / 组件 | 说明 | 锚点 |
|-------------|------|------|
| `elastic_ep_executor` | `ElasticEPScalingExecutor`，`elastic_ep_execute` 委托入口 | [gpu_worker.py:163-165, 1390-1394](d:\design\vllm\vllm\v1\worker\gpu_worker.py) |
| `worker_sentinel` | `WorkerSentinel`，`enable_fault_tolerance` 时创建，`handle_ft_command` 入口（本期新增，见增量小节） | [gpu_worker.py:166-168, 444-446](d:\design\vllm\vllm\v1\worker\gpu_worker.py) |
| `weight_transfer_engine` | 在线权重更新引擎，可为 `None`；`__init__` 只置 `None`，实际在 `load_model` 中创建（本期变更） | [gpu_worker.py:173-177, 459-464](d:\design\vllm\vllm\v1\worker\gpu_worker.py) |
| `profiler` | `TorchProfilerWrapper` / `CudaProfilerWrapper` / `ProtonProfilerWrapper`，惰性创建 | [gpu_worker.py:179-183, 1146-1208](d:\design\vllm\vllm\v1\worker\gpu_worker.py) |
| `_pp_send_work` | PP 非最后 stage 时 `isend_tensor_dict` 返回的 `Handle` 列表，下一轮 `execute_model` 开头 `wait` | [gpu_worker.py:186-187, 1057-1060, 1135-1139](d:\design\vllm\vllm\v1\worker\gpu_worker.py) |
| `model_runner` | V1 [`GPUModelRunner`](d:\design\vllm\vllm\v1\worker\gpu_model_runner.py) 或 V2 [`gpu/model_runner.GPUModelRunner`](d:\design\vllm\vllm\v1\worker\gpu\model_runner.py)，由 `vllm_config.use_v2_model_runner` 决定（本期机制变更，见增量小节） | [gpu_worker.py:185, 423-438](d:\design\vllm\vllm\v1\worker\gpu_worker.py) |

## __init__ 与设备初始化

- **基类状态**：`WorkerBase.__init__` 保存 `vllm_config` 及子配置引用，设置 `rank` / `local_rank` / `distributed_init_method`，`model_runner` 与 `device` 初值为 `None`（[worker_base.py:45-96](d:\design\vllm\vllm\v1\worker\worker_base.py)）。
- **精度与弹性 EP**：`torch.set_float32_matmul_precision`（[gpu_worker.py:159-161](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）；构造 `ElasticEPScalingExecutor(self)`（[gpu_worker.py:163-165](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）。
- **sleep 缓冲、权重传输、profiler 配置**：`_sleep_saved_buffers` 与 `_sleep_saved_draft_buffers`（[gpu_worker.py:169-171](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）；`weight_transfer_engine` 占位（[gpu_worker.py:173-177](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）；`profiler` 占位与 `profiler_config`（[gpu_worker.py:179-183](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）。
- **`init_device`（CUDA）**：清理 `NCCL_ASYNC_ERROR_HANDLING`（[gpu_worker.py:316-318](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）；在单节点非 Ray 等条件下按 DP 调整 `local_rank` 并校验 `assigned_physical_gpu_ids`（[gpu_worker.py:319-373](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）；`torch.device` 与 `set_device_index`（[gpu_worker.py:375-379](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）；调用 `init_worker_distributed_environment`（内部再调 `init_distributed_environment`、`ensure_model_parallel_initialized`、`ensure_ec_transfer_initialized` 等）（[gpu_worker.py:383-393, 1397-1439](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）；`set_random_seed`、`MemorySnapshot` 与 `request_memory`（[gpu_worker.py:398-411](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）；`init_workspace_manager`（DBO 时 2 个 ubatch，DSpark 时 2 条 workspace lane，[gpu_worker.py:95-101, 415-421](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）；构造 `GPUModelRunner`（V1/V2 二选一，[gpu_worker.py:423-438](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）；rank 0 时 `report_usage_stats`（[gpu_worker.py:440-442](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）。

## execute_model 主流程

```mermaid
sequenceDiagram
    participant EC as EngineCore / Scheduler
    participant EX as MultiprocExecutor
    participant MQ as rpc_broadcast_mq
    participant WP as WorkerProc.worker_busy_loop
    participant WW as WorkerWrapperBase
    participant W as Worker
    participant MR as GPUModelRunner

    EC->>EX: execute_model(scheduler_output)
    EX->>MQ: enqueue execute_model
    MQ->>WP: dequeue
    WP->>WW: execute_model(scheduler_output)
    Note over WW: _apply_mm_cache 后转发
    WW->>W: execute_model(scheduler_output)
    W->>W: wait _pp_send_work; PP irecv
    W->>MR: execute_model(scheduler_output, intermediate_tensors)
    MR-->>W: ModelRunnerOutput | IntermediateTensors | None
    alt PP 非末段
        W->>W: isend_tensor_dict; return None
    else 末段或单卡
        W-->>WW: ModelRunnerOutput | AsyncModelRunnerOutput | None
    end
    WP->>EX: handle_output via response_mq
```

- **PP 与异步中间张量**：先 `wait` 上一轮 `_pp_send_work`（[gpu_worker.py:1057-1060](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）；非首 PP rank 时 `irecv_tensor_dict` 包进 `AsyncIntermediateTensors`（[gpu_worker.py:1098-1110](d:\design\vllm\vllm\v1\worker\gpu_worker.py)，类定义 [gpu_worker.py:110-139](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）；`model_runner.execute_model`（[gpu_worker.py:1112-1115](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）；若返回 `IntermediateTensors` 则非末段 PP `isend_tensor_dict` 并 `return None`（[gpu_worker.py:1127-1141](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）。
- **Profiling 注解**：`annotate_profile` 包装迭代（[gpu_worker.py:931-1044, 1112](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）；`execute_model` 另有 `@with_gpu_sync_check` 装饰器（`VLLM_GPU_SYNC_CHECK`，本期新增，[gpu_worker.py:1051-1052](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）。
- **返回**：与 `WorkerBase.execute_model` 约定一致，可为 `ModelRunnerOutput` / `AsyncModelRunnerOutput` / `None`（[worker_base.py:146-155](d:\design\vllm\vllm\v1\worker\worker_base.py)）；Executor 侧通过 `MultiprocExecutor.execute_model` 的 `unique_reply_rank` 只收输出 rank（[multiproc_executor.py:340-351](d:\design\vllm\vllm\v1\executor\multiproc_executor.py)）。

## KV cache 初始化与 profile

| 阶段 | 行为 | 锚点 |
|------|------|------|
| 可用显存 | `determine_available_memory`：`kv_cache_memory_bytes` 短路仍 `profile_run`；否则 `memory_profiling` + `profile_run` + 可选 CUDAGraph 显存估计；本期新增 opt-in 的画像结果持久化复用 | [gpu_worker.py:475-647](d:\design\vllm\vllm\v1\worker\gpu_worker.py) |
| KV 规格 | `get_kv_cache_spec` 委托 `model_runner` | [gpu_worker.py:649-650](d:\design\vllm\vllm\v1\worker\gpu_worker.py) |
| 分配 | `initialize_from_config`：更新 `cache_config.num_gpu_blocks`、`ensure_kv_transfer_initialized`、`model_runner.initialize_kv_cache` 等 | [gpu_worker.py:664-692](d:\design\vllm\vllm\v1\worker\gpu_worker.py) |
| Warmup / 编译 | `compile_or_warm_up_model`：dummy run、`kernel_warmup`、`capture_model`、V2 `warmup_kernels` 等；现返回 `CompilationTimes`（本期变更） | [gpu_worker.py:694-929](d:\design\vllm\vllm\v1\worker\gpu_worker.py) |

Executor 侧本期已拆分：`Executor.initialize_from_config` 只 RPC `initialize_from_config`（[abstract.py:120-122](d:\design\vllm\vllm\v1\executor\abstract.py)），`Executor.compile_or_warm_up_model` 单独 RPC 并聚合各 worker 的 `CompilationTimes`（[abstract.py:124-139](d:\design\vllm\vllm\v1\executor\abstract.py)）。

## hidden state（§9）

| 类别 | 列表 | 行号（锚点） |
|------|------|----------------|
| CUDA Stream / Event | `Worker` 类体内 **无** 直接 `torch.cuda.Stream` / `Event`；异步 copy 流在 `GPUModelRunner` 的 `AsyncGPUModelRunnerOutput`（[gpu_model_runner.py:292-324](d:\design\vllm\vllm\v1\worker\gpu_model_runner.py)） | `gpu_worker.py` 内 grep `Stream(`/`Event(`：**0**（2026-08-18 复核） |
| 后台线程 | **无**；多进程下异步输出线程在 `WorkerProc.__init__`（`async_output_copy_thread`）（[multiproc_executor.py:685-691](d:\design\vllm\vllm\v1\executor\multiproc_executor.py)），非 `Worker` 类 | — |
| 跨进程 IPC | `Worker` **不**持有 `MessageQueue`；子进程内为 `WorkerProc` 的 `rpc_broadcast_mq` / `worker_response_mq`（见 [MultiprocExecutor.md](MultiprocExecutor.md)） | [multiproc_executor.py:1029-1040](d:\design\vllm\vllm\v1\executor\multiproc_executor.py) |
| Profiler 对象 | `self.profiler`：`TorchProfilerWrapper` / `CudaProfilerWrapper` / `ProtonProfilerWrapper` | [gpu_worker.py:179-183, 1146-1208](d:\design\vllm\vllm\v1\worker\gpu_worker.py) |
| 分布式异步句柄 | `_pp_send_work: list[Handle]`（PP send） | [gpu_worker.py:186-187, 1135-1139](d:\design\vllm\vllm\v1\worker\gpu_worker.py) |
| Sleep 模式缓冲 | `_sleep_saved_buffers` / `_sleep_saved_draft_buffers`；sleep 后端经 `SleepModeBackendFactory` 惰性解析（本期新增抽象） | [gpu_worker.py:169-171, 189-195](d:\design\vllm\vllm\v1\worker\gpu_worker.py) |

## 与 MultiprocExecutor 的协议方法清单

`synthesis:` 下列字符串方法由 `Executor` / `MultiprocExecutor` 通过 `collective_rpc` 调度到 **worker 包装器**（`WorkerWrapperBase`），再落到 `Worker` 或其基类；`WorkerProc.worker_busy_loop` 用 `getattr(self.worker, method)` 解析字符串（[multiproc_executor.py:1029-1054](d:\design\vllm\vllm\v1\executor\multiproc_executor.py)）。

**`Executor` 抽象层常用 RPC（方法名字符串）**（[abstract.py:120-334](d:\design\vllm\vllm\v1\executor\abstract.py)）：`initialize_from_config`、`compile_or_warm_up_model`、`determine_available_memory`、`get_kv_cache_spec`、`get_kv_connector_handshake_metadata`、`execute_model`、`sample_tokens`、`execute_dummy_batch`、`take_draft_token_ids`、`profile`、`shutdown`、`get_supported_tasks`、`supports_draft_weight_updates`（本期新增）、`add_lora`、`remove_lora`、`pin_lora`、`list_loras`、`reset_mm_cache`、`reset_encoder_cache`、`sleep`、`wake_up`、`save_sharded_state`。

**`MultiprocExecutor` 显式覆盖**（[multiproc_executor.py:340-373, 537-539](d:\design\vllm\vllm\v1\executor\multiproc_executor.py)）：`execute_model`、`sample_tokens`、`execute_dummy_batch`、`take_draft_token_ids`、`check_health`。

**其它入口（经 `EngineCore` / `LLMEngine` / 工具链 RPC 到同一 worker）**：`update_max_model_len`（[core.py:316](d:\design\vllm\vllm\v1\engine\core.py)，worker 实现 [gpu_worker.py:652-663](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）；`apply_model`（[llm_engine.py:437-438](d:\design\vllm\vllm\v1\engine\llm_engine.py)）；`get_model_inspection`（[llm.py:913](d:\design\vllm\vllm\entrypoints\llm.py)）；`get_encoder_timing_stats`（[benchmarks/mm_processor.py:82](d:\design\vllm\vllm\benchmarks\mm_processor.py)）；`init_weight_transfer_engine` / `update_weights`（[async_llm.py:1122-1151](d:\design\vllm\vllm\v1\engine\async_llm.py)）；`elastic_ep_execute`（[elastic_state.py:127-213](d:\design\vllm\vllm\distributed\elastic_ep\elastic_state.py) 等）。

**多进程子进程构造时直接调用（非 `collective_rpc`）**：`WorkerProc.__init__` 中 `wrapper.init_worker` → `init_device` → `load_model` 或 `elastic_ep_execute("load_model")`（[multiproc_executor.py:651-680](d:\design\vllm\vllm\v1\executor\multiproc_executor.py)）。

**`Worker` 已实现但非上述字符串清单的常见扩展**：`update_config`、`reload_weights`（[gpu_worker.py:467-473](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）、`save_tensorized_model`（[gpu_worker.py:1245](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）、`handle_ft_command`（[gpu_worker.py:444-446](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）等，供 Ray/容错/其它路径或 `collective_rpc` 透传调用。

## 与 GPUModelRunner 的关系

- **归属**：`Worker.init_device` 构造 `self.model_runner`（[gpu_worker.py:423-438](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）。
- **前向与采样**：`execute_model` / `sample_tokens` 委托 `GPUModelRunner`（[gpu_worker.py:1046-1049, 1112-1115](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）；`GPUModelRunner.execute_model` 签名见 [gpu_model_runner.py:4294-4299](d:\design\vllm\vllm\v1\worker\gpu_model_runner.py)。

## §5 step 3 hidden cross-reference grep 结果

| # | 类别 | 结论 |
|---|------|------|
| 1 | 跨语言绑定 | `gpu_worker` / `GPUWorker` / `init_worker_distributed_environment`：在 `d:\design\vllm\csrc\` 全 C++/CUDA 树 grep **0** 命中（无针对 `Worker` 的 pybind 名）。 |
| 2 | 协作伙伴跨子系统 | `vllm.v1.worker.gpu_worker`：在 `d:\design\vllm\vllm\` 全 `*.py` 树 **13** 命中（10 个文件）；`d:\design\vllm\vllm\v1\executor\` 内对该模块路径 grep **0** 命中（Executor 经 `worker_cls` 字符串与 `WorkerWrapperBase` 动态构造 worker，见 [worker_base.py:191-305](d:\design\vllm\vllm\v1\worker\worker_base.py)）。`init_worker_distributed_environment`：在 `d:\design\vllm\vllm\` 全 `*.py` 树 **6** 命中（`gpu_worker` / `cpu_worker` / `xpu_worker`）。`d:\design\vllm\vllm\entrypoints\` 全树对上述模块路径 grep **0** 命中。`d:\design\vllm\benchmarks\` 全树 grep **0** 命中。（2026-04-18 结果；本期未重扫，见 VERIFY） |
| 3 | 配置 / IPC 共享结构 | `ParallelConfig` / `SchedulerOutput` / `ModelRunnerOutput` / `KVCacheConfig` / `DeviceConfig`：在 `d:\design\vllm\vllm\` 全 `*.py` 树均为**大量**跨模块命中（数十至上百文件级，IDE 全局统计为准）。`MessageQueue`：在 `d:\design\vllm\vllm\v1\worker\` 全树 grep **0** 命中；worker 子进程收包在 `WorkerProc`（[multiproc_executor.py:1029-1040](d:\design\vllm\vllm\v1\executor\multiproc_executor.py)）。 |
| 4 | 测试覆盖 | `tests/v1/worker/test_gpu_worker.py`：**不存在**（glob 0）。导入 `vllm.v1.worker.gpu_worker` 的测试文件包括 [test_worker_memory_snapshot.py](d:\design\vllm\tests\v1\worker\test_worker_memory_snapshot.py)、[test_abort_final_step.py](d:\design\vllm\tests\v1\engine\test_abort_final_step.py)、[test_worker.py](d:\design\vllm\tests\lora\test_worker.py)、[test_weight_transfer_llm.py](d:\design\vllm\tests\entrypoints\weight_transfer\test_weight_transfer_llm.py)、[test_comm_ops.py](d:\design\vllm\tests\distributed\test_comm_ops.py)（`AsyncIntermediateTensors`）。（2026-04-18 结果；本期未重扫，见 VERIFY） |
| 5 | doc / config | `gpu_worker.py` / `gpu_worker`：在 `d:\design\vllm\docs\` 全树 grep **1** 命中（[arch_overview.md:107](d:\design\vllm\docs\design\arch_overview.md)）。（2026-04-18 结果；本期未重扫，见 VERIFY） |

## Increment 2026-08-18 (vLLM 5f7fab88 → d29dc3ab)

本期 `gpu_worker.py` 1060 → 1439 行（53 个 commit，+559/-180 量级），全页行号锚点已按新版重排。要点：

- **V1/V2 model runner 切换机制变更**：旧 pin 时 worker 直接读 env（`self.use_v2_model_runner = envs.VLLM_USE_V2_MODEL_RUNNER`，env 默认 `0`、纯 opt-in）；现改为读 `vllm_config.use_v2_model_runner`（[d:\design\vllm\vllm\v1\worker\gpu_worker.py:L185](d:\design\vllm\vllm\v1\worker\gpu_worker.py)），该属性在 [d:\design\vllm\vllm\config\vllm.py:L614-660](d:\design\vllm\vllm\config\vllm.py) 中自动判定——env `VLLM_USE_V2_MODEL_RUNNER` 显式设置时优先（env 默认已改为 `None`，[d:\design\vllm\vllm\envs.py:L294](d:\design\vllm\vllm\envs.py)、[L2030-2031](d:\design\vllm\vllm\envs.py)）；否则 PCP>1、`dspark` 投机、DFlash 混合 KV group、diffusion 模型**强制 V2**；其余按默认 V2 架构白名单 + Triton 可用性 + 不支持特性清单自动选择。V2 已从"实验性 opt-in"变为**受支持架构上的默认选择**。细节与 V2 包结构见 [GPUModelRunner.md](GPUModelRunner.md) 增量小节。
- **容错框架（fault tolerance）**：新增 `WorkerSentinel`（`parallel_config.enable_fault_tolerance` 时构造，[d:\design\vllm\vllm\v1\worker\gpu_worker.py:L166-168](d:\design\vllm\vllm\v1\worker\gpu_worker.py)），`handle_ft_command` RPC 入口（[d:\design\vllm\vllm\v1\worker\gpu_worker.py:L444-446](d:\design\vllm\vllm\v1\worker\gpu_worker.py)），实现在 [d:\design\vllm\vllm\v1\worker\sentinel\gpu_worker_sentinel.py](d:\design\vllm\vllm\v1\worker\sentinel\gpu_worker_sentinel.py)（新目录 `v1/worker/sentinel/`）。
- **权重传输引擎构造时机后移**：`weight_transfer_engine` 从 `__init__` 移到 `load_model`（需要 model 引用，[d:\design\vllm\vllm\v1\worker\gpu_worker.py:L173-177](d:\design\vllm\vllm\v1\worker\gpu_worker.py)、[L459-464](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）；`load_model` 新增 `load_dummy_weights` 参数（[L450](d:\design\vllm\vllm\v1\worker\gpu_worker.py)），支持投机 draft 权重运行时更新。
- **Sleep 模式**：新增可插拔 sleep 后端抽象（`_get_sleep_mode_backend` / `SleepModeBackendFactory`，[d:\design\vllm\vllm\v1\worker\gpu_worker.py:L189-195](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）；level-2 sleep 保留 draft 缓冲 `_sleep_saved_draft_buffers`（[L171](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）。
- **Profiler**：新增 Triton Proton 后端 `ProtonProfilerWrapper`（[d:\design\vllm\vllm\v1\worker\gpu_worker.py:L1185-1186](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）；`profile()` 按 `profiler_config.profiler` 三选一（[L1146-1208](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）。
- **执行路径**：`execute_model` 增加 `@with_gpu_sync_check`（env `VLLM_GPU_SYNC_CHECK`，CI 检测 GPU↔CPU 同步，[d:\design\vllm\vllm\v1\worker\gpu_worker.py:L1051-1052](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）；V2 pooling 模型 `output is None` 时补 `model_runner.pool()`（[L1116-1121](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）。
- **Workspace 管理**：`init_workspace_manager` 增加 lane 维度——DSpark（V2）需要 2 条 workspace lane（[d:\design\vllm\vllm\v1\worker\gpu_worker.py:L95-101](d:\design\vllm\vllm\v1\worker\gpu_worker.py)、[L415-421](d:\design\vllm\vllm\v1\worker\gpu_worker.py)）。
- **显存画像**：`determine_available_memory` 支持画像结果跨启动持久化复用（opt-in，fast-startup），区间扩至 [d:\design\vllm\vllm\v1\worker\gpu_worker.py:L475-647](d:\design\vllm\vllm\v1\worker\gpu_worker.py)。
- **Executor 协议拆分**：`Executor.initialize_from_config` 不再顺带触发 warmup；`compile_or_warm_up_model` 独立 RPC 并回传聚合 `CompilationTimes`（[d:\design\vllm\vllm\v1\executor\abstract.py:L120-139](d:\design\vllm\vllm\v1\executor\abstract.py)；worker 端返回值 [d:\design\vllm\vllm\v1\worker\gpu_worker.py:L694](d:\design\vllm\vllm\v1\worker\gpu_worker.py)、[d:\design\vllm\vllm\v1\worker\worker_base.py:L102-108](d:\design\vllm\vllm\v1\worker\worker_base.py)）。

## Notes / Caveats

> [!todo] VERIFY: **`reset_prefix_cache`** 引擎与调度器侧 API（[core.py:786](d:\design\vllm\vllm\v1\engine\core.py)、[scheduler.py:2511](d:\design\vllm\vllm\v1\core\sched\scheduler.py)），**不是** `Worker` 上的 RPC；KV 块池在 scheduler / `kv_cache_manager` 重置。

> [!warning] CONTRADICTION: **Profiling API 名**对外为 `profile(is_start, profile_prefix)`（[gpu_worker.py:1146-1208](d:\design\vllm\vllm\v1\worker\gpu_worker.py)），Executor 亦 RPC `"profile"`（[abstract.py:258-259](d:\design\vllm\vllm\v1\executor\abstract.py)），与 `LLMEngine.start_profile` / `stop_profile` 字面名不一致；后者在 [llm_engine.py:338-341](d:\design\vllm\vllm\v1\engine\llm_engine.py) 是 client 侧高层 API，并非 worker 直接方法名。

> [!todo] VERIFY: **`get_cache_block_size_bytes`** 仍仅在 `WorkerBase` 声明为 `NotImplementedError`（[worker_base.py:163-167](d:\design\vllm\vllm\v1\worker\worker_base.py)），`gpu_worker.py` **未**覆盖。是否被实际调用需在 v1 路径上确认。

> [!todo] VERIFY: 本次 2026-08-18 增量复核了 `gpu_worker.py` / `worker_base.py` / `multiproc_executor.py` / `abstract.py` 主锚点与"其它入口"跨文件锚点；§hidden cross-reference 表中标注"2026-04-18 结果"的全仓 grep 统计（命中数 / 文件清单）未按 HEAD d29dc3ab 重跑，仅确认主要论断（Executor 不直接 import gpu_worker、worker 无 MessageQueue）仍成立。

## See also

- [vllm/entities/MultiprocExecutor.md](MultiprocExecutor.md)
- [vllm/entities/GPUModelRunner.md](GPUModelRunner.md)
- [vllm/entities/OutputProcessor.md](OutputProcessor.md)
- [vllm/entities/EngineCore.md](EngineCore.md)
- [vllm/topics/multiproc-ipc.md](../topics/multiproc-ipc.md)
- [vllm/topics/request-lifecycle.md](../topics/request-lifecycle.md)
- [vllm/modules/executor.md](../modules/executor.md)
