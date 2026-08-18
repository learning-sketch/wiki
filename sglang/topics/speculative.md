---
type: topic
project: sglang
status: stale
confidence: medium
verified_against: 2026-08-18
sources:
  - d:\design\sglang\python\sglang\srt\speculative\spec_info.py:L31-L61
  - d:\design\sglang\python\sglang\srt\speculative\spec_info.py:L254-L345
  - d:\design\sglang\python\sglang\srt\speculative\spec_registry.py
  - d:\design\sglang\python\sglang\srt\speculative\base_spec_worker.py
  - d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py:L129, L1011
  - d:\design\sglang\python\sglang\srt\speculative\multi_layer_eagle_worker_v2.py:L110, L918
  - d:\design\sglang\python\sglang\srt\speculative\standalone_worker_v2.py:L35, L147
  - d:\design\sglang\python\sglang\srt\speculative\frozen_kv_mtp_worker_v2.py:L91, L676
  - d:\design\sglang\python\sglang\srt\speculative\dflash_worker_v2.py:L168
  - d:\design\sglang\python\sglang\srt\speculative\dspark_components\dspark_worker_v2.py:L76
  - d:\design\sglang\python\sglang\srt\speculative\ngram_worker.py:L71
  - d:\design\sglang\python\sglang\srt\speculative\eagle_info.py:L16-L272
  - d:\design\sglang\python\sglang\srt\speculative\eagle_utils.py:L341-L441, L649
  - d:\design\sglang\python\sglang\srt\speculative\dflash_utils.py
  - d:\design\sglang\python\sglang\srt\speculative\dflash_disaggregation.py
  - d:\design\sglang\python\sglang\srt\managers\scheduler.py:L905-L1043
  - d:\design\sglang\python\sglang\srt\managers\tp_worker.py:L308-L340, L481
  - d:\design\sglang\python\sglang\srt\server_args.py:L2074-L2243
  - d:\design\sglang\python\sglang\srt\arg_groups\speculative_hook.py
related:
  - sglang/modules/speculative.md
  - sglang/entities/TpModelWorker.md
  - sglang/entities/Scheduler.md
  - sglang/modules/sampling.md
  - sglang/modules/layers.md
  - sglang/modules/managers.md
  - sglang/modules/distributed.md
  - comparison/topics/speculative-decoding.md
  - comparison/dimensions.md
---

# Speculative Decoding 架构（SGLang 内部）

## Summary

~~SGLang 投机解码用 **`SpeculativeAlgorithm` 6 成员枚举 + 5 个算法族实现**（EAGLE / MultiLayer EAGLE / Standalone / DFlash / NGRAM），通过 **2 种 worker 接入协议**进入调度器：EAGLE / Standalone / MultiLayer-EAGLE 是 `TpModelWorker` 的真正子类；DFlash / NGRAM 是 duck-typed 包装类。~~ **已失效（pin `06f32bab` 前大重构，2026-08-18 verify 确认）**：HEAD 上 `SpeculativeAlgorithm` 为 **8 成员枚举**（[spec_info.py:L31-45](d:\design\sglang\python\sglang\srt\speculative\spec_info.py)：`DFLASH` / **`DSPARK`** / `EAGLE` / `EAGLE3` / **`FROZEN_KV_MTP`** / `STANDALONE` / `NGRAM` / `NONE`）+ [`spec_registry.py`](d:\design\sglang\python\sglang\srt\speculative\spec_registry.py) 的 **`CustomSpecAlgo` 插件注册机制**（`register_algorithm` [L222](d:\design\sglang\python\sglang\srt\speculative\spec_registry.py)，`from_string` 对内建/插件统一分发 [spec_info.py:L48-61](d:\design\sglang\python\sglang\srt\speculative\spec_info.py)）；**V1 worker 已全部删除，V2 是唯一路径**（`create_worker` [L254-307](d:\design\sglang\python\sglang\srt\speculative\spec_info.py)：V2 worker 同时驱动 overlap 与非 overlap，非 overlap 时由 scheduler 同步驱动）；**duck-typed 双轨也已消失**——所有 worker 统一继承 [`BaseSpecWorker(ABC)`](d:\design\sglang\python\sglang\srt\speculative\base_spec_worker.py) [L147]（含 `NGRAMWorker(BaseSpecWorker)` [ngram_worker.py:L71](d:\design\sglang\python\sglang\srt\speculative\ngram_worker.py) 与 `DFlashWorkerV2(BaseSpecWorker)` [dflash_worker_v2.py:L168](d:\design\sglang\python\sglang\srt\speculative\dflash_worker_v2.py)）。仍成立的论断：`Scheduler.init_model_worker` 在启用 spec 时把 `model_worker` 切到 `draft_worker`，**资源信息仍从 `tp_worker.get_worker_info()` 读取**（[scheduler.py:L993-1043](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）。verify 算子 `verify_tree_greedy` / `tree_speculative_sampling_target_only` 仍在，但 **sgl-kernel 目录已整体移出本仓库**（`sgl_kernel` 现为外部包 import，[eagle_utils.py:L386](d:\design\sglang\python\sglang\srt\speculative\eagle_utils.py)），并新增 Triton fallback [`verify_tree_greedy_triton` L341](d:\design\sglang\python\sglang\srt\speculative\eagle_utils.py) 与 CPU 版 [L57](d:\design\sglang\python\sglang\srt\speculative\eagle_utils.py)。

> ~~synthesis: SGLang 的 spec 实现 **算法 × 是否 overlap** 形成 V1/V2 双轨（同一算法两份 worker 类，通过 `create_worker(server_args)` 路由）。~~ **已失效（pin 前重构）**：V1/V2 双轨已坍缩为 **V2 单轨**——`eagle_worker.py` / `multi_layer_eagle_worker.py` / `standalone_worker.py` / `dflash_worker.py` / `eagle_info_v2.py` 等 V1/重复文件已删除，`create_worker` 不再按 overlap 分路（[spec_info.py:L254-307](d:\design\sglang\python\sglang\srt\speculative\spec_info.py)）。与 vLLM 单一 proposer 多态、MindIE Plugin + C++ placeholder 的三范式对比仍需按新结构重校（见 §Increment）。

## Sources

（行号已按 HEAD `f7101b0a` 校正；~~划线~~ = 文件/符号已删除）

- 算法枚举 / 工厂：[`SpeculativeAlgorithm` enum L31-45 + `from_string` L48-61 + `create_worker` L254-307](d:\design\sglang\python\sglang\srt\speculative\spec_info.py)；插件注册 [`spec_registry.py`](d:\design\sglang\python\sglang\srt\speculative\spec_registry.py)（`CustomSpecAlgo` L25 / `register_algorithm` L222）
- 抽象：[`base_spec_worker.py`](d:\design\sglang\python\sglang\srt\speculative\base_spec_worker.py)（`BaseSpecWorker(ABC)` L147 / `EagleDraftWorkerBase(ABC)` L57 / `HiCacheDraftMode` L26）
- ~~EAGLE V1（subclass）：`EAGLEWorker(TpModelWorker)` eagle_worker.py L79~~ **文件已删**（pin 前 V1→V2 合并）
- EAGLE V2（唯一路径）：[`EagleDraftWorker(EagleDraftWorkerBase) L129` + `EAGLEWorkerV2(BaseSpecWorker) L1011` + `forward_batch_generation L1108`](d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py)
- MultiLayer EAGLE：~~`MultiLayerEagleWorker(TpModelWorker)` multi_layer_eagle_worker.py L70~~ **文件已删**；现仅 [`MultiLayerEagleDraftWorker(EagleDraftWorkerBase) L110` + `MultiLayerEagleWorkerV2(BaseSpecWorker) L918`](d:\design\sglang\python\sglang\srt\speculative\multi_layer_eagle_worker_v2.py)
- Standalone：~~`StandaloneWorker(EAGLEWorker)` standalone_worker.py L24~~ **文件已删**；现仅 [`StandaloneDraftWorker(EagleDraftWorker) L35` / `StandaloneWorkerV2(EAGLEWorkerV2) L147`](d:\design\sglang\python\sglang\srt\speculative\standalone_worker_v2.py)
- Frozen-KV MTP（pin 前新增算法族）：[`FrozenKVMTPDraftWorker(EagleDraftWorkerBase, TpModelWorker) L91` + `FrozenKVMTPWorkerV2(EAGLEWorkerV2) L676`](d:\design\sglang\python\sglang\srt\speculative\frozen_kv_mtp_worker_v2.py)
- DFlash：~~duck-typed `class DFlashWorker:` dflash_worker.py L50~~ **文件已删**；现 [`DFlashWorkerV2(BaseSpecWorker) L168` + `forward_batch_generation L1439`](d:\design\sglang\python\sglang\srt\speculative\dflash_worker_v2.py) + [`dflash_utils.py`](d:\design\sglang\python\sglang\srt\speculative\dflash_utils.py)
- DSpark（pin 前新增算法族，12 文件子包）：[`DSparkWorkerV2(BaseSpecWorker) L76`](d:\design\sglang\python\sglang\srt\speculative\dspark_components\dspark_worker_v2.py) + [`dspark_components/`](d:\design\sglang\python\sglang\srt\speculative\dspark_components)（draft / verify / planner / sampler / sps / sts / kv_inject / observability 等）
- NGRAM：~~duck-typed `class NGRAMWorker:` L25~~ 现 [`NGRAMWorker(BaseSpecWorker) L71` + `forward_batch_generation L425`](d:\design\sglang\python\sglang\srt\speculative\ngram_worker.py)
- Scheduler 切换：[`init_tp_model_worker L905` + `maybe_init_draft_worker L923` + `init_model_worker L993` + `tp_worker.get_worker_info() L1043`](d:\design\sglang\python\sglang\srt\managers\scheduler.py)
- TpModelWorker 多 runner：[`model_runner_list L334` + `_init_multi_layer_eagle_model_runners L481`](d:\design\sglang\python\sglang\srt\managers\tp_worker.py)
- Verify（greedy）：[`verify_tree_greedy_func` L374-441](d:\design\sglang\python\sglang\srt\speculative\eagle_utils.py) → CUDA 走外部包 `from sgl_kernel import verify_tree_greedy`（L386）/ CPU L57 / NPU `sgl_kernel_npu` L415 / **Triton fallback [`verify_tree_greedy_triton` L341-373](d:\design\sglang\python\sglang\srt\speculative\eagle_utils.py)**；~~sgl-kernel/python + csrc 仓内锚点~~ **sgl-kernel 树已整体移出本仓库（pin 前）**
- Verify（sample）：[`eagle_sample` L649 内调 `tree_speculative_sampling_target_only` L759, L822](d:\design\sglang\python\sglang\srt\speculative\eagle_utils.py)（verify 逻辑已从 `EagleVerifyInput.verify` 移到 `eagle_utils` 模块函数；`TREE_SPEC_KERNEL_AVAILABLE` 在 [spec_utils.py:L185](d:\design\sglang\python\sglang\srt\speculative\spec_utils.py)）
- CLI：[ServerArgs `speculative_*` 注解式字段 L2074-2243](d:\design\sglang\python\sglang\srt\server_args.py) + 规范化 hook [`arg_groups/speculative_hook.py`](d:\design\sglang\python\sglang\srt\arg_groups\speculative_hook.py)（`_handle_dflash` / `_handle_dspark` / `_handle_eagle_family` / `_handle_frozen_kv_mtp` / `_handle_ngram`，经 [`SpeculativeAlgorithm.handle_server_args` spec_info.py:L198+](d:\design\sglang\python\sglang\srt\speculative\spec_info.py) 分发）
- 模块全景（48 .py）：[`speculative/`](d:\design\sglang\python\sglang\srt\speculative)（详见 [modules/speculative.md](../modules/speculative.md)）

## Architecture / Data flow

> [!warning] CONTRADICTION（对 HEAD 而言整节 stale，2026-08-18 verify 标记）：下图描绘的 **V1/V2 双轨 + duck-typed 分支** 是 pin 前旧结构。HEAD 现实：`create_worker`（[spec_info.py:L254-307](d:\design\sglang\python\sglang\srt\speculative\spec_info.py)）按算法**直接**返回 7 个 V2 worker 之一（`DFlashWorkerV2` / `DSparkWorkerV2` / `FrozenKVMTPWorkerV2` / `MultiLayerEagleWorkerV2` / `EAGLEWorkerV2` / `StandaloneWorkerV2` / `NGRAMWorker`），不再按 overlap 分路；所有 worker 统一继承 `BaseSpecWorker`。本节保留供历史对照，重画留待 re-ingest（见 §Increment）。

```mermaid
flowchart TB
    SA["CLI: --speculative-algorithm + --disable-overlap-schedule<br/>+ --enable-multi-layer-eagle"]
    SA --> CW["SpeculativeAlgorithm.create_worker(server_args)<br/>(spec_info.py L59-122)<br/>路由: 算法 × overlap × multi_layer"]

    Sch["Scheduler.init_model_worker (scheduler.py L681-705)"]
    Sch -->|step1| TW["TpModelWorker (target)"]
    Sch -->|step2 maybe_init_draft_worker| CW
    CW --> DW{接入方式}

    DW -->|EAGLE / Standalone<br/>(V1 单层)| W1["subclass:<br/>EAGLEWorker(TpModelWorker)<br/>独立 KV pool"]
    DW -->|EAGLE multi-layer<br/>(V1)| W2["subclass:<br/>MultiLayerEagleWorker(TpModelWorker)<br/>+ TpModelWorker.model_runner_list<br/>(N=speculative_num_steps)"]
    DW -->|EAGLE/Standalone<br/>(V2 / overlap)| W3["composition:<br/>EAGLEWorkerV2(BaseSpecWorker)<br/>持有 EagleDraftWorker<br/>内嵌 TpModelWorker(is_draft_worker=True)"]
    DW -->|DFLASH| W4["duck-typed:<br/>class DFlashWorker:<br/>self.model_runner = target.model_runner"]
    DW -->|NGRAM| W5["duck-typed:<br/>class NGRAMWorker:<br/>纯 CPU trie + GPU mask"]

    Sch -->|step3 model_worker = draft_worker<br/>(scheduler.py L686-689)| MW["self.model_worker"]
    Sch -->|step4 get_worker_info() 从 TARGET 读| TW

    W1 & W2 & W3 & W4 --> VR["verify"]
    W5 -. 不走 verify_tree_greedy .-> SK2
    VR --> SK1["sgl_kernel.verify_tree_greedy (greedy)"]
    VR --> SK2["sgl_kernel.tree_speculative_sampling_target_only (sample)"]
    SK1 -.NPU.-> NPU["sgl_kernel_npu.verify_tree_greedy"]
```

**关键观察**：
- `init_model_worker` 第 4 步 [`tp_worker.get_worker_info()` L1043](d:\design\sglang\python\sglang\srt\managers\scheduler.py) 从 **target** 读资源信息（**不是 draft**）——该论断在 HEAD 复核**仍成立**。
- ~~spec v2 用"组合"；spec v1 用"继承"（`EAGLEWorker(TpModelWorker)`）——两种 draft 架构并存。~~ **已失效（pin 前）**：V1 继承路线已删，现统一为「`*WorkerV2(BaseSpecWorker)` 组合 + `*DraftWorker(EagleDraftWorkerBase)` 承载 draft」两层结构（[base_spec_worker.py:L57, L147](d:\design\sglang\python\sglang\srt\speculative\base_spec_worker.py)）。

## 5 算法族对照矩阵

> [!warning] CONTRADICTION（对 HEAD 而言 stale，2026-08-18 verify 标记）：算法族已从 ~~5~~ 扩为 **7**（EAGLE/EAGLE3、MultiLayer EAGLE、Standalone、**Frozen-KV MTP**、DFlash、**DSpark**、NGRAM），且下表引用的 `eagle_worker.py` / `multi_layer_eagle_worker.py` / `standalone_worker.py` / `dflash_worker.py` 等 V1 文件均已删除、"subclass vs duck-typed"接入维度已消失（统一 `BaseSpecWorker`）。HEAD 的 worker 类清单见上方 Sources 与 §Increment；矩阵重建留待 re-ingest。

| 算法族 | draft 模型 | verify | Worker 接入方式 | KV pool | forward dispatch 入参 |
|---|---|---|---|---|---|
| **EAGLE / EAGLE3 (V1)** | EAGLE 小 transformer（可选 hot vocab） | tree greedy / sampling (CUDA) | **subclass**：[`EAGLEWorker(TpModelWorker) L79`](d:\design\sglang\python\sglang\srt\speculative\eagle_worker.py) | 独立 KV pool；共享 `req_to_token_pool` + `allocator`（[L115-117](d:\design\sglang\python\sglang\srt\speculative\eagle_worker.py)） | 重写 [`forward_batch_generation(batch: ScheduleBatch)` L279](d:\design\sglang\python\sglang\srt\speculative\eagle_worker.py)（**签名变 ScheduleBatch**） |
| **EAGLE / EAGLE3 (V2 / overlap)** | 同上 | 同上 + plan_stream 异步 | **组合**：[`EAGLEWorkerV2(BaseSpecWorker) L624`](d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py) 持有 `EagleDraftWorker`，内嵌 `TpModelWorker(is_draft_worker=True)` ([L138-152](d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py)) | 同 V1 | [`forward_batch_generation(mwb: ModelWorkerBatch)` L690](d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py) |
| **MultiLayer EAGLE (V1/V2)** | MTP，每步独立 `ModelRunner` | 同 EAGLE | V1 [`(TpModelWorker) L70`](d:\design\sglang\python\sglang\srt\speculative\multi_layer_eagle_worker.py) + `TpModelWorker.model_runner_list[]` 由 [`_init_multi_layer_eagle_model_runners` L363-388](d:\design\sglang\python\sglang\srt\managers\tp_worker.py) 追加 N-1 个 `ModelRunner`；V2 [`(BaseSpecWorker) L568`](d:\design\sglang\python\sglang\srt\speculative\multi_layer_eagle_worker_v2.py) | 多 `ModelRunner` 共享 `memory_pool_config` | `mtp_model_runner(layer_id) → model_runner_list[layer_id]` ([L236-237](d:\design\sglang\python\sglang\srt\speculative\multi_layer_eagle_worker.py)) |
| **Standalone (V1/V2)** | 任意独立 draft 模型（不共享 embed/lm_head） | 同 EAGLE | V1 [`StandaloneWorker(EAGLEWorker) L24`](d:\design\sglang\python\sglang\srt\speculative\standalone_worker.py)（间接 TpModelWorker 子类）；V2 [`StandaloneWorkerV2(EAGLEWorkerV2) L132` + `StandaloneDraftWorker(EagleDraftWorker) L35`](d:\design\sglang\python\sglang\srt\speculative\standalone_worker_v2.py) | 同 EAGLE | 复用 EAGLE 路径 |
| **DFLASH** | 独立 draft + 可选 sliding window draft KV | DFlash 专用（mask policy + Triton fused KV materialize） | **duck-typed**：[`class DFlashWorker:` L50](d:\design\sglang\python\sglang\srt\speculative\dflash_worker.py)（无基类），`self.model_runner = target.model_runner` ([L73-74](d:\design\sglang\python\sglang\srt\speculative\dflash_worker.py)) | 共享 target KV pool；`--speculative-dflash-draft-window-size` 时另开私有 compact `req_to_token` ([L92-100](d:\design\sglang\python\sglang\srt\speculative\dflash_worker.py)) | [`forward_batch_generation(batch, **kwargs)` L1101-1113](d:\design\sglang\python\sglang\srt\speculative\dflash_worker.py)：`ModelWorkerBatch` fallback 给 target |
| **NGRAM** | **无 draft 模型**（CPU n-gram trie / SAM） | `reconstruct_indices_from_tree_mask` (CUDA)，**不调** `verify_tree_greedy` | **duck-typed**：[`class NGRAMWorker:` L25](d:\design\sglang\python\sglang\srt\speculative\ngram_worker.py)（无基类），`self.model_runner = target.model_runner` ([L39](d:\design\sglang\python\sglang\srt\speculative\ngram_worker.py)) | 完全共享 target KV pool（无 draft 前向） | [`forward_batch_generation(batch: ScheduleBatch)` L252](d:\design\sglang\python\sglang\srt\speculative\ngram_worker.py)；底层 [`NgramCorpus`](d:\design\sglang\python\sglang\srt\speculative\cpp_ngram\ngram_corpus.py) C++ |

**关键文件分组**（详 [modules/speculative.md File inventory](../modules/speculative.md)）：EAGLE 系 = `eagle_*.py` 共 7 个；MultiLayer = `multi_layer_eagle_*.py` 共 4 个；Standalone = `standalone_worker{,_v2}.py`；DFlash = `dflash_*.py` 共 3 个 + `triton_ops/fused_kv_materialize.py`；NGRAM = `ngram_{worker,info}.py` + `external_corpus_manager.py` + `cpp_ngram/`（2 .py）。

> ~~synthesis: `create_worker(server_args)` 路由维度 = **算法 × `enable_overlap` × `enable_multi_layer_eagle`**（spec_info.py L59-122）。同一 EAGLE 算法在 `enable_overlap=True` 时返回 V2、否则返回 V1。DFlash / NGRAM 在 V2 路径直接 raise。~~ **已失效（pin 前重构，2026-08-18 标注）**：`create_worker`（[spec_info.py:L254-307](d:\design\sglang\python\sglang\srt\speculative\spec_info.py)）现在**只按算法分发**（overlap 不再是路由维度——"V2 worker drives both overlap and non-overlap"，源码注释 L261-262）；`enable_multi_layer_eagle` 仍是 EAGLE 内部的二选一维度（L286-292）；DFlash / NGRAM 不再 raise，DFlash 走 `DFlashWorkerV2`、NGRAM 走 `NGRAMWorker`（均 `BaseSpecWorker` 子类）。

## TpModelWorker subclass vs duck-typed worker

> [!warning] CONTRADICTION（对 HEAD 而言整节 stale，2026-08-18 verify 标记）：本节描述的二分法已在 pin 前重构中消失——`NGRAMWorker` / `DFlashWorkerV2` 现在都是 [`BaseSpecWorker`](d:\design\sglang\python\sglang\srt\speculative\base_spec_worker.py) 子类（[ngram_worker.py:L71](d:\design\sglang\python\sglang\srt\speculative\ngram_worker.py) / [dflash_worker_v2.py:L168](d:\design\sglang\python\sglang\srt\speculative\dflash_worker_v2.py)），不再是无基类 duck-typed 包装；draft 侧真模型统一由 `EagleDraftWorkerBase` 派生类承载。节内「资源信息从 target 读」（现 [scheduler.py:L1043](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）与「NGRAM 无 draft 前向」两条**仍成立**。本节保留供历史对照。

> synthesis: 这是 SGLang spec 设计的**核心架构选择**，理解它就理解了为什么 ~~27~~（HEAD：48）个 .py 分成两批截然不同的模式。

**5 维差异**：

| 维度 | EAGLE 系（subclass / 组合） | DFlash / NGRAM（duck-typed） |
|---|---|---|
| 类继承 | 继承 `TpModelWorker` 或 `BaseSpecWorker`+内嵌 | **无任何基类**：`class DFlashWorker:` / `class NGRAMWorker:` |
| ModelRunner | `super().__init__(is_draft_worker=True, ...)` **新建** draft（[L142-156](d:\design\sglang\python\sglang\srt\speculative\eagle_worker.py)） | **复用** `target_worker.model_runner` |
| KV pool | draft 独立 KVCache；req_to_token + allocator 共享 | DFlash 共享 target；NGRAM 无 draft 前向 |
| Scheduler 协议 | 完整 `TpModelWorker` 接口 | 仅 forward 子集（`forward_batch_generation` / `model_runner` / `clear_cache_pool`） |
| draft 模型 | 必须有 | DFlash 有；**NGRAM 完全无前向**（纯 CPU trie + GPU mask） |

**为什么两条路径并存（4 条 synthesis）**：

1. **资源信息从 target 读**：[`Scheduler.init_model_worker` L691-705](d:\design\sglang\python\sglang\srt\managers\scheduler.py) 调 `self.tp_worker.get_worker_info()`（**target**，非 draft）拿 `max_total_num_tokens` / `forward_stream` 等。NGRAM 无 model、DFlash 共享 target pool ⇒ 没必要做完整 TpModelWorker。
2. **forward dispatch 是协议而非继承**：scheduler 在 [L686-689](d:\design\sglang\python\sglang\srt\managers\scheduler.py) `self.model_worker = self.draft_worker` 后，调用栈只用 `self.model_worker.forward_batch_generation(...)` / `.model_runner.<...>` —— **duck typing**。
3. **EAGLE 做 subclass 是因为有真 draft 模型 + 真 KV pool**：要走 `update_weights_*` / `memory_pool_config` / CUDA graph capture / `init_attention_backend` 等全套基础设施，直接继承最省。
4. **MultiLayer-EAGLE 走第三条路**：仍 subclass，但 `is_multi_layer_eagle=True` 触发 [`_init_multi_layer_eagle_model_runners` L363-388](d:\design\sglang\python\sglang\srt\managers\tp_worker.py) 把 "1 worker = 1 ModelRunner" 暗约束打破为 "1 worker = N ModelRunner"。

**V1/V2 双轨**：正交于 subclass/duck-typed，EAGLE 系沿 `enable_overlap` 拆 V1（继承）/ V2（组合），后者匹配 overlap scheduler（独立 plan_stream、`ModelWorkerBatch` 入参、异步 verify）。**5 算法族 × V1/V2 × subclass/duck-typed** 三维落到 27 .py：每个 `_v2.py` 对应一个 V1 兄弟；DFlash / NGRAM 因强制 disable overlap 只有 V1。

## sgl-kernel CUDA 绑定

> [!warning] CONTRADICTION（锚点失效，2026-08-18 verify 标记）：**sgl-kernel 目录已在 pin 前整体移出本仓库**（HEAD 顶层无 `sgl-kernel/`；`git ls-tree 06f32bab` 亦无）——下表所有 `d:\design\sglang\sgl-kernel\...` 锚点已死，`sgl_kernel` 现为**外部 pip 包**。2 个算子名与语义不变，但 Python 侧调用点已重构：greedy 走 [`verify_tree_greedy_func` eagle_utils.py:L374-441](d:\design\sglang\python\sglang\srt\speculative\eagle_utils.py)（CUDA 外部包 / CPU L57 / NPU L415 / **新增 Triton fallback L341**），sampling 走 [`eagle_sample` eagle_utils.py:L649](d:\design\sglang\python\sglang\srt\speculative\eagle_utils.py)（L759, L822 调 `tree_speculative_sampling_target_only`；DFlash 家族在 [dflash_utils.py:L878](d:\design\sglang\python\sglang\srt\speculative\dflash_utils.py) 另有调用）——**`EagleVerifyInput.verify` 方法已不存在**，eagle_info.py 现仅存 dataclass（[L16/L142/L272](d:\design\sglang\python\sglang\srt\speculative\eagle_info.py)）。

SGLang spec 验证调用 **2 个独立 sgl-kernel CUDA 算子**（下表为 pin 前旧锚点，保留供历史对照）：

| 算子 | 入参 schema | Python 包装 → torch.ops → C++ 注册 → CUDA 实现 | 主调用点 |
|---|---|---|---|
| **`verify_tree_greedy`**（贪婪树验证） | `(Tensor! predicts, Tensor! accept_index, Tensor! accept_token_num, Tensor candidates, Tensor retrive_index, Tensor retrive_next_token, Tensor retrive_next_sibling, Tensor target_predict) -> ()` | [`verify_tree_greedy_func` L161-199](d:\design\sglang\python\sglang\srt\speculative\eagle_utils.py) → [`sgl_kernel.verify_tree_greedy` L38-57](d:\design\sglang\sgl-kernel\python\sgl_kernel\speculative.py) → [`m.def + m.impl(..., torch::kCUDA)` L255-259](d:\design\sglang\sgl-kernel\csrc\common_extension.cc)（ROCm 镜像 [L136-139](d:\design\sglang\sgl-kernel\csrc\common_extension_rocm.cc)）→ [CUDA `verify_tree_greedy` L323](d:\design\sglang\sgl-kernel\csrc\speculative\eagle_utils.cu) | [`EagleVerifyInput.verify` L317-330](d:\design\sglang\python\sglang\srt\speculative\eagle_info.py) `is_all_greedy or not TREE_SPEC_KERNEL_AVAILABLE` 时 |
| **`tree_speculative_sampling_target_only`**（带温度树采样） | 上面 8 个张量 + `target_probs` / `draft_probs` / `uniform_samples` / `threshold_single` / `threshold_acc` / `deterministic` | [`sgl_kernel.tree_speculative_sampling_target_only` L4-35](d:\design\sglang\sgl-kernel\python\sgl_kernel\speculative.py) ← 由 [eagle_info.py L49-54 import](d:\design\sglang\python\sglang\srt\speculative\eagle_info.py)（is_cuda gated）→ [`m.def + m.impl(..., torch::kCUDA)` L247-253](d:\design\sglang\sgl-kernel\csrc\common_extension.cc) → [CUDA L31](d:\design\sglang\sgl-kernel\csrc\speculative\speculative_sampling.cu) | [`EagleVerifyInput.verify` L332+](d:\design\sglang\python\sglang\srt\speculative\eagle_info.py) non-greedy 分支 |

**路由**（[eagle_info.py L317](d:\design\sglang\python\sglang\srt\speculative\eagle_info.py)）：batch 全 greedy → `verify_tree_greedy`；含非 greedy → `tree_speculative_sampling_target_only`；AMD/HIP 不可用时 fallback 到 greedy（warning [L312-315](d:\design\sglang\python\sglang\srt\speculative\eagle_info.py)）；NPU 路径走 [`sgl_kernel_npu.verify_tree_greedy` L186-198](d:\design\sglang\python\sglang\srt\speculative\eagle_utils.py)。

> synthesis: 这是 SGLang 第二组已确认的 sgl-kernel C++ 算子绑定（第一组是 [`shm_allreduce`](../modules/distributed.md)）—— **verify 路径是 SGLang 唯一通过 C++ kernel 加速的 spec 子环节**，draft 前向仍走 PyTorch + Triton + CUDA Graph。

## EAGLE multi-layer 特殊性

`TpModelWorker` 默认 1 worker = 1 `ModelRunner`，`is_multi_layer_eagle=True` 时打破暗约束：[`model_runner_list: List[ModelRunner] = []` L257](d:\design\sglang\python\sglang\srt\managers\tp_worker.py) 由 [`_init_multi_layer_eagle_model_runners` L363-388](d:\design\sglang\python\sglang\srt\managers\tp_worker.py) 填充：先 `append(self.model_runner)` (idx=0)，再 `for i in range(1, speculative_num_steps): append(ModelRunner(..., draft_model_idx=i))`；按层访问 [`mtp_model_runner(layer_id) → model_runner_list[layer_id]` L236-237](d:\design\sglang\python\sglang\srt\speculative\multi_layer_eagle_worker.py)；draft TP context 用 `mtp_model_runner(0).tp_group`（[L259-262](d:\design\sglang\python\sglang\srt\speculative\multi_layer_eagle_worker.py)）。**KV 共享**：N 个 `ModelRunner` 同传 `req_to_token_pool` / `token_to_kv_pool_allocator` / `memory_pool_config`（[L383-385](d:\design\sglang\python\sglang\srt\managers\tp_worker.py)）—— 多 MTP 层模型的 KV 物理上是一份。

> synthesis: 这是 [comparison/topics/executor-worker.md](../../comparison/topics/executor-worker.md) "draft model runner 持有位置差异" 中 SGLang 的精确实现：**target 与 draft 都是 `ModelRunner`，由 `model_runner_list[]` 共享在同一 `TpModelWorker` 进程内**；vLLM 走 `EagleProposer/Speculator` 内含 cudagraph 管理；MindIE 走 `MtpWorker.draft_model_runner` 装饰。三家都用 `ModelRunner` 抽象，但持有方式截然不同。

## CLI 全表

> [!warning] CONTRADICTION（锚点整体失效，2026-08-18 verify 标记）：server_args 已重构——spec 字段现为注解式 `A[...]` 字段（[server_args.py:L2074-2243](d:\design\sglang\python\sglang\srt\server_args.py)，argparse 自动生成，下表 L5099-5271 手写 argparse 锚点全部失效）；~~`_handle_speculative_decoding` L3107-3160~~ 已拆为 [`arg_groups/speculative_hook.py`](d:\design\sglang\python\sglang\srt\arg_groups\speculative_hook.py) 的 `_handle_dflash` / `_handle_dspark` / `_handle_eagle_family` / `_handle_frozen_kv_mtp` / `_handle_ngram`，经 [`SpeculativeAlgorithm.handle_server_args`](d:\design\sglang\python\sglang\srt\speculative\spec_info.py) 按算法分发。参数语义大体延续，逐项行号重建留待 re-ingest。

完整 spec 相关参数（下表锚点为 pin 前旧值，保留供历史对照）。

### 算法选择 / Draft 加载

| 参数 | 锚点 | 说明 |
|---|---|---|
| `--speculative-algorithm` | [L5101-5104](d:\design\sglang\python\sglang\srt\server_args.py) | choices `{DFLASH, EAGLE, EAGLE3, NEXTN, STANDALONE, NGRAM}`；`NEXTN` 在 [L3131-3132](d:\design\sglang\python\sglang\srt\server_args.py) 被规范化为 `EAGLE`（enum 无 `NEXTN` 成员） |
| `--enable-multi-layer-eagle` | [L5267-5270](d:\design\sglang\python\sglang\srt\server_args.py) | EAGLE 系切换到 MultiLayer 路径 |
| `--disable-overlap-schedule` | (顶层) | 决定 V1 / V2 路由 |
| `--speculative-draft-model-path` / `--speculative-draft-model` | [L5106-5111](d:\design\sglang\python\sglang\srt\server_args.py) | EAGLE3/Standalone/DFlash 必填 |
| `--speculative-draft-model-revision` / `--speculative-draft-load-format` / `--speculative-draft-model-quantization` | [L5112-5128](d:\design\sglang\python\sglang\srt\server_args.py) / [L5207-5213](d:\design\sglang\python\sglang\srt\server_args.py) | draft 加载控制 |
| `--speculative-token-map` | [L5174-5179](d:\design\sglang\python\sglang\srt\server_args.py) | draft hot vocab 表（EAGLE / Standalone） |

### EAGLE 参数

| 参数 | 锚点 | 说明 |
|---|---|---|
| `--speculative-num-steps` | [L5129-5134](d:\design\sglang\python\sglang\srt\server_args.py) | draft 步数；MultiLayer 下 = `model_runner_list` 长度 |
| `--speculative-eagle-topk` | [L5135-5140](d:\design\sglang\python\sglang\srt\server_args.py) | 每步 draft topk；spec v2 强制 = 1 |
| `--speculative-num-draft-tokens` | [L5141-5146](d:\design\sglang\python\sglang\srt\server_args.py) | 树总叶节点数 |
| `--speculative-accept-threshold-single` / `-acc` | [L5162-5173](d:\design\sglang\python\sglang\srt\server_args.py) | `tree_speculative_sampling_target_only` 入参 |
| `--speculative-attention-mode` `{prefill, decode}` | [L5180-5186](d:\design\sglang\python\sglang\srt\server_args.py) | verify+draft 共用 backend mode |
| `--speculative-draft-attention-backend` | [L5187-5192](d:\design\sglang\python\sglang\srt\server_args.py) | draft 独立 attention backend |
| `--speculative-moe-runner-backend` / `-a2a-backend` | [L5193-5206](d:\design\sglang\python\sglang\srt\server_args.py) | draft 端 MoE 后端 |

### DFlash / NGRAM 专用

| 参数 | 锚点 | 说明 |
|---|---|---|
| `--speculative-dflash-block-size` | [L5147-5152](d:\design\sglang\python\sglang\srt\server_args.py) | verify 窗长（DFlash 替代 num_draft_tokens） |
| `--speculative-dflash-draft-window-size` | [L5153-5161](d:\design\sglang\python\sglang\srt\server_args.py) | draft sliding window；启用即开 compact `req_to_token` |
| `--speculative-ngram-{min,max}-bfs-breadth` (默认 1 / 10) | [L5216-5227](d:\design\sglang\python\sglang\srt\server_args.py) | NGRAM trie BFS 宽度 |
| `--speculative-ngram-match-type` `{BFS, PROB}` (默认 BFS) | [L5228-5234](d:\design\sglang\python\sglang\srt\server_args.py) | trie 匹配策略 |
| `--speculative-ngram-max-trie-depth` (默认 18) | [L5235-5240](d:\design\sglang\python\sglang\srt\server_args.py) | trie 深度上限 |
| `--speculative-ngram-capacity` (默认 10M) | [L5241-5246](d:\design\sglang\python\sglang\srt\server_args.py) | 内部缓存容量 |
| `--speculative-ngram-external-corpus-{path,max-tokens}` / `-external-sam-budget` | [L5247-5264](d:\design\sglang\python\sglang\srt\server_args.py) | 启动时预加载 JSONL 语料 + SAM 预算 |

DFlash 校验：[`_handle_speculative_decoding` L3134-3160](d:\design\sglang\python\sglang\srt\server_args.py) 强制 `dp_attention=False` / `pp_size=1` / `speculative_num_steps = 1` / `speculative_eagle_topk = 1`、必须设 draft model。DFlash 与 NGRAM 在 [`create_worker` L70-72 / L113-116](d:\design\sglang\python\sglang\srt\speculative\spec_info.py) 强制 disable overlap（V2 路径会 raise）。

## Increment 2026-08-18 (06f32bab → f7101b0a)

本期 `speculative/` 30 文件 / ~1082 行 churn（+1547/-620 与 disaggregation 合计）。**先声明分界**：本页正文大面积失效源自 **pin 前**（34fef07a→06f32bab 区间）的 V1→V2 大重构——枚举 6→**8** 成员（+`DSPARK` +`FROZEN_KV_MTP`）、V1 worker 全删、duck-typed 消失、`dspark_components/`（12 文件）与 `frozen_kv_mtp_*` / `adaptive_*` / `decoupled_spec_io.py` / `ragged_verify.py` / `spec_registry.py` 新增、`triton_ops/` 目录删除、sgl-kernel 移出仓库；`06f32bab..HEAD` 本期区间 `git log --diff-filter=AD` **仅新增 1 文件**（`dflash_disaggregation.py`），无删除。本期增量本身：

- **算法族清单核对**（`ls /tmp/sglang/python/sglang/srt/speculative/`）：目录 **48 .py**（34 根 + 12 [`dspark_components/`](d:\design\sglang\python\sglang\srt\speculative\dspark_components) + 2 [`cpp_ngram/`](d:\design\sglang\python\sglang\srt\speculative\cpp_ngram)）；pin 时 47，本期 +1。**本页此前完全没提 DSpark 与 Frozen-KV MTP**——两者均为 pin 前新增算法族（DSpark 初始 commit #31847 "support inkling dspark"），已补入 Sources；worker 工厂 7 分支见 [`create_worker` spec_info.py:L254-307](d:\design\sglang\python\sglang\srt\speculative\spec_info.py)。
- **ngram accept tokens 经 FutureMap relay**（#35198）：overlap 模式下 ngram 的 accept tokens 改经 [`managers/overlap_utils.py` `FutureMap`](d:\design\sglang\python\sglang\srt\managers\overlap_utils.py) relay（+53 行），[`ngram_info.py`](d:\design\sglang\python\sglang\srt\speculative\ngram_info.py) +4；`ngram_worker.py` 本期 +58/-40（另含 #35207 降低 ngram draft prep 的 host 开销）。
- **DSpark 支持 logprobs**（#34696）+ `compute_spec_v2_logprobs` 签名简化（#35058）：`dspark_components/dspark_worker_v2.py` +66/-35；[`draft_utils.py`](d:\design\sglang\python\sglang\srt\speculative\draft_utils.py) +167/-72（现以 [`DraftBackendFactory` L27](d:\design\sglang\python\sglang\srt\speculative\draft_utils.py) 为主体）。
- **WAR → shared-read 重命名链**（#34916 → #34982 → #35059 → #35057）：`model_executor/runner_utils/war_event.py` → [`shared_read_event.py`](d:\design\sglang\python\sglang\srt\model_executor\runner_utils\shared_read_event.py)（"WAR read-done fastpath" 更名 "shared-read-done"）；#35059 起 shared-read ends **仅由 attention backend 声明解析**；#35057 把 multi-layer eagle 的最后一个 shared-read runner 指向 draft runner；spec_info.py 中旧 `is_war_publish_phase` 方法已删（本期 diff 可见）。
- **spec buffer 尺寸改从 config-bags 读取**（#35024）：dummy verify `SpecInput` 构造改用 `runtime_context.get_spec()` 的 `speculative_num_steps` / `eagle_topk` / `num_draft_tokens`（[spec_info.py 本期 diff](d:\design\sglang\python\sglang\srt\speculative\spec_info.py)），不再读 startup `server_args` 记录——adaptive spec 每个候选 step config 可独立 capture。
- **新文件 [`dflash_disaggregation.py`](d:\design\sglang\python\sglang\srt\speculative\dflash_disaggregation.py)**（+32，#33676 引入）：`build_dflash_family_disagg_draft_input`——DFlash 家族在 PD-decode prebuilt batch 上构造 `DFlashDraftInputV2` 并经 FutureMap stash bonus tokens，与既有 [`eagle_disaggregation.py`](d:\design\sglang\python\sglang\srt\speculative\eagle_disaggregation.py) / [`dspark_disaggregation.py`](d:\design\sglang\python\sglang\srt\speculative\dspark_disaggregation.py) 同族；**但 [`build_disagg_draft_input` spec_info.py:L173-193](d:\design\sglang\python\sglang\srt\speculative\spec_info.py) 目前只分发 EAGLE / DSPARK**（见下方 VERIFY）。
- **dflash_utils.py +217**：新增 [`DFlashDraftConfig` L485 / `parse_dflash_draft_config` L536](d:\design\sglang\python\sglang\srt\speculative\dflash_utils.py)、fused QKV 判定 [`can_dflash_use_fused_qkv_proj` L638](d:\design\sglang\python\sglang\srt\speculative\dflash_utils.py)、NPU top-k/p renorm [L73-138](d:\design\sglang\python\sglang\srt\speculative\dflash_utils.py) 等；`dflash_worker_v2.py` +74/-14。
- **平台面**：#7216/#9401（ROCm 上 EAGLE greedy-verify 决策做 TP 广播并 scope 到 ROCm）、#29202（AMD MTP draft-extend CUDA graph）、#33676（NPU DeepSeek-V4 DSpark）。
- synthesis: 本期 spec 主线是 **overlap 语义收口**（FutureMap relay / shared-read 重命名与统一解析）+ **配置读取迁移到 bags** + **DFlash/DSpark 功能补齐**；算法族集合与 worker 类拓扑本期无变化（上述结构性变化全部发生在 pin 前区间）。

> [!todo] VERIFY: [`dflash_disaggregation.py`](d:\design\sglang\python\sglang\srt\speculative\dflash_disaggregation.py) 的 `build_dflash_family_disagg_draft_input` 在 `d:\design\sglang\python\` 全树 grep **0 树内调用方**（`build_disagg_draft_input` 仅分发 EAGLE/DSPARK，DFlash 家族 `return None`）——疑为 #33676 预留接线；后续 increment 需复查是否接入。

> [!todo] VERIFY: 本页正文（Architecture 图 / 5 算法族矩阵 / subclass-vs-duck-typed 节 / CLI 表）已按 CONTRADICTION 块标注 stale，**需要一次完整 re-ingest** 才能按 V2 单轨 + 8 枚举 + 插件注册的新结构重写；本次 increment pass 仅完成锚点校正与失效标注，未重建正文结构（遵循"不重写页面结构"约束）。

## Notes / Caveats

> ~~[!todo] VERIFY: pin 从 `34fef07a` → `06f32bab`（2026-08-10 increment）后本页未深 verify；文件数量/行号可能漂移。~~ **RESOLVED 2026-08-18**：已深 verify——确认 pin 前发生 V1→V2 大重构（枚举 8 成员 / V1 文件全删 / sgl-kernel 移出仓库 / 新增 DSpark、Frozen-KV MTP 两算法族），失效论断已在各节以 CONTRADICTION 块划线标注，HEAD 锚点见 Sources 与 §Increment。

> [!warning] CONTRADICTION（cross-page lint follow-up pending）
> [comparison/topics/speculative-decoding.md L80](d:\design\wiki\comparison\topics\speculative-decoding.md) 写"SGLang **24** 文件 / vLLM **12** 文件"；本轮 Glob 实测 **SGLang 27 .py**（23 根 + 2 `triton_ops/` + 2 `cpp_ngram/`）、**vLLM 11 .py**（[`d:\design\vllm\vllm\v1\spec_decode\`](d:\design\vllm\vllm\v1\spec_decode)）。L439 `vllm/topics/spec-decode-eagle.md` 链接描述同样写"12"。
> 本任务范围为 sglang 内部 topic，不修改 comparison 页。**已记录为下次 lint pass / 该 compare 页 verify pass 修复目标（`24 → 27` SGLang、`12 → 11` vLLM）**；详见 [modules/speculative.md §Notes](../modules/speculative.md) 已 RESOLVED 的同名 VERIFY 块。

> [!warning] CONTRADICTION（worker 抽象命名陷阱）
> [comparison L82](../../comparison/topics/speculative-decoding.md) 顶层 synthesis "SGLang draft 用独立 TpModelWorker(is_draft_worker=True)" **仅适用于 EAGLE / Standalone / MultiLayer 系**；DFLASH / NGRAM 是 duck-typed 包装，无 `is_draft_worker` 标志、不构造独立 `ModelRunner`。精化措辞详见本页 §"TpModelWorker subclass vs duck-typed worker"。

> [!todo] VERIFY: `MultiLayerEagleWorkerV2(BaseSpecWorker)` 在 V2 路径下 `model_runner_list` 由谁持有——本轮仅确认 V1 multi-layer 走 `TpModelWorker.model_runner_list`，V2 走 `BaseSpecWorker` 而非直接继承 `TpModelWorker`，需要查 V2 多 layer 内部组织。

> [!todo] VERIFY: `forward_batch_generation` **签名差异**：`EAGLEWorker.forward_batch_generation(batch: ScheduleBatch)` ([eagle_worker.py L279](d:\design\sglang\python\sglang\srt\speculative\eagle_worker.py)) vs 父类 `TpModelWorker.forward_batch_generation(model_worker_batch: ModelWorkerBatch, forward_batch=None, ...)` ([tp_worker.py L443](d:\design\sglang\python\sglang\srt\managers\tp_worker.py))——子类改第一参数类型；scheduler 调用栈如何按 worker 类型选用不同入参，留待 [`Scheduler.run_batch`](../entities/Scheduler.md) 深入。

## See also

- [sglang/modules/speculative.md](../modules/speculative.md) — 模块全景：48 .py 文件（HEAD 实测；旧文 27 已失效）、`SpeculativeAlgorithm` enum 8 成员、`SpecInput` dataclass 注入 ScheduleBatch / ForwardBatch、N-gram C++ 扩展、DSpark 子包
- [sglang/entities/TpModelWorker.md](../entities/TpModelWorker.md) — `model_runner_list` 字段语义 + EAGLE multi-layer 联动
- [sglang/entities/Scheduler.md](../entities/Scheduler.md) — `init_tp_model_worker` / `maybe_init_draft_worker` / `init_model_worker` 三步走 + `model_worker = draft_worker` 切换语义
- [sglang/modules/sampling.md](../modules/sampling.md) — sgl-kernel 通用采样 kernel 路径（`top_k_renorm_prob` / `top_p_renorm_prob` 与 spec 共用）
- [sglang/modules/layers.md](../modules/layers.md) — attention backend；spec draft / verify 通过 `--speculative-draft-attention-backend` 与 `--speculative-attention-mode` 选择独立 backend
- [sglang/modules/distributed.md](../modules/distributed.md) — sgl-kernel C++ 绑定模式参照（`shm_allreduce` 是另一组已确认绑定）
- [comparison/topics/speculative-decoding.md](../../comparison/topics/speculative-decoding.md) — 三家 spec 9 子维度对比（**本页是 SGLang 内部 deep-dive，对比页负责跨项目**）
- [comparison/topics/executor-worker.md](../../comparison/topics/executor-worker.md) — draft model runner 持有位置三家差异（`model_runner_list` 是 SGLang 答案）
- [comparison/dimensions.md §dim-spec](../../comparison/dimensions.md)
