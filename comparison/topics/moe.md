---
type: comparison
project: cross
status: draft
confidence: high
verified_against: 2026-04-19
sources:
  - d:\design\wiki\sglang\topics\moe.md
  - d:\design\wiki\mindie\topics\moe.md
  - d:\design\sglang\python\sglang\srt\layers\moe
  - d:\design\sglang\python\sglang\srt\eplb
  - d:\design\sglang\python\sglang\srt\elastic_ep
  - d:\design\sglang\python\sglang\srt\batch_overlap
  - d:\design\MindIE-LLM\mindie_llm\runtime\layers\fused_moe
  - d:\design\MindIE-LLM\src\kernels\mie_ops\csrc\mc2\dispatch_ffn_combine
  - d:\design\vllm\vllm\model_executor\layers\fused_moe  # 68 .py
  - d:\design\vllm\vllm\config\parallel.py
  - d:\design\vllm\vllm\config\kernel.py
  - d:\design\vllm\vllm\distributed\eplb  # 9 .py
  - d:\design\vllm\vllm\distributed\elastic_ep  # 3 .py
  - d:\design\vllm\vllm\distributed\device_communicators\all2all.py
  - d:\design\vllm\vllm\v1\worker\gpu\eplb_utils.py
  - d:\design\vllm\csrc\moe  # 13 .cu
related:
  - sglang/topics/moe.md
  - mindie/topics/moe.md
  - comparison/topics/distributed.md
  - comparison/topics/pd-disaggregation.md
  - comparison/topics/flashcomm.md
  - comparison/dimensions.md
---

# MoE 体系跨项目对比

> 三项目 MoE 实现对比。覆盖 [§dim-moe](../dimensions.md#dim-moe)。

## Summary

> synthesis: 三家用**完全不同的"product space"**切 MoE：MindIE 单枚举 `MoECommType` 把通信形态 + kernel 二合一（5 enum，**实际可达 3**，2 dead branch）；vLLM 双轴 `all2all_backend (7) × moe_backend (9)` 正交 + `oracle/` 子包查 (quant×backend) 选 expert；SGLang 三层正交 `MoeA2ABackend (8 enum / 7 CLI) × MoeRunnerBackend (11) × method` + `FusedOpPool` 注册表查 `(a2a, runner)` 命中即融合。

> synthesis: **三件 MoE 高级特性**（EPLB / Elastic EP / TBO+SBO Overlap）三家覆盖：**SGLang 三件全有**（独有阈值门控、SBO、离线 simulator、Mooncake EP）；**vLLM 三件 EPLB+Elastic EP+DBO 都有但单档**（独有 async EPLB、3 EPLB 通信 backend、`SharedFusedMoE` 抽象）；**MindIE 三件全 N/A**（runtime 主线无 EPLB / 无 Elastic EP / 无 TBO 等价物，但**唯一**用 `npu_dispatch_ffn_combine` 把 dispatch+FFN+combine 合进同一 ACLNN 算子）。

## Sources

| 项目 | 主目录 / 文件 |
|---|---|
| **MindIE** | [`runtime/layers/fused_moe/`](d:\design\MindIE-LLM\mindie_llm\runtime\layers\fused_moe)（6 .py）+ [`runtime/models/{deepseek_v3,deepseek_v32,qwen3_moe}/`](d:\design\MindIE-LLM\mindie_llm\runtime\models) + [`src/kernels/mie_ops/csrc/mc2/dispatch_ffn_combine/`](d:\design\MindIE-LLM\src\kernels\mie_ops\csrc\mc2\dispatch_ffn_combine)（6 .cpp）。详 [mindie/topics/moe.md](../../mindie/topics/moe.md) |
| **vLLM**（fresh grep） | [`model_executor/layers/fused_moe/`](d:\design\vllm\vllm\model_executor\layers\fused_moe) **68 .py**（含 `prepare_finalize/` 7 + `runner/` 6 + `router/` 13 + `experts/` 8 + `oracle/` 6 子包）+ [`distributed/eplb/`](d:\design\vllm\vllm\distributed\eplb) 9 .py + [`distributed/elastic_ep/`](d:\design\vllm\vllm\distributed\elastic_ep) 3 .py + [`csrc/moe/`](d:\design\vllm\csrc\moe) 13 .cu。配置入口 [config/parallel.py](d:\design\vllm\vllm\config\parallel.py) + [config/kernel.py](d:\design\vllm\vllm\config\kernel.py)；通信 [device_communicators/all2all.py](d:\design\vllm\vllm\distributed\device_communicators\all2all.py)；EPLB 触发 [v1/worker/gpu/eplb_utils.py](d:\design\vllm\vllm\v1\worker\gpu\eplb_utils.py) |
| **SGLang** | [`srt/layers/moe/`](d:\design\sglang\python\sglang\srt\layers\moe) 42 .py + [`srt/eplb/`](d:\design\sglang\python\sglang\srt\eplb) 12 + [`srt/elastic_ep/`](d:\design\sglang\python\sglang\srt\elastic_ep) 3 + [`srt/batch_overlap/`](d:\design\sglang\python\sglang\srt\batch_overlap) 2 + [`sgl-kernel/csrc/moe/`](d:\design\sglang\sgl-kernel\csrc\moe)。详 [sglang/topics/moe.md](../../sglang/topics/moe.md)（30 KB seed） |

## 三方对照表（13 子维度）

> 每 cell 的具体行号锚点，对 SGLang 见 [sglang/topics/moe.md](../../sglang/topics/moe.md)、对 MindIE 见 [mindie/topics/moe.md](../../mindie/topics/moe.md)；vLLM 锚点 inline 给出（无 vLLM seed 页）。

### 1. MoE 实现 / 抽象层

| 维度 | MindIE | vLLM | SGLang |
|---|---|---|---|
| 顶层 nn.Module 数 | **1**：`FusedMoE` ([fused_moe.py:38](d:\design\MindIE-LLM\mindie_llm\runtime\layers\fused_moe\fused_moe.py)) | **2**：`FusedMoE(PluggableLayer)` + `SharedFusedMoE(FusedMoE)` ([layer.py:217](d:\design\vllm\vllm\model_executor\layers\fused_moe\layer.py), [shared_fused_moe.py:10](d:\design\vllm\vllm\model_executor\layers\fused_moe\shared_fused_moe.py)) | **4**：`FusedMoE` + `DeepEPMoE(FusedMoE)` + `NpuFuseEPMoE(DeepEPMoE)` + `MoriEPMoE(DeepEPMoE)` |
| Method-base 抽象 | `FusedMoEMethodBase` 单层 | `FusedMoEMethodBase` + `FusedMoEModularMethod` ([fused_moe_modular_method.py](d:\design\vllm\vllm\model_executor\layers\fused_moe\fused_moe_modular_method.py))**双层**——后者把 prepare/run/finalize 串成 modular pipeline | `FusedMoEMethodBase` + `KTEPWrapperMethod`（KTransformers AMX/CPU 卸载） |
| 抽象划分粒度 | "通信+kernel 二合一" | "**三段 modular**"：`FusedMoEPrepareAndFinalize` ABC ([modular_kernel.py:181](d:\design\vllm\vllm\model_executor\layers\fused_moe\modular_kernel.py)) + `FusedMoEModularKernel` + `FusedMoEMethodBase`，由 `maybe_init_modular_kernel` ([layer.py:600](d:\design\vllm\vllm\model_executor\layers\fused_moe\layer.py)) 装配 | "**三层正交**" dispatcher × runner × method（详 sglang seed） |
| 共享 expert 抽象 | 模型层 inline（`DeepseekV3Moe.shared_experts = DeepseekV3MLP`） | **`SharedFusedMoE` 第一类抽象** + runner/`SharedExperts` 模块——三家中**唯一**抽象成第一类对象 | 模型层 inline；无 `SharedFusedMoE` 等价类 |

### 2. a2a / token dispatcher backend（CLI 可选）

| 维度 | MindIE | vLLM | SGLang |
|---|---|---|---|
| 选择枚举 | `MoECommType` **5 enum**（[moe_comm_strategy.py:43-50](d:\design\MindIE-LLM\mindie_llm\runtime\layers\fused_moe\moe_comm_strategy.py)） | `All2AllBackend Literal` **10 字面值**（[parallel.py:40-51](d:\design\vllm\vllm\config\parallel.py)）含 2 deprecated（`naive`/`pplx`）+ 1 alias（`flashinfer_all2allv`） | `MoeA2ABackend` **8 enum** ([utils.py:23-32](d:\design\sglang\python\sglang\srt\layers\moe\utils.py)) vs CLI Literal **7**（`customized` 预留不可达） |
| 实际可达 backend 数 | **3**（`AllGather` / `MC2` / `All2AllV`；`FUSED_MC2` + `FUSED_ALLTOALL` dead branch） | **7**：`allgather_reducescatter`（默认）/ `deepep_high_throughput` / `deepep_low_latency` / `mori` / `nixl_ep` / `flashinfer_nvlink_two_sided` / `flashinfer_nvlink_one_sided` | **7**：`none` / `deepep` / `mooncake` / `nixl` / `mori` / `ascend_fuseep` / `flashinfer` |
| Dispatcher 抽象 | `MoETokenDispatcher` + 3 子类 `TokenDispatcherWith{AllGather,MC2,All2AllV}` ([token_dispatcher.py](d:\design\MindIE-LLM\mindie_llm\runtime\layers\fused_moe\token_dispatcher.py)) | `FusedMoEPrepareAndFinalizeModular` 派生 + `All2AllManagerBase` ([base_device_communicator.py:28](d:\design\vllm\vllm\distributed\device_communicators\base_device_communicator.py)) 双层；7 派生在 [prepare_finalize/](d:\design\vllm\vllm\model_executor\layers\fused_moe\prepare_finalize) 与 [device_communicators/all2all.py:41-775](d:\design\vllm\vllm\distributed\device_communicators\all2all.py)（`Naive`/`AgRs`/`DeepEPHT`/`DeepEPLL`/`NixlEP`/`FlashInferNVLink{One,Two}Sided`/`Mori`） | `BaseDispatcher` + 7 子类（详 sglang seed §7-取值矩阵） |
| 选择时机 | **运行时动态**——每次 `FusedMoE.forward` 调 `select_moe_comm_method` ([fused_moe.py:151-153](d:\design\MindIE-LLM\mindie_llm\runtime\layers\fused_moe\fused_moe.py))，按 `is_prefill` / token 数 选 | **静态**——`maybe_make_prepare_finalize` ([all2all_utils.py:89](d:\design\vllm\vllm\model_executor\layers\fused_moe\all2all_utils.py)) 在 `FusedMoE.maybe_init_modular_kernel` 构造期一次决定 | **静态**——`create_moe_dispatcher` 在 init 时按全局 `MOE_A2A_BACKEND` 实例化 |

### 3. Runner / Expert kernel backend（独立维度）

| 维度 | MindIE | vLLM | SGLang |
|---|---|---|---|
| 是否独立维度 | **N/A**——绑定在 `MoECommType` 内 | ✅ **9 取值** `KernelConfig.moe_backend`：`auto` / `triton` / `deep_gemm` / `cutlass` / `flashinfer_trtllm` / `flashinfer_cutlass` / `flashinfer_cutedsl` / `marlin` / `aiter` ([kernel.py:108-118](d:\design\vllm\vllm\config\kernel.py)) | ✅ **11 取值** `MoeRunnerBackend` |
| Runner 抽象 | N/A | `MoERunner` ABC + 唯一实现 `DefaultMoERunner` ([runner/default_moe_runner.py:13](d:\design\vllm\vllm\model_executor\layers\fused_moe\runner\default_moe_runner.py))；`create_moe_runner` 工厂 ([moe_runner_factory.py:24](d:\design\vllm\vllm\model_executor\layers\fused_moe\runner\moe_runner_factory.py)) | `MoeRunner` + `RunnerCore` + `FusedOpPool` + `PermuteMethodPool`（5 backend core） |
| (a2a, runner) 融合查表 | N/A | 通过 `oracle/` 子包（[oracle/{unquantized,fp8,mxfp4,mxfp8,nvfp4}.py](d:\design\vllm\vllm\model_executor\layers\fused_moe\oracle)）按 quant×backend 选 expert 实现 | `FusedOpPool.get_fused_func(a2a_name, runner_name)` 命中即直走融合算子 |

### 4. Expert parallelism (EP) 实现

| 维度 | MindIE | vLLM | SGLang |
|---|---|---|---|
| EP 启用 CLI | `moe_ep` server config + `moe_tp * moe_ep == world_size` 强校验 ([parallel_info_manager.py:208-214](d:\design\MindIE-LLM\mindie_llm\runtime\utils\distributed\parallel_info_manager.py)) | `--enable-expert-parallel` bool ([parallel.py:148](d:\design\vllm\vllm\config\parallel.py))；EP world = `data_parallel_size * tensor_parallel_size` 隐式派生 | `--ep-size` int + `--moe-a2a-backend` ([server_args.py:527-530](d:\design\sglang\python\sglang\srt\server_args.py)) |
| EP 进程组 | `ParallelType.{MOE_TP, MOE_EP, MOE_EP_MC2}` 3 子组；**MC2 专用组 `is_reusable=False`**（不复用 PG 缓存） | `_EP` GroupCoordinator + 独立 `_EPLB` ([parallel_state.py:1251-1272](d:\design\vllm\vllm\distributed\parallel_state.py))；EPLB 用独立组**防死锁** ([parallel_state.py:1688-1708](d:\design\vllm\vllm\distributed\parallel_state.py)) | `moe_ep_group` + `moe_dp_group` + `moe_tp_group` 3 子组 |
| Expert 切分 | `assign_experts`：均匀；不整除时**前 N-1 rank 各 ceil，最后 rank 收余数** ([fused_moe.py:276-288](d:\design\MindIE-LLM\mindie_llm\runtime\layers\fused_moe\fused_moe.py)) | `expert_placement_strategy ∈ {linear, round_robin}` ([parallel.py:161-170](d:\design\vllm\vllm\config\parallel.py))；`enable_ep_weight_filter` 跳过非 local expert disk I/O | `ExpertLocationMetadata.init_by_*` + `--init-expert-location ∈ {trivial, ...}` |

### 5. EPLB（Expert load balancing）

| 维度 | MindIE | vLLM | SGLang |
|---|---|---|---|
| 是否实现 | **N/A (verified 2026-04-19)**——`mindie_llm/runtime/` grep `eplb` **0 命中**；ATB 侧另有 EPLB 不混用（详 [mindie/topics/moe.md §EPLB 缺失](../../mindie/topics/moe.md#eplb-缺失重要-caveat)） | ✅ [`distributed/eplb/`](d:\design\vllm\vllm\distributed\eplb) **9 .py** + [`v1/worker/gpu/eplb_utils.py`](d:\design\vllm\vllm\v1\worker\gpu\eplb_utils.py) `EPLBController` + `step_eplb_after` 装饰器 | ✅ [`srt/eplb/`](d:\design\sglang\python\sglang\srt\eplb) **12 .py** |
| 算法数 | N/A | **1**：`DefaultEplbPolicy.balanced_packing` ([policy/default.py:21-60](d:\design\vllm\vllm\distributed\eplb\policy\default.py)) | **3 + hierarchical = 6 enum**：`deepseek` / `deepseek_vec` / `elasticity_aware` |
| 通信 backend | N/A | **3 子类**：`TorchDistNcclEplbCommunicator` / `TorchDistGlooStagedEplbCommunicator` / `PyNcclEplbCommunicator` ([eplb_communicator.py:51,96,170](d:\design\vllm\vllm\distributed\eplb\eplb_communicator.py)) | 复用 `MOE_EP` 通信组（无独立通信抽象） |
| 触发机制 | N/A | `step_eplb_after` 装饰 `model_runner` 三处（[gpu/model_runner.py:405,1125,1244](d:\design\vllm\vllm\v1\worker\gpu\model_runner.py)）；`step_interval=3000` 默认 | `EPLBManager._entrypoint` 每 `eplb_rebalance_num_iterations` 次 `yield from rebalance()` |
| 阈值门控 | N/A | **N/A (verified 2026-04-19)**——`EplbConfig` 仅 `step_interval` + `window_size`，无利用率阈值 | ✅ `eplb_min_rebalancing_utilization_threshold`（利用率 > 阈值时跳过 rebalance） |
| Async EPLB | N/A | ✅ `EplbConfig.use_async` + `async_worker.py` ([parallel.py:80-83](d:\design\vllm\vllm\config\parallel.py), [async_worker.py:25](d:\design\vllm\vllm\distributed\eplb\async_worker.py)) | **N/A (verified 2026-04-19)**——rebalance 通过 `yield from` 在 forward 间隙 chunk 化但同步 |
| 离线 simulator | N/A | **N/A (verified 2026-04-19)** | ✅ [`eplb_simulator/reader.py`](d:\design\sglang\python\sglang\srt\eplb\eplb_simulator) 从 `*.pt` 离线 expert 分布做模拟优化 |

### 6. Elastic EP（容错 / 扩缩）

| 维度 | MindIE | vLLM | SGLang |
|---|---|---|---|
| 是否实现 | **N/A (verified 2026-04-19)**——全仓库 grep `elastic_ep` **0 命中** | ✅ "**进程组重组型**" | ✅ "**Backup + active_ranks 型**" |
| 顶层目录 | N/A | [`distributed/elastic_ep/`](d:\design\vllm\vllm\distributed\elastic_ep) **3 .py**：`elastic_state.py`（4 状态机 enum：ScaleUpExisting 9 步 / ScaleUpNew 4 步 / ScaleDownRemaining 4 步 / ScaleDownRemoving 3 步）+ `elastic_execute.py:ElasticEPScalingExecutor` + `standby_state.py`（standby DP/EP/EPLB/WORLD 四组） | [`srt/elastic_ep/`](d:\design\sglang\python\sglang\srt\elastic_ep) **3 .py** |
| 容灾机制 | N/A | **NCCL transfer 重组型**——`ScaleUpExistingEngineState.TRANSFER_WEIGHTS` 走节点间 NCCL ([elastic_state.py:38](d:\design\vllm\vllm\distributed\elastic_ep\elastic_state.py))；`StatelessGroupCoordinator` 重建 DP/EP/EPLB | **Mooncake/NIXL register_memory 型**——`ExpertBackupManager` 子进程 + `register_memory`；worker `ExpertBackupClient` 通过 `batch_transfer_sync_read` 拉取 |
| Active rank 状态 | N/A | 通过 barrier 同步（[elastic_state.py:73](d:\design\vllm\vllm\distributed\elastic_ep\elastic_state.py) `_BarrierTimeoutError`） | `active_ranks` GPU tensor + `active_ranks_cpu` 镜像；NIXL dispatcher 缓存引用 `copy_(1 - mask_buffer)` 原地写 |
| Expert weight backup（独立进程） | N/A | **N/A (verified 2026-04-19)**——全 vllm/ grep `expert_backup` / `expert_rebalance` / `expert_redistribute` **0 命中**；权重 transfer 走临时 NCCL，无独立 backup 进程 | ✅ `ExpertBackupManager` 子进程 |
| CLI 开关 | N/A | `--enable-elastic-ep` ([parallel.py:190](d:\design\vllm\vllm\config\parallel.py)) | `--elastic-ep-backend ∈ {mooncake, nixl}` + `--enable-elastic-expert-backup` |

### 7. TBO / SBO / Dual-Batch Overlap

| 维度 | MindIE | vLLM | SGLang |
|---|---|---|---|
| 是否实现 | **N/A (verified 2026-04-19)**——全仓库 grep `enable_dbo` / `two_batch_overlap` / `single_batch_overlap` **0 命中**；通信/计算重叠走 ATB `flashcomm`（详 [comparison/topics/flashcomm.md](flashcomm.md)） | ✅ DBO（dual batch overlap，**仅 micro-batch 粒度**） | ✅ **TBO + SBO 双层** |
| Micro-batch overlap（跨层） | N/A | `--enable-dbo` + `--ubatch-size` + `dbo_decode_token_threshold=32` + `dbo_prefill_token_threshold=512` 双阈值 ([parallel.py:193-207](d:\design\vllm\vllm\config\parallel.py))；运行时 [`v1/worker/ubatching.py:20`](d:\design\vllm\vllm\v1\worker\ubatching.py) `UBatchContext` + [`gpu_ubatch_wrapper.py:96`](d:\design\vllm\vllm\v1\worker\gpu_ubatch_wrapper.py) `UBatchWrapper` | TBO：`MaybeTboDeepEPDispatcher`（持两 EP dispatcher 实例）+ `TboForwardBatchPreparer` + `execute_overlapped_operations` 按 `tbo_delta_stages` 步进 |
| 单 batch 内 combine ↔ down-gemm overlap（层内） | N/A | **N/A (verified 2026-04-19)**——无 SBO 等价物 | ✅ SBO：`CombineOverlapArgs` + `DownGemmOverlapArgs`（`torch.cuda.Stream` + `Event`），经 `MoeRunner.set_overlap_args` 注入 |
| 与 MoE 通信耦合 | N/A | DBO 通过 `enable_expert_parallel` 后路径互通；按 token 数自动 fallback | TBO 与 `moe_a2a_backend == "none"` **互斥**强校验 ([server_args.py:6629-6633](d:\design\sglang\python\sglang\srt\server_args.py))；SBO 无此约束 |

### 8. DeepEP / Mooncake / Mori / NIXL / FlashInfer / Ascend MC2 集成矩阵

| 协议 | MindIE | vLLM | SGLang |
|---|---|---|---|
| **DeepEP HT (NVSHMEM+IB)** | N/A | ✅ `DeepEPHTPrepareAndFinalize` + `DeepEPHTAll2AllManager` ([deepep_ht.py:28](d:\design\vllm\vllm\model_executor\layers\fused_moe\prepare_finalize\deepep_ht.py), [all2all.py:306](d:\design\vllm\vllm\distributed\device_communicators\all2all.py)) | ✅ `DeepEPDispatcher Normal` impl |
| **DeepEP LL** | N/A | ✅ `DeepEPLLPrepareAndFinalize` + `DeepEPLLAll2AllManager` ([deepep_ll.py:53](d:\design\vllm\vllm\model_executor\layers\fused_moe\prepare_finalize\deepep_ll.py), [all2all.py:370](d:\design\vllm\vllm\distributed\device_communicators\all2all.py)) | ✅ `DeepEPDispatcher LowLatency` impl |
| **Mooncake TE** | **MoE all2all N/A**（Mooncake 仅 mempool 用） | **MoE all2all N/A (verified 2026-04-19)**——Mooncake 限定在 KV transfer 域 | ✅ `MooncakeEPDispatcher`；elastic EP 时透传 `active_ranks` |
| **NIXL EP** | N/A | ✅ `NixlEPPrepareAndFinalize` + `NixlEPAll2AllManager` ([nixl_ep_prepare_finalize.py:49](d:\design\vllm\vllm\model_executor\layers\fused_moe\nixl_ep_prepare_finalize.py), [all2all.py:443](d:\design\vllm\vllm\distributed\device_communicators\all2all.py)) | ✅ `NixlEPDispatcher` |
| **Mori (AMD)** | N/A | ✅ `MoriPrepareAndFinalize` + `MoriAll2AllManager` ([mori_prepare_finalize.py:15](d:\design\vllm\vllm\model_executor\layers\fused_moe\mori_prepare_finalize.py), [all2all.py:775](d:\design\vllm\vllm\distributed\device_communicators\all2all.py)) | ✅ `MoriEPDispatcher Normal+LowLatency` 双 impl |
| **FlashInfer NVLink** | N/A | ✅ 双向：`FlashInferNVLinkOneSidedPrepareAndFinalize` + `FlashInferNVLinkTwoSidedPrepareAndFinalize`（后者带 `flashinfer_all2allv` alias） | ✅ `FlashinferDispatcher`（与 cutlass / cutedsl runner 配对） |
| **AllGather + ReduceScatter (默认)** | ✅ `MoECommType.ALLGATHER` 后 `all_reduce` ([fused_moe.py:194-201](d:\design\MindIE-LLM\mindie_llm\runtime\layers\fused_moe\fused_moe.py)) | ✅ **默认** `allgather_reducescatter` + `AgRsAll2AllManager` ([all2all.py:150](d:\design\vllm\vllm\distributed\device_communicators\all2all.py))；`use_sequence_parallel_moe` 加速路径 ([parallel.py:609-624](d:\design\vllm\vllm\config\parallel.py)) | ✅ `StandardDispatcher`；DP+EP=DP_size 时 `reduce_scatterv` |
| **Ascend MC2 (HCCL)** | ✅ **唯一深度集成**：`TokenDispatcherWithMC2` + `MOE_EP_MC2` 独立通信组 + C++ `npu_dispatch_ffn_combine` ACLNN op（**FFN 也合进同一 fused op**） | **N/A (verified 2026-04-19)**——无 NPU 平台 | ✅ `NpuFuseEPDispatcher` + `NpuFuseEPMoE`——但**未把 FFN 合进 dispatch**（仍走独立 expert kernel） |

### 9. C++ / CUDA 专属 kernel 与 DeepSeek-V3 优化

| 维度 | MindIE | vLLM | SGLang |
|---|---|---|---|
| 顶层 C++/CUDA 目录 | [`src/kernels/mie_ops/csrc/mc2/dispatch_ffn_combine/`](d:\design\MindIE-LLM\src\kernels\mie_ops\csrc\mc2\dispatch_ffn_combine) **6 .cpp**（op_host + op_kernel + op_api 三层） | [`csrc/moe/`](d:\design\vllm\csrc\moe) **13 .cu**（含 `permute_unpermute_kernels/` + `marlin_moe_wna16/` + `mxfp8_moe/` 子目录） | [`sgl-kernel/csrc/moe/`](d:\design\sglang\sgl-kernel\csrc\moe) ~14 算子 + [`sgl-kernel/csrc/cpu/moe.cpp`](d:\design\sglang\sgl-kernel\csrc\cpu\moe.cpp) + 量化变体 |
| **DeepSeek-V3 专属 router gemm** | **N/A**（DeepSeek 路由走 `experts_selector.select_experts` Python 调 `torch_npu.npu_moe_gating_top_k`） | ✅ [`csrc/moe/dsv3_router_gemm_{bf16,float}_out.cu`](d:\design\vllm\csrc\moe) + 入口 `dsv3_router_gemm_entry.cu`——**与 SGLang 同款命名** | ✅ [`sgl-kernel/csrc/gemm/dsv3_router_gemm_{bf16,float}_out.cu`](d:\design\sglang\sgl-kernel\csrc\gemm) + `dsv3_fused_a_gemm`（RMSNorm+量化+下投影 GEMM） |
| **dispatch+FFN+combine 单算子融合** | ✅ **唯一**：`npu_dispatch_ffn_combine` ACLNN op | **N/A (verified 2026-04-19)**——prepare_finalize 与 expert kernel 仍独立 | **N/A (verified 2026-04-19)**——`fuseep` 仅融合 dispatch+combine，FFN 独立 |
| **moe_align / topk / moe_sum** | NPU 算子 (`npu_moe_init_routing_v2` / `npu_moe_token_unpermute`) | `moe_align_sum_kernels.cu` + `topk_softmax_kernels.cu` + `grouped_topk_kernels.cu` + `moe_permute_unpermute_op.cu` | `moe_align_block_size` + `topk_softmax`/`topk_sigmoid`/`fast_topk` + `moe_sum`/`moe_sum_reduce` + `moe_fused_gate` + `kimi_k2_moe_fused_gate` |
| **量化 grouped MM** | NPU 算子（依赖 ATB） | Marlin (`marlin_moe_wna16/ops.cu`) + Mxfp8 cutlass + WNA16 (`moe_wna16.cu`) | Cutlass W4A8 (`cutlass_w4a8_moe_mm`) + GGUF (`gguf/moe.cuh`+`moe_vec.cuh`) + Marlin (`gemm/marlin/dequant.h`) |

### 10. Topk / Router 子系统

| 维度 | MindIE | vLLM | SGLang |
|---|---|---|---|
| Router 类层级 | 单函数 `select_experts` → `torch_npu.npu_moe_gating_top_k` | **8 类 `BaseRouter` 派生** ([router/](d:\design\vllm\vllm\model_executor\layers\fused_moe\router) 13 .py)：`FusedTopKRouter` / `FusedTopKBiasRouter` / `GroupedTopKRouter` / `CustomRoutingRouter` / `RoutingSimulatorRouter`（**离线模拟**） / `ZeroExpertRouter` / `BaseRouter` / `FusedMoERouter` ABC | 单 `TopK(MultiPlatformOp)` 多后端 + Triton `fused_moe_router_cudacore_kernel` |
| Sigmoid+group-topk 融合 | NPU 算子内部 | `GroupedTopKRouter` + `FusedTopKBiasRouter`（DeepSeek 风格）+ csrc `grouped_topk_kernels.cu` | sgl-kernel `moe_fused_gate` + `kimi_k2_moe_fused_gate` 模型变体 |
| Customized routing | N/A | `CustomRoutingRouter` ([custom_routing_router.py:12](d:\design\vllm\vllm\model_executor\layers\fused_moe\router\custom_routing_router.py)) + `register_router` 工厂 | **N/A (verified 2026-04-19)**——`MoeA2ABackend.CUSTOMIZED` enum 预留但 CLI 不允许 |

### 11. Expert distribution recorder / 激活捕获

| 维度 | MindIE | vLLM | SGLang |
|---|---|---|---|
| 录制实现 | **N/A (verified 2026-04-19)**——`mindie_llm/runtime/` grep `expert_distribution`/`expert_balance` **0 命中** | ✅ via EPLB `EplbState.add_model` 内建 ([eplb_state.py:241](d:\design\vllm\vllm\distributed\eplb\eplb_state.py))；`EplbConfig.window_size=1000` + `log_balancedness` ([parallel.py:58,71-78](d:\design\vllm\vllm\config\parallel.py)) | ✅ `ExpertDistributionRecorder` + [`routed_experts_capturer.py`](d:\design\sglang\python\sglang\srt\layers\moe\routed_experts_capturer.py) `_RoutedExpertsDeviceCache` |
| 跨项目继承 | N/A | **vLLM 端 [`routed_experts_capturer.py:4`](d:\design\vllm\vllm\model_executor\layers\fused_moe\routed_experts_capturer.py) 文件头注释 `# https://github.com/sgl-project/sglang/blob/.../routed_experts_capturer.py` —— 直接拷贝自 SGLang**（synthesis：跨项目代码继承的稀有显式案例） | ✅ 原始实现 |

### 12. 量化 MoE 方法

| 维度 | MindIE | vLLM | SGLang |
|---|---|---|---|
| 量化 method 类数 | TODO（runtime fused_moe quant 主要走 NPU 算子；待量化 ingest pass） | **15+ `*MoEMethod`**：Quark FP8/Int8/W4A8/MX 系列 6 + Online FP8/Mxfp8 3 + Mxfp4 1 + ModelOpt 2 + GPTQMarlin 1 + GGUF 1 + compressed_tensors W4A8 2 | 通过 `MoeRunnerConfig.quant_method` + fused 函数文件 `cutlass_moe.py` / `cutlass_w4a8_moe.py` / `flashinfer_*` / `marlin` 注册 |
| W4A8 / FP4 路径 | N/A | `compressed_tensors_moe_w4a8_{int8,fp8}.py` + `quark_w4a8_fp8`；FP4 `oracle/nvfp4.py` + `experts/trtllm_nvfp4_moe.py` | sgl-kernel `cutlass_w4a8_moe_mm`；FP4 `flashinfer_cutedsl_moe.py` |

### 13. 模型族 / MTP-spec 与 EPLB 协作

| 维度 | MindIE | vLLM | SGLang |
|---|---|---|---|
| MoE 模型族数 | **3 族**：DeepSeek-V3 + DeepSeek-V3.2（复用 V3 主干）+ Qwen3-MoE | **20+ 族**：mixtral / phimoe / olmoe / qwen2_moe / qwen3_moe / qwen3_5_mtp / qwen3_next{,_mtp} / nemotron_h{,_mtp} / minimax_m2 / mllama4 / openpangu{,_mtp} / param2moe / sarvam / step3 等 | **8+ 族**：DeepSeek V2/V3/V3.2 + Qwen3-MoE + MiMo + GLM4-MoE + BailingMoE + Llama4 + MiniMax-M2 |
| draft / MTP MoE 与 EPLB | MTP draft 不影响 expert 路由（详 [mindie/topics/speculative.md](../../mindie/topics/speculative.md)） | `EPLBController.maybe_register_speculator` 显式支持 draft MoE 加入 EPLB ([eplb_utils.py:51-81](d:\design\vllm\vllm\v1\worker\gpu\eplb_utils.py))；**`assert not enable_elastic_ep` —— elastic EP 与 draft MoE 互斥** | `speculative_moe_backend_context` / `speculative_moe_a2a_backend_context` 两 contextmanager 在 draft 路径切换 backend |

---

## §跨项目 synthesis（5 条 cross-cut 论断）

- **synthesis: 三家把 MoE 切成完全不同的"product space"**——MindIE 单枚举二合一（5 enum 实际可达 3）；vLLM `all2all_backend (7) × moe_backend (9)` 双轴正交 + `oracle/` 子包按 (quant×backend) 选 expert；SGLang 三层正交 dispatcher × runner × method + `FusedOpPool` 注册表查 (a2a, runner) 命中即融合。**vLLM 与 SGLang 的"两维独立"哲学一致**，命名差异掩盖结构同构；MindIE "单枚举"是 Ascend MC2 硬件强约束的产物（dispatch+FFN+combine 必须同一算子）。

- **synthesis: dispatcher 抽象的同构性 ≫ 命名差异**——MindIE `MoETokenDispatcher` 三派生 ≈ vLLM `FusedMoEPrepareAndFinalizeModular` 七派生 ≈ SGLang `BaseDispatcher` 七派生。**三家都把"通信形态"独立成抽象基类**，差异仅在派生数与是否绑定 kernel：vLLM/SGLang 走"dispatcher 不持 kernel，prepare/run/finalize 三段拼装"；MindIE 走"通信形态绑 kernel + 单算子融合 FFN"。前者灵活后者深度。

- **synthesis: 三家都有"枚举先于实现"的 dead-branch 问题，但 vLLM 处理最优**——MindIE `FUSED_MC2`/`FUSED_ALLTOALL` 在 enum 但生产策略表不含 `FusedMC2Strategy`（dead branch + 三方源互相矛盾）；SGLang `MoeA2ABackend.CUSTOMIZED` 在 enum 但 CLI Literal 不允许；vLLM `naive`/`pplx` 显式 deprecated 在 `__post_init__` 自动 fallback 到 `allgather_reducescatter`（[parallel.py:417-423](d:\design\vllm\vllm\config\parallel.py)）——**唯一显式处理 deprecation 而非留 dead branch 的家**。

- **synthesis: EPLB / Elastic EP / Overlap 三件套覆盖度排序 SGLang 三件全有 > vLLM 三件全有但单档 >> MindIE 三件全 N/A**。SGLang 独有：阈值门控 + SBO + 离线 simulator + Mooncake EP；vLLM 独有：async EPLB + 3 EPLB 通信 backend + draft MoE EPLB；MindIE 独有：dispatch+FFN+combine 单算子。**EPLB 缺失是 MindIE 大 MoE 部署的产品级风险**（与 [comparison/topics/distributed.md §9](distributed.md#9-与你-pd-优化的关联synthesis) 一致）。

- **synthesis: DeepSeek-V3 是三家共同的"灯塔模型"，但 csrc 优化只在 NV/AMD 两家**——`dsv3_router_gemm_{bf16,float}_out.cu` 在 vLLM `csrc/moe/` 与 SGLang `sgl-kernel/csrc/gemm/` **同款命名**，MindIE 因走 NPU 算子无独立 dsv3 csrc。同时 vLLM `routed_experts_capturer.py` 文件头**显式 attribution 引用 SGLang 同名文件**——三家中难得"代码继承显式标注"的实例，可视为"SGLang 的 expert 行为捕获方法已成跨项目 de facto"。

---

## §Anchor-driven cross-check（按 [AGENTS.md §8 rule 6](../../AGENTS.md) 必跑）

### Anchor 1: SGLang `MoeA2ABackend` enum 8 vs CLI Literal 7（"customized 预留不可达"）

| 项目 | 等价 pattern grep 结果 |
|---|---|
| **SGLang（已知锚）** | enum 8 成员，CLI Literal 7 字面值；`customized` 在 srt/ 内 grep `is_customized()` 仅本类自身定义命中（无消费方分支）—— **dead enum**。锚点 [utils.py:23-32, 64-65](d:\design\sglang\python\sglang\srt\layers\moe\utils.py) |
| **MindIE 反扫** | `MoECommType` 5 enum，但生产 `MOE_COMM_STRATEGIES` 列表 [moe_comm_strategy.py:210-214](d:\design\MindIE-LLM\mindie_llm\runtime\layers\fused_moe\moe_comm_strategy.py) 仅含 3 Strategy 类；`FusedMC2Strategy` 类存在但未加入列表 → `FUSED_MC2`/`FUSED_ALLTOALL` 是 **dead branch**（架构文档 + 单测 + 生产代码 三方互不一致；详 [mindie/topics/moe.md §MoECommType 完整清单 RECONFIRMED](../../mindie/topics/moe.md#moecommtype-完整清单)）—— **SGLang 同型问题** |
| **vLLM 反扫** | `All2AllBackend Literal` 10 字面值含 `naive`+`pplx`，但 `__post_init__` 显式 `if self.all2all_backend in ["pplx", "naive"]: ... self.all2all_backend = "allgather_reducescatter"` ([parallel.py:417-423](d:\design\vllm\vllm\config\parallel.py))——**显式处理 deprecation**；额外 grep `register_dispatcher`/`customized`/`register_runner` 在 fused_moe/ 全树 **0 命中**——**无 "customized 预留" 模式** |

> **synthesis (anchor 1)**: 三家**都有 enum 与实际可达路径不一致的问题**，但严重度与处理方式不同：vLLM 显式 fallback（最优）→ SGLang enum/CLI 不同步（中等）→ MindIE 三方源互相矛盾（最差）。**vLLM 的 deprecation 模式应被另两家借鉴**。补充：vLLM 独有 `RoutingSimulatorRouter`（离线模拟）+ `ZeroExpertRouter`（消融实验）两类 router——是 SGLang/MindIE 完全没有的"研究/调试型 router"。

### Anchor 2: SGLang `is_deepep_class_backend()` 漏 nixl（[utils.py:260-263](d:\design\sglang\python\sglang\srt\layers\moe\utils.py)）

| 项目 | 等价 pattern grep 结果 |
|---|---|
| **SGLang（已知锚）** | `is_deepep_class_backend()` 把 deepep/mooncake/mori 视为同族；但 `create_moe_dispatcher` 把 deepep/mooncake/mori/**nixl** 一同包成 `MaybeTboDeepEPDispatcher`——**两处分类不一致** ([utils.py:260-263](d:\design\sglang\python\sglang\srt\layers\moe\utils.py) vs [fused_moe_triton/layer.py:84-88](d:\design\sglang\python\sglang\srt\layers\moe\fused_moe_triton\layer.py)) |
| **vLLM 反扫** | grep `use_deepep_ht_kernels`/`use_deepep_ll_kernels`/`use_nixl_ep_kernels`/`use_mori_kernels`——**每 backend 独立 bool flag** ([all2all_utils.py:135-289](d:\design\vllm\vllm\model_executor\layers\fused_moe\all2all_utils.py))，无"族归类"函数；`use_batched_dp_moe` `in ("deepep_low_latency", "nixl_ep")` 把 nixl 与 deepep_ll **显式归同族** ([parallel.py:625-631](d:\design\vllm\vllm\config\parallel.py))——**没有 SGLang 那种"漏 nixl"的隐式分类不一致** |
| **MindIE 反扫** | `MoECommType` 单枚举无"族"概念；`get_cached_dispatcher` ([moe_comm_method.py](d:\design\MindIE-LLM\mindie_llm\runtime\layers\fused_moe\moe_comm_method.py)) 按 enum 直接映射——**无 SGLang 那种 wrapper 抽象族归类问题** |

> **synthesis (anchor 2)**: SGLang 的"漏 nixl"是**抽象层级溢出的代价**——把多 backend 统一包成 `MaybeTboDeepEPDispatcher` wrapper 后，"deepep 族"语义边界变模糊。vLLM 用"每 backend 独立 bool flag + 显式同族集合"避开；MindIE 用"单枚举无族"避开。**SGLang 应补 `is_nixl()` 进 `is_deepep_class_backend()` 或拆分两个分类函数**（已在 sglang seed [`>[!todo] VERIFY`](../../sglang/topics/moe.md#notes--caveats) 标记）。

---

## §综合 cheat sheet

| 行 | MindIE | vLLM | SGLang |
|---|---|---|---|
| 总 .py 数（fused_moe + EPLB + Elastic + Overlap） | 6 + 0 + 0 + 0 = **6** | 68 + 9 + 3 + 0 = **80** | 42 + 12 + 3 + 2 = **59** |
| 核心抽象数（Module + Method-base） | 1 + 1 = 2 | 2 + 2 = 4 | 4 + 2 = 6 |
| a2a backend（CLI 可选） | 3 | 7 | 7 |
| Runner backend（独立维度） | 0（绑定枚举） | 9 | 11 |
| EPLB 算法数 | 0 | 1 | 3 (+3 hierarchical) |
| EPLB 通信 backend | N/A | 3 | 1（复用 EP 组） |
| Elastic EP 类型 | N/A | 进程组重组型 | Backup + active_ranks 型 |
| Micro-batch overlap | N/A | DBO 单档 | TBO + SBO 双档 |
| DeepSeek-V3 专属 csrc | NPU 算子路径 | dsv3_router_gemm 同款 | dsv3_router_gemm + dsv3_fused_a_gemm |
| dispatch+FFN+combine 单算子 | ✅ Ascend MC2 唯一 | N/A | N/A |
| Mooncake 用于 MoE all2all | N/A（仅 KV transfer） | N/A（仅 KV transfer） | ✅ `MooncakeEPDispatcher` |
| 共享 expert 抽象第一类对象 | N/A（模型层 inline） | ✅ `SharedFusedMoE` | N/A（模型层 inline） |
| 离线 EPLB simulator | N/A | N/A | ✅ `eplb_simulator/reader.py` |
| MoE 模型族数 | 3 | 20+ | 8+ |
| 量化 *MoEMethod 类数 | TODO（NPU 算子路径） | 15+ | 通过 `MoeRunnerConfig.quant_method` 注入 |

---

## §与 PD 优化的关联（synthesis）

> 本节是综合性建议，不是任何一方源码原文（仿 [comparison/topics/prefix-cache.md](prefix-cache.md) / [engine-architecture.md](engine-architecture.md) 模式）。

1. **MindIE EPLB 缺口仍是大 MoE 部署首要风险**（与 [comparison/topics/distributed.md §9.1](distributed.md#9-与你-pd-优化的关联synthesis) 一致）：DeepSeek-V3 / Qwen3-MoE 在高并发 + 长尾 prompt 下 hot expert 可吃掉 30%+ 吞吐。短期借 [`sglang/srt/eplb/eplb_simulator/reader.py`](d:\design\sglang\python\sglang\srt\eplb\eplb_simulator) 离线分析；长期参考 vLLM `DefaultEplbPolicy.balanced_packing` 简版或 SGLang `eplb_algorithms/deepseek_vec.py` 完整版。

2. **MoE PD 分离的核心：P 节点 vs D 节点 (TP, EP, DP) 配比**：P 节点（prefill bound）EP 大化以摊薄通信；D 节点（decode bound）attention DP（[comparison/topics/distributed.md §4 DP-attention](distributed.md#4-dp-data-parallel--两种语义--三家映射)）+ MoE TP/EP 混合。**vLLM 没有 DP-attention，所以 vLLM 的大 MoE PD 优化空间比 MindIE/SGLang 小**——这点在 MoE 维度独立于 EPLB 又一次浮现。

3. **MindIE `npu_dispatch_ffn_combine` 单算子融合是 PD-D 端 latency 优化的"独家底牌"**：少 2 次 kernel 启动 + 数据局部性；但代价是无法独立替换 expert kernel——量化升级（W4A8 → MXFP4）必须改 ACLNN op，灵活性 < vLLM `oracle/` + `experts/` 子包模式。

4. **TBO/DBO 与 PD 互补不互斥**：DBO/TBO 切两 micro-batch 在 dispatch/expert/combine 三阶段交错；P 节点 micro-batch 切分粒度可激进（prefill batch friendly），D 节点需小阈值（vLLM 默认 `dbo_decode_token_threshold=32`）。**MindIE PD-D 节点引入 TBO 等价物**（基于 `flashcomm` + 自研 micro-batch）是吞吐 vs latency 平衡的下一步关键改造点。

5. **跨项目可借鉴的 "MoE+PD 协同" 具体技术**：(a) 从 SGLang 借 `routed_experts_capturer.py`（vLLM 已经拷贝了）做 PD 部署前的 expert hotness profiling；(b) 从 vLLM 借 `EplbConfig.use_async` + `async_worker.py` 让 EPLB 不阻塞 forward（PD-D 低 latency 关键）；(c) 从 MindIE 借 `MOE_EP_MC2 is_reusable=False` 独立通信组的 buffer 配额隔离思路，避免大 token 突发把 EPLB rebalance 通信卡住。

---

## Notes / Caveats

> [!warning] CONTRADICTION: **三家 "MoE backend" 命名陷阱**——vLLM `KernelConfig.moe_backend` = expert kernel（与 a2a 通信无关）；SGLang `MoeRunnerBackend` = expert kernel（与 `MoeA2ABackend` 正交）；MindIE `MoECommType` = 通信形态 + kernel **二合一**。跨项目讨论"MoE backend"时务必先明确指 a2a 还是 kernel。

> [!warning] CONTRADICTION: **三家 "FUSED" 命名陷阱**——MindIE `FUSED_MC2` = dispatch+FFN+combine 单算子（**真融合**，但 dead branch）；SGLang `fuseep`/`NpuFuseEPDispatcher` = dispatch+combine 融合（**FFN 仍独立**）；vLLM 无"fused dispatch+combine"等价命名（DeepEP HT 模式接近但未在命名上突出）。直接看 "FUSE/FUSED" 关键字会被名词漂移误导。

> [!warning] CONTRADICTION: **`Mooncake` 在三家覆盖域不一致**——SGLang：Mooncake 同时用于 KV transfer **和** MoE all2all（`MooncakeEPDispatcher`）；vLLM：Mooncake **只**用于 KV transfer（`distributed/kv_transfer/`），MoE all2all **N/A**；MindIE：Mooncake 只用于 mempool，MoE all2all 不依赖 Mooncake。跨项目 "Mooncake 集成度" 比较时务必拆 KV vs MoE 两域。

> [!todo] VERIFY: vLLM `EplbConfig.window_size=1000` + `step_interval=3000` 固定周期 vs SGLang `eplb_min_rebalancing_utilization_threshold` 利用率自适应的 ROI 对比——同 DeepSeek-V3 + 同 expert 分布下 benchmark 哪种更优未见。

> [!todo] VERIFY: vLLM `routed_experts_capturer.py` 注释引用 SGLang commit `bed301a5acaa9577c9aa706468bdf242f6a43051`——与 SGLang 当前 HEAD 之间是否有功能 drift？若 drift，vLLM 端是否落后？

> [!todo] VERIFY: MindIE `mie_ops.npu_dispatch_ffn_combine` 与 SGLang `NpuFuseEPDispatcher`（[fuseep.py:43](d:\design\sglang\python\sglang\srt\layers\moe\token_dispatcher\fuseep.py)）在 NPU 上的实测吞吐差异——前者把 FFN 合进单算子（理论 latency 低），后者 FFN 独立（quant 切换灵活）。生产 DeepSeek-V3 部署上谁 ROI 高需 benchmark。

> [!todo] VERIFY: vLLM `enable_dbo` 默认 `False`、SGLang `--enable-two-batch-overlap` 默认 `False`——**为什么默认关？** 推测 micro-batch 切分对小 batch 不友好，但具体阈值（`dbo_*_token_threshold`）以下的 ROI 数据未见。

> [!note] **代码继承显式案例**：vLLM [`routed_experts_capturer.py:4`](d:\design\vllm\vllm\model_executor\layers\fused_moe\routed_experts_capturer.py) 文件头注释直接引用 SGLang GitHub URL —— 三家中难得"代码 attribution 显式标注"的实例。

---

## See also

- 三家本地实现页：
  - [sglang/topics/moe.md](../../sglang/topics/moe.md)（30 KB seed，最完整；本页 SGLang 列均锚此页）
  - [mindie/topics/moe.md](../../mindie/topics/moe.md)（含 V32 RESOLVED + FUSED_MC2 dead branch 详解 + 三方 anchor cross-check）
  - vLLM 暂无 `vllm/topics/moe.md`（**本页 vLLM 行的 fresh grep 是事实上首份合成**——未来可考虑独立 ingest）
- 维度索引与姊妹对比：
  - [comparison/dimensions.md §dim-moe](../dimensions.md#dim-moe)（已 verified 2026-04-18，含 prepare_finalize 子包描述）
  - [comparison/topics/distributed.md §5 EP+EPLB+Elastic-EP](distributed.md#5-epexpert-parallel--eplb--elastic-ep)（同主题分布式视角）
  - [comparison/topics/pd-disaggregation.md](pd-disaggregation.md)（与本页 §与 PD 优化的关联 互补）
  - [comparison/topics/flashcomm.md](flashcomm.md)（MindIE 通信/计算重叠路径，本页 TBO/SBO 维度的姊妹）
  - [comparison/topics/cp-sp.md](cp-sp.md)（attention 内并行视角，与 MoE 内 EP 互补）
- 模块页：
  - [sglang/modules/eplb.md](../../sglang/modules/eplb.md) / [elastic_ep.md](../../sglang/modules/elastic_ep.md) / [batch_overlap.md](../../sglang/modules/batch_overlap.md)
  - [mindie/entities/ParallelInfoManager.md](../../mindie/entities/ParallelInfoManager.md)
