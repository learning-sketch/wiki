---
type: topic
project: sglang
status: stale
confidence: high
verified_against: 2026-04-19
sources:
  - d:\design\sglang\python\sglang\srt\speculative\spec_info.py:15-122
  - d:\design\sglang\python\sglang\srt\speculative\base_spec_worker.py
  - d:\design\sglang\python\sglang\srt\speculative\eagle_worker.py:79-156
  - d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py:85-152
  - d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py:624-739
  - d:\design\sglang\python\sglang\srt\speculative\multi_layer_eagle_worker.py:70
  - d:\design\sglang\python\sglang\srt\speculative\multi_layer_eagle_worker.py:236-237
  - d:\design\sglang\python\sglang\srt\speculative\multi_layer_eagle_worker_v2.py:568
  - d:\design\sglang\python\sglang\srt\speculative\standalone_worker.py:24
  - d:\design\sglang\python\sglang\srt\speculative\standalone_worker_v2.py:35-37
  - d:\design\sglang\python\sglang\srt\speculative\standalone_worker_v2.py:132
  - d:\design\sglang\python\sglang\srt\speculative\dflash_worker.py:50-100
  - d:\design\sglang\python\sglang\srt\speculative\dflash_worker.py:1101-1113
  - d:\design\sglang\python\sglang\srt\speculative\ngram_worker.py:25-80
  - d:\design\sglang\python\sglang\srt\speculative\eagle_info.py:49-54
  - d:\design\sglang\python\sglang\srt\speculative\eagle_info.py:317-330
  - d:\design\sglang\python\sglang\srt\speculative\eagle_utils.py:161-199
  - d:\design\sglang\python\sglang\srt\managers\scheduler.py:615-689
  - d:\design\sglang\python\sglang\srt\managers\scheduler.py:691-705
  - d:\design\sglang\python\sglang\srt\managers\tp_worker.py:240-396
  - d:\design\sglang\python\sglang\srt\managers\tp_worker.py:443-533
  - d:\design\sglang\python\sglang\srt\server_args.py:497-524
  - d:\design\sglang\python\sglang\srt\server_args.py:3107-3160
  - d:\design\sglang\python\sglang\srt\server_args.py:5099-5271
  - d:\design\sglang\sgl-kernel\python\sgl_kernel\speculative.py:4-57
  - d:\design\sglang\sgl-kernel\csrc\common_extension.cc:247-259
  - d:\design\sglang\sgl-kernel\csrc\speculative\eagle_utils.cu:323-331
  - d:\design\sglang\sgl-kernel\csrc\speculative\speculative_sampling.cu:31
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

SGLang 投机解码用 **`SpeculativeAlgorithm` 6 成员枚举 + 5 个算法族实现**（EAGLE / MultiLayer EAGLE / Standalone / DFlash / NGRAM），通过 **2 种 worker 接入协议**进入调度器：**EAGLE / Standalone / MultiLayer-EAGLE 是 `TpModelWorker` 的真正子类（独立 KV pool + 独立 `ModelRunner`）**；**DFlash / NGRAM 是 duck-typed 包装类**（持有 `target_worker` 引用、复用 `target_worker.model_runner`，通过暴露 `forward_batch_generation` / `model_runner` 等属性满足 scheduler 的调用协议）。验证阶段用 sgl-kernel 的 **2 个 CUDA 算子**：`verify_tree_greedy`（贪婪树验证）与 `tree_speculative_sampling_target_only`（带温度的树投机采样）；`Scheduler.init_model_worker` 在启用 spec 时把 `model_worker` 切到 `draft_worker`，但**资源信息仍从 `tp_worker.get_worker_info()` 读取**。

> synthesis: SGLang 的 spec 实现 **算法 × 是否 overlap** 形成 V1/V2 双轨（同一算法两份 worker 类，通过 `create_worker(server_args)` 路由），与 vLLM 单一 `SpecDecodeBaseProposer` 多态、MindIE Plugin + C++ placeholder 形成三种范式。

## Sources

- 算法枚举 / 工厂：[`SpeculativeAlgorithm` enum L15-23 + `create_worker` L59-122](d:\design\sglang\python\sglang\srt\speculative\spec_info.py)
- 抽象：[`BaseSpecWorker` / `BaseDraftWorker`](d:\design\sglang\python\sglang\srt\speculative\base_spec_worker.py)
- EAGLE V1（subclass）：[`EAGLEWorker(TpModelWorker) L79` + `super().__init__(is_draft_worker=True) L142-156`](d:\design\sglang\python\sglang\srt\speculative\eagle_worker.py)
- EAGLE V2（组合 + overlap）：[`EAGLEWorkerV2(BaseSpecWorker) L624` + 内嵌 `TpModelWorker(is_draft_worker=True)` L138-152](d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py)
- MultiLayer EAGLE：[`MultiLayerEagleWorker(TpModelWorker) L70` + `mtp_model_runner L236-237`](d:\design\sglang\python\sglang\srt\speculative\multi_layer_eagle_worker.py)、[`MultiLayerEagleWorkerV2(BaseSpecWorker) L568`](d:\design\sglang\python\sglang\srt\speculative\multi_layer_eagle_worker_v2.py)
- Standalone：[`StandaloneWorker(EAGLEWorker) L24`](d:\design\sglang\python\sglang\srt\speculative\standalone_worker.py)、[`StandaloneDraftWorker(EagleDraftWorker) L35` / `StandaloneWorkerV2(EAGLEWorkerV2) L132`](d:\design\sglang\python\sglang\srt\speculative\standalone_worker_v2.py)
- DFlash duck-typed：[`class DFlashWorker: L50` + `self.model_runner = target.model_runner L73-74` + `forward_batch_generation L1101`](d:\design\sglang\python\sglang\srt\speculative\dflash_worker.py)
- NGRAM duck-typed：[`class NGRAMWorker: L25` + `self.model_runner = target.model_runner L38-39` + `forward_batch_generation L252`](d:\design\sglang\python\sglang\srt\speculative\ngram_worker.py)
- Scheduler 切换：[`init_tp_model_worker L615-637` + `maybe_init_draft_worker L639-679` + `init_model_worker L681-689` + `tp_worker.get_worker_info() L691-705`](d:\design\sglang\python\sglang\srt\managers\scheduler.py)
- TpModelWorker 多 runner：[`model_runner_list L257` + `_init_multi_layer_eagle_model_runners L363-388` + `forward_batch_generation L443-533`](d:\design\sglang\python\sglang\srt\managers\tp_worker.py)
- Verify CUDA（greedy）：[`verify_tree_greedy_func` L161-199](d:\design\sglang\python\sglang\srt\speculative\eagle_utils.py) → [`sgl_kernel.verify_tree_greedy` L38-57](d:\design\sglang\sgl-kernel\python\sgl_kernel\speculative.py) → [`m.impl(... torch::kCUDA)` L255-259](d:\design\sglang\sgl-kernel\csrc\common_extension.cc) → [CUDA L323](d:\design\sglang\sgl-kernel\csrc\speculative\eagle_utils.cu)
- Verify CUDA（sample）：[import L49-54](d:\design\sglang\python\sglang\srt\speculative\eagle_info.py) → [`sgl_kernel.tree_speculative_sampling_target_only` L4-35](d:\design\sglang\sgl-kernel\python\sgl_kernel\speculative.py) → [`m.impl(... torch::kCUDA)` L247-253](d:\design\sglang\sgl-kernel\csrc\common_extension.cc) → [CUDA L31](d:\design\sglang\sgl-kernel\csrc\speculative\speculative_sampling.cu)
- CLI：[ServerArgs 字段 L497-524 + argparse L5099-5271 + 规范化 L3107-3160](d:\design\sglang\python\sglang\srt\server_args.py)
- 模块全景（27 .py）：[`speculative/`](d:\design\sglang\python\sglang\srt\speculative)（详见 [modules/speculative.md](../modules/speculative.md)）

## Architecture / Data flow

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
- `init_model_worker` 第 4 步 [`tp_worker.get_worker_info()` L705](d:\design\sglang\python\sglang\srt\managers\scheduler.py) 从 **target** 读资源信息（**不是 draft**）—— DFlash/NGRAM 不需要完整 `TpModelWorker` 子类的根本原因。
- spec v2 用"组合"（`EagleDraftWorker` 内嵌 `TpModelWorker(is_draft_worker=True)`，[L138-152](d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py)）；spec v1 用"继承"（`EAGLEWorker(TpModelWorker)`）—— **两种 draft 架构并存**。

## 5 算法族对照矩阵

| 算法族 | draft 模型 | verify | Worker 接入方式 | KV pool | forward dispatch 入参 |
|---|---|---|---|---|---|
| **EAGLE / EAGLE3 (V1)** | EAGLE 小 transformer（可选 hot vocab） | tree greedy / sampling (CUDA) | **subclass**：[`EAGLEWorker(TpModelWorker) L79`](d:\design\sglang\python\sglang\srt\speculative\eagle_worker.py) | 独立 KV pool；共享 `req_to_token_pool` + `allocator`（[L115-117](d:\design\sglang\python\sglang\srt\speculative\eagle_worker.py)） | 重写 [`forward_batch_generation(batch: ScheduleBatch)` L279](d:\design\sglang\python\sglang\srt\speculative\eagle_worker.py)（**签名变 ScheduleBatch**） |
| **EAGLE / EAGLE3 (V2 / overlap)** | 同上 | 同上 + plan_stream 异步 | **组合**：[`EAGLEWorkerV2(BaseSpecWorker) L624`](d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py) 持有 `EagleDraftWorker`，内嵌 `TpModelWorker(is_draft_worker=True)` ([L138-152](d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py)) | 同 V1 | [`forward_batch_generation(mwb: ModelWorkerBatch)` L690](d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py) |
| **MultiLayer EAGLE (V1/V2)** | MTP，每步独立 `ModelRunner` | 同 EAGLE | V1 [`(TpModelWorker) L70`](d:\design\sglang\python\sglang\srt\speculative\multi_layer_eagle_worker.py) + `TpModelWorker.model_runner_list[]` 由 [`_init_multi_layer_eagle_model_runners` L363-388](d:\design\sglang\python\sglang\srt\managers\tp_worker.py) 追加 N-1 个 `ModelRunner`；V2 [`(BaseSpecWorker) L568`](d:\design\sglang\python\sglang\srt\speculative\multi_layer_eagle_worker_v2.py) | 多 `ModelRunner` 共享 `memory_pool_config` | `mtp_model_runner(layer_id) → model_runner_list[layer_id]` ([L236-237](d:\design\sglang\python\sglang\srt\speculative\multi_layer_eagle_worker.py)) |
| **Standalone (V1/V2)** | 任意独立 draft 模型（不共享 embed/lm_head） | 同 EAGLE | V1 [`StandaloneWorker(EAGLEWorker) L24`](d:\design\sglang\python\sglang\srt\speculative\standalone_worker.py)（间接 TpModelWorker 子类）；V2 [`StandaloneWorkerV2(EAGLEWorkerV2) L132` + `StandaloneDraftWorker(EagleDraftWorker) L35`](d:\design\sglang\python\sglang\srt\speculative\standalone_worker_v2.py) | 同 EAGLE | 复用 EAGLE 路径 |
| **DFLASH** | 独立 draft + 可选 sliding window draft KV | DFlash 专用（mask policy + Triton fused KV materialize） | **duck-typed**：[`class DFlashWorker:` L50](d:\design\sglang\python\sglang\srt\speculative\dflash_worker.py)（无基类），`self.model_runner = target.model_runner` ([L73-74](d:\design\sglang\python\sglang\srt\speculative\dflash_worker.py)) | 共享 target KV pool；`--speculative-dflash-draft-window-size` 时另开私有 compact `req_to_token` ([L92-100](d:\design\sglang\python\sglang\srt\speculative\dflash_worker.py)) | [`forward_batch_generation(batch, **kwargs)` L1101-1113](d:\design\sglang\python\sglang\srt\speculative\dflash_worker.py)：`ModelWorkerBatch` fallback 给 target |
| **NGRAM** | **无 draft 模型**（CPU n-gram trie / SAM） | `reconstruct_indices_from_tree_mask` (CUDA)，**不调** `verify_tree_greedy` | **duck-typed**：[`class NGRAMWorker:` L25](d:\design\sglang\python\sglang\srt\speculative\ngram_worker.py)（无基类），`self.model_runner = target.model_runner` ([L39](d:\design\sglang\python\sglang\srt\speculative\ngram_worker.py)) | 完全共享 target KV pool（无 draft 前向） | [`forward_batch_generation(batch: ScheduleBatch)` L252](d:\design\sglang\python\sglang\srt\speculative\ngram_worker.py)；底层 [`NgramCorpus`](d:\design\sglang\python\sglang\srt\speculative\cpp_ngram\ngram_corpus.py) C++ |

**关键文件分组**（详 [modules/speculative.md File inventory](../modules/speculative.md)）：EAGLE 系 = `eagle_*.py` 共 7 个；MultiLayer = `multi_layer_eagle_*.py` 共 4 个；Standalone = `standalone_worker{,_v2}.py`；DFlash = `dflash_*.py` 共 3 个 + `triton_ops/fused_kv_materialize.py`；NGRAM = `ngram_{worker,info}.py` + `external_corpus_manager.py` + `cpp_ngram/`（2 .py）。

> synthesis: `create_worker(server_args)` 路由维度 = **算法 × `enable_overlap` × `enable_multi_layer_eagle`**（[spec_info.py L59-122](d:\design\sglang\python\sglang\srt\speculative\spec_info.py)）。同一 EAGLE 算法在 `enable_overlap=True` 时返回 V2、否则返回 V1 —— **这是源码中每个 EAGLE 类都有 `_v2.py` 双胞胎的根因**。DFlash / NGRAM 在 V2 路径直接 raise（[L70-72 / L113-116](d:\design\sglang\python\sglang\srt\speculative\spec_info.py)）。

## TpModelWorker subclass vs duck-typed worker

> synthesis: 这是 SGLang spec 设计的**核心架构选择**，理解它就理解了为什么 27 个 .py 分成两批截然不同的模式。

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

SGLang spec 验证调用 **2 个独立 sgl-kernel CUDA 算子**：

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

完整 spec 相关参数（dataclass [server_args.py L497-524](d:\design\sglang\python\sglang\srt\server_args.py)；argparse 注册 [L5099-5271](d:\design\sglang\python\sglang\srt\server_args.py)；规范化 [`_handle_speculative_decoding` L3107-3160](d:\design\sglang\python\sglang\srt\server_args.py)）。

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

## Notes / Caveats

> [!todo] VERIFY: pin 从 `34fef07a` → `06f32bab`（2026-08-10 increment）后本页未深 verify；文件数量/行号可能漂移。优先对照 [entities/Scheduler.md](../entities/Scheduler.md) / 新模块页。

> [!warning] CONTRADICTION（cross-page lint follow-up pending）
> [comparison/topics/speculative-decoding.md L80](d:\design\wiki\comparison\topics\speculative-decoding.md) 写"SGLang **24** 文件 / vLLM **12** 文件"；本轮 Glob 实测 **SGLang 27 .py**（23 根 + 2 `triton_ops/` + 2 `cpp_ngram/`）、**vLLM 11 .py**（[`d:\design\vllm\vllm\v1\spec_decode\`](d:\design\vllm\vllm\v1\spec_decode)）。L439 `vllm/topics/spec-decode-eagle.md` 链接描述同样写"12"。
> 本任务范围为 sglang 内部 topic，不修改 comparison 页。**已记录为下次 lint pass / 该 compare 页 verify pass 修复目标（`24 → 27` SGLang、`12 → 11` vLLM）**；详见 [modules/speculative.md §Notes](../modules/speculative.md) 已 RESOLVED 的同名 VERIFY 块。

> [!warning] CONTRADICTION（worker 抽象命名陷阱）
> [comparison L82](../../comparison/topics/speculative-decoding.md) 顶层 synthesis "SGLang draft 用独立 TpModelWorker(is_draft_worker=True)" **仅适用于 EAGLE / Standalone / MultiLayer 系**；DFLASH / NGRAM 是 duck-typed 包装，无 `is_draft_worker` 标志、不构造独立 `ModelRunner`。精化措辞详见本页 §"TpModelWorker subclass vs duck-typed worker"。

> [!todo] VERIFY: `MultiLayerEagleWorkerV2(BaseSpecWorker)` 在 V2 路径下 `model_runner_list` 由谁持有——本轮仅确认 V1 multi-layer 走 `TpModelWorker.model_runner_list`，V2 走 `BaseSpecWorker` 而非直接继承 `TpModelWorker`，需要查 V2 多 layer 内部组织。

> [!todo] VERIFY: `forward_batch_generation` **签名差异**：`EAGLEWorker.forward_batch_generation(batch: ScheduleBatch)` ([eagle_worker.py L279](d:\design\sglang\python\sglang\srt\speculative\eagle_worker.py)) vs 父类 `TpModelWorker.forward_batch_generation(model_worker_batch: ModelWorkerBatch, forward_batch=None, ...)` ([tp_worker.py L443](d:\design\sglang\python\sglang\srt\managers\tp_worker.py))——子类改第一参数类型；scheduler 调用栈如何按 worker 类型选用不同入参，留待 [`Scheduler.run_batch`](../entities/Scheduler.md) 深入。

## See also

- [sglang/modules/speculative.md](../modules/speculative.md) — 模块全景：27 .py 文件分类、`SpeculativeAlgorithm` enum 完整 6 成员、`SpecInput` dataclass 注入 ScheduleBatch / ForwardBatch、Triton ops 子目录、N-gram C++ 扩展
- [sglang/entities/TpModelWorker.md](../entities/TpModelWorker.md) — `model_runner_list` 字段语义 + EAGLE multi-layer 联动
- [sglang/entities/Scheduler.md](../entities/Scheduler.md) — `init_tp_model_worker` / `maybe_init_draft_worker` / `init_model_worker` 三步走 + `model_worker = draft_worker` 切换语义
- [sglang/modules/sampling.md](../modules/sampling.md) — sgl-kernel 通用采样 kernel 路径（`top_k_renorm_prob` / `top_p_renorm_prob` 与 spec 共用）
- [sglang/modules/layers.md](../modules/layers.md) — attention backend；spec draft / verify 通过 `--speculative-draft-attention-backend` 与 `--speculative-attention-mode` 选择独立 backend
- [sglang/modules/distributed.md](../modules/distributed.md) — sgl-kernel C++ 绑定模式参照（`shm_allreduce` 是另一组已确认绑定）
- [comparison/topics/speculative-decoding.md](../../comparison/topics/speculative-decoding.md) — 三家 spec 9 子维度对比（**本页是 SGLang 内部 deep-dive，对比页负责跨项目**）
- [comparison/topics/executor-worker.md](../../comparison/topics/executor-worker.md) — draft model runner 持有位置三家差异（`model_runner_list` 是 SGLang 答案）
- [comparison/dimensions.md §dim-spec](../../comparison/dimensions.md)
