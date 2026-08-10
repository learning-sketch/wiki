---
type: comparison
project: cross
status: verified
confidence: high
verified_against: 2026-04-18
sources:
  - d:\design\MindIE-LLM\mindie_llm
  - d:\design\MindIE-LLM\src
  - d:\design\vllm\vllm
  - d:\design\sglang\python\sglang\srt
related:
  - comparison/index.md
  - vllm/overview.md
  - sglang/overview.md
  - topics/scheduler.md
  - topics/sync-schedule.md
  - topics/kv-cache.md
  - topics/chunked-prefill.md
  - topics/cp-sp.md
  - topics/flashcomm.md
  - topics/pd-disaggregation.md
  - topics/distributed.md
  - topics/speculative-decoding.md
  - topics/prefix-cache.md
  - topics/multiproc-ipc.md
  - topics/engine-architecture.md
  - topics/executor-worker.md
  - topics/async-schedule.md
---

# Comparison Dimensions

> 三项目对比的 18 个维度，每个维度先标"项目→实现位置"，**深度对比留给 ingest 时建独立 topic 页**。
> 本页是 dimensions 注册表，新增维度先在这里注册再去建 topic。

> 标记说明：
> - **anchor**：已确认在该项目存在，给出代码位置
> - **N/A**：经查无对应实现
> - **TODO**：本轮未确认，等 ingest 时澄清

---

## §dim-overall：整体进程模型

| 项目 | 进程模型 | 锚点 |
|---|---|---|
| MindIE | **1 进程**（generator + C++ LlmEngine 嵌入式调度线程 + PluginManager.forward_thread + ModelRunner 全在一起）；PD 时 +1 connector 子进程 | `mindie/entities/Generator.md`（已删）, `LlmEngine.md`（已删）, `PluginManager.md`（已删） |
| vLLM | **可插拔 4 种**：inproc（1 进程）/ uniproc + EngineCore（2 进程）/ MultiprocExecutor（1+1+N 进程）/ DPLB（1 + DP×1 + DP×N 进程）；ZMQ + MessageQueue shm 双 IPC | [vllm/entities/EngineCoreClient.md](../vllm/entities/EngineCoreClient.md), [v1/engine/core.py](../../vllm/vllm/v1/engine/core.py), [v1/executor/multiproc_executor.py](../../vllm/vllm/v1/executor/multiproc_executor.py) |
| SGLang | **严格 3 进程**：TokenizerManager（主）+ Scheduler 子进程（×TP×PP×DP，scheduler 与 worker 同进程）+ DetokenizerManager 子进程；不可降为 inproc | [sglang/entities/TokenizerManager.md](../sglang/entities/TokenizerManager.md), [Scheduler.md](../sglang/entities/Scheduler.md), [entrypoints/engine.py:143-209](../../sglang/python/sglang/srt/entrypoints/engine.py) |

**深度对比**：见 [comparison/topics/engine-architecture.md §1 顶层进程拓扑](topics/engine-architecture.md)。

---

## §dim-engine：Engine 抽象与请求生命周期

| 项目 | 入口 | 锚点 |
|---|---|---|
| MindIE | **`Generator`**（Python，~1438 行，**装配 + 调度入口 + PD 接口**三重角色）+ **`LlmEngine`**（C++，独立调度线程） | `mindie/entities/Generator.md`（已删）, `LlmEngine.md`（已删）, [text_generator/generator.py](../../MindIE-LLM/mindie_llm/text_generator/generator.py), [src/engine/llm_engine.cpp](../../MindIE-LLM/src/engine/llm_engine.cpp) |
| vLLM | **`LLMEngine`** (sync) **+ `AsyncLLM`** (async) **双前端**，共享 `EngineCore` 后端；`EngineCoreClient` 6 子类作 IPC 桥（详 [vllm/entities/EngineCoreClient.md](../vllm/entities/EngineCoreClient.md)） | [vllm/entities/LLMEngine.md](../vllm/entities/LLMEngine.md), [AsyncLLM.md](../vllm/entities/AsyncLLM.md), [EngineCore.md](../vllm/entities/EngineCore.md), [EngineCoreClient.md](../vllm/entities/EngineCoreClient.md) |
| SGLang | **`Engine`**（[entrypoints/engine.py:143](../../sglang/python/sglang/srt/entrypoints/engine.py)）= **launcher**（fork 3 子进程后退出 `__init__`）；运行时职责在 `TokenizerManager` + `Scheduler` + `DetokenizerManager` 三独立进程 | [sglang/entities/TokenizerManager.md](../sglang/entities/TokenizerManager.md), [Scheduler.md](../sglang/entities/Scheduler.md), [entrypoints/engine.py:164-209](../../sglang/python/sglang/srt/entrypoints/engine.py) |

**深度对比**：见 [comparison/topics/engine-architecture.md](topics/engine-architecture.md)（13 个子维度：进程拓扑 / 顶层入口 / sync vs async / Engine↔Scheduler 边界 / Engine↔Worker 边界 / Tokenize-Detokenize 进程位置 / 跨语言 / 启动序列 / 关闭语义 / PD 关系 / 综合 cheat sheet / anchor-driven cross-check / 与 PD 优化的关联）。

---

## §dim-executor：Executor / Worker 模型

| 项目 | 实现 | 锚点 |
|---|---|---|
| MindIE | **无独立 Executor 抽象**（`Generator` 直接调 `PluginManager.generate_token` → `ModelRunner`）；worker 是同进程后台线程 `forward_thread`（[plugin_manager.py:107-115](../../MindIE-LLM/mindie_llm/text_generator/plugins/plugin_manager.py)） | `mindie/entities/Generator.md`（已删）, `PluginManager.md`（已删）, `ModelRunner.md`（已删）, [runtime/model_runner/model_runner.py](../../MindIE-LLM/mindie_llm/runtime/model_runner/model_runner.py) |
| vLLM | **`Executor` 抽象 + 4 种实现**：UniProc / MultiProc / Ray / RayV2；Worker 是 `Worker(WorkerBase)` 持 `GPUModelRunner`；`MultiprocExecutor` 用 `MessageQueue.enqueue + getattr(self.worker, method)` 字符串 RPC（[multiproc_executor.py:953-979](../../vllm/vllm/v1/executor/multiproc_executor.py)） | [vllm/entities/MultiprocExecutor.md](../vllm/entities/MultiprocExecutor.md), [GPUWorker.md](../vllm/entities/GPUWorker.md), [GPUModelRunner.md](../vllm/entities/GPUModelRunner.md), [v1/executor/abstract.py](../../vllm/vllm/v1/executor/abstract.py) |
| SGLang | **无独立 Executor 抽象**（`Scheduler` 直接持 `TpModelWorker`，同进程）；多 worker 通过多 scheduler 进程 + NCCL 实现 | [sglang/entities/Scheduler.md](../sglang/entities/Scheduler.md), [managers/tp_worker.py](../../sglang/python/sglang/srt/managers/tp_worker.py) |

**深度对比**：见 [comparison/topics/executor-worker.md](topics/executor-worker.md)（11 个子维度：Executor 抽象 / Worker 类层次 / 进程模型 / RPC 调用机制 / forward+sample 拆分 / 多 worker 并行 / 启动序列 / 错误处理 / 综合 cheat sheet / anchor-driven cross-check / 与 PD 优化的关联）；轻量版本见 [engine-architecture.md §5](topics/engine-architecture.md)。

---

## §dim-scheduler：Scheduler 策略

| 项目 | 实现 | 锚点 |
|---|---|---|
| MindIE | **C++ `Scheduler`**（namespace mindie_llm）+ 2D 正交 policy（StagePolicy × Per-stage Policy，6×4） | [src/scheduler/scheduler.h](../../MindIE-LLM/src/scheduler/scheduler.h), [src/scheduler/policy/](../../MindIE-LLM/src/scheduler/policy)（15 .h） |
| vLLM | `Scheduler` + `AsyncScheduler` + `RequestQueue`（FCFS / PRIORITY） | [v1/core/sched/scheduler.py](../../vllm/vllm/v1/core/sched/scheduler.py), [async_scheduler.py](../../vllm/vllm/v1/core/sched/async_scheduler.py), [request_queue.py](../../vllm/vllm/v1/core/sched/request_queue.py) |
| SGLang | `Scheduler` + 11 mixin（OutputProcessor / UpdateWeights / Profiler / Metrics / DisaggregationDecode / DisaggregationPrefill / Multiplex / RuntimeChecker / PP / DPAttn / Dllm） | [managers/scheduler.py:317-329](../../sglang/python/sglang/srt/managers/scheduler.py) |

**深度对比**：见 [comparison/topics/scheduler.md](topics/scheduler.md)（覆盖进程位置、Schedule 主循环、队列、策略模块化、CPU/GPU overlap、PD 支持、抢占、Pause 状态机 8 个子维度）。

---

## §dim-kv：KV cache 管理

| 项目 | 实现 | 锚点 |
|---|---|---|
| MindIE | C++ `BlockSpaceManager` 抽象（**`BlockManagerType` 5 枚举槽位，但工厂仅产 2 种** `SelfAttn`/`LwdSelfAttn` + 1 独立类 `RequestSingle` 未接工厂；详 `entities/BlockSpaceManager.md`（已删）） + Python `KVCachePool` + `MemPool` 2 后端 (memcache/mooncake) | `entities/BlockSpaceManager.md`（已删）, [src/include/block_manager/block_manager_interface.h](../../MindIE-LLM/src/include/block_manager/block_manager_interface.h), [adapter/torch_utils/kvcache_pool.py](../../MindIE-LLM/mindie_llm/text_generator/adapter/torch_utils/kvcache_pool.py), [text_generator/mempool/](../../MindIE-LLM/mindie_llm/text_generator/mempool) |
| vLLM | `KVCacheManager` + `KVCacheCoordinator`(3 实现) + `BlockPool` + `BlockHashToBlockMap` + `KVCacheSpec` 11+ 子类 + `EncoderCacheManager`（独立） | [v1/core/kv_cache_manager.py](../../vllm/vllm/v1/core/kv_cache_manager.py), [block_pool.py](../../vllm/vllm/v1/core/block_pool.py), [kv_cache_coordinator.py](../../vllm/vllm/v1/core/kv_cache_coordinator.py), [v1/kv_cache_interface.py](../../vllm/vllm/v1/kv_cache_interface.py) |
| SGLang | `BasePrefixCache` 多实现（RadixCache / HiRadixCache / SWA / Mamba / ChunkCache）+ `BaseTokenToKVPoolAllocator` 2 实现 + `KVCache` 多 pool（MHA/MLA/NSA/DoubleSparse/Hybrid+FP4 子类）+ `HiCacheStorage` 7 后端 + `sparsity/` 子系统 | [mem_cache/](../../sglang/python/sglang/srt/mem_cache)（62 .py） |

**深度对比**：见 [comparison/topics/kv-cache.md](topics/kv-cache.md)（10 个子维度：架构 / Prefix 数据结构 / Eviction / 多 attention 类型 / HiCache / 量化 / PD 原生支持 / 稀疏 / Cache events / 与 PD 优化的关联）。

---

## §dim-distributed：分布式并行（TP/PP/EP/DP）

| 项目 | 实现 | 锚点 |
|---|---|---|
| MindIE | `runtime/utils/distributed/{pipeline_parallel,parallel_info_manager,communication_op,model_cache_pool}.py` + 顶层 `mindie_llm/distributed/`；**`ParallelType` 12 种 enum，无 PP；EPLB / Elastic-EP 均 N/A** | [runtime/utils/distributed/](../../MindIE-LLM/mindie_llm/runtime/utils/distributed), [parallel_info_manager.py:33-47](../../MindIE-LLM/mindie_llm/runtime/utils/distributed/parallel_info_manager.py) |
| vLLM | `vllm/distributed/`（顶层）+ `eplb/` + `elastic_ep/` + v1 worker 内 `dp_utils.py` / `cp_utils.py` / `gpu/pp_utils.py` / `gpu/eplb_utils.py`；**18 个 device_communicators 后端** | [vllm/distributed/](../../vllm/vllm/distributed), [v1/worker/dp_utils.py](../../vllm/vllm/v1/worker/dp_utils.py), [v1/worker/gpu/pp_utils.py](../../vllm/vllm/v1/worker/gpu/pp_utils.py), [v1/worker/gpu/eplb_utils.py](../../vllm/vllm/v1/worker/gpu/eplb_utils.py) |
| SGLang | `srt/distributed/`（22 .py + 12+ device_communicators） + `srt/eplb/`（含 simulator + 多算法）+ `srt/elastic_ep/` + `layers/dp_attention.py`；**Scheduler 显式 6 rank 维度** | [srt/distributed/](../../sglang/python/sglang/srt/distributed), [srt/eplb/](../../sglang/python/sglang/srt/eplb), [srt/elastic_ep/](../../sglang/python/sglang/srt/elastic_ep), [layers/dp_attention.py](../../sglang/python/sglang/srt/layers/dp_attention.py), [managers/scheduler.py:332-360](../../sglang/python/sglang/srt/managers/scheduler.py) |

**深度对比**：见 [topics/distributed.md](topics/distributed.md)（9 个子维度：进程组抽象 / TP collectives / PP 完成度 / DP 双语义 / EP+EPLB+Elastic / scheduler rank 维度数 / CP-SP 链回 / 目录布局 / 与 PD 优化关联）。

---

## §dim-pd：PD 分离（Disaggregated Serving）

| 项目 | 实现 | 锚点 |
|---|---|---|
| MindIE | `text_generator/utils/separate_deployment_engine.py` + `connector/` 多组件 + 设计文档 | [text_generator/utils/separate_deployment_engine.py](../../MindIE-LLM/mindie_llm/text_generator/utils/separate_deployment_engine.py), [docs/mindie_generator_aclgraph_pp_design.md](../../MindIE-LLM/docs/mindie_generator_aclgraph_pp_design.md) |
| vLLM | 入口在 `entrypoints/serve/disagg/`，KV 传输落到 `distributed/kv_transfer/kv_connector/v1/`（13+ backend）+ `v1/kv_offload/`（worker 端块管理） | [entrypoints/serve/disagg/](../../vllm/vllm/entrypoints/serve/disagg), [v1/kv_offload/](../../vllm/vllm/v1/kv_offload), [distributed/kv_transfer/kv_connector/v1/base.py](../../vllm/vllm/distributed/kv_transfer/kv_connector/v1/base.py) |
| SGLang | `disaggregation/` 顶层模块 + 7 个后端（base/common/nixl/mooncake/mori/ascend/fake）+ `managers/disagg_service.py` | [disaggregation/](../../sglang/python/sglang/srt/disaggregation), [managers/disagg_service.py](../../sglang/python/sglang/srt/managers/disagg_service.py) |

**深度对比**：见 [comparison/topics/pd-disaggregation.md](topics/pd-disaggregation.md)（14 个子维度）。

---

## §dim-kv-transfer：KV 传输协议

| 项目 | 协议 | 锚点 |
|---|---|---|
| MindIE | LLMDataDist (Ascend 自研，含 RDMA + cache_manager) + Mooncake (mempool) | [text_generator/utils/separate_deployment_engine.py](../../MindIE-LLM/mindie_llm/text_generator/utils/separate_deployment_engine.py), [text_generator/mempool/mooncake_mempool.py](../../MindIE-LLM/mindie_llm/text_generator/mempool/mooncake_mempool.py) |
| vLLM | `KVConnectorBase_V1` 抽象 + **14 个 v1 backend**（NIXL / Mooncake / MoRIIO / LMCache×3 / HF3FS / P2pNccl / Offloading / SimpleCPUOffload / MultiConnector / FlexKV / DecodeBench / Example×2）+ 独立 `v1/kv_offload/` 块管理层 + `entrypoints/serve/disagg/` HTTP 前端 | [distributed/kv_transfer/kv_connector/v1/base.py](../../vllm/vllm/distributed/kv_transfer/kv_connector/v1/base.py), [v1/kv_offload/](../../vllm/vllm/v1/kv_offload) — 深度对比见 [vllm/topics/kv-connector.md](../vllm/topics/kv-connector.md) |
| SGLang | NIXL / Mooncake / MORI / Ascend / Fake 五后端可插拔（统一 `BaseKVManager`/`Sender`/`Receiver`/`BootstrapServer` 抽象） | [disaggregation/base/conn.py](../../sglang/python/sglang/srt/disaggregation/base/conn.py), [disaggregation/{nixl,mooncake,mori,ascend,fake}/](../../sglang/python/sglang/srt/disaggregation) |

**深度对比**：见 [comparison/topics/pd-disaggregation.md §3 KV 传输栈](topics/pd-disaggregation.md)；vLLM 14 backend 单家深度见 [vllm/topics/kv-connector.md](../vllm/topics/kv-connector.md)。

---

## §dim-spec：Speculative decoding

| 项目 | 实现 | 锚点 |
|---|---|---|
| MindIE | `text_generator/plugins/{mtp,la,memory_decoding}/` + `runtime/model_runner/spec_worker.py`（实际类名 `MtpWorker`，仅 `num_speculative_tokens > 0` 启用）+ C++ Scheduler `speculationGamma` placeholder 机制；详 `mindie/topics/speculative.md`（已删） | `mindie/topics/speculative.md`（已删）, [text_generator/plugins/mtp/](../../MindIE-LLM/mindie_llm/text_generator/plugins/mtp), [plugins/la/](../../MindIE-LLM/mindie_llm/text_generator/plugins/la), [plugins/memory_decoding/](../../MindIE-LLM/mindie_llm/text_generator/plugins/memory_decoding), [runtime/model_runner/spec_worker.py](../../MindIE-LLM/mindie_llm/runtime/model_runner/spec_worker.py) |
| vLLM | `v1/spec_decode/` 12 .py（`SpecDecodeBaseProposer` 派生 EAGLE/EAGLE3/DraftModel/DFlash + 独立 Medusa/Ngram CPU/Ngram GPU/Suffix/ExtractHiddenStates）+ `v1/worker/gpu/spec_decode/eagle/speculator.py` 新版 GPU 路径（独立 `InputBuffers` + 独立 prefill/decode `EagleCudaGraphManager`）+ 双层占位（scheduler `scheduled_spec_decode_tokens` dict + `AsyncScheduler` `num_output_placeholders` int + 共享 `[-1]` list）+ 3 模式 `RejectionSampler`（strict / probabilistic / synthetic）；详 [vllm/topics/spec-decode-eagle.md](../vllm/topics/spec-decode-eagle.md) | [vllm/topics/spec-decode-eagle.md](../vllm/topics/spec-decode-eagle.md), [v1/spec_decode/](../../vllm/vllm/v1/spec_decode), [v1/worker/gpu/spec_decode/eagle/](../../vllm/vllm/v1/worker/gpu/spec_decode/eagle), [config/speculative.py](../../vllm/vllm/config/speculative.py) |
| SGLang | `srt/speculative/` (含 triton_ops) | [srt/speculative/](../../sglang/python/sglang/srt/speculative) |

**深度对比**：见 [topics/speculative-decoding.md](topics/speculative-decoding.md)（9 个子维度：算法目录 / Draft 模型集成 / Verify 机制 / Scheduler 集成 / gamma 配置 / 兼容性矩阵 / KV 槽位管理 / anchor-driven cross-check / 与 PD 优化的关联）。

---

## §dim-compile：Compilation / Graph capture

| 项目 | 实现 | 锚点 |
|---|---|---|
| MindIE | `runtime/compilation/` + `modeling/model_wrapper/aclgraph/` (NPU aclgraph) | [runtime/compilation/](../../MindIE-LLM/mindie_llm/runtime/compilation), [modeling/model_wrapper/aclgraph/](../../MindIE-LLM/mindie_llm/modeling/model_wrapper/aclgraph) |
| vLLM | `vllm/compilation/` + `v1/worker/gpu/cudagraph_utils.py` + `v1/worker/encoder_cudagraph.py` | [vllm/compilation/](../../vllm/vllm/compilation), [v1/worker/gpu/cudagraph_utils.py](../../vllm/vllm/v1/worker/gpu/cudagraph_utils.py) |
| SGLang | `srt/compilation/` | [srt/compilation/](../../sglang/python/sglang/srt/compilation) |

---

## §dim-sampling：Sampling

| 项目 | 实现 | 锚点 |
|---|---|---|
| MindIE | `text_generator/samplers/` (sampler + token_selectors{cpu,pta} + logits_handlers) | [text_generator/samplers/](../../MindIE-LLM/mindie_llm/text_generator/samplers) |
| vLLM | `v1/sample/` + `v1/sample/logits_processor/` + `v1/worker/gpu/sample/` | [v1/sample/](../../vllm/vllm/v1/sample), [v1/worker/gpu/sample/](../../vllm/vllm/v1/worker/gpu/sample) |
| SGLang | `srt/sampling/` (含 penaltylib) | [srt/sampling/](../../sglang/python/sglang/srt/sampling) |

---

## §dim-lora：LoRA / Multi-LoRA

| 项目 | 实现 | 锚点 |
|---|---|---|
| MindIE | `runtime/lora/` | [runtime/lora/](../../MindIE-LLM/mindie_llm/runtime/lora) |
| vLLM | `vllm/lora/` + `v1/worker/lora_model_runner_mixin.py` + `v1/worker/gpu/lora_utils.py` | [vllm/lora/](../../vllm/vllm/lora), [v1/worker/lora_model_runner_mixin.py](../../vllm/vllm/v1/worker/lora_model_runner_mixin.py) |
| SGLang | `srt/lora/` (含 triton_ops + torch_ops 双实现) | [srt/lora/](../../sglang/python/sglang/srt/lora) |

---

## §dim-quant：量化

| 项目 | 后端 | 锚点 |
|---|---|---|
| MindIE | TODO（runtime 下 conf/config 提到，需 ingest） | TODO |
| vLLM | 在 `model_executor/layers/quantization/`（待 ingest 确认） | [vllm/model_executor/](../../vllm/vllm/model_executor) |
| SGLang | `srt/layers/quantization/{quark,modelslim,compressed_tensors}/schemes/` 多套 | [srt/layers/quantization/](../../sglang/python/sglang/srt/layers/quantization) |

---

## §dim-moe：MoE

> verified 2026-04-18 (verify pass on `mindie/topics/moe.md`（已删）)

| 项目 | 实现 | 锚点 |
|---|---|---|
| MindIE | `runtime/models/{qwen3_moe,deepseek_v3,deepseek_v32}/` + `runtime/layers/fused_moe/` (5 类 `MoECommType`，**实际仅 3 路可达**；`FUSED_MC2`/`FUSED_ALLTOALL` 为 dead branch) + 3 `TokenDispatcherWith{AllGather,MC2,All2AllV}` + C++ `dispatch_ffn_combine` MC2 融合算子；**deepseek_v32 复用 v3 主干**（router 显式 `_get_model_cls=DeepseekV3ForCausalLM`）；**runtime 主线无 EPLB** + ATB 侧另有 EPLB 不混用；详 `mindie/topics/moe.md`（已删） | `mindie/topics/moe.md`（已删）, [runtime/layers/fused_moe/](../../MindIE-LLM/mindie_llm/runtime/layers/fused_moe), [runtime/models/deepseek_v3/](../../MindIE-LLM/mindie_llm/runtime/models/deepseek_v3) |
| vLLM | `model_executor/layers/fused_moe/` + 独立 `prepare_finalize/` 子包（`FusedMoEPrepareAndFinalizeModular` 基类 + `deepep_ht`/`deepep_ll`/`naive_dp_ep`/`nixl_ep_prepare_finalize`）+ `runner/` 子包（`MoeRunner` 多 backend）+ `fused_moe_modular_method.py` 把 prepare/run/finalize modular 化；EPLB 在 [`distributed/eplb/`](../../vllm/vllm/distributed/eplb) + [`v1/worker/gpu/eplb_utils.py`](../../vllm/vllm/v1/worker/gpu/eplb_utils.py) | [model_executor/layers/fused_moe/](../../vllm/vllm/model_executor/layers/fused_moe), [model_executor/layers/fused_moe/prepare_finalize/](../../vllm/vllm/model_executor/layers/fused_moe/prepare_finalize), [v1/worker/gpu/eplb_utils.py](../../vllm/vllm/v1/worker/gpu/eplb_utils.py) |
| SGLang | `srt/layers/moe/token_dispatcher/`（`standard`/`deepep`/`mooncake`/`nixl`/`flashinfer`/`moriep`/**`fuseep`**——后者最贴近 MindIE FUSED_MC2）+ `moe_runner/`（`triton`/`deep_gemm`/`flashinfer_cutedsl`/`flashinfer_trtllm`/`marlin`）+ `fused_moe_triton/` + `ep_moe/layer.py` + [`srt/eplb/`](../../sglang/python/sglang/srt/eplb) + [`srt/elastic_ep/`](../../sglang/python/sglang/srt/elastic_ep) | [srt/layers/moe/](../../sglang/python/sglang/srt/layers/moe), [srt/eplb/](../../sglang/python/sglang/srt/eplb) |

---

## §dim-multimodal：多模态

| 项目 | 实现 | 锚点 |
|---|---|---|
| MindIE | TODO | TODO |
| vLLM | `vllm/multimodal/` + `v1/worker/gpu/mm/{encoder_cache,encoder_runner,rope}.py` | [vllm/multimodal/](../../vllm/vllm/multimodal), [v1/worker/gpu/mm/](../../vllm/vllm/v1/worker/gpu/mm) |
| SGLang | `srt/multimodal/` (含 evs) + `managers/multimodal_processor.py` + `managers/mm_utils.py` | [srt/multimodal/](../../sglang/python/sglang/srt/multimodal) |

---

## §dim-serving：服务化 / API 协议

| 项目 | 协议 | 锚点 |
|---|---|---|
| MindIE | `server/main.py` (具体协议待 ingest) + `connector/` | [server/main.py](../../MindIE-LLM/mindie_llm/server/main.py) |
| vLLM | OpenAI / Pooling / SageMaker / serve（disagg/elastic_ep/instrumentator/...） + 流式 | [vllm/entrypoints/](../../vllm/vllm/entrypoints) |
| SGLang | OpenAI / Anthropic / Ollama 三套 + HTTP / gRPC | [srt/entrypoints/openai/](../../sglang/python/sglang/srt/entrypoints/openai), [anthropic/](../../sglang/python/sglang/srt/entrypoints/anthropic), [ollama/](../../sglang/python/sglang/srt/entrypoints/ollama), [grpc_server.py](../../sglang/python/sglang/srt/entrypoints/grpc_server.py) |

---

## §dim-hardware：硬件后端

| 项目 | 支持 | 锚点 |
|---|---|---|
| MindIE | Ascend NPU (主) | （NPU 是默认后端，无独立 backend 目录） |
| vLLM | CUDA / CPU / TPU / XPU + `vllm/platforms/` 抽象 | [v1/worker/gpu_*](../../vllm/vllm/v1/worker), [cpu_*](../../vllm/vllm/v1/worker), [xpu_*](../../vllm/vllm/v1/worker), [tpu_*](../../vllm/vllm/v1/worker), [vllm/platforms/](../../vllm/vllm/platforms) |
| SGLang | CUDA + MUSA (Moore Threads) + Ascend | [srt/hardware_backend/](../../sglang/python/sglang/srt/hardware_backend), [srt/hardware_backend/musa/](../../sglang/python/sglang/srt/hardware_backend/musa) |

---

## §dim-prefix-cache：Prefix caching

| 项目 | 实现 | 锚点 |
|---|---|---|
| MindIE | **C++ `PrefixCacheBlockAllocator`**（`unordered_map<HashValue,BlockId>` hash table + LRU evictor + boost-style `HashCombine`）+ **Python `PrefixCachePlugin`**（input 裁剪 + 异步 `MemPool.put/get`）；调度器 `GetCommonComputedBlockIds`（本地） / `GetRemoteComputedBlockIds`（KV pool 跨实例命中，pybind 反向调 Python `MemPool.LookUp`）；C++↔Python 协议靠 protobuf `computed_block_lens` / `remote_computed_block_lens`；详见 `mindie/topics/prefix-cache.md`（已删） | [block_manager/prefix_cache_block_allocator.cpp](../../MindIE-LLM/src/block_manager/prefix_cache_block_allocator.cpp), [block_manager/prefix_cache_block.cpp](../../MindIE-LLM/src/block_manager/prefix_cache_block.cpp), [scheduler.cpp:1242-1274](../../MindIE-LLM/src/scheduler/scheduler.cpp), [plugins/prefix_cache/](../../MindIE-LLM/mindie_llm/text_generator/plugins/prefix_cache) |
| vLLM | **`KVCacheManager` 内置**（[v1/core/kv_cache_manager.py:106-553](../../vllm/vllm/v1/core/kv_cache_manager.py)）+ **`BlockPool.BlockHashToBlockMap`**（hash table，[v1/core/block_pool.py:34-128](../../vllm/vllm/v1/core/block_pool.py)，**不是** trie）+ 3 种 `KVCacheCoordinator`（NoPrefixCache / Unitary / Hybrid，[v1/core/kv_cache_coordinator.py:256-545](../../vllm/vllm/v1/core/kv_cache_coordinator.py)）+ 6 种 `SingleTypeKVCacheManager.find_longest_cache_hit`（FullAttn / SlidingWindow / ChunkedLocal / Mamba / CrossAttn / SinkFull，[v1/core/single_type_kv_cache_manager.py:420-1124](../../vllm/vllm/v1/core/single_type_kv_cache_manager.py)）；hash 算法 `hash_block_tokens(caching_hash_fn, prev, tokens, extra_keys)`（[v1/core/kv_cache_utils.py:535-562](../../vllm/vllm/v1/core/kv_cache_utils.py)），4 种可选 hash function（sha256/sha256_cbor/xxhash/xxhash_cbor）；**全 Python 一段抽象**（csrc/ 0 命中）；KV connector 复用同一 `BlockHashToBlockMap` mirror 到 CPU；详见 [vllm/topics/prefix-cache.md](../../vllm/topics/prefix-cache.md) | [v1/core/kv_cache_manager.py](../../vllm/vllm/v1/core/kv_cache_manager.py), [v1/core/block_pool.py](../../vllm/vllm/v1/core/block_pool.py), [v1/core/kv_cache_coordinator.py](../../vllm/vllm/v1/core/kv_cache_coordinator.py), [v1/core/single_type_kv_cache_manager.py](../../vllm/vllm/v1/core/single_type_kv_cache_manager.py), [v1/core/kv_cache_utils.py:535](../../vllm/vllm/v1/core/kv_cache_utils.py), [config/cache.py:78-95](../../vllm/vllm/config/cache.py)（`enable_prefix_caching=True` 默认） |
| SGLang | **`RadixCache(BasePrefixCache)` trie**（[radix_cache.py:285](../../sglang/python/sglang/srt/mem_cache/radix_cache.py)）—— 三家中**唯一用 trie 不是 hash table** 的实现；`Scheduler.init_cache_with_memory_pool` 内 **8+ 路工厂分支**（[managers/scheduler.py:783-890](../../sglang/python/sglang/srt/managers/scheduler.py)）选 `RadixCache` / `HiRadixCache`（HiCache 三层）/ `RadixCacheCpp`（C++ 加速 env 触发）/ `SWARadixCache` / `MambaRadixCache` / `HiMambaRadixCache` / `UnifiedRadixCache` / `LMCRadixCache` / `ChunkCache`（disable_radix_cache + chunked_prefill 时 fallback）+ `SessionAwareCache` 包装；`--disable-radix-cache` 反向开关（默认启用，[server_args.py:618](../../sglang/python/sglang/srt/server_args.py)）；`--radix-eviction-policy` CLI 暴露 3 种 `{lru, lfu, slru}`（`evict_policy.py` 注册 7 类可扩展）；详见 [comparison/topics/prefix-cache.md](topics/prefix-cache.md) | [srt/mem_cache/radix_cache.py](../../sglang/python/sglang/srt/mem_cache/radix_cache.py), [hiradix_cache.py](../../sglang/python/sglang/srt/mem_cache/hiradix_cache.py), [evict_policy.py](../../sglang/python/sglang/srt/mem_cache/evict_policy.py), [managers/scheduler.py:783-890](../../sglang/python/sglang/srt/managers/scheduler.py) |

**深度对比**：见 [comparison/topics/prefix-cache.md](topics/prefix-cache.md)（11 个子维度：数据结构与抽象 / hash 算法 / 命中查询 API / eviction 策略 / hybrid 模型 / 多层存储 HiCache / 配置开关与默认 / 兼容性矩阵 / 跨语言绑定 / anchor-driven cross-check / 与 PD 优化的关联）。

---

## §dim-batching：Continuous batching / Chunked prefill

| 项目 | 实现 | 锚点 |
|---|---|---|
| MindIE | `text_generator/plugins/splitfuse/` (chunked prefill) | [plugins/splitfuse/](../../MindIE-LLM/mindie_llm/text_generator/plugins/splitfuse) |
| vLLM | 在 `v1/core/sched/scheduler.py` 内（待 ingest） | [v1/core/sched/scheduler.py](../../vllm/vllm/v1/core/sched/scheduler.py) |
| SGLang | 在 `managers/scheduler.py` + `managers/schedule_policy.py` 内 + `prefill_delayer.py` | [managers/scheduler.py](../../sglang/python/sglang/srt/managers/scheduler.py), [managers/prefill_delayer.py](../../sglang/python/sglang/srt/managers/prefill_delayer.py) |

---

## §dim-structured：Structured output

| 项目 | 实现 | 锚点 |
|---|---|---|
| MindIE | `text_generator/plugins/structured_output/` 仅 xgrammar 单后端，仅 `json_object`/`json_schema`；目录在 plugins/ 下但**不**走 `PluginManager.importlib` 循环；C++ 仅校验 + 透传 `responseFormat`/`predictedTokenIds_`，与 mtp 互斥；详见 `mindie/topics/structured-output.md`（已删） | [plugins/structured_output/](../../MindIE-LLM/mindie_llm/text_generator/plugins/structured_output)（[manager](../../MindIE-LLM/mindie_llm/text_generator/plugins/structured_output/structured_output_manager.py), [grammar](../../MindIE-LLM/mindie_llm/text_generator/plugins/structured_output/structured_output_grammar.py), [bitmask](../../MindIE-LLM/mindie_llm/text_generator/plugins/structured_output/structured_output_bitmask.py)）+ [PluginManager `_init_structured_output_manager` plugin_manager.py:1046-1095](../../MindIE-LLM/mindie_llm/text_generator/plugins/plugin_manager.py) + [sampler `GuidedDecodingLogitsHandler` pta_handlers.py:87-125](../../MindIE-LLM/mindie_llm/text_generator/samplers/logits_handlers/pta_handlers.py) + [mtp 互斥 infer_param.cpp:216-225](../../MindIE-LLM/src/server/endpoint/utils/infer_param.cpp) |
| vLLM | `v1/structured_output/` + `v1/worker/gpu/structured_outputs.py` | [v1/structured_output/](../../vllm/vllm/v1/structured_output), [v1/worker/gpu/structured_outputs.py](../../vllm/vllm/v1/worker/gpu/structured_outputs.py) |
| SGLang | `srt/constrained/` | [srt/constrained/](../../sglang/python/sglang/srt/constrained) |

---

## §dim-async-schedule：异步调度（CPU/GPU overlap）

> 把 [§dim-scheduler 的 CPU/GPU Overlap 子节](topics/scheduler.md) 提升为独立维度，方便对照。

| 项目 | 实现 | 锚点 |
|---|---|---|
| MindIE | `maxScheduledBatch_=2`（同步=1，异步单发=2）+ Generator `async_inference` 路径 + `plugin_manager.generate_token_async` | [src/scheduler/scheduler.h:292-293](../../MindIE-LLM/src/scheduler/scheduler.h), [text_generator/generator.py:319-323, 636-648](../../MindIE-LLM/mindie_llm/text_generator/generator.py) |
| vLLM | `AsyncScheduler` 子类（覆盖 `_update_after_schedule` + `_update_request_with_output`，用 placeholder token 提前调度下一步） | [v1/core/sched/async_scheduler.py](../../vllm/vllm/v1/core/sched/async_scheduler.py)（61 行） |
| SGLang | `event_loop_overlap`（用 `result_queue: deque` 把上一 batch 的 process_batch_result 与本 batch 的 run_batch overlap）+ `is_disable_overlap_for_batch` 显式禁用条件 | [managers/scheduler.py:1412-1497](../../sglang/python/sglang/srt/managers/scheduler.py) |

**深度对比**：见 [topics/async-schedule.md](topics/async-schedule.md)（8 个子维度：配置开关 / in-flight 上限 / 主循环 / 关键行为 / 三种 overlap 范式 / spec+grammar 协议 / 与 PD 优化的关联 / 默认值汇总）。轻量版本见 [topics/scheduler.md §5 CPU/GPU Overlap](topics/scheduler.md)。

---

## §dim-sync-schedule：同步调度（默认 / 无 overlap 路径）

> §dim-async-schedule 的反向视角：每家"sync schedule（单 in-flight batch、CPU schedule 与 GPU forward 严格串行）"路径的入口、配置、自动 fallback 条件。

| 项目 | sync 默认？ | 配置开关 | 入口函数 | 锚点 |
|---|---|---|---|---|
| MindIE | ✅ 默认 sync | `activateAsyncInference=false` (engine config) → `maxScheduledBatch_=1` | `LlmEngine` 主循环 → `Scheduler::Schedule(needSync)` | [config_info.h:171, 388](../../MindIE-LLM/src/include/config/config_info.h), [scheduler.cpp:122-128](../../MindIE-LLM/src/scheduler/scheduler.cpp), [scheduler.h:292-293](../../MindIE-LLM/src/scheduler/scheduler.h), [llm_engine.cpp:462-540](../../MindIE-LLM/src/engine/llm_engine.cpp) |
| vLLM | ❌ 默认 async（auto） | `SchedulerConfig.async_scheduling: bool \| None`，**4 种条件 auto-disable**（pooling/不支持 spec/不支持 executor） → 用 `Scheduler` (而非 `AsyncScheduler`) → `EngineCore.step` (而非 `step_with_batch_queue`) | `EngineCore.run_busy_loop → step` | [config/scheduler.py:146-174](../../vllm/vllm/config/scheduler.py), [config/vllm.py:798-832](../../vllm/vllm/config/vllm.py), [v1/engine/core.py:212-214, 404-433](../../vllm/vllm/v1/engine/core.py) |
| SGLang | ❌ 默认 overlap | `--disable-overlap-schedule` (`server_args.py:639`)，**10+ 种条件 auto-disable**（mps/sparse head/attention 组合/PP/特定 spec） → `dispatch_event_loop` 路由到 `event_loop_normal*` | `event_loop_normal` / `event_loop_normal_disagg_prefill` / `event_loop_normal_disagg_decode` | [server_args.py:639, 1132-1134, 2185-2187, 2300-2309, 3001-3005, 3249-3293, 3378-3382](../../sglang/python/sglang/srt/server_args.py), [scheduler.py:376, 1383-1409, 3628-3654](../../sglang/python/sglang/srt/managers/scheduler.py) |

**深度对比**：见 [comparison/topics/sync-schedule.md](topics/sync-schedule.md)（覆盖配置开关、in-flight batch 上限、6 个事件循环差异、与其它特性兼容性、命名陷阱 `Schedule(needSync)` ≠ "sync schedule"，共 8 个子维度）。

> [!warning] CONTRADICTION: MindIE 的 `Schedule(needSync)` 参数中的 `needSync` 是**跨 DP rank 同步**标志（[llm_engine.cpp:462](../../MindIE-LLM/src/engine/llm_engine.cpp)），与本维度的 "sync schedule（无 overlap）" **不是**一回事。详见 [topics/sync-schedule.md §6](topics/sync-schedule.md)。

---

## §dim-cp-sp：Context Parallel / Sequence Parallel

| 项目 | 实现 | 锚点 |
|---|---|---|
| MindIE | `ParallelType.ATTN_CP` + `ParallelType.ATTN_INNER_SP` 显式枚举 + `cp_size`/`sp_size` 配置 + `maybe_allgather_cp` runtime gather | [parallel_info_manager.py:34-47](../../MindIE-LLM/mindie_llm/runtime/utils/distributed/parallel_info_manager.py), [base/model.py:51-61](../../MindIE-LLM/mindie_llm/runtime/models/base/model.py), [model_runner.py:374-376, 650-661](../../MindIE-LLM/mindie_llm/runtime/model_runner/model_runner.py), [aclgraph_model_wrapper.py:62-65](../../MindIE-LLM/mindie_llm/modeling/model_wrapper/aclgraph/aclgraph_model_wrapper.py) |
| vLLM | **PCP**（Prefill CP）+ **DCP**（Decode CP）双轴 + `cp_kv_cache_interleave_size` 配置 + 每 attention impl 必须 `supports_pcp` / `need_to_return_lse_for_decode` | [v1/worker/cp_utils.py](../../vllm/vllm/v1/worker/cp_utils.py), [v1/worker/gpu/cp_utils.py](../../vllm/vllm/v1/worker/gpu/cp_utils.py), [v1/executor/multiproc_executor.py:115-121, 257-268](../../vllm/vllm/v1/executor/multiproc_executor.py)（`prefill_context_parallel_size`） |
| SGLang | `attn_cp` group + `enable_prefill_context_parallel` server_arg + `prefill_cp_mode = "in-seq-split"` + zigzag 切分（`ContextParallelMetadata.zigzag_index`）+ `attn_dp` 独立 DP | [layers/utils/cp_utils.py](../../sglang/python/sglang/srt/layers/utils/cp_utils.py)（含 `ContextParallelMetadata`, `is_prefill_context_parallel_enabled`, `cp_split_and_rebuild_data`），[layers/dp_attention.py](../../sglang/python/sglang/srt/layers/dp_attention.py), [communicator_nsa_cp.py](../../sglang/python/sglang/srt/layers/communicator_nsa_cp.py) |

**深度对比**：见 [comparison/topics/cp-sp.md](topics/cp-sp.md)。

---

## §dim-flashcomm：FlashComm 通信优化

> Ascend 专有名词，命名层面**仅 MindIE 出现**——vLLM 与 SGLang 没有同名实现。

| 项目 | 实现 | 锚点 |
|---|---|---|
| MindIE | **专门 `is_flashcomm_supported` 模型描述符**（[model_descriptor.py:24-26](../../MindIE-LLM/mindie_llm/runtime/models/base/model_descriptor.py)）+ `BaseModelForCausalLM.maybe_gather_and_unpad_for_flashcomm`（[base/model.py:38-49](../../MindIE-LLM/mindie_llm/runtime/models/base/model.py)）+ `forward_context.batch_descriptor.is_flash_comm_enabled` 运行时开关 + `linear_op.maybe_all_gather_and_maybe_unpad` 底层算子 | [model_descriptor.py:24-43](../../MindIE-LLM/mindie_llm/runtime/models/base/model_descriptor.py), [base/model.py:38-49](../../MindIE-LLM/mindie_llm/runtime/models/base/model.py), [model_runner.py:305-310, 637-649](../../MindIE-LLM/mindie_llm/runtime/model_runner/model_runner.py), [aclgraph_backend.py:208-217](../../MindIE-LLM/mindie_llm/runtime/compilation/aclgraph_backend.py), [linear_op.py](../../MindIE-LLM/mindie_llm/runtime/layers/linear/linear_op.py) |
| vLLM | **N/A**（命名层面无 flashcomm）— 等价的"通信与 FFN/LM_HEAD overlap"思路散在 attention backend / TP 通信原语里，待 ingest 时澄清 | TODO |
| SGLang | **N/A**（命名层面无 flashcomm）— 等价思路在 [layers/communicator.py](../../sglang/python/sglang/srt/layers/communicator.py) + [layers/communicator_nsa_cp.py](../../sglang/python/sglang/srt/layers/communicator_nsa_cp.py) | TODO |

**深度对比**：见 [comparison/topics/flashcomm.md](topics/flashcomm.md)。

---

## 维度索引（共 24 项）

1. §dim-overall — 整体进程模型
2. §dim-engine — Engine 抽象
3. §dim-executor — Executor / Worker
4. §dim-scheduler — Scheduler 策略
5. §dim-kv — KV cache 管理
6. §dim-distributed — 分布式并行
7. §dim-pd — **PD 分离**
8. §dim-kv-transfer — KV 传输协议
9. §dim-spec — Speculative decoding
10. §dim-compile — Compilation / Graph capture
11. §dim-sampling — Sampling
12. §dim-lora — LoRA
13. §dim-quant — 量化
14. §dim-moe — MoE
15. §dim-multimodal — 多模态
16. §dim-serving — 服务化 / API
17. §dim-hardware — 硬件后端
18. §dim-prefix-cache — Prefix caching
19. §dim-batching — Continuous batching / Chunked prefill（深度对比见 [comparison/topics/chunked-prefill.md](topics/chunked-prefill.md)）
20. §dim-structured — Structured output
21. §dim-async-schedule — 异步调度 / CPU-GPU overlap（深度对比见 [comparison/topics/scheduler.md §5](topics/scheduler.md)）
22. §dim-sync-schedule — 同步调度 / 默认无 overlap 路径（深度对比见 [comparison/topics/sync-schedule.md](topics/sync-schedule.md)）
23. §dim-cp-sp — Context Parallel / Sequence Parallel（深度对比见 [comparison/topics/cp-sp.md](topics/cp-sp.md)）
24. §dim-flashcomm — FlashComm 通信优化（深度对比见 [comparison/topics/flashcomm.md](topics/flashcomm.md)）

> [!todo] VERIFY: 表中很多 cell 是基于 overview 的初步映射，深度对比时务必逐条 ingest 校正。
