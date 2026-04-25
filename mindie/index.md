---
type: index
project: mindie
status: verified
confidence: high
verified_against: 2026-04-18
sources:
  - d:\design\MindIE-LLM\mindie_llm
  - d:\design\MindIE-LLM\src
  - d:\design\MindIE-LLM\docs
related:
  - ../index.md
  - overview.md
  - ../comparison/index.md
---

# MindIE-LLM Index

> 项目内所有 wiki 页的目录。
> 入口架构鸟瞰：[overview.md](overview.md)。
> 主代码根：[d:\design\MindIE-LLM\mindie_llm\](d:\design\MindIE-LLM\mindie_llm)。

---

## Modules（一级代码模块概念页）

按 [overview.md](overview.md) §Top-level layout 列出，待 ingest 填充。

| 模块 | wiki 页 | 状态 |
|---|---|---|
| `connector` | [modules/connector.md](modules/connector.md) | **DONE** |
| `server` | `modules/server.md` | TODO |
| `text_generator` | [modules/text_generator.md](modules/text_generator.md) | **DONE** |
| `runtime` | `modules/runtime.md` | TODO |
| `runtime.model_runner` | [modules/runtime_model_runner.md](modules/runtime_model_runner.md) | **DONE** |
| `runtime.models` | `modules/runtime_models.md` | TODO |
| `runtime.layers` | `modules/runtime_layers.md` | TODO |
| `runtime.ops` | `modules/runtime_ops.md` | TODO |
| `runtime.compilation` | `modules/runtime_compilation.md` | TODO |
| `runtime.utils.distributed` | `modules/distributed.md` | TODO |
| `runtime.lora` | `modules/lora.md` | TODO |
| `modeling` | `modules/modeling.md` | TODO |
| `tokenizer` | `modules/tokenizer.md` | TODO |
| `utils` | `modules/utils.md` | TODO |

---

## Entities（关键 class / 函数）

按层次分组：

### Python 顶层（生成栈）

| 实体 | 源文件 | wiki 页 | 状态 |
|---|---|---|---|
| `Generator` | [generator.py](d:\design\MindIE-LLM\mindie_llm\text_generator\generator.py) | [entities/Generator.md](entities/Generator.md) | **DONE** |
| `PluginManager` | [plugin_manager.py](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py) | [entities/PluginManager.md](entities/PluginManager.md) | **DONE** |
| `SeparateDeploymentEngine` | [separate_deployment_engine.py](d:\design\MindIE-LLM\mindie_llm\text_generator\utils\separate_deployment_engine.py) | [entities/SeparateDeploymentEngine.md](entities/SeparateDeploymentEngine.md) | **DONE** |

### Python runtime（模型执行）

| 实体 | 源文件 | wiki 页 | 状态 |
|---|---|---|---|
| `ModelRunner` | [model_runner.py](d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\model_runner.py) | [entities/ModelRunner.md](entities/ModelRunner.md) | **DONE** |
| `ModelRunnerExp` | [model_runner_exp.py](d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\model_runner_exp.py) | `entities/ModelRunnerExp.md` | TODO |
| `SpecWorker` | [spec_worker.py](d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\spec_worker.py) | `entities/SpecWorker.md` | TODO |
| `AclGraphModelWrapper` | [aclgraph_model_wrapper.py](d:\design\MindIE-LLM\mindie_llm\modeling\model_wrapper\aclgraph\aclgraph_model_wrapper.py) | [entities/AclGraphModelWrapper.md](entities/AclGraphModelWrapper.md) | **DONE**（含 Exp 对照） |
| `AclGraphModelWrapperExp` | [aclgraph_model_wrapper_exp.py](d:\design\MindIE-LLM\mindie_llm\modeling\model_wrapper\aclgraph\aclgraph_model_wrapper_exp.py) | → [entities/AclGraphModelWrapper.md](entities/AclGraphModelWrapper.md)（同页 §AclGraphModelWrapperExp） | **DONE** |
| `ParallelInfoManager` | [parallel_info_manager.py](d:\design\MindIE-LLM\mindie_llm\runtime\utils\distributed\parallel_info_manager.py) | [entities/ParallelInfoManager.md](entities/ParallelInfoManager.md) | **DONE** |
| `PipelineParallel` | [pipeline_parallel.py](d:\design\MindIE-LLM\mindie_llm\runtime\utils\distributed\pipeline_parallel.py) | `entities/PipelineParallel.md` | TODO（草稿，未接入主链路；详 [topics/aclgraph-pp.md](topics/aclgraph-pp.md)） |
| `Qwen3MoE` | [qwen3_moe.py](d:\design\MindIE-LLM\mindie_llm\runtime\models\qwen3_moe\qwen3_moe.py) | `entities/Qwen3MoE.md` | TODO |

### C++ 引擎层

| 实体 | 源文件 | wiki 页 | 状态 |
|---|---|---|---|
| `LlmEngine` (C++) | [src/engine/llm_engine.cpp](d:\design\MindIE-LLM\src\engine\llm_engine.cpp), [llm_engine.h](d:\design\MindIE-LLM\src\engine\llm_engine.h) | [entities/LlmEngine.md](entities/LlmEngine.md) | **DONE** |
| `BatchScheduler` (C++) | [src/scheduler/scheduler.h](d:\design\MindIE-LLM\src\scheduler\scheduler.h) | [entities/BatchScheduler.md](entities/BatchScheduler.md) | **DONE** |
| `BlockSpaceManager` (C++) | [src/include/block_manager/block_manager_interface.h](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | [entities/BlockSpaceManager.md](entities/BlockSpaceManager.md) | **DONE**（含 5 实现 vs 实际 2 实现 CONTRADICTION） |
| `LayerwiseSelfAttnBlockManager` (C++) | [lwd_self_attn_block_manager.h](d:\design\MindIE-LLM\src\block_manager\lwd_self_attn_block_manager.h) | `entities/LayerwiseSelfAttnBlockManager.md` | TODO（P2，layerwise PD 独有） |

### 其它候选（来自 anchor 红利分析，按需建）

| 实体 | 源文件 | 优先级 | 触发时机 |
|---|---|---|---|
| `MooncakeMempool` | [mooncake_mempool.py](d:\design\MindIE-LLM\mindie_llm\text_generator\mempool\mooncake_mempool.py) | P2 | KV transfer 优化时 |
| `RouterImpl` | [router_impl.py](d:\design\MindIE-LLM\mindie_llm\connector\request_router\router_impl.py) | P2 | connector 层调度优化时 |

---

## Topics（跨模块主题）

| 主题 | wiki 页 | 状态 |
|---|---|---|
| **AclGraph + Pipeline Parallel**（基于设计文档） | [topics/aclgraph-pp.md](topics/aclgraph-pp.md) | **DONE** |
| **Request lifecycle**（含 PD 分离链路） | [topics/request-lifecycle.md](topics/request-lifecycle.md) | **DONE** |
| Aclgraph capture & replay | `topics/aclgraph.md` | TODO |
| Pipeline parallel（深入实现） | `topics/pipeline-parallel.md` | TODO |
| **KV cache（C++ BlockSpaceManager + Python KVCachePool + MemPool）** | [topics/kv-cache.md](topics/kv-cache.md) | **DONE** |
| Tensor parallel | `topics/tensor-parallel.md` | TODO |
| **MoE 实现（Qwen3-MoE / DeepSeek + fused_moe + MC2）** | [topics/moe.md](topics/moe.md) | **DONE**（含 EPLB 缺失分析 + FUSED_MC2 漂移 CONTRADICTION） |
| KV cache & mempool（含 Mooncake） | （已合并入 [topics/kv-cache.md](topics/kv-cache.md)） | — |
| Splitfuse / chunked prefill | `topics/splitfuse.md` | TODO（已被 [comparison/topics/chunked-prefill.md](../comparison/topics/chunked-prefill.md) 项目级覆盖） |
| **Prefix cache**（C++ hash table + LRU + Python plugin + KV pool 二级存储） | [topics/prefix-cache.md](topics/prefix-cache.md) | **DONE**（含 hash table vs trie 辨析 + PD 仅 P 端开启 CONTRADICTION + Multi-LoRA 互斥未运行时强校验 caveat） |
| **Speculative decoding (MTP / LA / Memory Decoding)** | [topics/speculative.md](topics/speculative.md) | **DONE**（3 算法对比表 + C++ placeholder 机制 + UNSUPPORTED_PLUGINS 列表 caveat） |
| **Structured output（xgrammar / response_format / PD replay）** | [topics/structured-output.md](topics/structured-output.md) | **DONE**（含 mtp 互斥 + 双 proto wire 编号 CONTRADICTION + LRU=FIFO VERIFY） |
| Sampler 体系（PTA / CPU selectors） | `topics/sampler.md` | TODO（P2） |
| 模型加载 / 权重 prefetch | `topics/weight-loader.md` | TODO（P3） |
| **Connector / 请求路由** | [topics/connector.md](topics/connector.md) | **DONE**（含 PD 拉 KV 链 + Layerwise PD 路径 + dp_rank_id 公式 caveat） |

---

## Sources of truth

- 主代码根：[d:\design\MindIE-LLM\mindie_llm\](d:\design\MindIE-LLM\mindie_llm)
- 内部设计文档：[d:\design\MindIE-LLM\docs\](d:\design\MindIE-LLM\docs)
- 测试参考：[d:\design\MindIE-LLM\tests\pythontest\](d:\design\MindIE-LLM\tests\pythontest)
- 示例：[d:\design\MindIE-LLM\mindie_llm\examples\](d:\design\MindIE-LLM\mindie_llm\examples)
