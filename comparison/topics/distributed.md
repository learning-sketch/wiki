---
type: comparison
project: cross
status: verified
confidence: high
verified_against: 2026-04-18
sources:
  - d:\design\MindIE-LLM\mindie_llm\runtime\utils\distributed\parallel_info_manager.py
  - d:\design\MindIE-LLM\mindie_llm\runtime\utils\distributed\__init__.py
  - d:\design\MindIE-LLM\mindie_llm\runtime\utils\distributed\communication_op.py
  - d:\design\MindIE-LLM\mindie_llm\runtime\utils\distributed\pipeline_parallel.py
  - d:\design\MindIE-LLM\mindie_llm\runtime\layers\linear\linear.py
  - d:\design\MindIE-LLM\mindie_llm\runtime\layers\fused_moe\fused_moe.py
  - d:\design\MindIE-LLM\mindie_llm\runtime\layers\fused_moe\token_dispatcher.py
  - d:\design\MindIE-LLM\mindie_llm\distributed\kv_transfer\kv_transfer_agent.py
  - d:\design\MindIE-LLM\src\utils\process_group.cpp
  - d:\design\MindIE-LLM\src\engine\llm_engine.cpp
  - d:\design\vllm\vllm\distributed\parallel_state.py
  - d:\design\vllm\vllm\distributed\communication_op.py
  - d:\design\vllm\vllm\distributed\device_communicators
  - d:\design\vllm\vllm\distributed\eplb\policy\default.py
  - d:\design\vllm\vllm\distributed\eplb\eplb_state.py
  - d:\design\vllm\vllm\distributed\elastic_ep
  - d:\design\vllm\vllm\v1\worker\dp_utils.py
  - d:\design\vllm\vllm\v1\worker\cp_utils.py
  - d:\design\vllm\vllm\v1\worker\gpu\pp_utils.py
  - d:\design\vllm\vllm\v1\worker\gpu\eplb_utils.py
  - d:\design\vllm\vllm\v1\executor\abstract.py
  - d:\design\sglang\python\sglang\srt\distributed\parallel_state.py
  - d:\design\sglang\python\sglang\srt\distributed\communication_op.py
  - d:\design\sglang\python\sglang\srt\distributed\device_communicators
  - d:\design\sglang\python\sglang\srt\layers\dp_attention.py
  - d:\design\sglang\python\sglang\srt\managers\scheduler.py
  - d:\design\sglang\python\sglang\srt\managers\scheduler_pp_mixin.py
  - d:\design\sglang\python\sglang\srt\managers\data_parallel_controller.py
  - d:\design\sglang\python\sglang\srt\eplb\eplb_manager.py
  - d:\design\sglang\python\sglang\srt\eplb\eplb_simulator
  - d:\design\sglang\python\sglang\srt\eplb\eplb_algorithms
  - d:\design\sglang\python\sglang\srt\elastic_ep\elastic_ep.py
  - d:\design\sglang\python\sglang\srt\elastic_ep\expert_backup_manager.py
related:
  - comparison/dimensions.md
  - comparison/topics/cp-sp.md
  - comparison/topics/flashcomm.md
  - vllm/entities/MultiprocExecutor.md
  - sglang/entities/Scheduler.md
  - sglang/modules/distributed.md
  - sglang/modules/eplb.md
  - sglang/modules/elastic_ep.md
---

# Cross-project Comparison: Distributed Parallelism (TP / PP / DP / EP)

> 三项目分布式并行对比。覆盖 [§dim-distributed](../dimensions.md)。
>
> **本页只覆盖 4 大并行轴**：TP（tensor）、PP（pipeline）、DP（data，含两种语义）、EP（MoE expert）+ EPLB / Elastic-EP。
> **CP/SP** 已有 [comparison/topics/cp-sp.md](cp-sp.md)（zigzag CP / PCP / DCP / ATTN_CP / ATTN_INNER_SP），本页只在 §7 简短列出对照。
> **TP 通信优化** 已有 [comparison/topics/flashcomm.md](flashcomm.md)，本页只在 §3 末尾链回。
>
> 命名澄清（贯穿全文）：
> - **DP** 这个词在三家有 **2 种不同语义**——下文 §5 单独区分 "DP-attention（attention 内）" vs "DP-batch（每 GPU 进程跑一份请求）"。

---

## TL;DR

| 轴 | MindIE | vLLM | SGLang |
|---|---|---|---|
| **TP** | ✅，`ParallelType.{ATTN_TP, ATTN_O_PROJ_TP, MLP_TP, WORLD_EMBED_TP, LM_HEAD_TP, MOE_TP}` 6 种细分 | ✅，统一 `tp_group`（[parallel_state.py:1219-1221](d:\design\vllm\vllm\distributed\parallel_state.py)） | ✅，`tp_group` + 派生 `attention_tp` / `moe_tp` 两个子组（[parallel_state.py:1710-1981](d:\design\sglang\python\sglang\srt\distributed\parallel_state.py)） |
| **PP** | **草稿/未接入主链路**（`pipeline_parallel.py` 引用的 `ParallelType.PP` 不存在；`has_pp()` 恒 False） | ✅，`pp_group` + `pp_broadcast` / `pp_receive` 原语（[v1/worker/gpu/pp_utils.py:10-41](d:\design\vllm\vllm\v1\worker\gpu\pp_utils.py)）；**无 1F1B / interleaved 关键字** | ✅ **最完整**，独立 `event_loop_pp` + `SchedulerPPMixin`（[scheduler_pp_mixin.py:47-146](d:\design\sglang\python\sglang\srt\managers\scheduler_pp_mixin.py)），含 microbatch + async-send/sync-recv + `pp_async_batch_depth` |
| **DP-attention** | ✅，`ParallelType.ATTN_DP` 一等公民（[parallel_info_manager.py:33-47](d:\design\MindIE-LLM\mindie_llm\runtime\utils\distributed\parallel_info_manager.py)） | ❌ N/A，全仓库无 `enable_dp_attention` / `dp_attention` | ✅，独立模块 [layers/dp_attention.py](d:\design\sglang\python\sglang\srt\layers\dp_attention.py) + `enable_dp_attention` server arg + `SchedulerDPAttnMixin` |
| **DP-batch** | ✅，`dp_size` 配置 + `dp_rank_id` 路由 | ✅，`dp_utils.coordinate_batch_across_dp` + `request_wave` 跨 DP rank（[dp_utils.py:102-229](d:\design\vllm\vllm\v1\worker\dp_utils.py), [v1/engine/core.py:317-321](d:\design\vllm\vllm\v1\engine\core.py)） | ✅，`dp_size` server arg + `data_parallel_controller.py` 多进程路由 |
| **EP** | ✅，`ParallelType.{MOE_TP, MOE_EP, MOE_EP_MC2}` 3 种 | ✅，`ep_group`（[parallel_state.py:1254-1260](d:\design\vllm\vllm\distributed\parallel_state.py)） + 独立 `eplb_group` 防死锁 | ✅，`moe_ep_group` + `moe_dp_group` + `moe_tp_group` 3 个子组（[parallel_state.py:1885-1908](d:\design\sglang\python\sglang\srt\distributed\parallel_state.py)） |
| **EPLB（Expert load balancing）** | **N/A**（grep 全仓库无 `eplb`） | ✅，[`distributed/eplb/`](d:\design\vllm\vllm\distributed\eplb)（policy/state/rebalance_execute）+ [`v1/worker/gpu/eplb_utils.py`](d:\design\vllm\vllm\v1\worker\gpu\eplb_utils.py)（`EPLBController`、`step_eplb_after` 装饰器） | ✅ **最完整**，[`srt/eplb/`](d:\design\sglang\python\sglang\srt\eplb)（manager/algorithms/simulator/expert_distribution）+ 周期+阈值门控（[eplb_manager.py:39-106](d:\design\sglang\python\sglang\srt\eplb\eplb_manager.py)） |
| **Elastic EP（容错/扩缩）** | **N/A** | ✅，[`distributed/elastic_ep/`](d:\design\vllm\vllm\distributed\elastic_ep) + `StatelessGroupCoordinator` | ✅，[`srt/elastic_ep/`](d:\design\sglang\python\sglang\srt\elastic_ep)（`active_ranks` + `ExpertBackupManager` 跨节点 expert 备份） |
| **进程组 backend** | HCCL（`ProcessGroupHCCL.Options`，[parallel_info_manager.py:278-304](d:\design\MindIE-LLM\mindie_llm\runtime\utils\distributed\parallel_info_manager.py)） + Gloo（C++ side, [process_group.h:19](d:\design\MindIE-LLM\src\include\utils\process_group.h)） | NCCL / pynccl / custom_all_reduce / symm_mem / flashinfer / Ray / SHM 等 **18 个 communicator**（[device_communicators/](d:\design\vllm\vllm\distributed\device_communicators)） | NCCL / pynccl / custom / symm / mscclpp / shm + npu/xpu/hpu 后端（[device_communicators/](d:\design\sglang\python\sglang\srt\distributed\device_communicators)） |
| **顶层 `distributed/` 行数 / 文件数** | ~6 .py（runtime 侧）+ 1 .py（顶层 KV agent 占位） | ~30 .py（不含子目录）+ 18 device_communicators + 4 子目录（eplb / elastic_ep / kv_transfer / weight_transfer / ec_transfer） | 22 .py（含 device_communicators） + 单独 `eplb/` `elastic_ep/` 子目录 |

---

## 1. 进程组 / Group 抽象对照

| 项目 | 抽象对象 | 创建函数 | 主要 group 名 |
|---|---|---|---|
| MindIE | `ParallelInfoManager` 单例 + 12 种 `ParallelType` enum | [`parallel_info_manager.py:33-47, 192-193, 387-396`](d:\design\MindIE-LLM\mindie_llm\runtime\utils\distributed\parallel_info_manager.py)；初始化 [`__init__.py:65-69`](d:\design\MindIE-LLM\mindie_llm\runtime\utils\distributed\__init__.py) | `WORLD / ATTN_TP / ATTN_DP / ATTN_CP / ATTN_INNER_SP / ATTN_O_PROJ_TP / MLP_TP / WORLD_EMBED_TP / LM_HEAD_TP / MOE_TP / MOE_EP / MOE_EP_MC2`（**12 种，不含 PP**） |
| vLLM | `GroupCoordinator` 实例 × N | [`parallel_state.py:1484-1582`](d:\design\vllm\vllm\distributed\parallel_state.py) `initialize_model_parallel(...)` | `tp / pp / dp / ep / eplb / dcp / pcp`（7 种） |
| SGLang | `GroupCoordinator` 实例 × N | [`parallel_state.py:1710-1981`](d:\design\sglang\python\sglang\srt\distributed\parallel_state.py) `initialize_model_parallel(...)` | `world / tp / pp / attn_cp / attention_tp / moe_dp / moe_ep / moe_tp`（**8 种**） |

> synthesis: **MindIE 把"哪一层用哪种 TP"枚举到 enum 里**（如 `ATTN_O_PROJ_TP` 与 `MLP_TP` 是不同 enum），而 vLLM/SGLang 用一个统一 `tp_group`，靠模型代码自己决定要不要做 all_reduce。MindIE 的方式更细粒度（可以在 attention 与 MLP 用不同 TP 策略），代价是 `ParallelType` 与新模型架构耦合度高。
>
> SGLang 把 attention 与 MoE **各自再分 (TP, DP, CP)** 子组（`attention_tp` + `attn_cp` + `attn_dp` 用于 attention；`moe_tp` + `moe_dp` + `moe_ep` 用于 MoE），这是 vLLM 没有的（vLLM 没拆 attn_tp / moe_tp）。

---

## 2. TP（Tensor Parallel）

### 实现位置

| 项目 | rank/world API | all_reduce 落地 |
|---|---|---|
| MindIE | `ParallelInfoManager.get(ParallelType.ATTN_TP).rank/world_size`（[parallel_info_manager.py:387-396](d:\design\MindIE-LLM\mindie_llm\runtime\utils\distributed\parallel_info_manager.py)） | **散布在 layer 内**，例如 [linear.py:299-300](d:\design\MindIE-LLM\mindie_llm\runtime\layers\linear\linear.py) `dist.all_reduce(..., group=self.parallel_info.process_group)`；[`communication_op.py`](d:\design\MindIE-LLM\mindie_llm\runtime\utils\distributed\communication_op.py) **只有 all_gather/gather**，无 all_reduce 封装 |
| vLLM | `get_tp_group()` / `get_tensor_model_parallel_world_size()` / `get_tensor_model_parallel_rank()`（[parallel_state.py:1219-1221, 1827-1834](d:\design\vllm\vllm\distributed\parallel_state.py)） | `tensor_model_parallel_all_reduce(...)` → `get_tp_group().all_reduce` → `device_communicator.all_reduce`（[communication_op.py:9-43](d:\design\vllm\vllm\distributed\communication_op.py), [parallel_state.py:492-519](d:\design\vllm\vllm\distributed\parallel_state.py)） |
| SGLang | `get_tp_group()` / `get_tensor_model_parallel_world_size()` / `_rank()`（[parallel_state.py:1447-1492, 2109-2116](d:\design\sglang\python\sglang\srt\distributed\parallel_state.py)） | `communication_op.py:16-55` 对 `get_tp_group()` 的 `all_reduce` / fused allreduce+RMSNorm / `all_gather` / `gather` / `broadcast_tensor_dict` |

### Device communicators 后端清单

| 项目 | backend 数 | 重点 |
|---|---|---|
| MindIE | 1（HCCL via `torch.distributed`） + 1 C++ Gloo | C++ 侧 [process_group.cpp:26](d:\design\MindIE-LLM\src\utils\process_group.cpp) 用 `TCPStore`；Python 侧 NPU 通信全走 HCCL（[__init__.py:65-69](d:\design\MindIE-LLM\mindie_llm\runtime\utils\distributed\__init__.py) `backend='hccl'`） |
| vLLM | **18 个** [`device_communicators/`](d:\design\vllm\vllm\distributed\device_communicators) | 含：`base / cpu / cuda / cuda_wrapper / custom_all_reduce / flashinfer_all_reduce / quick_all_reduce / pynccl / pynccl_wrapper / pynccl_allocator / symm_mem / all_reduce_utils / all2all / shm_broadcast / shm_object_storage / ray_communicator / xpu / mnnvl_compat` |
| SGLang | **17 .py / 13 逻辑后端** [`device_communicators/`](d:\design\sglang\python\sglang\srt\distributed\device_communicators) | 含：`pynccl / pynccl_wrapper / pynccl_allocator / custom_all_reduce (×4 文件) / quick_all_reduce / shm_broadcast / mooncake_transfer_engine / torch_symm_mem / pymscclpp / cuda_wrapper / npu / xpu / hpu`；详见 [sglang/modules/distributed.md §Device communicators](../../sglang/modules/distributed.md#device-communicators13-logical-backends--17-py) |

> synthesis: **vLLM 的多 backend 是为了 NVIDIA 各代卡**（custom_all_reduce 走 IPC、symm_mem 走 NVLINK SHARP、quick_all_reduce 走小消息路径、flashinfer 走 fused 路径）。**SGLang 多 backend 是为多硬件**（NPU/XPU/HPU）。**MindIE 单 HCCL** 因为只面向 Ascend NPU。

### TP 通信优化（链回）

→ 详见 [comparison/topics/flashcomm.md](flashcomm.md)：
- MindIE 有专门 `is_flashcomm_supported` 模型描述符
- vLLM / SGLang 走 attention backend / `LayerCommunicator` 等等价路径

---

## 3. PP（Pipeline Parallel）

| 项目 | 状态 | 关键代码 |
|---|---|---|
| MindIE | ⚠️ **草稿，未接入主链路** | [pipeline_parallel.py:7-39](d:\design\MindIE-LLM\mindie_llm\runtime\utils\distributed\pipeline_parallel.py) 三个函数（`recv_pipeline_tensors` / `send_pipeline_tensors` / `broadcast_pipeline_tokens`）都调 `ParallelType.PP`，但 `ParallelType` enum **没有 PP 成员**（[parallel_info_manager.py:33-47](d:\design\MindIE-LLM\mindie_llm\runtime\utils\distributed\parallel_info_manager.py)）；`has_pp()` 恒 False（[parallel_info_manager.py:243-257](d:\design\MindIE-LLM\mindie_llm\runtime\utils\distributed\parallel_info_manager.py)）；`model_runner.py` 内**未** grep 到 `pipeline_parallel` 调用 |
| vLLM | ✅ 进程组就绪，原语轻量 | `get_pp_group()`（[parallel_state.py:1238-1240](d:\design\vllm\vllm\distributed\parallel_state.py)）；PP 组初始化 [parallel_state.py:1625-1641](d:\design\vllm\vllm\distributed\parallel_state.py) `init_model_parallel_group(..., group_name="pp")`；原语 [v1/worker/gpu/pp_utils.py:10-41](d:\design\vllm\vllm\v1\worker\gpu\pp_utils.py) `pp_broadcast` + `pp_receive`（基于 `torch.distributed.broadcast`）；调度算法关键字 `1F1B` / `interleaved` **N/A**——但有 "Async scheduled PP" 注释（[gpu_model_runner.py:1270-1273, 3498-3501](d:\design\vllm\vllm\v1\worker\gpu_model_runner.py)） |
| SGLang | ✅ **最完整** | 进程组：[parallel_state.py:1964-1980](d:\design\sglang\python\sglang\srt\distributed\parallel_state.py) `group_name="pp"`；调度：[scheduler_pp_mixin.py:47-146](d:\design\sglang\python\sglang\srt\managers\scheduler_pp_mixin.py) 实现完整 `event_loop_pp`，含：(a) 多 stage 顺序执行；(b) async send + sync recv；(c) 可选 `pp_async_batch_depth` 重叠；调度入口 [scheduler.py:3635-3636](d:\design\sglang\python\sglang\srt\managers\scheduler.py) `if pp_size > 1: scheduler.event_loop_pp()` |

> [!warning] CONTRADICTION: MindIE 的 PP 现状有**双重描述**：
> - 代码层：`pipeline_parallel.py` 存在但与 `ParallelType` enum 不一致（API 残缺）
> - 设计层：`docs/mindie_generator_aclgraph_pp_design.md` 1733 行设计文档详细描述 PP 改造方案
>
> 详见 `mindie/topics/aclgraph-pp.md`（已删）（已建专题页，覆盖：当前主线为何不能用 PP、推荐 7 层架构、可复用资产）。

> synthesis: **PP 实现完成度排序：SGLang > vLLM >> MindIE（仅设计文档）**。三家都没有声明用 1F1B / interleaved schedule，SGLang 的 microbatch + async-send/sync-recv 是常规非 1F1B 流水。

---

## 4. DP（Data Parallel）—— 两种语义 + 三家映射

> **本节最容易踩坑**。"DP" 在三家代码里实际指 **2 种不同的事**：
>
> | 语义 | 它在做什么 | 类比 |
> |---|---|---|
> | **DP-batch** | 多 GPU 进程各跑一份独立 batch，scheduler 路由请求到不同 DP rank | "多副本部署" |
> | **DP-attention** | 同一 batch 在 attention 层切 sequence，但在 MLP / MoE 层 all_gather 回完整 batch | "attention 内的 DP，MLP/MoE 仍是 TP" |

### DP-batch（多进程跑独立 batch）

| 项目 | 配置 | 路由位置 |
|---|---|---|
| MindIE | `dp_size = parse("dp", default_value=1)`（config.py:88-91） | `dp_rank_id = (rank // (cp_size * tp_size)) % dp_size`（router_impl.py:214-215） |
| vLLM | `parallel_config.data_parallel_size` | `request_wave` 跨 DP rank 协调请求 wave（[v1/engine/core.py:317-321](d:\design\vllm\vllm\v1\engine\core.py), [v1/engine/core_client.py:296-297](d:\design\vllm\vllm\v1\engine\core_client.py)）；运行时同步 token 数 / ubatch / cudagraph 模式 [v1/worker/dp_utils.py:20-36, 39-55, 102-229](d:\design\vllm\vllm\v1\worker\dp_utils.py) |
| SGLang | `--dp-size` server arg | [`managers/data_parallel_controller.py`](d:\design\sglang\python\sglang\srt\managers\data_parallel_controller.py)（多 worker 进程 + ZMQ 路由），[447-462](d:\design\sglang\python\sglang\srt\managers\data_parallel_controller.py) 处理 `enable_dp_attention` 时给每 GPU 进程一个 dp_rank |

### DP-attention（attention 内 DP，MLP/MoE 仍 TP）

| 项目 | 实现 | 锚点 |
|---|---|---|
| MindIE | `ParallelType.ATTN_DP` 一等公民；`get_num_tokens_across_dp_cpu/npu` 用 ATTN_DP 的 `cpu_process_group` 做跨 DP rank 同步 | [parallel_info_manager.py:33-47](d:\design\MindIE-LLM\mindie_llm\runtime\utils\distributed\parallel_info_manager.py), [model_runner.py:609-625](d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\model_runner.py) |
| vLLM | **N/A**（无 `enable_dp_attention` / `dp_attention` 字符串） | — |
| SGLang | 独立模块 [`layers/dp_attention.py`](d:\design\sglang\python\sglang\srt\layers\dp_attention.py)（含 `compute_dp_attention_world_info`、`initialize_dp_attention`、`dp_gather_*` / `dp_scatter` / `dp_reduce_scatter_tensor`）+ `enable_dp_attention` server arg + Scheduler 继承 `SchedulerDPAttnMixin`（[scheduler.py:317-328](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）；mixin `prepare_mlp_sync_batch` 在 TP 组上 all_gather batch 元数据 [scheduler_dp_attn_mixin.py:228-241](d:\design\sglang\python\sglang\srt\managers\scheduler_dp_attn_mixin.py) | [layers/dp_attention.py:237-251, 271-302, 311-360, 443-560](d:\design\sglang\python\sglang\srt\layers\dp_attention.py), [data_parallel_controller.py:475-493](d:\design\sglang\python\sglang\srt\managers\data_parallel_controller.py) |

> [!warning] CONTRADICTION: vLLM 的 `vllm/distributed/parallel_state.py` 有 `_DP` group（[1643-1654 附近](d:\design\vllm\vllm\distributed\parallel_state.py)），但 **它是 DP-batch group，不是 DP-attention**。如果你看到 vLLM 文档说 "DP"，几乎都指 DP-batch。

> synthesis: **DP-attention 这件事 MindIE 与 SGLang 都做了，vLLM 没做**。DP-attention 的好处是 MoE 模型下 attention TP 维度可以缩到比 MoE TP 维度小（attention 是 memory-bound、TP 切多了反而慢），需要在 MLP/MoE 入口前把 sequence 拼回完整。**MindIE `ParallelType.ATTN_DP` 与 SGLang `dp_attention` 几乎是同一件事的两套实现**——这是三家中难得"理念一致、代码各做"的例子。

---

## 5. EP（Expert Parallel）+ EPLB + Elastic-EP

### 基础 EP

| 项目 | rank/group API | MoE 层入口 |
|---|---|---|
| MindIE | `ParallelType.{MOE_TP, MOE_EP, MOE_EP_MC2}` | [fused_moe.py:64-90](d:\design\MindIE-LLM\mindie_llm\runtime\layers\fused_moe\fused_moe.py) `moe_ep_rank` / `moe_ep_size` / `assign_experts`；[token_dispatcher.py](d:\design\MindIE-LLM\mindie_llm\runtime\layers\fused_moe\token_dispatcher.py) `ParallelType.MOE_EP`；模型 [models/qwen3_moe/](d:\design\MindIE-LLM\mindie_llm\runtime\models\qwen3_moe), [models/deepseek_v3/](d:\design\MindIE-LLM\mindie_llm\runtime\models\deepseek_v3) |
| vLLM | `get_ep_group()`（[parallel_state.py:1254-1260](d:\design\vllm\vllm\distributed\parallel_state.py)）；EP 组创建 [parallel_state.py:1684-1686 附近](d:\design\vllm\vllm\distributed\parallel_state.py) | `model_executor/layers/` 内 MoE 层 |
| SGLang | `get_moe_ep_group()` + `get_moe_dp_group()` + `get_moe_tp_group()`（[parallel_state.py:1885-1908, 2152-2170](d:\design\sglang\python\sglang\srt\distributed\parallel_state.py)） | [`layers/moe/token_dispatcher/`](d:\design\sglang\python\sglang\srt\layers\moe\token_dispatcher) 多 backend（`standard / deepep / flashinfer / fuseep / mooncake / nixl`）；`ep_moe/layer.py` 实现 EP 路径 |

### EPLB（Expert Load Balancing）

| 项目 | 状态 | 关键 |
|---|---|---|
| MindIE | **N/A**（grep 全仓库无 `eplb`） | — |
| vLLM | ✅ | 独立目录 [`distributed/eplb/`](d:\design\vllm\vllm\distributed\eplb)；策略 [`policy/default.py:21-60`](d:\design\vllm\vllm\distributed\eplb\policy\default.py) `DefaultEplbPolicy.balanced_packing`；状态 `eplb_state.py`；执行 `rebalance_execute.py`；worker 侧 [`v1/worker/gpu/eplb_utils.py:18-120`](d:\design\vllm\vllm\v1\worker\gpu\eplb_utils.py) `step_eplb_after` 装饰器 + `EPLBController`；**独立 `eplb_group` 防死锁**（[parallel_state.py:1688-1708](d:\design\vllm\vllm\distributed\parallel_state.py)） |
| SGLang | ✅ **最完整** | [`srt/eplb/`](d:\design\sglang\python\sglang\srt\eplb) 含：`expert_distribution.py` / `expert_location*.py` / `eplb_manager.py` / `eplb_simulator/`（含 `reader.py`，从 `*.pt` 离线读取 expert 分布）/ `eplb_algorithms/`（`deepseek.py`, `deepseek_vec.py`, `elasticity_aware.py` 等多算法）；触发：[eplb_manager.py:39-50](d:\design\sglang\python\sglang\srt\eplb\eplb_manager.py) 每 `eplb_rebalance_num_iterations` 次 forward；门控：[eplb_manager.py:93-106](d:\design\sglang\python\sglang\srt\eplb\eplb_manager.py) 平均利用率高于 `eplb_min_rebalancing_utilization_threshold` 时跳过——**周期性 + 阈值门控的动态重平衡** |

### Elastic EP（容错 / 扩缩）

| 项目 | 状态 | 关键 |
|---|---|---|
| MindIE | **N/A** | — |
| vLLM | ✅ | [`distributed/elastic_ep/`](d:\design\vllm\vllm\distributed\elastic_ep) `elastic_execute.py`；`StatelessGroupCoordinator` 用于 elastic 时 DP 组 |
| SGLang | ✅ | [`srt/elastic_ep/`](d:\design\sglang\python\sglang\srt\elastic_ep) 三个文件：[elastic_ep.py:30-73](d:\design\sglang\python\sglang\srt\elastic_ep\elastic_ep.py) `ElasticEPStateManager` 维护 `active_ranks`（健康 rank 掩码）；`expert_backup_manager.py` 跨节点 expert 权重备份（ZMQ）；`expert_backup_client.py` 客户端 |

> synthesis: **EP 完成度排序**：
> - **基础 EP**：三家都有
> - **EPLB**：vLLM 与 SGLang 有，**MindIE 完全没有**——这是 MindIE 跑大 MoE 模型（DeepSeek-V3 等）的潜在短板
> - **Elastic EP**：vLLM 与 SGLang 有，MindIE 没有——但 MindIE 的 `SwitchRole()`（`mindie/entities/BatchScheduler.md`（已删））覆盖了 PD 角色弹性切换的场景，与 elastic EP **不正交**
> - **EPLB 算法多样性**：SGLang 有多个 algorithm（deepseek / deepseek_vec / elasticity_aware）+ 离线 simulator，vLLM 只有 `DefaultEplbPolicy.balanced_packing`

---

## 6. Scheduler 接收的 rank 维度对照

三家 scheduler / executor 在 init 时显式接收的"rank 轴"数量反映其并行模型的复杂度：

| 项目 | rank 维度数 | 具体参数 | 锚点 |
|---|---|---|---|
| MindIE | **2 个**（隐含） | `tp_rank` + `dp_rank_id`；其它（`ATTN_DP`, `MOE_EP` 等）由 `ParallelInfoManager` 在 init 时按 enum 自动派生 | [parallel_info_manager.py:387-396](d:\design\MindIE-LLM\mindie_llm\runtime\utils\distributed\parallel_info_manager.py), [router_impl.py:214-215] |
| vLLM | **3-5 个**（看 executor） | `parallel_config.{tensor_parallel_size, pipeline_parallel_size, data_parallel_size, expert_parallel_size}` 经 executor 派生 worker rank | [v1/executor/multiproc_executor.py:257-274](d:\design\vllm\vllm\v1\executor\multiproc_executor.py) |
| SGLang | **6 个** | `Scheduler.__init__(server_args, port_args, gpu_id, tp_rank, moe_ep_rank, pp_rank, attn_cp_rank, moe_dp_rank, dp_rank)` | [scheduler.py:332-360](d:\design\sglang\python\sglang\srt\managers\scheduler.py) |

> synthesis: **SGLang 是三家中显式 rank 维度最多的**——6 个独立 rank 参数对应 6 个独立 process group。这反映了 SGLang 的设计哲学："每个并行轴独立可配置"。代价是用户需要懂 6 个 size。MindIE 的"配置 enum 自动派生"对用户友好但灵活性受限（要新增并行轴必须改 enum）。vLLM 介于两者之间。

---

## 7. CP / SP（仅锚点，详见 [cp-sp.md](cp-sp.md)）

| 项目 | CP/SP API |
|---|---|
| MindIE | `ParallelType.{ATTN_CP, ATTN_INNER_SP}`（[parallel_info_manager.py:33-47](d:\design\MindIE-LLM\mindie_llm\runtime\utils\distributed\parallel_info_manager.py)） |
| vLLM | `get_dcp_group()` / `get_pcp_group()` 双 group + [`v1/worker/cp_utils.py:14-58`](d:\design\vllm\vllm\v1\worker\cp_utils.py) `check_attention_cp_compatibility` / `get_total_cp_world_size` |
| SGLang | `get_attn_cp_group()`（[parallel_state.py:1464](d:\design\sglang\python\sglang\srt\distributed\parallel_state.py)） + `--enable-prefill-context-parallel`（[server_args.py:689](d:\design\sglang\python\sglang\srt\server_args.py)） + zigzag 切分 |

→ 详见 [comparison/topics/cp-sp.md](cp-sp.md)。

---

## 8. 顶层目录布局对照

```text
MindIE
├── mindie_llm/runtime/utils/distributed/   # 6 .py: ParallelInfoManager + collectives
│   ├── parallel_info_manager.py            # 12 enum + ProcessGroupHCCL
│   ├── communication_op.py                 # 仅 all_gather 系列
│   ├── pipeline_parallel.py                # 草稿（API 不一致）
│   ├── model_cache_pool.py                 # KV cache 池
│   └── __init__.py / utils.py
├── mindie_llm/distributed/                 # 仅 KV transfer agent 占位
└── src/utils/process_group.cpp             # C++ TCPStore + Gloo

vLLM
└── vllm/distributed/
    ├── parallel_state.py                   # GroupCoordinator + initialize_model_parallel
    ├── communication_op.py
    ├── stateless_coordinator.py            # elastic 支持
    ├── device_communicators/               # 18 backend
    ├── eplb/                               # policy + state + rebalance_execute
    ├── elastic_ep/                         # elastic EP
    ├── kv_transfer/                        # PD 分离（详见 PD 对比页）
    ├── ec_transfer/                        # encoder cache transfer
    └── weight_transfer/                    # 权重 IPC/NCCL

SGLang
└── srt/
    ├── distributed/                        # 22 .py（含 device_communicators 子目录）
    │   ├── parallel_state.py               # 8 group + DP-attention 派生
    │   ├── communication_op.py
    │   ├── naive_distributed.py            # 文件 rendezvous（无 NCCL）
    │   └── device_communicators/           # 17 .py / 13 逻辑后端（含 mooncake / mscclpp）
    ├── layers/dp_attention.py              # DP-attention 独立模块
    ├── eplb/                               # 完整 EPLB 体系
    │   ├── eplb_manager.py
    │   ├── eplb_simulator/                 # 离线分析
    │   └── eplb_algorithms/                # 多算法
    └── elastic_ep/                         # active_ranks + expert backup
```

---

## 9. 与你 PD 优化的关联（synthesis）

> 本节是综合性建议，不是任何一方源码原文。

如果你在 MindIE 上做大模型 + PD 分离的优化：

1. **MindIE EPLB 缺口**：
   - DeepSeek-V3 / Qwen3-MoE 这类大 MoE 模型，**expert 不均衡可吃掉 30%+ 吞吐**。MindIE 当前没有 EPLB，意味着一旦上线高并发 + 长尾 prompt 分布，hot expert 会成为瓶颈。
   - 短期 workaround：用 [SGLang `eplb_simulator/reader.py`](d:\design\sglang\python\sglang\srt\eplb\eplb_simulator\reader.py) 的离线分析方法，先确认 MindIE 部署下 expert 不均衡的程度，再决定要不要自研。
   - 长期：参考 [vLLM `DefaultEplbPolicy.balanced_packing`](d:\design\vllm\vllm\distributed\eplb\policy\default.py)（更简单）或 [SGLang `eplb_algorithms/deepseek_vec.py`](d:\design\sglang\python\sglang\srt\eplb\eplb_algorithms\deepseek_vec.py)（更完整）。

2. **MindIE PP 现状对 PD 的影响**：
   - PP **未接入 aclgraph 主线**（详 `aclgraph-pp.md`（已删））。意味着大模型只能靠 TP 切，TP 切到 16/32 时 attention 通信开销爆炸。
   - 你看到的 PD 分离能"绕开 TP 通信瓶颈"——P 节点只 prefill 一次（TP 通信开销摊薄），D 节点逐 token 解码（TP 通信占比小）。**这对 MindIE 是 PP 缺失的补偿，不是 PP 的替代**。

3. **DP-attention 是 MoE 模型 PD 分离的关键**：
   - MindIE `ATTN_DP` 与 SGLang `dp_attention` 都做了。具体到 PD 场景，**P 节点用 attention TP=4 + MoE EP=8**（attention 短，TP 通信不亏；MoE 大，EP 摊薄），**D 节点用 attention DP=4 + MoE TP/EP 混合**（attention 决定 latency，DP 比 TP 快；MoE 摊薄）。
   - vLLM 没有 DP-attention，所以 vLLM 跑大 MoE 的 PD 优化空间比 MindIE / SGLang 小。

4. **跨项目可借鉴**：
   - **从 SGLang 借鉴 `event_loop_pp` 的 microbatch 思路**（[scheduler_pp_mixin.py:47-146](d:\design\sglang\python\sglang\srt\managers\scheduler_pp_mixin.py)），可作为 MindIE PP 落地的参考实现。
   - **从 vLLM 借鉴独立 `eplb_group` 防死锁**（[parallel_state.py:1688-1708](d:\design\vllm\vllm\distributed\parallel_state.py)）—— EPLB 通信不能复用 EP 组，否则可能在 expert 重映射时阻塞 forward。

---

## Notes / Caveats

> [!todo] VERIFY: MindIE Python 侧 `ATTN_DP` 与产品文档/外部讨论中的"全局 DP"是否完全同义。
> [!todo] VERIFY: vLLM 是否有等价于 SGLang `dp_attention` 的 attention DP 实现（grep 范围已穷尽 `enable_dp_attention` / `dp_attention`，未命中，但可能在 attention backend 内有自己的轮子）。
> [!todo] VERIFY: SGLang `event_loop_pp` 的 `pp_async_batch_depth` 参数对 P50/P99 latency 的实际影响（仅看到代码注释，未见 benchmark）。
> [!todo] VERIFY: MindIE C++ 端 `distributedEnable` flag（[llm_engine.cpp:44-343](d:\design\MindIE-LLM\src\engine\llm_engine.cpp)）与 Python `parallel_info_manager` 的协作链。
> [!warning] CONTRADICTION: MindIE PP 状态——代码层（草稿、API 残缺）与设计层（1733 行设计文档）双重描述，需以 `aclgraph-pp.md`（已删） 为准。
> [!warning] CONTRADICTION: 三家 "DP" 命名分裂——同名两种语义（DP-batch vs DP-attention），跨项目讨论时务必先澄清。

## See also
- [comparison/dimensions.md §dim-distributed](../dimensions.md)
- [comparison/topics/cp-sp.md](cp-sp.md) — Context / Sequence Parallel 详细对比
- [comparison/topics/flashcomm.md](flashcomm.md) — TP 通信优化
- [comparison/topics/scheduler.md](scheduler.md) — scheduler 与 distributed rank 维度的关联
- [comparison/topics/pd-disaggregation.md](pd-disaggregation.md) — PD 分离（影响 TP/EP/DP 配比决策）
- [vllm/entities/MultiprocExecutor.md](../../vllm/entities/MultiprocExecutor.md) — vLLM executor 的 worker 拓扑
- [sglang/entities/Scheduler.md](../../sglang/entities/Scheduler.md) — SGLang Scheduler 6 rank 维度
