---
type: comparison
project: cross
status: verified
confidence: high
verified_against: 2026-04-19 (verify pass: 2026-04-18; lint fix: 2026-04-19)
sources:
  - d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\mtp\mtp_plugin.py
  - d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\mtp\decoding_policy.py
  - d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\la\la_plugin.py
  - d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\memory_decoding\memory_decoding_plugin.py
  - d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_utils.py
  - d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\spec_worker.py
  - d:\design\MindIE-LLM\mindie_llm\runtime\models\deepseek_v3\deepseek_v3_mtp.py
  - d:\design\MindIE-LLM\src\scheduler\scheduler.cpp
  - d:\design\MindIE-LLM\src\include\config\config_info.h
  - d:\design\MindIE-LLM\src\config_manager\config_interaction.cpp
  - d:\design\vllm\vllm\config\speculative.py
  - d:\design\vllm\vllm\config\vllm.py
  - d:\design\vllm\vllm\v1\spec_decode\eagle.py
  - d:\design\vllm\vllm\v1\spec_decode\medusa.py
  - d:\design\vllm\vllm\v1\spec_decode\dflash.py
  - d:\design\vllm\vllm\v1\spec_decode\draft_model.py
  - d:\design\vllm\vllm\v1\spec_decode\ngram_proposer.py
  - d:\design\vllm\vllm\v1\spec_decode\suffix_decoding.py
  - d:\design\vllm\vllm\v1\spec_decode\metadata.py
  - d:\design\vllm\vllm\v1\worker\gpu\spec_decode\rejection_sampler.py
  - d:\design\vllm\vllm\v1\worker\gpu\spec_decode\eagle\speculator.py
  - d:\design\vllm\vllm\v1\core\sched\scheduler.py
  - d:\design\vllm\vllm\v1\core\sched\async_scheduler.py
  - d:\design\sglang\python\sglang\srt\speculative\spec_info.py
  - d:\design\sglang\python\sglang\srt\speculative\base_spec_worker.py
  - d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py
  - d:\design\sglang\python\sglang\srt\speculative\eagle_worker.py
  - d:\design\sglang\python\sglang\srt\speculative\standalone_worker_v2.py
  - d:\design\sglang\python\sglang\srt\speculative\multi_layer_eagle_worker_v2.py
  - d:\design\sglang\python\sglang\srt\speculative\dflash_worker.py
  - d:\design\sglang\python\sglang\srt\speculative\ngram_worker.py
  - d:\design\sglang\python\sglang\srt\speculative\eagle_utils.py
  - d:\design\sglang\python\sglang\srt\speculative\eagle_info.py
  - d:\design\sglang\python\sglang\srt\speculative\eagle_info_v2.py
  - d:\design\sglang\python\sglang\srt\speculative\external_corpus_manager.py
  - d:\design\sglang\python\sglang\srt\speculative\cpp_ngram\ngram_corpus.py
  - d:\design\sglang\python\sglang\srt\server_args.py
  - d:\design\sglang\python\sglang\srt\managers\scheduler.py
related:
  - comparison/index.md
  - comparison/dimensions.md
  - comparison/topics/async-schedule.md
  - comparison/topics/pd-disaggregation.md
  - comparison/topics/sync-schedule.md
  - comparison/topics/scheduler.md
  - mindie/topics/speculative.md
  - mindie/entities/PluginManager.md
  - mindie/entities/BatchScheduler.md
  - mindie/entities/ModelRunner.md
  - vllm/topics/spec-decode-eagle.md
  - sglang/modules/speculative.md
  - sglang/topics/speculative.md
---

# Cross-project Comparison: Speculative Decoding（MTP / EAGLE / N-gram / Medusa / DFlash / Suffix）

> 三项目「投机解码 / 多 token 预测」实现对比，覆盖 [§dim-spec](../dimensions.md)。
> 深入展开 [mindie/topics/speculative.md](../../mindie/topics/speculative.md)（MindIE 专题），并补全 vLLM 与 SGLang 的等价物。
>
> **核心问题**：三家如何（a）枚举草稿算法、（b）把 draft 模型和主模型挂在一起、（c）做 verify、（d）让 scheduler / KV cache 能容纳 draft token？
>
> **核心差异 TL;DR**（synthesis）：
>
> - **MindIE**：Python `Plugin` 体系（3 个独立子目录 `mtp` / `la` / `memory_decoding`）+ 可选 `MtpWorker` 双 ModelRunner + **C++ Scheduler `speculationGamma` placeholder 机制**统一管 KV 槽位。
> - **vLLM**：v1 把 6 类 proposer（Eagle / Medusa / N-gram / N-gram-GPU / DraftModel / Suffix / DFlash）收进 `v1/spec_decode/` 单包 + `EagleSpeculator`（GPU）独立 + 共享 `RejectionSampler`（strict / probabilistic / synthetic 三 sampler）。
> - **SGLang**：`SpeculativeAlgorithm` 5 enum + spec_v2 / 非 v2 二选一 worker（11+ worker / draft / cuda graph runner 文件）+ 与 overlap scheduler 强耦合（spec v2 = overlap）。

---

## TL;DR 三方对照

| 维度 | MindIE | vLLM | SGLang |
|---|---|---|---|
| **组织形式** | Plugin 子目录（[plugins/mtp/](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\mtp), [plugins/la/](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\la), [plugins/memory_decoding/](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\memory_decoding)）+ `MtpWorker` proxy（[spec_worker.py](d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\spec_worker.py)） | 单包 `v1/spec_decode/` **11 .py**（不含 `__init__.py`）+ `v1/worker/gpu/spec_decode/eagle/` 单独子树（[v1/spec_decode/](d:\design\vllm\vllm\v1\spec_decode), [v1/worker/gpu/spec_decode/](d:\design\vllm\vllm\v1\worker\gpu\spec_decode)） | 独立 `srt/speculative/` **27 .py**（递归含 `cpp_ngram/` 子目录）+ spec_v2 / 非 v2 双 worker 体系（[srt/speculative/](d:\design\sglang\python\sglang\srt\speculative)） |
| **算法枚举** | `plugin_list` 字符串（`mtp` / `la` / `memory_decoding`）+ `PluginParameterValidator` JSON 校验（[plugin_utils.py:63-89](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_utils.py)） | `SpeculativeMethod` Literal：`ngram` / `medusa` / `mlp_speculator` / `draft_model` / `suffix` / `EagleModelTypes`（13 种 MTP 子串 + `eagle` / `eagle3` / `extract_hidden_states` / `dflash`）/ `ngram_gpu`（[config/speculative.py:34-64](d:\design\vllm\vllm\config\speculative.py)） | `SpeculativeAlgorithm` Enum 5 项：`DFLASH` / `EAGLE` / `EAGLE3` / `STANDALONE` / `NGRAM`（[spec_info.py:15-23](d:\design\sglang\python\sglang\srt\speculative\spec_info.py)） |
| **Draft 模型集成** | **双 ModelRunner**：`MtpWorker.main_model_runner` + `draft_model_runner`（`is_draft_model=True`）（[spec_worker.py:43-64](d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\spec_worker.py)）；MTP 模型实现绑定 DeepSeek MTP 层 [deepseek_v3_mtp.py:131-170](d:\design\MindIE-LLM\mindie_llm\runtime\models\deepseek_v3\deepseek_v3_mtp.py) | `SpecDecodeBaseProposer`（[v1/spec_decode/eagle.py:60-130](d:\design\vllm\vllm\v1\spec_decode\eagle.py)）+ `MedusaProposer` / `DFlashProposer` / `DraftModelProposer` / `SuffixDecodingProposer` 各自 `load_model` `get_model(...)`；`MedusaProposer.load_model` 用 `set_model_tag("medusa_head")`（[medusa.py:57-64](d:\design\vllm\vllm\v1\spec_decode\medusa.py)） | **独立 TpModelWorker(is_draft_worker=True) 仅 EAGLE 系**[^tpw-precision]：`EagleDraftWorker` 内创建 `self.draft_worker = TpModelWorker(..., is_draft_worker=True, req_to_token_pool=..., token_to_kv_pool_allocator=...)` —— **共享 allocator 但有独立 KV pool**（[eagle_worker_v2.py:125-152](d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py)）；`DFlashWorker` / `NGRAMWorker` 走 duck-typed 直接包装 target_worker，**不开第二个 TpModelWorker** |

[^tpw-precision]: **2026-04-19 lint 精确化**：原文笼统说"SGLang 用独立 TpModelWorker"易误解为所有算法都开第二个 worker。实际只有 **EAGLE / EAGLE3 / STANDALONE**（统称 "EAGLE 系"，spec_v2 路径，[`eagle_worker_v2.py`](d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py) / [`standalone_worker_v2.py`](d:\design\sglang\python\sglang\srt\speculative\standalone_worker_v2.py)）走"独立 TpModelWorker(is_draft_worker=True)"路径；**DFLASH** ([`dflash_worker.py`](d:\design\sglang\python\sglang\srt\speculative\dflash_worker.py)) 与 **NGRAM** ([`ngram_worker.py`](d:\design\sglang\python\sglang\srt\speculative\ngram_worker.py)) 是 duck-typed wrapper —— DFLASH 用主模型自身的 hidden states + KV，NGRAM 走 cpp_ngram 语料 trie 不需 GPU draft 模型，二者均无第二个 TpModelWorker（[sglang/topics/speculative.md](../../sglang/topics/speculative.md) §worker 体系）。
| **Verify 机制** | Python 贪婪 verify：`MtpPlugin.plugin_verify` → `decoding_policy.verify_greedy_one_batch`（线性比对 draft 与 target）（[decoding_policy.py:60-69](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\mtp\decoding_policy.py)） | **`RejectionSampler` 三模式 Triton kernel**：`strict` / `probabilistic` / `synthetic`（[rejection_sampler.py:100-235](d:\design\vllm\vllm\v1\worker\gpu\spec_decode\rejection_sampler.py), [config/speculative.py:65](d:\design\vllm\vllm\config\speculative.py)） | **Tree verify CUDA kernel**：`verify_tree_greedy_func` → `sgl_kernel.verify_tree_greedy`（CUDA / NPU 双实现）+ `tree_speculative_sampling_target_only`（采样路径）（[eagle_utils.py:161-199](d:\design\sglang\python\sglang\srt\speculative\eagle_utils.py), [eagle_info_v2.py:51-56](d:\design\sglang\python\sglang\srt\speculative\eagle_info_v2.py)） |
| **Scheduler 集成** | C++ `Scheduler::AddNextTokenPlaceHolder` / `ReplacePlaceHolderWithToken` —— 用 `PLACEHOLDER_TOKEN`(=-1) 占 KV 槽，`tokenNumPerIter = 1 + speculationGamma`（[scheduler.cpp:760-776, 802-915](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)） | Python `Scheduler.num_lookahead_tokens` + `scheduled_spec_decode_tokens: dict[str, list[int]]` 显式传递（[v1/core/sched/scheduler.py:213-222, 376, 525-535, 905-924](d:\design\vllm\vllm\v1\core\sched\scheduler.py)）；async path 用 `_spec_token_placeholders = [-1]*num_spec_tokens`（[async_scheduler.py:16, 32-35](d:\design\vllm\vllm\v1\core\sched\async_scheduler.py)） | `ModelWorkerBatch.spec_info: SpecInput`（dataclass）—— 在 batch 上挂 `EagleDraftInput` / `EagleVerifyInput` / `NgramVerifyInput`（[spec_info.py:125-167](d:\design\sglang\python\sglang\srt\speculative\spec_info.py)）；scheduler 在 `dispatch_event_loop` 里按 `is_spec_v2` 分流（[managers/scheduler.py:1486-1495, 2797-2800](d:\design\sglang\python\sglang\srt\managers\scheduler.py)） |
| **Draft KV 槽位预留** | C++ 端按 `maxScheduledBatch_ * tokenNumPerIter + tokenNumPerIter` 预留占位符（[scheduler.cpp:769-771](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)） | `extra_slots_per_request = 1 if not parallel_drafting else num_speculative_tokens` + `needs_extra_input_slots` 标志（[v1/spec_decode/eagle.py:91-97, 656, 690-739](d:\design\vllm\vllm\v1\spec_decode\eagle.py)） | `EagleDraftInput.ALLOC_LEN_PER_DECODE = max(num_steps*topk, num_draft_tokens)`（[standalone_worker_v2.py:73-75](d:\design\sglang\python\sglang\srt\speculative\standalone_worker_v2.py)）+ `alloc_token_slots` / `alloc_paged_token_slots_extend`（[eagle_info_v2.py:20-21, 137-146](d:\design\sglang\python\sglang\srt\speculative\eagle_info_v2.py)） |
| **关键参数** | `speculationGamma` (uint32_t)（[config_info.h:118, 282](d:\design\MindIE-LLM\src\include\config\config_info.h)）+ plugin JSON `num_speculative_tokens` / `level` / `window` / `guess_set_size` / `decoding_length` | `num_speculative_tokens` (>0) + `method` + `prompt_lookup_min/max` + `parallel_drafting` + `disable_padded_drafter_batch`（[config/speculative.py:75-100](d:\design\vllm\vllm\config\speculative.py)） | `speculative_algorithm` + `speculative_num_steps` + `speculative_eagle_topk` + `speculative_num_draft_tokens` + ngram 8 个独立参数（[server_args.py:497-525](d:\design\sglang\python\sglang\srt\server_args.py)） |
| **CUDA Graph** | 通过 `aclgraph` 整体 capture（不单为 spec）—— spec_worker 调 `model_runner.forward` 自动复用 | `EagleSpeculator` 内含独立 `EagleCudaGraphManager`（[gpu/spec_decode/eagle/cudagraph.py](d:\design\vllm\vllm\v1\worker\gpu\spec_decode\eagle\cudagraph.py)） + `CudagraphDispatcher` PIECEWISE 模式（[v1/spec_decode/eagle.py:127-130](d:\design\vllm\vllm\v1\spec_decode\eagle.py)） | **3 个独立 CUDA graph runner**：`EAGLEDraftCudaGraphRunner` / `EAGLEDraftExtendCudaGraphRunner` / `MultiLayerEagleDraftExtendCudaGraphRunner`（[eagle_worker_v2.py:250-317](d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py)） + 独立 plan_stream（[eagle_worker_v2.py:74-82, 564-576](d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py)） |
| **与 overlap scheduler 兼容** | 与 async（`activateAsyncInference`）通过 `tokenNumPerIter` placeholder 长度统一（[scheduler.cpp:881-883](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)） | async + spec **仅 EagleModelTypes / NgramGPUTypes 允许**；其它（Medusa / draft_model / suffix）会被 auto-disable async（[config/vllm.py:801-812](d:\design\vllm\vllm\config\vllm.py)） | spec v2（EAGLE / EAGLE3 / STANDALONE）+ overlap 互启 + 需要 `SGLANG_ENABLE_SPEC_V2=True`（[server_args.py:3275-3295](d:\design\sglang\python\sglang\srt\server_args.py)）；DFLASH / NGRAM **强制 disable overlap**（[server_args.py:3251-3253, 3380](d:\design\sglang\python\sglang\srt\server_args.py)） |
| **结构化输出兼容** | 服务端 `mtpEnabled` 与 `response_format` 互斥（[mindie/topics/speculative.md §配置依赖](../../mindie/topics/speculative.md)） | async path 用 `pending_structured_output_tokens` flag 协调（[async_scheduler.py:26-28](d:\design\vllm\vllm\v1\core\sched\async_scheduler.py)） | `is_disable_overlap_for_batch` 在 `spec_v2 + grammar + decode + queue非空` 强制禁 overlap（[managers/scheduler.py:1486-1495](d:\design\sglang\python\sglang\srt\managers\scheduler.py)） |

---

## 1. 算法目录（class layout）

### MindIE：Plugin 子目录 + 单 worker proxy

| 子目录 / 文件 | 主类 | 行号 |
|---|---|---|
| [plugins/mtp/mtp_plugin.py](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\mtp\mtp_plugin.py) | `MtpPlugin(Plugin)` | [mtp_plugin.py:24-25](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\mtp\mtp_plugin.py) |
| [plugins/la/la_plugin.py](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\la\la_plugin.py) | `LaPlugin(Plugin)`（Lookahead/Jacobi） | [la_plugin.py:26-27](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\la\la_plugin.py) |
| [plugins/memory_decoding/memory_decoding_plugin.py](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\memory_decoding\memory_decoding_plugin.py) | `MemoryDecodingPlugin(Plugin)`（Trie 历史 IO） | [memory_decoding_plugin.py:21-22](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\memory_decoding\memory_decoding_plugin.py) |
| [runtime/model_runner/spec_worker.py](d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\spec_worker.py) | `MtpWorker` / `MtpWorkerExp`（仅 MTP 用） | [spec_worker.py:43-64, 323-659, 693-707](d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\spec_worker.py) |

详见 [mindie/topics/speculative.md](../../mindie/topics/speculative.md)。

### vLLM：v1/spec_decode/ 单包

| 文件 | 主类 | 行号 / 备注 |
|---|---|---|
| [v1/spec_decode/eagle.py](d:\design\vllm\vllm\v1\spec_decode\eagle.py) | `SpecDecodeBaseProposer` —— **全家通用基类**，含 EAGLE / EAGLE3 / DraftModel / DFlash | [eagle.py:60-130](d:\design\vllm\vllm\v1\spec_decode\eagle.py) |
| [v1/spec_decode/medusa.py](d:\design\vllm\vllm\v1\spec_decode\medusa.py) | `MedusaProposer`（多头并行） | [medusa.py:18-79](d:\design\vllm\vllm\v1\spec_decode\medusa.py) |
| [v1/spec_decode/dflash.py](d:\design\vllm\vllm\v1\spec_decode\dflash.py) | `DFlashProposer(SpecDecodeBaseProposer)` | [dflash.py:20-60](d:\design\vllm\vllm\v1\spec_decode\dflash.py) |
| [v1/spec_decode/draft_model.py](d:\design\vllm\vllm\v1\spec_decode\draft_model.py) | `DraftModelProposer(SpecDecodeBaseProposer)` —— 通用 draft 模型 | [draft_model.py:17-40](d:\design\vllm\vllm\v1\spec_decode\draft_model.py) |
| [v1/spec_decode/ngram_proposer.py](d:\design\vllm\vllm\v1\spec_decode\ngram_proposer.py) | `NgramProposer`（CPU + numba JIT，prompt lookup） | [ngram_proposer.py:12-62](d:\design\vllm\vllm\v1\spec_decode\ngram_proposer.py) |
| [v1/spec_decode/ngram_proposer_gpu.py](d:\design\vllm\vllm\v1\spec_decode\ngram_proposer_gpu.py) | `NgramProposerGpu` | （ngram_gpu 路径） |
| [v1/spec_decode/suffix_decoding.py](d:\design\vllm\vllm\v1\spec_decode\suffix_decoding.py) | `SuffixDecodingProposer`（基于外部 `arctic_inference.SuffixDecodingCache`） | [suffix_decoding.py:9-65](d:\design\vllm\vllm\v1\spec_decode\suffix_decoding.py) |
| [v1/spec_decode/extract_hidden_states.py](d:\design\vllm\vllm\v1\spec_decode\extract_hidden_states.py) | hidden state 抽取辅助 | （Eagle3 专用） |
| [v1/spec_decode/metadata.py](d:\design\vllm\vllm\v1\spec_decode\metadata.py) | `SpecDecodeMetadata` dataclass | [metadata.py:9-66](d:\design\vllm\vllm\v1\spec_decode\metadata.py) |
| [v1/spec_decode/metrics.py](d:\design\vllm\vllm\v1\spec_decode\metrics.py) | `SpecDecodingStats` | （metrics 暴露） |
| [v1/spec_decode/utils.py](d:\design\vllm\vllm\v1\spec_decode\utils.py) | Triton kernel：`copy_and_expand_eagle_inputs_kernel` 等 | （worker 端 layout 工具） |
| [v1/worker/gpu/spec_decode/rejection_sampler.py](d:\design\vllm\vllm\v1\worker\gpu\spec_decode\rejection_sampler.py) | `RejectionSampler`（统一三 sampler） | [rejection_sampler.py:100-235](d:\design\vllm\vllm\v1\worker\gpu\spec_decode\rejection_sampler.py) |
| [v1/worker/gpu/spec_decode/eagle/speculator.py](d:\design\vllm\vllm\v1\worker\gpu\spec_decode\eagle\speculator.py) | `EagleSpeculator`（GPU 端 EAGLE 实现，独立 input buffer + cudagraph） | [speculator.py:37-90](d:\design\vllm\vllm\v1\worker\gpu\spec_decode\eagle\speculator.py) |

### SGLang：双 worker 体系（spec v1 / spec v2）

| 文件 | 主类 | 备注 |
|---|---|---|
| [base_spec_worker.py](d:\design\sglang\python\sglang\srt\speculative\base_spec_worker.py) | `BaseSpecWorker(ABC)` / `BaseDraftWorker(ABC)` | 抽象接口 [base_spec_worker.py:10-35](d:\design\sglang\python\sglang\srt\speculative\base_spec_worker.py) |
| [eagle_worker.py](d:\design\sglang\python\sglang\srt\speculative\eagle_worker.py) | `EAGLEWorker`（spec v1） | 与 sync schedule 配合 |
| [eagle_worker_v2.py](d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py) | `EAGLEWorkerV2(BaseSpecWorker)` + `EagleDraftWorker(BaseDraftWorker)`（spec v2，与 overlap scheduler 配合） | [eagle_worker_v2.py:85-152, 624-739](d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py) |
| [standalone_worker_v2.py](d:\design\sglang\python\sglang\srt\speculative\standalone_worker_v2.py) | `StandaloneWorkerV2 / StandaloneDraftWorker(EagleDraftWorker)`（不共享 embed/lm_head） | [standalone_worker_v2.py:35-100](d:\design\sglang\python\sglang\srt\speculative\standalone_worker_v2.py) |
| [multi_layer_eagle_worker_v2.py](d:\design\sglang\python\sglang\srt\speculative\multi_layer_eagle_worker_v2.py) | `MultiLayerEagleWorkerV2`（多层 EAGLE） | （`enable_multi_layer_eagle`） |
| [dflash_worker.py](d:\design\sglang\python\sglang\srt\speculative\dflash_worker.py) | `DFlashWorker`（DFLASH 路径，**仅 spec v1，禁 overlap**） | [spec_info.py:68-75](d:\design\sglang\python\sglang\srt\speculative\spec_info.py) |
| [ngram_worker.py](d:\design\sglang\python\sglang\srt\speculative\ngram_worker.py) | `NGRAMWorker`（含 cpp_ngram corpus + 可选 external corpus） | [ngram_worker.py:25-80](d:\design\sglang\python\sglang\srt\speculative\ngram_worker.py) |
| [external_corpus_manager.py](d:\design\sglang\python\sglang\srt\speculative\external_corpus_manager.py) | `ExternalCorpusManager`（NGRAM 外部语料 add/remove/list 异步加载） | [external_corpus_manager.py:23-71](d:\design\sglang\python\sglang\srt\speculative\external_corpus_manager.py) |
| [cpp_ngram/](d:\design\sglang\python\sglang\srt\speculative\cpp_ngram) | `NgramCorpus`（C++ 后端 trie） | [cpp_ngram/ngram_corpus.py](d:\design\sglang\python\sglang\srt\speculative\cpp_ngram\ngram_corpus.py) |
| [eagle_info.py](d:\design\sglang\python\sglang\srt\speculative\eagle_info.py) / [eagle_info_v2.py](d:\design\sglang\python\sglang\srt\speculative\eagle_info_v2.py) | `EagleDraftInput` / `EagleVerifyInput` 等 spec_info dataclass | [spec_info.py:125-167](d:\design\sglang\python\sglang\srt\speculative\spec_info.py) |
| [eagle_utils.py](d:\design\sglang\python\sglang\srt\speculative\eagle_utils.py) | `verify_tree_greedy_func` / `build_tree_kernel_efficient` 等 kernel 入口 | [eagle_utils.py:161-199](d:\design\sglang\python\sglang\srt\speculative\eagle_utils.py) |
| [draft_utils.py](d:\design\sglang\python\sglang\srt\speculative\draft_utils.py) | `DraftBackendFactory` | （选 attn backend） |
| [eagle_draft_cuda_graph_runner.py](d:\design\sglang\python\sglang\srt\speculative\eagle_draft_cuda_graph_runner.py) / [eagle_draft_extend_cuda_graph_runner.py](d:\design\sglang\python\sglang\srt\speculative\eagle_draft_extend_cuda_graph_runner.py) / [multi_layer_eagle_draft_extend_cuda_graph_runner.py](d:\design\sglang\python\sglang\srt\speculative\multi_layer_eagle_draft_extend_cuda_graph_runner.py) | 3 个独立 cuda graph runner | （[eagle_worker_v2.py:250-317](d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py) capture） |

> synthesis（**为什么三家组织方式不同**）：MindIE 把"算法"做成可热插拔的 Plugin（与 `splitfuse` / `prefix_cache` 等同样进 plugin_list）；vLLM 把不同 proposer 收进同一目录但每个一个 class，sampling 复用单个 `RejectionSampler`；SGLang 按"算法 × spec_v2 是否启用"做笛卡尔积，每个组合一个 worker 文件。SGLang 文件最多（**27 个 .py**，递归含 `cpp_ngram/`）反映出它把"spec v1 兼容旧 sync 路径"和"spec v2 与 overlap scheduler 配合"分别落地，未做单一抽象。

---

## 2. Draft 模型集成机制

### MindIE：双 ModelRunner（仅 MTP 走该路径）

`MtpWorker`（[spec_worker.py:43-64](d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\spec_worker.py)）：

```text
self.main_model_runner  = ModelRunner(...)                     # 主模型
self.draft_model_runner = ModelRunner(..., is_draft_model=True) # 复制权重的 draft 模型
```

draft 模型实现在 [deepseek_v3_mtp.py](d:\design\MindIE-LLM\mindie_llm\runtime\models\deepseek_v3\deepseek_v3_mtp.py)：单层 `DeepseekV3MtpLayer` + `embed_tokens` + `eh_proj` + `ParallelLMHead`。**当前实现绑定 DeepSeek V3 的 layer_idx=61**（[deepseek_v3_mtp.py:131-138](d:\design\MindIE-LLM\mindie_llm\runtime\models\deepseek_v3\deepseek_v3_mtp.py)）。

`speculative_worker_selector` 决定是否包装：仅 `num_speculative_tokens > 0` 时返回 `MtpWorker(Exp)`，否则返回原始 `ModelRunner`（[spec_worker.py:693-707](d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\spec_worker.py)）。LA / memory_decoding **不走 `MtpWorker`**，仍是单 ModelRunner。

### vLLM：`SpecDecodeBaseProposer` 单 model + `get_model` 加载

`SpecDecodeBaseProposer.__init__`（[v1/spec_decode/eagle.py:60-130](d:\design\vllm\vllm\v1\spec_decode\eagle.py)）：

- 持 `self.draft_model_config = self.speculative_config.draft_model_config`
- `load_model` 调 `vllm.model_executor.model_loader.get_model(...)` 加载 draft
- 通过 `pass_hidden_states_to_model` 区分 EAGLE vs DraftModel（DraftModel = `False`，EAGLE/DFlash = `True`）
- `parallel_drafting` flag 决定 `extra_slots_per_request` 公式（[eagle.py:91-97](d:\design\vllm\vllm\v1\spec_decode\eagle.py)）

`MedusaProposer.load_model`（[medusa.py:57-64](d:\design\vllm\vllm\v1\spec_decode\medusa.py)）：

```python
with set_model_tag("medusa_head"):
    self.model = get_model(vllm_config=self.vllm_config,
                           model_config=self.spec_config.draft_model_config)
```

vLLM 没有 MindIE 式"两个完整 ModelRunner"的概念 —— Medusa 只挂多头到同一 forward；EAGLE 由 `EagleSpeculator` 内部建专属 `BlockTables` / `InputBuffers` / `idx_mapping`（[gpu/spec_decode/eagle/speculator.py:63-80](d:\design\vllm\vllm\v1\worker\gpu\spec_decode\eagle\speculator.py)），而非复用 `GPUModelRunner`。

### SGLang：独立 `TpModelWorker(is_draft_worker=True)` 共享 allocator

`EagleDraftWorker.__init__`（[eagle_worker_v2.py:125-152](d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py)）：

```python
self.req_to_token_pool, self.token_to_kv_pool_allocator = target_worker.get_memory_pool()
...
self.draft_worker = TpModelWorker(
    server_args=server_args,
    ...
    is_draft_worker=True,
    req_to_token_pool=self.req_to_token_pool,             # 共享 allocator
    token_to_kv_pool_allocator=self.token_to_kv_pool_allocator,
    memory_pool_config=target_worker.model_runner.memory_pool_config,
)
```

注释明示"**Draft and target worker own their own KV cache pools**"（[eagle_worker_v2.py:124](d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py)）—— 两个 worker 各持 KV pool 但 allocator/req_to_token 共享。`StandaloneDraftWorker` 同样模式但**不共享 embed/lm_head**（[standalone_worker_v2.py:35-37](d:\design\sglang\python\sglang\srt\speculative\standalone_worker_v2.py)）。

> synthesis（**模型隔离的三种程度**）：MindIE「双 ModelRunner」最重；vLLM「proposer + load_model」最轻（与主模型公用 `GPUModelRunner` 元数据）；SGLang「独立 TpModelWorker 共享 allocator」中间路线。三家共同点是**主模型的 hidden states 与 lm_head 信息 必须传递给 draft 侧**（MindIE 通过 `forward_context.mtp_metadata.last_hidden_states` / vLLM 通过 `pass_hidden_states_to_model` / SGLang 通过 `EagleDraftInput.hidden_states`）。

---

## 3. Verify 机制

| 项目 | 实现路径 | Kernel / 算法 |
|---|---|---|
| MindIE | `MtpPlugin.plugin_verify` → `decoding_policy.verify_greedy_one_batch`（[mtp/decoding_policy.py:60-69](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\mtp\decoding_policy.py)） | **Python 线性循环**（贪婪逐 token 比对） |
| MindIE LA | `LaPlugin.plugin_verify` → `la_token_verify_not_sample` → `la_decoding_policy.la_verify_greedy_one_batch`（[la_plugin.py:119-144](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\la\la_plugin.py)） | **Python 贪婪 verify** |
| MindIE Memory | `MemoryDecodingPlugin.plugin_verify` → `decoding_policy.verify`（[memory_decoding_plugin.py:225-234](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\memory_decoding\memory_decoding_plugin.py)） | Python verify |
| vLLM | `RejectionSampler.__call__` 三模式分流（[rejection_sampler.py:159-235](d:\design\vllm\vllm\v1\worker\gpu\spec_decode\rejection_sampler.py)） | Triton：`_strict_rejection_sample_kernel`（贪婪，[rejection_sampler.py:23-77](d:\design\vllm\vllm\v1\worker\gpu\spec_decode\rejection_sampler.py)）/ `probabilistic_rejection_sample`（采样路径）/ `synthetic_rejection_sample`（合成测试） |
| SGLang | `verify_tree_greedy_func` → `sgl_kernel.verify_tree_greedy`（CUDA / HIP）/ `sgl_kernel_npu.sample.verify_tree_greedy.verify_tree_greedy`（NPU）（[eagle_utils.py:161-199](d:\design\sglang\python\sglang\srt\speculative\eagle_utils.py)） | **Tree verify CUDA kernel**（多分支同时验证）+ `tree_speculative_sampling_target_only`（采样路径，[eagle_info_v2.py:51-56](d:\design\sglang\python\sglang\srt\speculative\eagle_info_v2.py)） |

> synthesis（**verify 实现分布在不同层**）：
>
> - **MindIE**：verify **完全在 Python**（Plugin 层），逐 batch 串行循环。优点：易调试，与 plugin 体系一致；代价：CPU bound，accept_length 取决于 GIL 与 Python 速度。
> - **vLLM**：verify 在 **GPU triton kernel**，与 `Sampler` 同 stream。三 sampler（strict/probabilistic/synthetic）由 `RejectionSampler` 统一封装。
> - **SGLang**：verify 是 **专用 CUDA kernel**（`verify_tree_greedy`），且**支持 tree mask**（多分支并行 verify），覆盖率最高但要求 `sgl_kernel` / `sgl_kernel_npu` 两套实现。
>
> 三家选择反映"**算法稳定性 vs 性能**"取舍：MindIE Python verify 适合算法迭代期；vLLM/SGLang kernel verify 适合稳定后压榨延迟。

---

## 4. Scheduler 集成

### MindIE：C++ Placeholder 系统

完整 § 见 [mindie/topics/speculative.md §C++ Scheduler placeholder 机制](../../mindie/topics/speculative.md)。要点（[scheduler.cpp:760-915](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)）：

| 函数 | 行号 | 行为 |
|---|---|---|
| `CalculatePlaceHolderNum` | [scheduler.cpp:760-776](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp) | `tokenNumPerIter = 1 + speculationGamma`；`maxPlaceHolderNum = maxScheduledBatch_ * tokenNumPerIter + tokenNumPerIter` |
| `AddNextTokenPlaceHolder` | [scheduler.cpp:802-825](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp) | 调度出队前给 `outputTokenIds` 追加 `PLACEHOLDER_TOKEN`(=-1)，**ChunkedPrefill 非 last_chunk 跳过** |
| `ReplacePlaceHolderWithToken` | [scheduler.cpp:854-916](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp) | engine 回填后从尾部 `placeholderCount` 个槽位替换为 `predictedTokensBySeqId_` 的真实 token；`placeholderCount < numGenTokens` 或越界即 `throw` |

### vLLM：Python `scheduled_spec_decode_tokens` + AsyncScheduler placeholder

[v1/core/sched/scheduler.py:213-222](d:\design\vllm\vllm\v1\core\sched\scheduler.py) `__init__`：

```python
self.use_eagle = False
self.num_spec_tokens = self.num_lookahead_tokens = 0
if speculative_config:
    self.num_spec_tokens = speculative_config.num_speculative_tokens
    if speculative_config.use_eagle():
        self.use_eagle = True
        self.num_lookahead_tokens = self.num_spec_tokens
    if speculative_config.uses_draft_model():
        self.num_lookahead_tokens = self.num_spec_tokens
```

调度时维护 `scheduled_spec_decode_tokens: dict[str, list[int]]`（[scheduler.py:376, 525-535, 905-924](d:\design\vllm\vllm\v1\core\sched\scheduler.py)），随 `SchedulerOutput` 下发到 worker；preempt 时 `scheduled_spec_decode_tokens.pop(preempted_req_id, None)`（[scheduler.py:486](d:\design\vllm\vllm\v1\core\sched\scheduler.py)）。

异步路径再叠加：[v1/core/sched/async_scheduler.py:15-35](d:\design\vllm\vllm\v1\core\sched\async_scheduler.py)：

```python
self._spec_token_placeholders: list[int] = [-1] * self.num_spec_tokens
...
def _update_after_schedule(self, scheduler_output):
    super()._update_after_schedule(scheduler_output)
    spec_decode_tokens = scheduler_output.scheduled_spec_decode_tokens
    for req_id in scheduler_output.num_scheduled_tokens:
        request = self.requests[req_id]
        if request.is_prefill_chunk: continue
        ...
        cur_num_spec_tokens = len(spec_decode_tokens.get(req_id, ()))
        request.num_output_placeholders += 1 + cur_num_spec_tokens
        request.spec_token_ids = self._spec_token_placeholders   # 共享 read-only -1 list
```

### SGLang：`spec_info` dataclass + scheduler 分派

`ModelWorkerBatch.spec_info`（[spec_info.py:125-167](d:\design\sglang\python\sglang\srt\speculative\spec_info.py)）：5 种类型枚举（`EAGLE_DRAFT` / `EAGLE_VERIFY` / `DFLASH_DRAFT` / `DFLASH_VERIFY` / `NGRAM_VERIFY`）。

scheduler 在 overlap loop 里把 `next_draft_input` 透传到下一 batch（[managers/scheduler.py:2797-2800](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）：

```python
batch.spec_info = batch_result.next_draft_input
batch.spec_info.future_indices = future_indices
```

> synthesis（**Scheduler 接管深度对比**）：
>
> | 维度 | MindIE | vLLM | SGLang |
> |---|---|---|---|
> | 谁负责 KV 槽位预留 | C++ Scheduler 用占位符 | Python Scheduler `num_lookahead_tokens` + worker `extra_slots_per_request` 双层 | spec worker 内 `alloc_token_slots` |
> | 占位形态 | `PLACEHOLDER_TOKEN`(=-1) 写到 `outputTokenIds` 序列里 | `spec_token_ids = [-1, -1, ...]` 共享 list + `num_output_placeholders` int | `EagleDraftInput.hidden_states` + `topk_p` / `topk_index` 张量 |
> | 谁判断接受多少 token | Python Plugin `plugin_verify` | `RejectionSampler` triton kernel + scheduler 减 `num_output_placeholders` | `verify_tree_greedy_func` CUDA kernel + scheduler `next_draft_input.accept_length` |
>
> MindIE 把"占位长度"做到字符串 / int 序列层面（最浅 API），vLLM 把"占位"分到 scheduler（int 计数）+ worker（实张量）两层，SGLang 把"占位"完全做成 dataclass 字段，scheduler 不感知占位长度。

---

## 5. Speculation Gamma / 草稿长度配置

| 项目 | 关键参数 | 校验规则 | 锚点 |
|---|---|---|---|
| MindIE | `speculationGamma` (uint32_t) + plugin JSON | `validation_func_mtp`：`max(num_st, 2*num_st-2) <= gamma` 且 `0 <= num_st <= 5`；`validation_func_la`：`(level-1) * (window+guess_set_size) <= gamma`；memory_decoding：`decoding_length <= gamma` | [plugin_utils.py:46-89](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_utils.py), [config_info.h:118, 282](d:\design\MindIE-LLM\src\include\config\config_info.h) |
| vLLM | `num_speculative_tokens > 0`（Pydantic `Field(default=None, gt=0)`） + 各方法独有参数 | EAGLE 系列：`num_speculative_tokens > 1` 时 `n_predict % num_st == 0` 校验；NGRAM：`prompt_lookup_min/max` 必填 | [config/speculative.py:75-100, 525-606](d:\design\vllm\vllm\config\speculative.py) |
| SGLang | `speculative_num_steps` + `speculative_num_draft_tokens` + `speculative_eagle_topk`；NGRAM 8 个独立参数（`speculative_ngram_min_bfs_breadth` 等） | spec v2：`speculative_eagle_topk == 1`（其它会 raise，[server_args.py:3287-3289](d:\design\sglang\python\sglang\srt\server_args.py)）；`max_running_requests` 默认强制 48 | [server_args.py:497-525, 3245-3295](d:\design\sglang\python\sglang\srt\server_args.py) |

> synthesis：**MindIE 用单个 `gamma` 上限统一约束所有 plugin**（且对 MTP 还限定 `num_st <= 5`），最严格但最简单；vLLM 把约束散在 `SpeculativeConfig.__post_init__`（多模型类型 `n_predict` 校验，[config/speculative.py:578-606](d:\design\vllm\vllm\config\speculative.py)）；SGLang 用 server_args 全局配置 + spec v2 的 `topk == 1` 强约束（[server_args.py:3287-3289](d:\design\sglang\python\sglang\srt\server_args.py)）。

---

## 6. 兼容性矩阵（与 chunked prefill / PD / structured / async 的互斥）

| 特性组合 | MindIE | vLLM | SGLang |
|---|---|---|---|
| spec + **chunked prefill** | C++ scheduler 在非 last_chunk 跳过占位（[scheduler.cpp:806-808](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)）—— **同一请求可同时 chunked prefill + spec**；plugin 文档侧"并行解码与 SplitFuse 互斥"针对 LA/memory（[mindie/topics/speculative.md §三算法对比表](../../mindie/topics/speculative.md)） | scheduler 内 `num_computed_tokens` / `num_tokens_with_spec` 抽象统一处理（[v1/core/sched/scheduler.py:1075-1086](d:\design\vllm\vllm\v1\core\sched\scheduler.py)） | spec v2 + **mixed_chunk** **强制禁用**（EAGLE / EAGLE3 / STANDALONE / DFLASH 都禁），warning："Mixed chunked prefill is disabled because of using ... speculative decoding"（[server_args.py:3256-3260, 3297-3302, 3380-3381](d:\design\sglang\python\sglang\srt\server_args.py)） |
| spec + **PD 分离** | plugin 体系正交于 PD；layerwise PD 用 `maxDispatchBatchNum` 配 placeholder 长度（[mindie/topics/speculative.md](../../mindie/topics/speculative.md)） | spec hidden states 由 connector 自管（[comparison/topics/pd-disaggregation.md §13](pd-disaggregation.md)） | **专门预留 metadata buffer 字段** `output_topk_p` / `output_topk_index` / `output_hidden_states`（[disaggregation/utils.py:182-191](d:\design\sglang\python\sglang\srt\disaggregation\utils.py)） |
| spec + **structured output** | server 端 `mtpEnabled` 与 `response_format` **互斥**（[mindie/topics/speculative.md §配置依赖](../../mindie/topics/speculative.md)） | async path 用 `pending_structured_output_tokens` flag 协调（[async_scheduler.py:26-28](d:\design\vllm\vllm\v1\core\sched\async_scheduler.py)） | spec v2 + grammar + decode + 队列非空 → `is_disable_overlap_for_batch` **强制禁 overlap**，注释 "we do not support overlap + spec + grammar yet"（[managers/scheduler.py:1486-1495](d:\design\sglang\python\sglang\srt\managers\scheduler.py)） |
| spec + **async / overlap scheduler** | `tokenNumPerIter * maxScheduledBatch_` 占位长度自动适配（[scheduler.cpp:769-771, 881-883](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)） | 仅 `EagleModelTypes` / `NgramGPUTypes` 允许 async；`Medusa` / `draft_model` / `suffix` 触发 auto-disable async（[config/vllm.py:801-812](d:\design\vllm\vllm\config\vllm.py)） | EAGLE / EAGLE3 / STANDALONE + `SGLANG_ENABLE_SPEC_V2=True` → overlap 自动开；DFLASH / NGRAM 强制关 overlap（[server_args.py:3251-3253, 3275-3295, 3380](d:\design\sglang\python\sglang\srt\server_args.py)） |
| spec + **dp_attention** | 通过 `parallel_info_manager` 处理；详情待 ingest | 通过 `dp_size` / `dp_rank` 字段统一处理 | STANDALONE + `enable_dp_attention` **直接 raise**（[server_args.py:3263-3267](d:\design\sglang\python\sglang\srt\server_args.py)） |
| spec + **multi-modal** | 当前 MTP 绑定 DeepSeek，非多模态路径 | EAGLE 路径检查 `supports_multimodal`（[v1/spec_decode/eagle.py:115-119](d:\design\vllm\vllm\v1\spec_decode\eagle.py)） | EAGLE3 通过 `eagle_use_aux_hidden_state`（[eagle_worker_v2.py:156-163](d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py)） |

> synthesis：**SGLang 兼容性表是三家最多硬约束**（多个 force-disable + raise），表明它对"已知 buggy 组合"采取拒绝策略；vLLM 用 auto-disable 软降级；MindIE 用 plugin 白名单 + JSON 校验 + 服务端 `mtpEnabled` 检查。这反映了三家"**新功能默认是否 fail-fast**"的工程哲学差异。

---

## 7. KV 槽位管理（draft token 占多少 KV）

| 项目 | 槽位预留方式 | 占位语义 | 对账机制 |
|---|---|---|---|
| MindIE | C++ Scheduler `maxPlaceHolderNum = maxScheduledBatch_ * tokenNumPerIter + tokenNumPerIter`（[scheduler.cpp:769-771](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)） | `outputTokenIds.push_back(PLACEHOLDER_TOKEN)` 字面写 -1 到 token 序列尾 | `ReplacePlaceHolderWithToken` 用 `predictedTokensBySeqId_` 替换；`placeholderCount < numGenTokens` 即 `throw runtime_error`（[scheduler.cpp:882-901](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)） |
| vLLM | `extra_slots_per_request = 1 if not parallel_drafting else num_speculative_tokens`；`net_num_new_slots_per_request = extra_slots - (1 if pass_hidden_states_to_model and method != "dflash" else 0)`（[v1/spec_decode/eagle.py:91-97](d:\design\vllm\vllm\v1\spec_decode\eagle.py)） | worker 端 `needs_extra_input_slots` flag；`compute_new_slot_mapping` triton kernel 计算 slot mapping（[eagle.py:185-192, 656, 690-739](d:\design\vllm\vllm\v1\spec_decode\eagle.py)） | scheduler `_update_request_with_output` 减 `num_output_placeholders -= len(new_token_ids)` 并 assert ≥ 0（[async_scheduler.py:51-53](d:\design\vllm\vllm\v1\core\sched\async_scheduler.py)） |
| SGLang | `EagleDraftInput.ALLOC_LEN_PER_DECODE = max(num_steps*topk, num_draft_tokens)`（[standalone_worker_v2.py:73-75](d:\design\sglang\python\sglang\srt\speculative\standalone_worker_v2.py)）；运行时 `alloc_paged_token_slots_extend` / `alloc_token_slots`（[eagle_info_v2.py:20-21, 137-146](d:\design\sglang\python\sglang\srt\speculative\eagle_info_v2.py)） | 不写"占位 token id"，而是用 `EagleDraftInput.topk_p` / `topk_index` / `hidden_states` 张量直接持有 draft 状态 | scheduler 在下一 batch 用 `accept_length` 收回（[eagle_worker_v2.py:578-579, 600-621](d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py)） |

> synthesis（**三种"占位"哲学**）：
>
> - **MindIE**：占位 = "把 -1 写进 token 序列"，最具体、最易调试，但要求每个消费 `outputTokenIds` 的下游（如 detokenize / metrics）都能识别 -1 并跳过。
> - **vLLM**：占位 = "scheduler 维护 int 计数 + worker 维护 slot mapping"，**两层职责分离**；async path 共享同一 `[-1] * num_spec_tokens` 列表避免每 step 创建。
> - **SGLang**：占位 = "用 dataclass 字段直接持 draft state（topk_p / hidden_states 张量）"，**不需要占位 token**；下一 batch 直接消费 `next_draft_input`。
>
> 对应到错误模式：MindIE 的 `placeholderCount` 和 `numGenTokens` 不一致就 throw（fail-fast）；vLLM `assert num_output_placeholders >= 0`；SGLang 没有显式对账，依赖 dataclass 不变量。

---

## 8. Anchor-driven cross-check（mandatory per [§8 rule 6](../../AGENTS.md)）

> 选 MindIE 的 2 个具体 anchor，去 vLLM / SGLang 全仓库扫等价 pattern。

### Anchor #1：MindIE `PLACEHOLDER_TOKEN` (-1) 写入 token 序列尾的占位机制

> 源：[scheduler.cpp:802-825](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp) `AddNextTokenPlaceHolder` + [scheduler.cpp:854-916](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp) `ReplacePlaceHolderWithToken`。

**vLLM 反向扫**：

| 模式 | 范围 | 命中 |
|---|---|---|
| `PLACEHOLDER_TOKEN` / `placeholder` (spec 上下文) | `d:\design\vllm\vllm\v1\spec_decode\` | **0 命中**（`spec_decode/eagle.py:1341` 仅命中 `media_placeholder_token_id`，多模态语义无关） |
| `_spec_token_placeholders` / `num_output_placeholders` | `d:\design\vllm\vllm\v1\` | **5 文件命中**：`structured_output/__init__.py`、`simple_kv_offload/manager.py`、`request.py`、`outputs.py`、`core/sched/scheduler.py`、`core/sched/async_scheduler.py`、`sample/rejection_sampler.py` |
| `[-1] * num_spec_tokens` | `d:\design\vllm\vllm\v1\core\sched\async_scheduler.py:16` | **真等价物**：`self._spec_token_placeholders: list[int] = [-1] * self.num_spec_tokens` —— **共享 read-only `-1` 列表 + 每 req `num_output_placeholders` int 计数**（[async_scheduler.py:16, 32-35, 51-58](d:\design\vllm\vllm\v1\core\sched\async_scheduler.py)） |

**结果**：vLLM **有等价 placeholder 机制**，但实现在 **`AsyncScheduler`（async path 专属）**，不像 MindIE 那样写到字面 token 序列里。位于 scheduler 层，由 `request.num_output_placeholders` int 计数 + `request.spec_token_ids` 共享指向 `[-1, -1, ...]` list。

**SGLang 反向扫**：

| 模式 | 范围 | 命中 |
|---|---|---|
| `PLACEHOLDER` / `placeholder` (spec 上下文) | `d:\design\sglang\python\sglang\srt\speculative\` | **0 命中** |
| `placeholder` 全 srt | `d:\design\sglang\python\sglang\srt\managers\scheduler.py` | **1 命中**：[scheduler.py:3551](d:\design\sglang\python\sglang\srt\managers\scheduler.py) `# placeholder for override`（与 spec 无关，placeholder 函数桩） |
| `accept_length` / `accept_lens` | `d:\design\sglang\python\sglang\srt\speculative\` | 多次命中 `eagle_worker_v2.py:558-561, 578-579`、`eagle_info_v2.py` —— **真等价物（按 accept_length 在 verify 后回收 KV）** |

**结果**：SGLang **N/A**：在 `d:\design\sglang\python\sglang\srt\speculative\` 全树 grep `placeholder` 0 命中。**等价机制**是 `EagleDraftInput`（dataclass 持 draft 张量）+ `accept_length` 字段在下一 batch `_draft_extend_for_decode` 里收回（[eagle_worker_v2.py:547-622](d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py)）。**核心差异**：SGLang 不通过"占位 token id"实现 KV 槽预留，而是在 verify 后才把 accepted token 写入 token 序列。

### Anchor #2：MindIE `MtpWorker` 双 ModelRunner（main + draft）

> 源：[spec_worker.py:43-64](d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\spec_worker.py)。

**vLLM 反向扫**：

| 模式 | 范围 | 命中 |
|---|---|---|
| `draft_model_runner` / `MtpWorker` 等价命名 | `d:\design\vllm\vllm\v1\` | **N/A**（vLLM 无双 runner 概念；用 `SpecDecodeBaseProposer.load_model` + `get_model(...)` 加载 draft，与主模型共用 `GPUModelRunner`） |
| `draft_model_config` | `d:\design\vllm\vllm\v1\spec_decode\` | **多文件命中**：`eagle.py:71`、`draft_model.py`、`medusa.py:36`、`speculator.py:46`（**真等价物的"配置层"**：每个 proposer 持自己的 `draft_model_config`） |
| `EagleSpeculator.__init__` 独立 InputBuffer / BlockTables | [gpu/spec_decode/eagle/speculator.py:63-90](d:\design\vllm\vllm\v1\worker\gpu\spec_decode\eagle\speculator.py) | **真等价物**：EAGLE 路径有**独立 input_buffers + block_tables + idx_mapping + temperature + seeds + draft_tokens 张量** —— 等价于"独立 ModelRunner 状态"，但仍跑在主 worker 进程内 |

**结果**：vLLM **没有"双 ModelRunner"** 但**有"双状态对象"**——`EagleSpeculator` 持自己的 `InputBuffers` / `BlockTables` / `idx_mapping`，相当于把 ModelRunner 的核心状态拆出一份给 draft 用，而非整个 ModelRunner 复制。

**SGLang 反向扫**：

| 模式 | 范围 | 命中 |
|---|---|---|
| `is_draft_worker` | `d:\design\sglang\python\sglang\srt\speculative\` | **多次命中**：[eagle_worker_v2.py:148](d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py)、[standalone_worker_v2.py:99](d:\design\sglang\python\sglang\srt\speculative\standalone_worker_v2.py) `TpModelWorker(..., is_draft_worker=True, ...)` —— **真等价物（独立的第二个 TpModelWorker，和 MindIE 双 ModelRunner 最接近）** |
| `target_worker` / `draft_worker` | `d:\design\sglang\python\sglang\srt\speculative\` | 多次命中：`base_spec_worker.py:23-29`（抽象） + `EAGLEWorkerV2._target_worker / _draft_worker`（[eagle_worker_v2.py:679-684](d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py)） |
| `draft_runner` | `d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py` | [eagle_worker_v2.py:155, 168, 172, 247, 447, 534, 591](d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py) `self.draft_runner = self.draft_worker.model_runner` —— **正是 MindIE `draft_model_runner` 的对应物** |

**结果**：SGLang **有真等价物 + 命名几乎一致**：`EagleDraftWorker` 内有独立 `TpModelWorker(is_draft_worker=True)` + `draft_runner = self.draft_worker.model_runner`，与 MindIE `MtpWorker.draft_model_runner` 语义完全等价。**关键差异**：SGLang 共享 `req_to_token_pool` 与 `token_to_kv_pool_allocator`（`get_memory_pool()` 返回，[eagle_worker_v2.py:125-127](d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py)）；MindIE 当前实现待 ingest 验证 KV pool 共享细节。

### Anchor-driven cross-check 综合补漏入对比表

> 上述 cross-check 没有发现"一家有、另两家漏掉"的论断（三家在"placeholder/双状态"两条都有等价物），但**实现层级不同**：vLLM placeholder 在 scheduler 层（int 计数），SGLang 没有"占位 token id"而用 dataclass 持张量；vLLM "双状态"在 worker 内一个进程，SGLang 是真"双 worker"。这些差异已在 §4 / §7 的对照表中 explicit。**N/A 已在表里逐项标注，配 verified 日期 2026-04-18。**

---

## 9. 与 PD 优化的关联（synthesis）

> 注意：本节是综合性建议，不是任何一方源码原文。

### Spec + PD：三家关键差异对你的影响

| 场景 | 三家行为 | 对 PD 部署的影响 |
|---|---|---|
| **D 节点用 spec decode** | MindIE：`speculationGamma > 0` 时 D 节点 placeholder 长度自动适配；vLLM：D 节点跑 EAGLE/draft，与 connector 透传 hidden state；SGLang：disagg metadata buffer 专门预留 `output_topk_p` 等字段（[disaggregation/utils.py:182-191](d:\design\sglang\python\sglang\srt\disaggregation\utils.py)） | SGLang 把 spec hidden state 直接编码进 KV 传输 metadata，**对长上下文 + spec 场景吞吐最好**；MindIE 和 vLLM 都需要 P → D 端额外传 hidden states |
| **P 节点是否可以做 spec** | 三家都允许，但 P 阶段 spec accept rate 较低（通常 spec 在 decode 阶段更有价值） | 在大部分 PD 部署中，P 节点关 spec、D 节点开 spec 是合理选择 |
| **layerwise PD + spec** | MindIE `lwd_self_attn_block_manager` 注释明示**不支持 prefix caching**（[lwd.h:27](d:\design\MindIE-LLM\src\block_manager\lwd_self_attn_block_manager.h)）；spec 兼容性 **未在源码层显式禁用**，但与 layerwise PD 的双 manager Allocate 短板效应叠加可能有边角 | 如果你的 layerwise PD 路径打算开 MTP，先验证 `maxDispatchBatchNum * tokenNumPerIter` 是否被云端 manager 接受 |
| **spec + chunked prefill + PD** | MindIE：spec 占位在 last_chunk 才追加（[scheduler.cpp:806-808](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)）；SGLang：mixed_chunk 强制关闭（[server_args.py:3256-3260](d:\design\sglang\python\sglang\srt\server_args.py)） | 如果 PD 部署用 chunked prefill 切大请求，**MindIE 路径最自然**（splitfuse plugin 与 spec 正交）；SGLang 用户必须二选一 |

### 想从 vLLM / SGLang 借什么到 MindIE？

| 借鉴点 | 来源 | 借鉴价值 |
|---|---|---|
| **Tree verify CUDA kernel** | SGLang `verify_tree_greedy_func` → `sgl_kernel.verify_tree_greedy`（[eagle_utils.py:161-199](d:\design\sglang\python\sglang\srt\speculative\eagle_utils.py)） | MindIE 当前 `verify_greedy_one_batch` 是 Python 串行循环（[mtp/decoding_policy.py:60-69](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\mtp\decoding_policy.py)）；如果未来需要 tree-style spec（多分支并行 verify），需要 NPU 等价 kernel |
| **统一 RejectionSampler 三模式** | vLLM `RejectionSampler` strict / probabilistic / synthetic（[rejection_sampler.py:100-235](d:\design\vllm\vllm\v1\worker\gpu\spec_decode\rejection_sampler.py)） | MindIE 当前仅支持贪婪 verify；要支持非 greedy 采样下的 spec accept，需要 probabilistic rejection sampler |
| **proposer 通用基类** | vLLM `SpecDecodeBaseProposer`（[v1/spec_decode/eagle.py:60](d:\design\vllm\vllm\v1\spec_decode\eagle.py)） | MindIE 三 plugin 当前各写各的 `model_inputs_update` / `sample_preprocess` / `plugin_verify`，缺统一基类约束；如果未来加新算法（如 EAGLE）建议先抽象 base proposer |
| **spec_info dataclass 化** | SGLang `SpecInput` 抽象 + `EagleDraftInput / EagleVerifyInput / NgramVerifyInput`（[spec_info.py:135-167](d:\design\sglang\python\sglang\srt\speculative\spec_info.py)） | MindIE 当前 spec 状态散在 `PluginDataParam` 各字段，类型不显式；dataclass 化可提升 IDE 支持与跨进程序列化 |

### 想从 MindIE 借什么到另两家？

> synthesis：MindIE 的 **C++ Scheduler 显式占位机制**（`AddNextTokenPlaceHolder` + `ReplacePlaceHolderWithToken`）是非常清晰的"调度器 / 引擎契约"——调度器先在 KV 上 reserve，引擎之后 commit。vLLM `num_output_placeholders` int 计数与 SGLang `accept_length` 都是这个契约的不同实现。MindIE 把"占位 token 字面值"暴露到 outputTokenIds 序列里，对 detokenize / metrics 模块要求一致跳过 `-1`，但调试性最强（一眼能看出每个 req 当前有几个未确定的 token）。

---

## Notes / Caveats

> **NOTE 2026-04-19 lint fix**: 24→27 SGLang / 12→11 vLLM stale numbers + EAGLE-only TpModelWorker precision（详见 footnote `[^tpw-precision]`，table L82）。SGLang `srt/speculative/` 真实 .py 文件数 = **27**（递归含 `cpp_ngram/` 子目录）；vLLM `v1/spec_decode/` 真实业务 .py = **11**（不含 `__init__.py`）。"独立 TpModelWorker" 措辞补 EAGLE 系限定，区分 DFlash/NGRAM 的 duck-typed 包装路径。新增 `sglang/topics/speculative.md` 关联。

> **Verify pass 2026-04-18 摘要**：本页 5 个 markers（**2 CONTRADICTION + 5 VERIFY**，VERIFY 1 与 VERIFY 5 合并为单条；详 [log.md 2026-04-18 ingest comparison/topics/speculative-decoding 条目](../../log.md)）**全部 RESOLVED**——3 个借力新建的 [vllm/topics/spec-decode-eagle.md](../../vllm/topics/spec-decode-eagle.md) 直接关联，2 个通过复读 SGLang `eagle_worker_v2.py` + MindIE `spec_worker.py` 源码补 anchor 解。**仍有 1 个跨页待跟进**：MindIE 端 `is_draft_model=True` 的 KV pool 是否与主模型 alias 同一池（VERIFY 2 末尾备注），留作下一轮 `mindie/topics/speculative.md` ingest 跟进。

> ~~[!warning] CONTRADICTION: vLLM 的 `EagleSpeculator`（[gpu/spec_decode/eagle/speculator.py:37](d:\design\vllm\vllm\v1\worker\gpu\spec_decode\eagle\speculator.py)）与 `SpecDecodeBaseProposer`（[v1/spec_decode/eagle.py:60](d:\design\vllm\vllm\v1\spec_decode\eagle.py)）**两个 EAGLE 实现并存**——前者是 v1 worker GPU 路径独立 speculator（持独立 InputBuffer / BlockTables），后者是通用 proposer。实际选哪个取决于 worker 类型，本页未深入区分；**新读者注意不要混淆**。~~ **RESOLVED 2026-04-18**：vLLM v1 内部确实有"双 EAGLE GPU 实现"并存，并非本页错描，而是 vLLM 当前的 ModelRunner 重构未完。详见 [vllm/topics/spec-decode-eagle.md §4 EAGLE 双路径](../../vllm/topics/spec-decode-eagle.md)：(a) 老路径 `EagleProposer(SpecDecodeBaseProposer)`（[v1/spec_decode/eagle.py:1735-1747](d:\design\vllm\vllm\v1\spec_decode\eagle.py)），由老 `gpu_model_runner.py:561` 实例化，复用 `GPUModelRunner` 的 input/positions/hidden_states buffer，**支持 tree spec + parallel drafting + 7 method 通用 + EAGLE3 aux hidden state**；(b) 新路径 `EagleSpeculator`（[gpu/spec_decode/eagle/speculator.py:37-561](d:\design\vllm\vllm\v1\worker\gpu\spec_decode\eagle\speculator.py)），由新 `v1/worker/gpu/model_runner.py:172` 通过 `init_speculator()` 工厂创建（[__init__.py:8-15](d:\design\vllm\vllm\v1\worker\gpu\spec_decode\__init__.py)），独立 `InputBuffers` + 独立 `BlockTables` + 独立两个 `EagleCudaGraphManager`（prefill / decode），**仅支持链式 spec（无 tree）+ 仅 4 method `use_eagle()` = `eagle/eagle3/mtp/dflash`**。**选择由 ModelRunner 类型决定**（仍待官方 model_runner_v2 文档完整确认）。
> ~~[!warning] CONTRADICTION: SGLang `disable_overlap_schedule` 在 spec 配置中**多处自动开关**：默认 False → 但 SPEC_V2 disabled 时被设为 True → SPEC_V2 enabled 时设回 False（[server_args.py:3275-3295](d:\design\sglang\python\sglang\srt\server_args.py)）。**最终值取决于 env `SGLANG_ENABLE_SPEC_V2`**，不是命令行参数。~~ **RESOLVED 2026-04-18**：源码确认这不是矛盾，而是 **3 段顺序逻辑的有意 cascade**（[server_args.py:3251-3295, 3380](d:\design\sglang\python\sglang\srt\server_args.py)）：(a) DFLASH / NGRAM 算法路径**强制** disable overlap（早段，3251-3253 + 3380）；(b) EAGLE / EAGLE3 / STANDALONE + `SGLANG_ENABLE_SPEC_V2=False` 时**自动** disable overlap（spec v1 路径）；(c) EAGLE / EAGLE3 / STANDALONE + `SGLANG_ENABLE_SPEC_V2=True` 时**重新启用** overlap（spec v2 路径，与 overlap scheduler 强耦合）。**结论**：`disable_overlap_schedule` 最终值是**算法 × env**联合决定的派生量，不是矛盾。**用户视角的指引**：要 spec v2 + overlap，必须显式设 `SGLANG_ENABLE_SPEC_V2=1` 且不能用 DFLASH/NGRAM 算法。
> ~~[!todo] VERIFY: vLLM `SpeculativeMethod` 字面量包含 `dflash` 出现在 `EagleModelTypes` 内（[config/speculative.py:53-55](d:\design\vllm\vllm\config\speculative.py)），但 `DFlashProposer` 是独立类（[dflash.py:20](d:\design\vllm\vllm\v1\spec_decode\dflash.py)）；method 命名与类对应关系待 verify pass 详读 model_loader 路径。~~ **RESOLVED 2026-04-18**：详见 [vllm/topics/spec-decode-eagle.md §2 SpeculativeMethod 完整枚举](../../vllm/topics/spec-decode-eagle.md)。**字面归类与类映射澄清**：(a) `EagleModelTypes = Literal["eagle", "eagle3", "extract_hidden_states", MTPModelTypes, DFlashModelTypes]`（[speculative.py:53-55](d:\design\vllm\vllm\config\speculative.py)），所以 `dflash` **字面在 `EagleModelTypes` 集合内**；(b) `SpeculativeConfig.use_eagle()` 返回 `self.method in ("eagle", "eagle3", "mtp", "dflash")`（[speculative.py:866-869](d:\design\vllm\vllm\config\speculative.py)），把 dflash **当作 EAGLE 通路对待**；(c) 但实例化的 proposer 类是独立的 `DFlashProposer(SpecDecodeBaseProposer)`（[dflash.py:20-282](d:\design\vllm\vllm\v1\spec_decode\dflash.py)），override `set_inputs_first_pass` + 强制 `parallel_drafting=True` + 跨注意力（非因果） + override `_raise_if_multimodal` 允许多模态。**完整 method ↔ class 映射**：见 [vllm/topics/spec-decode-eagle.md §2 synthesis 表](../../vllm/topics/spec-decode-eagle.md)（7 种实现 + 1 个 `mlp_speculator` 字面但 v1 无 proposer 的 gap）。**入口选择**：[gpu_model_runner.py:528-579](d:\design\vllm\vllm\v1\worker\gpu_model_runner.py) 的 if/elif 链按 method 分流，dflash 专用分支在 L555-557；新路径 `init_speculator()` 仅返回 `EagleSpeculator`。
> ~~[!todo] VERIFY: SGLang `EagleDraftWorker` 与 target worker 共享 `req_to_token_pool` / `token_to_kv_pool_allocator` 但 KV pool 独立（[eagle_worker_v2.py:124-127, 149-151](d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py)）—— 实际 alloc/free 一致性如何保证未深读，可能存在边角 race。~~ **RESOLVED 2026-04-18**：源码（[eagle_worker_v2.py:123-152](d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py) + [tp_worker.py:340-361](d:\design\sglang\python\sglang\srt\managers\tp_worker.py)）确认**一致性由"单 allocator + 单 req_to_token_pool"两个共享对象保证**：(a) `target_worker.get_memory_pool()` 返回 `(req_to_token_pool, token_to_kv_pool_allocator)` 二元组（[eagle_worker_v2.py:125-127](d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py)）；(b) draft 端 `TpModelWorker(is_draft_worker=True, req_to_token_pool=..., token_to_kv_pool_allocator=..., memory_pool_config=target_worker.model_runner.memory_pool_config)`（[eagle_worker_v2.py:138-152](d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py)）；(c) `TpModelWorker._init_model_runner` **直接透传**这俩到 `ModelRunner(req_to_token_pool=self.req_to_token_pool, token_to_kv_pool_allocator=self.token_to_kv_pool_allocator, memory_pool_config=self.memory_pool_config, ...)`（[tp_worker.py:340-361](d:\design\sglang\python\sglang\srt\managers\tp_worker.py)）。**意义**：(i) 单 allocator 防双重分配（无 race）；(ii) 单 req_to_token_pool 让两 worker 看到同一份 req→slot 映射（draft 改的 slot，target 直接可见）；(iii) draft 与 target 各自的 ModelRunner 仍**独立 load 自己的 KV cache 张量层**（不同模型权重对应不同 layer 数 / hidden dim），但**索引到同一份 slot id 空间**。**核心差异 vs MindIE 双 ModelRunner**：MindIE 当前未在源码层 expose"共享 allocator"——本批次未深 ingest MindIE 端 `is_draft_model=True` 是否同样 alias 同一 KV pool（详 [mindie/topics/speculative.md §SpecWorker](../../mindie/topics/speculative.md)），留作下一轮 mindie ingest 跟进。
> ~~[!todo] VERIFY: MindIE `MtpWorker` 与 `MtpWorkerExp`（实验路径）在 `ENV.model_runner_exp` 控制下分流（[spec_worker.py:693-707](d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\spec_worker.py)），两者签名差异（`npu_cache` / `forward_context` / `build_forward_context`，[spec_worker.py:323-659](d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\spec_worker.py)）的实质语义差未在本轮 ingest 范围内详细解释。~~ **RESOLVED 2026-04-18**：复读 [spec_worker.py:23-707](d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\spec_worker.py) 后核心结论：**两类核心 dual-ModelRunner 模式完全相同**——`main_model_runner = create_main_model_runner(target_cls, ...)` + `draft_model_runner = create_draft_model_runner(target_cls, is_draft_model=True, ...)`（[spec_worker.py:43-64](d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\spec_worker.py) vs [spec_worker.py:323-355](d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\spec_worker.py)）。**关键差异**：(a) `MtpWorkerExp.__init__` 额外初始化 `_offsets`（NPU pre-allocated `arange` tensor）+ `mapping`（`get_parallel_info_manager()` 句柄）+ `is_dp_and_server_centralized` flag（[spec_worker.py:330-343](d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\spec_worker.py)）；(b) `MtpWorkerExp` 多 4 个方法（`compile()` / `set_eager_mode_with_padding()` / 等）以支持 **`aclgraph` capture + eager-padded mode**；(c) `MtpWorkerExp.forward_*` 方法签名接受 `npu_cache` / `forward_context` / `build_forward_context` 三参数，做 **sub_context 合并**（[spec_worker.py:323-659](d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\spec_worker.py)）以让 main + draft 共用一套 forward_context（避免每次 build 两次）。**结论**：`MtpWorkerExp` = `MtpWorker` + aclgraph/compile 支持 + sub_context 合并 + DP 集中模式适配；功能上**包含 `MtpWorker` 全部能力**（superset）。**选择**：[spec_worker.py:703-707](d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\spec_worker.py) `if ENV.model_runner_exp and num_speculative_tokens > 0: return MtpWorkerExp` —— env 显式开启 exp runner 时切到 Exp，否则走基础版。两版可视为 "**legacy MtpWorker**" + "**aclgraph-aware MtpWorkerExp**"。详细仍可补入 [mindie/topics/speculative.md §SpecWorker](../../mindie/topics/speculative.md) 下一轮 ingest。
> ~~[!todo] VERIFY: vLLM `Scheduler.use_eagle` flag（[v1/core/sched/scheduler.py:218](d:\design\vllm\vllm\v1\core\sched\scheduler.py)）会触发哪些独立分支，本页未列尽。~~ **RESOLVED 2026-04-18**：grep 全文确认 `Scheduler.use_eagle` 在 [v1/core/sched/scheduler.py](d:\design\vllm\vllm\v1\core\sched\scheduler.py) 共 **5 处** 使用 + 2 处独立分支：(a) **L214** init `False`；(b) **L218-219** `if speculative_config.use_eagle(): self.use_eagle = True`；(c) **L229** 透传给 `KVCacheManager(..., use_eagle=self.use_eagle, ...)`；(d) **L330-331** chunked prefill **`last_cache_position` 减 1 个 block**（"eagle prune"，避免 EAGLE 末尾 block 误命中导致 cache miss）；(e) **L434, L703** `shift_computed_tokens=1 if self.use_eagle else 0`（KV cache 查找时偏移 1 token，因 EAGLE draft 占 1 个 lookahead slot）。**额外配套**：L218-222 同段还设 `num_lookahead_tokens = num_spec_tokens`（`use_eagle()` 或 `uses_draft_model()`）。**全部下游影响**已锁定到这 5 处 + KVCacheManager 内的 use_eagle 分支（详 [vllm/topics/spec-decode-eagle.md §5.1 第一层 scheduler 端 placeholder 机制](../../vllm/topics/spec-decode-eagle.md)）。

## See also

- [comparison/dimensions.md §dim-spec](../dimensions.md)
- [sglang/topics/speculative.md](../../sglang/topics/speculative.md)（**SGLang 全景**：5 enum × spec_v2/v1 worker 体系 + `srt/speculative/` 27 .py 拆解；EAGLE 系 vs DFlash/NGRAM duck-typed 路径区分）
- [mindie/topics/speculative.md](../../mindie/topics/speculative.md)（MindIE 全景：3 plugin + MtpWorker + C++ placeholder）
- [vllm/topics/spec-decode-eagle.md](../../vllm/topics/spec-decode-eagle.md)（**vLLM 全景**：`v1/spec_decode/` 11 .py（不含 `__init__.py`）文件分类、`SpecDecodeBaseProposer` 类层次、`SpeculativeMethod` Literal 完整枚举与 `mlp_speculator` 缺失矛盾、双 EAGLE 路径（`EagleProposer` vs `EagleSpeculator`）并存、3 模式 `RejectionSampler` 双套并存、ngram CPU/GPU 双实现差异、`extract_hidden_states` 是"伪 spec"用于 PD KV 写入）
- [comparison/topics/async-schedule.md §7 spec decode + structured output 三家协议对照](async-schedule.md)（互锁）
- [comparison/topics/sync-schedule.md](sync-schedule.md)
- [comparison/topics/scheduler.md](scheduler.md)
- [comparison/topics/pd-disaggregation.md §9 Hybrid / 多 KV 组](pd-disaggregation.md)（spec + PD 的 metadata buffer）
- [comparison/topics/pd-disaggregation.md §13 与 chunked prefill / spec decoding 的协同](pd-disaggregation.md)
- [mindie/entities/PluginManager.md](../../mindie/entities/PluginManager.md)（`mtp_model_inputs` 传递 + plugin_verify 调用入口）
- [mindie/entities/BatchScheduler.md](../../mindie/entities/BatchScheduler.md)（`speculationGamma` / `tokenNumPerIter` 占位公式）
- [mindie/entities/ModelRunner.md](../../mindie/entities/ModelRunner.md)（`MtpWorker` 装饰器位点）
