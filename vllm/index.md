---
type: index
project: vllm
status: verified
confidence: high
verified_against: 2026-08-18 (vLLM d29dc3ab, increment from 5f7fab88)
sources:
  - d:\design\vllm\vllm
related:
  - ../index.md
  - overview.md
  - ../comparison/index.md
---

# vLLM Index

> 项目内所有 wiki 页的目录。
> 入口架构鸟瞰：[overview.md](overview.md)。
> 主代码根：[d:\design\vllm\vllm\](d:\design\vllm\vllm)（重点 [v1/](d:\design\vllm\vllm\v1)）。

---

## Modules

### v1 子系统（重点）

| 模块 | wiki 页 | 状态 |
|---|---|---|
| `v1/engine` | [modules/engine.md](modules/engine.md) | **DONE**（verify 2026-08-18 @d29dc3ab：14 .py 无增删；增量主线 fault tolerance / graceful shutdown / EEP 两阶段 / LB 重写） |
| `v1/executor` | [modules/executor.md](modules/executor.md) | **DONE**（verify 2026-08-18：`"ray"` 默认实现翻转为 `RayExecutorV2`；`initialize_from_config`/`compile_or_warm_up_model` 拆分；+`vllm_net_devices.py`） |
| `v1/core` (scheduler) | `modules/core_scheduler.md` | TODO |
| `v1/core` (kv_cache) | `modules/core_kv_cache.md` | TODO |
| `v1/worker` | `modules/worker.md` | TODO |
| `v1/attention` | `modules/attention.md` | TODO |
| `v1/sample` | `modules/sample.md` | TODO |
| `v1/structured_output` | `modules/structured_output.md` | TODO |
| `v1/spec_decode` | `modules/spec_decode.md` | TODO |
| `v1/kv_offload` | `modules/kv_offload.md` | TODO |
| `v1/pool` | `modules/pool.md` | TODO |
| `v1/metrics` | `modules/metrics.md` | TODO |

### 顶层模块

| 模块 | wiki 页 | 状态 |
|---|---|---|
| `entrypoints` | `modules/entrypoints.md` | TODO |
| `entrypoints/serve/disagg` | `modules/entrypoints_disagg.md` | TODO |
| `model_executor` | `modules/model_executor.md` | TODO |
| `attention` (旧 / 通用) | `modules/attention_legacy.md` | TODO |
| `distributed` | `modules/distributed.md` | TODO |
| `compilation` | `modules/compilation.md` | TODO |
| `kernels` | `modules/kernels.md` | TODO |
| `multimodal` | `modules/multimodal.md` | TODO |
| `lora` | `modules/lora.md` | TODO |
| `reasoning` | `modules/reasoning.md` | TODO |
| `tool_parsers` | `modules/tool_parsers.md` | TODO |
| `device_allocator` | `modules/device_allocator.md` | TODO |
| `platforms` | `modules/platforms.md` | TODO |
| `plugins` | `modules/plugins.md` | TODO |

---

## Entities（关键 class）

| 实体 | 源文件 | wiki 页 | 状态 |
|---|---|---|---|
| `EngineCore` | [v1/engine/core.py](d:\design\vllm\vllm\v1\engine\core.py) | [entities/EngineCore.md](entities/EngineCore.md) | **DONE**（verify 2026-08-18 @d29dc3ab：2536 行；类层次不变；+`EngineShutdownState` 优雅停机、`EngineCoreSentinel` 容错、两阶段 pause/sleep） |
| `MultiprocExecutor` | [v1/executor/multiproc_executor.py](d:\design\vllm\vllm\v1\executor\multiproc_executor.py) | [entities/MultiprocExecutor.md](entities/MultiprocExecutor.md) | **DONE**（verify 2026-08-18：1122 行；file:// rendezvous、EC 链式聚合；`max_concurrent_batches` 迁至 VllmConfig） |
| `Executor` (abstract) | [v1/executor/abstract.py](d:\design\vllm\vllm\v1\executor\abstract.py) | `entities/Executor.md` | TODO (P2) |
| `UniProcExecutor` | [v1/executor/uniproc_executor.py](d:\design\vllm\vllm\v1\executor\uniproc_executor.py) | `entities/UniProcExecutor.md` | TODO (P2) |
| `RayExecutor` | [v1/executor/ray_executor.py](d:\design\vllm\vllm\v1\executor\ray_executor.py) | `entities/RayExecutor.md` | TODO (P2) |
| `RayExecutorV2` | [v1/executor/ray_executor_v2.py](d:\design\vllm\vllm\v1\executor\ray_executor_v2.py) | `entities/RayExecutorV2.md` | TODO (P2) |
| `LLMEngine` (v1) | [v1/engine/llm_engine.py](d:\design\vllm\vllm\v1\engine\llm_engine.py) | [entities/LLMEngine.md](entities/LLMEngine.md) | **DONE** (P0；verify 2026-08-18：457 行；docstring 明示 Legacy；`use_cached_outputs` 参数已移除) |
| `AsyncLLM` | [v1/engine/async_llm.py](d:\design\vllm\vllm\v1\engine\async_llm.py) | [entities/AsyncLLM.md](entities/AsyncLLM.md) | **DONE** (P0；verify 2026-08-18：1166 行；+EEP 串行锁、`handle_fault`/`get_status`、PauseMode 签名) |
| `EngineCoreClient` | [v1/engine/core_client.py](d:\design\vllm\vllm\v1\engine\core_client.py) | [entities/EngineCoreClient.md](entities/EngineCoreClient.md) | **DONE** (P0)（含 `InprocClient` / `MPClient` / `SyncMPClient` / `AsyncMPClient` / `DPAsyncMPClient` / `DPLBAsyncMPClient` 6 子类） |
| `Coordinator` | [v1/engine/coordinator.py](d:\design\vllm\vllm\v1\engine\coordinator.py) | `entities/Coordinator.md` | TODO (P2) |
| `OutputProcessor` | [v1/engine/output_processor.py](d:\design\vllm\vllm\v1\engine\output_processor.py) | [entities/OutputProcessor.md](entities/OutputProcessor.md) | **DONE** (P1)（含 `RequestState` / `RequestOutputCollector`） |
| `Scheduler` (v1) | [v1/core/sched/scheduler.py](d:\design\vllm\vllm\v1\core\sched\scheduler.py) | [entities/Scheduler.md](entities/Scheduler.md) | **DONE**（含 `AsyncScheduler` / `RequestQueue`） |
| `AsyncScheduler` | [v1/core/sched/async_scheduler.py](d:\design\vllm\vllm\v1\core\sched\async_scheduler.py) | 见 [entities/Scheduler.md](entities/Scheduler.md) | DONE（合并） |
| `KVCacheManager` | [v1/core/kv_cache_manager.py](d:\design\vllm\vllm\v1\core\kv_cache_manager.py) | [entities/KVCacheManager.md](entities/KVCacheManager.md) | **DONE**（含 `BlockPool` / `BlockHashToBlockMap` / `KVCacheCoordinator` / `KVCacheSpec` 家族） |
| `BlockPool` | [v1/core/block_pool.py](d:\design\vllm\vllm\v1\core\block_pool.py) | 见 [entities/KVCacheManager.md](entities/KVCacheManager.md) | DONE（合并） |
| `GPUWorker` (Worker) | [v1/worker/gpu_worker.py](d:\design\vllm\vllm\v1\worker\gpu_worker.py) | [entities/GPUWorker.md](entities/GPUWorker.md) | **DONE** (P1；verify 2026-08-18：1439 行；V1/V2 runner 二选一构造 L423-438；+WorkerSentinel 容错) |
| `GPUModelRunner` | [v1/worker/gpu_model_runner.py](d:\design\vllm\vllm\v1\worker\gpu_model_runner.py) | [entities/GPUModelRunner.md](entities/GPUModelRunner.md) | **DONE** / **stale@d29dc3ab**（verify 2026-08-18：8018 行，主锚点已重排；**V2 runner（`worker/gpu/`）已转正为白名单架构默认**（`use_v2_model_runner` @config/vllm.py:614-660）；细粒度区间未逐行复核，VERIFY 已标；V2 建议未来单独建页） |

---

## Topics（跨模块主题）

| 主题 | wiki 页 | 状态 |
|---|---|---|
| **Request lifecycle**（端到端） | [topics/request-lifecycle.md](topics/request-lifecycle.md) | **DONE** / **stale@d29dc3ab**（verify 2026-08-18：主链路 add_request→step→collective_rpc→output ~30 锚点核准；跨进程统计表/时序图次级细节未逐项复核，VERIFY 已标） |
| **多进程 IPC** | [topics/multiproc-ipc.md](topics/multiproc-ipc.md) | **DONE**（verify 2026-08-18：5 类 IPC 分工不变；+EC 聚合回程、异步输出解包统一） |
| Continuous batching / chunked prefill | `topics/continuous-batching.md` | TODO |
| Paged attention / KV block | `topics/paged-attention.md` | TODO |
| Prefix caching | [topics/prefix-cache.md](topics/prefix-cache.md) | **DONE** / **stale@d29dc3ab**（verify 2026-08-18：核心链路 ~70 锚点核准；`spec_manager_map`→`KVCacheSpecRegistry` 等 4 处 RESOLVED；§跨子系统 grep 统计未重跑，VERIFY 已标） |
| PD 分离（disaggregated serving） | `topics/pd-disaggregation.md` | TODO |
| KV connector / KV offload | [topics/kv-connector.md](topics/kv-connector.md) | **STALE**（increment 2026-08-18 已写骨架：16 backend（P2P 删、NIXL pull/push 拆分、MooncakeStore 增）+ v1/kv_offload/ 块管理层 + entrypoints/scale_out/ 前端；backend 内部行号待 verify pass） |
| Speculative decoding (Eagle) | [topics/spec-decode-eagle.md](topics/spec-decode-eagle.md) | **STALE**（increment 2026-08-18 已写新架构主干：SpecDecodeBaseProposer 迁至 llm_base_proposer.py、worker/gpu/spec_decode Speculator 家族（BaseSpeculator→DraftModelSpeculator→AutoRegressiveSpeculator + dflash/dspark/gemma4/mtp）、RejectionSampler 换血为 standard/synthetic/block；正文旧锚点待 verify pass） |
| torch.compile / cuda graph | `topics/compilation.md` | TODO |
| Distributed parallel (TP/PP/DP/EP) | [comparison/topics/distributed.md](../comparison/topics/distributed.md) | **DONE**（跨项目对比） |
| Multi-modal | `topics/multimodal.md` | TODO |
| LoRA | `topics/lora.md` | TODO |
| OpenAI API 兼容层 | `topics/openai-api.md` | TODO |
| v0 vs v1 迁移路线 | `topics/v0-vs-v1.md` | TODO |

---

## Sources of truth

- 主代码根：[d:\design\vllm\vllm\](d:\design\vllm\vllm)
- v1 子树（重点）：[d:\design\vllm\vllm\v1\](d:\design\vllm\vllm\v1)
- 服务化层：[d:\design\vllm\vllm\entrypoints\](d:\design\vllm\vllm\entrypoints)
- 测试：[d:\design\vllm\tests\](d:\design\vllm\tests)
- 示例：[d:\design\vllm\examples\](d:\design\vllm\examples)
