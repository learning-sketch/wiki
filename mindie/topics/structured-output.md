---
type: topic
project: mindie
status: verified
confidence: high
verified_against: 2026-04-18
sources:
  - d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\__init__.py
  - d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_grammar.py
  - d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py
  - d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_bitmask.py
  - d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py
  - d:\design\MindIE-LLM\mindie_llm\text_generator\samplers\sampler.py
  - d:\design\MindIE-LLM\mindie_llm\text_generator\samplers\logits_handlers\pta_handlers.py
  - d:\design\MindIE-LLM\mindie_llm\text_generator\utils\sampling_metadata.py
  - d:\design\MindIE-LLM\mindie_llm\text_generator\utils\input_metadata.py
  - d:\design\MindIE-LLM\mindie_llm\text_generator\utils\batch_context.py
  - d:\design\MindIE-LLM\mindie_llm\text_generator\utils\tg_infer_context_store.py
  - d:\design\MindIE-LLM\mindie_llm\text_generator\utils\config.py
  - d:\design\MindIE-LLM\mindie_llm\connector\common\input_metadata_builder.py
  - d:\design\MindIE-LLM\src\server\endpoint\utils\infer_param.cpp
  - d:\design\MindIE-LLM\src\server\endpoint\utils\infer_param.h
  - d:\design\MindIE-LLM\src\server\endpoint\single_req_infer_interface\single_req_infer_interface_base.cpp
  - d:\design\MindIE-LLM\src\engine\seq_group_builder_from_infer_req.cpp
  - d:\design\MindIE-LLM\src\engine\construct_execute_request.cpp
  - d:\design\MindIE-LLM\src\scheduler\scheduler.cpp
  - d:\design\MindIE-LLM\src\include\request_response\request.h
  - d:\design\MindIE-LLM\src\include\sampling.h
  - d:\design\MindIE-LLM\src\include\dataclass\sequence_group.h
  - d:\design\MindIE-LLM\src\include\dataclass\sequence_group_meta_data.h
  - d:\design\MindIE-LLM\proto\model_execute_data.proto
  - d:\design\MindIE-LLM\docs\zh\user_guide\feature\structured_output.md
  - d:\design\MindIE-LLM\tests\pythontest\cpu\text_generator\plugins\structured_output\test_structured_output_manager.py
  - d:\design\MindIE-LLM\tests\pythontest\cpu\text_generator\plugins\structured_output\test_structured_output_grammar.py
  - d:\design\MindIE-LLM\tests\pythontest\cpu\text_generator\plugins\structured_output\test_structured_output_bitmask.py
  - d:\design\MindIE-LLM\tests\pythontest\cpu\text_generator\plugins\test_plugin_manager_structured.py
  - d:\design\MindIE-LLM\tests\pythontest\npu\text_generator\logits_handlers\test_pta_handlers.py
related:
  - ../entities/PluginManager.md
  - ../entities/Generator.md
  - ../topics/speculative.md
  - ../topics/connector.md
  - ../topics/request-lifecycle.md
  - ../../comparison/dimensions.md
---

# MindIE-LLM Structured Output（约束解码 / response_format / xgrammar）

## Summary

MindIE-LLM 的 **结构化输出**（structured output）是基于 **xgrammar** FSM 的 token 级约束解码子系统，仅支持 `json_object` 与 `json_schema` 两种 OpenAI 兼容的 `response_format` 类型（[`StructuredOutputType`，structured_output_grammar.py:20-22](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_grammar.py)；[功能特性表，docs/zh/user_guide/feature/structured_output.md:17-21](d:\design\MindIE-LLM\docs\zh\user_guide\feature\structured_output.md)）。它**不是常规 plugin**：`structured_output/` 目录虽然位于 `text_generator/plugins/` 下，但**不**走 `PluginManager.initialize` 的 `importlib` 循环，而是由 `_init_structured_output_manager` 单独懒加载 ([plugin_manager.py:204-205, 1046-1095](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py))。运行时把 schema 编译成 `xgr.CompiledGrammar`，每个请求维持一个 `XgrammarGrammar` FSM 状态，在 `preprocess` / `forward_loop` 阶段产出 `[batch_size, vocab_size//32]` 的 int32 bitmask，由 `GuidedDecodingLogitsHandler` 在 sampler 内将不允许的 logits 置 `-inf` ([structured_output_bitmask.py:45-62](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_bitmask.py))。C++ Server 侧仅做 JSON Schema 校验、把 `responseFormat` 透传到 `SequenceGroupMetaData.responseFormat_` 与 PD `predicted_token_ids`，**所有 FSM/bitmask 逻辑均在 Python 侧**。

## Sources

- 主源（Python 包，3 文件 + `__init__`）：
  - [d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\__init__.py](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\__init__.py)（44 行，导出符号清单）
  - [d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_grammar.py](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_grammar.py)（215 行）
  - [d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py)（1082 行）
  - [d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_bitmask.py](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_bitmask.py)（92 行）
- 集成点：[plugin_manager.py:124-125, 204-205, 607-616, 647-662, 818-832, 865-871, 915, 1046-1095](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py)
- Sampler：[pta_handlers.py:87-125](d:\design\MindIE-LLM\mindie_llm\text_generator\samplers\logits_handlers\pta_handlers.py)，[sampler.py:62-64, 255-267, 298](d:\design\MindIE-LLM\mindie_llm\text_generator\samplers\sampler.py)，[sampling_metadata.py:386-388](d:\design\MindIE-LLM\mindie_llm\text_generator\utils\sampling_metadata.py)
- Batch / context：[input_metadata.py:98-101](d:\design\MindIE-LLM\mindie_llm\text_generator\utils\input_metadata.py)，[batch_context.py:47, 57, 91, 111, 120-126, 508, 558, 653-656, 719-720, 843-847](d:\design\MindIE-LLM\mindie_llm\text_generator\utils\batch_context.py)，[tg_infer_context_store.py:94-95](d:\design\MindIE-LLM\mindie_llm\text_generator\utils\tg_infer_context_store.py)
- C++ Server / Engine / Scheduler：[infer_param.cpp:216-225, 844-977](d:\design\MindIE-LLM\src\server\endpoint\utils\infer_param.cpp)，[infer_param.h:101, 129](d:\design\MindIE-LLM\src\server\endpoint\utils\infer_param.h)，[single_req_infer_interface_base.cpp:945](d:\design\MindIE-LLM\src\server\endpoint\single_req_infer_interface\single_req_infer_interface_base.cpp)，[seq_group_builder_from_infer_req.cpp:85, 130](d:\design\MindIE-LLM\src\engine\seq_group_builder_from_infer_req.cpp)，[scheduler.cpp:1104-1113, 1463-1466](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)，[construct_execute_request.cpp:173-181](d:\design\MindIE-LLM\src\engine\construct_execute_request.cpp)
- 数据结构 / IPC：[request.h:69](d:\design\MindIE-LLM\src\include\request_response\request.h)，[sampling.h:58](d:\design\MindIE-LLM\src\include\sampling.h)，[sequence_group.h:110](d:\design\MindIE-LLM\src\include\dataclass\sequence_group.h)，[sequence_group_meta_data.h:99-101](d:\design\MindIE-LLM\src\include\dataclass\sequence_group_meta_data.h)，[model_execute_data.proto:170-171](d:\design\MindIE-LLM\proto\model_execute_data.proto)，[connector/common/input_metadata_builder.py:344, 478, 545, 661, 704, 803-812, 883](d:\design\MindIE-LLM\mindie_llm\connector\common\input_metadata_builder.py)
- 配置：[config.py:203-205](d:\design\MindIE-LLM\mindie_llm\text_generator\utils\config.py)
- 用户文档：[docs/zh/user_guide/feature/structured_output.md](d:\design\MindIE-LLM\docs\zh\user_guide\feature\structured_output.md)（214 行用户指南，唯一一篇 docs 提及）
- 测试：见下方 §跨子系统引用 类别 4。

## Architecture / Data flow

### 1. 模块拓扑

```
text_generator/plugins/structured_output/
├── __init__.py                       # 仅导出符号；不暴露 ...Plugin 类
├── structured_output_grammar.py      # FSM 抽象 + xgrammar 适配器
├── structured_output_manager.py      # 后端、缓存、批量 bitmask、状态同步
└── structured_output_bitmask.py      # logits 上的 bitmask 应用（NPU torch 实现）
```

`__init__.py` 仅导出符号清单，没有形如 `XxxPlugin` 的 plugin 主类（[`__init__.py:10-43`](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\__init__.py)）。这与 `splitfuse` / `prefix_cache` / `mtp` 等"标准 plugin"形成明显差异——后者有 `XxxPlugin` 类并被 `PluginManager.initialize` 通过 `importlib` 动态加载（[plugin_manager.py:186-200](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py)）；structured output **不**进 `plugin_list`，而由专属方法 `_init_structured_output_manager` 单独懒构建（[plugin_manager.py:204-205, 1046-1095](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py)；亦见 [PluginManager.md §第 176 行说明](../entities/PluginManager.md)）。

> synthesis: 这是一个"借用 plugins 目录但不走 plugin 机制"的子系统，**目录位置与加载机制不一致**，对 ingest 有迷惑性。

### 2. 三层抽象

| 层 | 类 | 责任 | 位置 |
|---|---|---|---|
| Schema 适配 | `StructuredOutputRequest` (`@dataclass`) | 解析 OpenAI 风格 `response_format` JSON 字符串 → `(StructuredOutputType, grammar_spec)` | [structured_output_grammar.py:25-87](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_grammar.py) |
| FSM 抽象 | `StructuredOutputGrammar` (ABC) | `accept_tokens` / `fill_bitmask` / `is_terminated` / 双游标 `num_processed_tokens` vs `num_tried_tokens` | [structured_output_grammar.py:90-138](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_grammar.py) |
| FSM 实现 | `XgrammarGrammar` | 包装 `xgr.GrammarMatcher`，在 reject 时仍推进 `num_tried_tokens` 以对齐 C++ replay buffer | [structured_output_grammar.py:141-215](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_grammar.py) |
| 后端 | `GrammarBackend` | 懒 `import xgrammar`，构造 `TokenizerInfo.from_huggingface` + `GrammarCompiler`，编译 `compile_json_schema` | [structured_output_manager.py:104-263](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py) |
| 配置 | `StructuredOutputConfig` | `backend=XGRAMMAR` / `xgrammar_any_whitespace=False` / `grammar_cache_size=100` / `bitmask_prealloc_batch=64` | [structured_output_manager.py:91-101](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py) |
| 编译产物 | `CompiledGrammar` (`@dataclass`) | `(backend_type, ctx, vocab_size, xgr_module)`，被多请求共享 | [structured_output_manager.py:266-276](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py) |
| 主管理 | `StructuredOutputManager` | 缓存编译结果（SHA-256 key + LRU 100）+ 维护 `state_key → grammar` 字典 + 预分配 bitmask 缓冲 | [structured_output_manager.py:279-1081](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py) |
| logits 应用 | `apply_token_bitmask_inplace[_npu]` | NPU 上展开 int32 bitmask 为 bit，再 `masked_fill_(-inf)`；对 `vocab_size > mask_coverage` 的尾部直接置 `-inf` | [structured_output_bitmask.py:45-91](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_bitmask.py) |

### 3. 端到端数据流（同步路径）

```mermaid
sequenceDiagram
    participant Client as HTTP Client
    participant InferAPI as C++ Server (infer_param.cpp)
    participant Sched as C++ Scheduler
    participant Builder as Python connector/input_metadata_builder
    participant PM as Python PluginManager
    participant Mgr as StructuredOutputManager
    participant Sampler as PTA Sampler

    Client->>InferAPI: POST /v1/chat/completions { response_format: {...} }
    Note over InferAPI: AssignResponseFormat 校验 type/json_schema/name/schema; 失败即拒
    InferAPI->>Sched: Request.responseFormat (string)
    Note over Sched: ValidateMtpConstraints: mtpEnabled && reqStructuredOutput → 拒绝
    Sched->>Sched: SequenceGroupMetaData.responseFormat_ = sampling->responseFormat
    Sched->>Sched: AddGeneratedToken: 同时追加 prefillReplayTokenIds_
    Sched->>Builder: protobuf SequenceGroupMetaData (response_format=34, predicted_token_ids=35)
    Builder->>PM: InputMetadata.batch_response_format / batch_predicted_token_ids
    PM->>Mgr: build_and_assign_structured_guided_bitmask(...)
    Note over Mgr: prefill: process_batch_for_generation → init grammar / fill bitmask<br/>decode: sync_states_for_decode 先回放 → 再 process_batch_for_generation
    Mgr->>Mgr: grammar.fill_bitmask([batch, vocab//32] int32)
    Mgr-->>PM: sampling_metadata.guided_bitmask
    PM->>Sampler: forward + sample(logits, sampling_metadata)
    Sampler->>Sampler: GuidedDecodingLogitsHandler: apply_token_bitmask_inplace_npu(logits, bm) → -inf
    Sampler-->>PM: sampling_output.token_ids
    PM->>Mgr: compute_structured_output_accepted(cache_ids, token_ids)
    Mgr->>Mgr: update_states_after_sampling: grammar.accept_tokens
```

### 4. 集成点（PluginManager / sampler / batch_context）

#### 4.1 初始化（PluginManager 端）

- 默认开启：`self._structured_output_enabled = kwargs.get("enable_structured_output", True)`（[plugin_manager.py:125](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py)）。
- 在 `initialize()` 末尾调用 `_init_structured_output_manager()`（[plugin_manager.py:204-205](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py)）。
- `_init_structured_output_manager` 从 `kwargs["guided_decoding_backend"]` 取后端字符串（默认 `"xgrammar"`，参见 [config.py:204-205](d:\design\MindIE-LLM\mindie_llm\text_generator\utils\config.py)），构造 `StructuredOutputManager`，并通过 `infer_context.set_structured_output_manager(...)` 把管理器塞到 `BatchContext` 中（[plugin_manager.py:1075-1088](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py)；[tg_infer_context_store.py:94-95](d:\design\MindIE-LLM\mindie_llm\text_generator\utils\tg_infer_context_store.py)；[batch_context.py:653-656](d:\design\MindIE-LLM\mindie_llm\text_generator\utils\batch_context.py)）。
- 任何异常（`ImportError` / 通用 `Exception`）都把 `_structured_output_enabled` 置 `False`、降级为透传（[plugin_manager.py:1090-1095](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py)）。

#### 4.2 同步路径（`preprocess` / `postprocess`）

- **preprocess**：组装 `model_inputs / sampling_metadata` 后，从 `input_metadata.batch_response_format`（prefill）或 `infer_context.get_response_format(cache_ids)`（decode）取 `response_format_array`，调 `build_and_assign_structured_guided_bitmask` 把 `guided_bitmask` 写入 `sampling_metadata`（[plugin_manager.py:607-615](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py)）。
- **postprocess**：在 sampler 输出后，调 `compute_structured_output_accepted` 推进 FSM；返回 `is_accepted` 数组用于 `output_filter.filter_finished_sequences`（[plugin_manager.py:647-668](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py)）。

#### 4.3 异步路径（`forward_loop`）

- 异步 worker 在 `input_queue.get()` 后**也**要 build bitmask，因为 sampler 走的是同一份 `sampling_metadata` 字段（[plugin_manager.py:818-832](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py)）。
- sample 后 `compute_structured_output_accepted` 写到 `sampling_output.is_structured_accepted`（[plugin_manager.py:865-871, 915](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py)）；同步路径再读这个字段（[plugin_manager.py:647](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py)）。

#### 4.4 Sampler

- `SamplingMetadata.guided_bitmask` 字段（[sampling_metadata.py:386-388](d:\design\MindIE-LLM\mindie_llm\text_generator\utils\sampling_metadata.py)）。
- Sampler 在 split 子批时按 `row_mask` 同步切片 `guided_bitmask`，并对维度不一致的情况记 warning 并**不**切（[sampler.py:255-267](d:\design\MindIE-LLM\mindie_llm\text_generator\samplers\sampler.py)）。
- 注册名 `'guided_decoding'` 的 PTA logits handler `GuidedDecodingLogitsHandler`：bitmask 为 None 时直接返回；否则懒导入 `apply_token_bitmask_inplace` 并原地修改 logits（[pta_handlers.py:87-125](d:\design\MindIE-LLM\mindie_llm\text_generator\samplers\logits_handlers\pta_handlers.py)）。导入失败仅 warning，不抛出（[pta_handlers.py:101-113](d:\design\MindIE-LLM\mindie_llm\text_generator\samplers\logits_handlers\pta_handlers.py)）。
- handler 默认 backend = `HandlingBackend.PTA`（[config.py:215](d:\design\MindIE-LLM\mindie_llm\text_generator\utils\config.py)），即在 device 端做 `masked_fill_`。

#### 4.5 Batch context

- `DictContext.response_format`（dict[context_handle → str]）：fork / clear / reset / `get_response_format` 五个方法配套（[batch_context.py:47, 57, 91, 111, 120-126](d:\design\MindIE-LLM\mindie_llm\text_generator\utils\batch_context.py)）。Prefill 时 `add_context` 用 `input_metadata.batch_response_format` 写入（[batch_context.py:843-847](d:\design\MindIE-LLM\mindie_llm\text_generator\utils\batch_context.py)），Decode 直接 `get_response_format(cache_ids)` 取。
- `BatchContext.structured_output_manager` 槽位 + `clear_context_by_handles` 调 `clear_finished_requests`（[batch_context.py:508, 558, 719-720](d:\design\MindIE-LLM\mindie_llm\text_generator\utils\batch_context.py)）。

### 5. 关键算法（`StructuredOutputManager` 内）

- **缓存键**：`f"{output_type}:{sha256(grammar_spec)}"` + 存入 `(spec, compiled)` 用作哈希碰撞校验（[structured_output_manager.py:326-333, 1050-1080](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py)）。LRU 简化为"超过容量删第一个"（[structured_output_manager.py:1075-1078](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py)）。
- **预分配 bitmask 缓冲**：`shape=(64, ceil(vocab/32))` int32，`fill(-1)` 表示全允许；批超时动态扩容（[structured_output_manager.py:1015-1024, 487-490](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py)）。
- **双游标设计**（关键不变量）：
  - `num_processed_tokens` = FSM 合法接受的 token 数（仅日志）。
  - `num_tried_tokens` = replay buffer 游标，**含**被 reject 的 token，用作回放下标（[structured_output_grammar.py:98-102, 171-203](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_grammar.py)；同步逻辑 [structured_output_manager.py:819-845](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py)）。
  - 这两个游标分离是为了对齐 **C++ 侧无条件存储 rejected token 的 replay buffer**，防止下标错位/重复 feed（行内注释 [structured_output_manager.py:819-822, 840-842](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py)）。
- **`build_and_assign_structured_guided_bitmask` 决策序**（[structured_output_manager.py:644-705](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py)）：
  1. Decode：先 `sync_states_for_decode` 再 `process_batch_for_generation`，否则会先把"无 grammar 的 sequence"误初始化为初始态（行内注释 [structured_output_manager.py:663-671](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py)）。
  2. Prefill 且有 `predicted_token_ids`（PD 分离 / 重计算）：build 之后再 `replay_predicted_tokens_after_init`，**然后**重算 bitmask，否则会用回放前的初始态约束（行内注释 [structured_output_manager.py:676-686](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py)）。
- **`sync_states_for_decode` 五分支**（[structured_output_manager.py:801-890](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py)）：
  1. 无 replay token → continue。
  2. `replay_pos == len(replay_tokens)` → 已对齐，continue。
  3. `replay_pos > len(replay_tokens)`（grammar 比 predicted 超前，常见于 D 节点采样后）→ keep grammar，continue（[structured_output_manager.py:829-839](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py)）。
  4. `0 < replay_pos < len(replay_tokens)` → 增量推进 `replay_tokens[replay_pos:]`（[structured_output_manager.py:843-859](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py)）。
  5. 其它（如游标错位）→ rebuild：弹出旧 grammar 后调 `_build_and_replay_structured_output_state`（[structured_output_manager.py:861-889](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py)）。
- **rebuild 失败的 suffix search**：完整重放失败时，先取 grammar 已接受的部分前缀；若全部在起始位置就被拒，则在尾部 512-token 窗口内逐步往后切，找最后一个能成功重放的 suffix（[structured_output_manager.py:932-957](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py)）。
  - synthesis: 这个 suffix-window 设计是为了应对"replay_tokens 被 prompt 噪声污染"的场景，作者在注释里点出"避免 suffix search 清空 grammar 导致状态归零"。

### 6. C++ 侧（仅校验 + 透传）

- **请求落地**：`AssignResponseFormat` 把整个 `response_format` JSON `dump()` 写入 `Request.responseFormat`（[infer_param.cpp:915-977](d:\design\MindIE-LLM\src\server\endpoint\utils\infer_param.cpp)；[request.h:69](d:\design\MindIE-LLM\src\include\request_response\request.h)）。Type 仅允许 `json_object` / `json_schema`（[infer_param.cpp:933-940](d:\design\MindIE-LLM\src\server\endpoint\utils\infer_param.cpp)）。
- **JSON Schema 校验**：
  - `name` 必须为 1-64 字符的非空字符串（[infer_param.cpp:844-862](d:\design\MindIE-LLM\src\server\endpoint\utils\infer_param.cpp)）。
  - `ValidateJsonSchemaTypes` 递归校验白名单 `{string, integer, number, boolean, array, object, null}` + `properties/items/required` 类型（[infer_param.cpp:864-913](d:\design\MindIE-LLM\src\server\endpoint\utils\infer_param.cpp)）。
- **互斥**：`ValidateMtpConstraints` —— `mtpEnabled && reqStructuredOutput` → "structured output (response_format) cannot be used with mtp"（[infer_param.cpp:216-225](d:\design\MindIE-LLM\src\server\endpoint\utils\infer_param.cpp)；`reqStructuredOutput` 字段 [infer_param.h:101](d:\design\MindIE-LLM\src\server\endpoint\utils\infer_param.h)；置位 [single_req_infer_interface_base.cpp:945](d:\design\MindIE-LLM\src\server\endpoint\single_req_infer_interface\single_req_infer_interface_base.cpp)）。这条互斥被 [topics/speculative.md §第 107 行](speculative.md) 引用。
- **传到采样参数 / SequenceGroup**：
  - `SamplingParams.responseFormat`（[sampling.h:58](d:\design\MindIE-LLM\src\include\sampling.h)）：`SeqGroupBuilderFromInferReq::CreateSampleParam` 中 `sampleParamSptr->responseFormat = request->responseFormat`（[seq_group_builder_from_infer_req.cpp:85](d:\design\MindIE-LLM\src\engine\seq_group_builder_from_infer_req.cpp)）。
  - `SequenceGroupMetaData.responseFormat_` + `predictedTokenIds_`（[sequence_group_meta_data.h:99-101](d:\design\MindIE-LLM\src\include\dataclass\sequence_group_meta_data.h)）：`Scheduler::FillScheduledSeqGrpMetaData` 复制（[scheduler.cpp:1104, 1110-1113](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)）。
- **PD replay 状态维护**：
  - `SequenceGroup.prefillReplayTokenIds_`（[sequence_group.h:110](d:\design\MindIE-LLM\src\include\dataclass\sequence_group.h)）由 D 节点接收 decode 请求时用 P 节点 prefill 已输出 token 初始化（[seq_group_builder_from_infer_req.cpp:130](d:\design\MindIE-LLM\src\engine\seq_group_builder_from_infer_req.cpp)）。
  - `Scheduler::AddGeneratedToken` 在每次新 token 出来时同时 `push_back` 进 `prefillReplayTokenIds_`，**仅当** `seqGroup->sampling->responseFormat.has_value()`（[scheduler.cpp:1463-1466](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)）。
- **proto 编码**：`response_format = 34`（optional string）+ `predicted_token_ids = 35`（repeated int64），`ConstructExecuteRequest` 写入（[construct_execute_request.cpp:173-181](d:\design\MindIE-LLM\src\engine\construct_execute_request.cpp)；[model_execute_data.proto:170-171](d:\design\MindIE-LLM\proto\model_execute_data.proto)）。
- **Python 解码**：`InputMetadataBuilder` 在 prefill 路径上 `seq_group_metadata.HasField("response_format")` 取出并按 sequence 数复制（[input_metadata_builder.py:803-812](d:\design\MindIE-LLM\mindie_llm\connector\common\input_metadata_builder.py)），在 decode 路径上注释明示"已在 prefill 阶段存入 DictContext"故跳过（[input_metadata_builder.py:482](d:\design\MindIE-LLM\mindie_llm\connector\common\input_metadata_builder.py)）。

> [!warning] CONTRADICTION: `prefillAndDecodeCommunication.proto:87` 也有一个 `optional string responseFormat = 21`，但它属于另一个采样参数 message（[prefillAndDecodeCommunication.proto:75-89](d:\design\MindIE-LLM\src\server\endpoint\grpc_wrapper\prefillAndDecodeCommunication.proto)），与 `model_execute_data.proto` 的 `response_format=34` 是 **两个独立 wire 编号 / 命名风格**（前者驼峰、后者下划线）。同一概念在两套 proto 中并存，未在源码注释中说明哪条链路实际生效；当前可见的 Python 解码端 `input_metadata_builder.py:805` 用的是 `seq_group_metadata.HasField("response_format")`，对应 `model_execute_data.proto:170`。`prefillAndDecodeCommunication.proto` 侧的 `responseFormat=21` 字段在仓库 Python 代码中 grep 0 命中（即未被 Python 端读取），疑似 PD gRPC 透传链路上的对照字段或预留。

### 7. 仅支持的特性 vs 用户文档对齐

| 维度 | 实现现状 | 锚点 |
|---|---|---|
| 后端 | 仅 `xgrammar`（`GuidedDecodingBackendType` 单 enum） | [structured_output_manager.py:87-89](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py) |
| 输出类型 | `json_object` / `json_schema` | [structured_output_grammar.py:20-22](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_grammar.py)；[infer_param.cpp:933-940](d:\design\MindIE-LLM\src\server\endpoint\utils\infer_param.cpp)；[docs:17-21](d:\design\MindIE-LLM\docs\zh\user_guide\feature\structured_output.md) |
| 缺失 vs vLLM | 无 regex / EBNF / pydantic / structural_tag / outlines / lm_format_enforcer / guidance / jump-forward | grep `outlines\|llguidance\|lm_format_enforcer` 在 `mindie_llm/` 全树 0 命中（仅 plugin_manager.py 里 `enable_guided_decoding` 标志） |
| `xgrammar_any_whitespace` | 默认 `False`（更严格 JSON 间距） | [structured_output_manager.py:99](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py) |
| 启用方式 | 无插件配置；自动 on，仅看请求里有无 `response_format` | [docs:35-36](d:\design\MindIE-LLM\docs\zh\user_guide\feature\structured_output.md) |

### 8. 跨项目对比（synthesis，brief）

> synthesis: 三家都以 xgrammar 为主流后端，但抽象层级与可选后端差异显著。

| 维度 | MindIE | vLLM | SGLang |
|---|---|---|---|
| 后端类数 | 1 (xgrammar) | 5 (xgrammar / outlines / lm_format_enforcer / guidance / + base) | 5 (xgrammar / outlines / llguidance / reasoner / outlines_jump_forward) |
| 入口位置 | [text_generator/plugins/structured_output/](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output) | [vllm/v1/structured_output/](d:\design\vllm\vllm\v1\structured_output) | [srt/constrained/](d:\design\sglang\python\sglang\srt\constrained) |
| 入口文件名 | `structured_output_manager.py` | `__init__.py` + `backend_*.py` | `grammar_manager.py` + `*_backend.py` |
| Schema 类型 | json_object / json_schema | json / regex / EBNF / structural_tag / pydantic | json / regex / EBNF / structural_tag |
| jump-forward | 无 | 无（不属于 structured_output 模块） | 有（`outlines_jump_forward.py` + `reasoner_grammar_backend.py`） |
| C++ 侧职责 | 仅 JSON Schema 静态校验 + proto 透传 | N/A（vLLM 无 C++ engine） | N/A（python-only） |
| 与 spec decoding 协同 | **互斥**（`mtpEnabled && reqStructuredOutput → 拒绝`） | 兼容（结构化输出与 spec decoding 在 v1/sample/ops 协同） | 兼容 |
| PD 场景的 replay | C++ `prefillReplayTokenIds_` + Python `predicted_token_ids` 双游标对齐 | 由 v1 scheduler 内部维护 | 由 grammar_manager 内部 |

详见 [comparison/dimensions.md §dim-structured](../../comparison/dimensions.md) 与（待新建的）`comparison/topics/structured-output.md`。

## Key APIs / Entities

### Python（主源）

| 名称 | 位置 | 作用 |
|---|---|---|
| `StructuredOutputType` (Enum) | [structured_output_grammar.py:20-22](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_grammar.py) | `JSON_OBJECT` / `JSON_SCHEMA` |
| `StructuredOutputRequest.from_response_format` | [structured_output_grammar.py:33-87](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_grammar.py) | 解析 OpenAI `response_format` JSON 字符串 |
| `StructuredOutputGrammar` (ABC) | [structured_output_grammar.py:90-138](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_grammar.py) | FSM 抽象 |
| `XgrammarGrammar.accept_tokens` | [structured_output_grammar.py:171-203](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_grammar.py) | 推进 FSM；reject 时仍 +tried |
| `XgrammarGrammar.fill_bitmask` | [structured_output_grammar.py:205-211](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_grammar.py) | 调 `matcher.fill_next_token_bitmask`；终止状态填 `-1`（全允许） |
| `parse_bitmask_allowed_tokens` | [structured_output_manager.py:29-52](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py) | int32 bitmask → 允许 token id 列表（仅日志） |
| `GuidedDecodingBackendType` | [structured_output_manager.py:87-89](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py) | 单值 enum: `XGRAMMAR` |
| `StructuredOutputConfig` | [structured_output_manager.py:95-101](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py) | 默认 `cache_size=100, prealloc_batch=64, any_whitespace=False` |
| `GrammarBackend.compile_grammar` | [structured_output_manager.py:166-181](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py) | 仅走 xgrammar 路径 |
| `GrammarBackend._init_xgrammar` | [structured_output_manager.py:187-230](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py) | `TokenizerInfo.from_huggingface` + `GrammarCompiler` |
| `CompiledGrammar` | [structured_output_manager.py:266-276](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py) | 跨请求复用编译产物 |
| `StructuredOutputManager.grammar_init` | [structured_output_manager.py:407-463](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py) | 为单请求初始化 grammar |
| `StructuredOutputManager.grammar_bitmask` | [structured_output_manager.py:465-517](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py) | 批量填 bitmask |
| `StructuredOutputManager.process_batch_for_generation` | [structured_output_manager.py:593-625](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py) | 入口 helper（被 `build_and_assign_...` 调） |
| `StructuredOutputManager.build_and_assign_structured_guided_bitmask` | [structured_output_manager.py:644-705](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py) | PluginManager 调；写 `sampling_metadata.guided_bitmask` |
| `StructuredOutputManager.compute_structured_output_accepted` | [structured_output_manager.py:627-642](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py) | sample 后推进 FSM |
| `StructuredOutputManager.update_states_after_sampling` | [structured_output_manager.py:707-743](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py) | 逐 token 推进，返回 `is_accepted_array` |
| `StructuredOutputManager.replay_predicted_tokens_after_init` | [structured_output_manager.py:755-799](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py) | PD prefill 后回放 |
| `StructuredOutputManager.sync_states_for_decode` | [structured_output_manager.py:801-890](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py) | Decode 同步 5 分支 |
| `StructuredOutputManager._build_and_replay_structured_output_state` | [structured_output_manager.py:915-957](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py) | rebuild + suffix search |
| `StructuredOutputManager._compile_grammar` | [structured_output_manager.py:1045-1081](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py) | SHA-256 LRU 缓存 |
| `apply_token_bitmask_inplace_npu` | [structured_output_bitmask.py:45-62](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_bitmask.py) | NPU 上展位 + `masked_fill_(-inf)` |
| `apply_token_bitmask_inplace` | [structured_output_bitmask.py:65-91](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_bitmask.py) | numpy → torch tensor 包装 |
| `GuidedDecodingLogitsHandler` | [pta_handlers.py:87-125](d:\design\MindIE-LLM\mindie_llm\text_generator\samplers\logits_handlers\pta_handlers.py) | sampler 端 logits 修改器 |

### C++（仅校验/透传）

| 名称 | 位置 | 作用 |
|---|---|---|
| `Request.responseFormat` | [request.h:69](d:\design\MindIE-LLM\src\include\request_response\request.h) | `optional<string>` 顶层载体 |
| `SamplingParams.responseFormat` | [sampling.h:58](d:\design\MindIE-LLM\src\include\sampling.h) | 落到采样参数 |
| `SequenceGroup.prefillReplayTokenIds_` | [sequence_group.h:110](d:\design\MindIE-LLM\src\include\dataclass\sequence_group.h) | PD replay 缓冲 |
| `SequenceGroupMetaData.responseFormat_` / `predictedTokenIds_` | [sequence_group_meta_data.h:99-101](d:\design\MindIE-LLM\src\include\dataclass\sequence_group_meta_data.h) | 调度元数据 |
| `InferParam::ValidateMtpConstraints` | [infer_param.cpp:216-225](d:\design\MindIE-LLM\src\server\endpoint\utils\infer_param.cpp) | mtp 互斥 |
| `AssignResponseFormat` | [infer_param.cpp:915-977](d:\design\MindIE-LLM\src\server\endpoint\utils\infer_param.cpp) | 校验 + dump 字符串 |
| `ValidateJsonSchemaName` / `ValidateJsonSchemaTypes` | [infer_param.cpp:844-913](d:\design\MindIE-LLM\src\server\endpoint\utils\infer_param.cpp) | 递归 schema 校验 |
| `Scheduler::AddGeneratedToken` | [scheduler.cpp:1456-1467](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp) | 仅当有 `responseFormat` 时维护 replay |
| `ConstructSeqGrpMetaData` 中 protobuf 写入 | [construct_execute_request.cpp:173-181](d:\design\MindIE-LLM\src\engine\construct_execute_request.cpp) | 写入 `response_format=34, predicted_token_ids=35` |
| Proto 字段 | [model_execute_data.proto:170-171](d:\design\MindIE-LLM\proto\model_execute_data.proto) | wire 编号 34/35 |

## Notes / Caveats

> [!warning] CONTRADICTION: 两套 proto 同时存在 `responseFormat`/`response_format`：`prefillAndDecodeCommunication.proto:87` 用驼峰命名 wire=21；`model_execute_data.proto:170` 用下划线 wire=34。仓库 Python 端只读取后者（[input_metadata_builder.py:805](d:\design\MindIE-LLM\mindie_llm\connector\common\input_metadata_builder.py)）。前者在 `mindie_llm/` 全 Python 树 grep 0 命中，疑似 PD gRPC 链路对照字段或预留。

> [!warning] CONTRADICTION: 目录 vs 加载机制。`structured_output/` 位于 `text_generator/plugins/` 下，但**不是**通过 `PluginManager` 的 `plugin_list` + `importlib` 循环加载（[plugin_manager.py:186-200](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py)），而是单独走 `_init_structured_output_manager`（[plugin_manager.py:1046-1095](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py)）。`__init__.py` 不导出 `XxxPlugin` 类（[`__init__.py:10-43`](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\__init__.py)）。已在 [PluginManager.md §第 176 行](../entities/PluginManager.md) 提到。

> [!todo] VERIFY: `_compile_grammar` 的"超容量删第一个"严格意义上是 FIFO 而非 LRU——`dict` 自 Python 3.7 保持插入顺序，但**重复访问命中**不会把 key 移动到末尾（[structured_output_manager.py:1053-1080](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py) 中 cache hit 路径**没有**重新 `pop` + 插入）。所以 `grammar_cache_size=100` 实际是 FIFO 容量，长 schema 复用率高时可能误删活跃项。

> [!todo] VERIFY: `XgrammarGrammar.fill_bitmask` 在 `_is_terminated` 时把整行写 `-1`（[structured_output_grammar.py:207-209](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_grammar.py)）= 全允许；理论上一旦终止，sampler 应该让 EOS 自然出现，但 `is_structured_accepted` 仍可能在终止后被推 token——`accept_tokens` 在 `_is_terminated` 时直接返回 True（[structured_output_grammar.py:173-174](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_grammar.py)），未推进 `num_tried_tokens`，可能让 PD replay 下标与 C++ 侧不一致。

> [!todo] VERIFY: `enable_structured_output` 与 `enable_guided_decoding` 是两个独立开关——`PluginManager` 用前者控制 manager 实例化（[plugin_manager.py:125, 1047](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py)），`SamplerConfig` 用后者控制 logits handler（[config.py:204](d:\design\MindIE-LLM\mindie_llm\text_generator\utils\config.py)、[sampler.py:64](d:\design\MindIE-LLM\mindie_llm\text_generator\samplers\sampler.py)）。若用户只关掉一个，行为是"bitmask 生成但不应用"或反之，未在用户文档中说明。

> [!todo] VERIFY: PD 场景下 `prefillReplayTokenIds_` 由 D 节点接收 decode 请求时初始化（[seq_group_builder_from_infer_req.cpp:130](d:\design\MindIE-LLM\src\engine\seq_group_builder_from_infer_req.cpp)），但 `Request.prefillReplayTokenIds` 字段本身在 `request.h` 中未单独 grep 到（grep `prefillReplayTokenIds` 在 `src/include/request_response/` 0 命中），需确认它来自上游 `Request` 结构体的哪个字段或哪条 proto。

## 跨子系统引用（§5 step 3 强制 5 类 grep）

> 范围：`d:\design\MindIE-LLM\` 全仓库（包含 `src/` C++、`mindie_llm/` Python、`tests/`、`docs/`、`proto/`、`config*/`）。

### 1) 跨语言绑定（C++ ↔ Python）

- Python 类名 `StructuredOutputManager` / `StructuredOutputGrammar` / `XgrammarGrammar` / `GuidedDecodingBackendType`：在 `d:\design\MindIE-LLM\src\` 全 C++ 树 grep **0 命中**（即 C++ 侧不直接构造或调用任何 Python 类，纯通过 `responseFormat`/`predictedTokenIds_` 字符串字段 + proto wire 编号 34/35 传递语义，由 Python 端在 `input_metadata_builder.py` 重新解析）。
- 关键库 `xgrammar` / `XGrammar` / `outlines` / `llguidance` / `lm_format_enforcer`：在 `d:\design\MindIE-LLM\src\` 全 C++ 树 grep **0 命中**；在 `d:\design\MindIE-LLM\mindie_llm\` 全 Python 树仅 5 个文件命中（[config.py:204-205, plugin_manager.py:1054, structured_output 子包 3 文件](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\structured_output_manager.py)）。
- `pybind` / `ctypes` / `cffi` / `Cython` / `capsule`：在 `d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\structured_output\` 全树 grep **0 命中**——本子系统纯 Python，不直接调 C++ 扩展，仅通过 `xgrammar` Python 包间接使用其 native 后端。

### 2) 协作伙伴跨子系统引用

- `PluginManager` ↔ `StructuredOutputManager`：在 `mindie_llm/text_generator/plugins/plugin_manager.py` 全树 grep `_structured_output_manager` 共 **9 命中**（行 124, 125, 205, 607, 613, 650, 818, 827, 865, 1046, 1051, 1071, 1081, 1086, 1092, 1095，跨同步 / 异步 / 初始化 / 失败 fallback 三类路径）。
- `BatchScheduler` (C++) ↔ structured output：在 `d:\design\MindIE-LLM\src\scheduler\scheduler.cpp` 全树 grep `responseFormat` 共 **3 命中**（[L1104, L1110, L1464](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)），且在 [L1463-1466](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp) `AddGeneratedToken` 内**仅**当有 `responseFormat` 才追加 `prefillReplayTokenIds_`（即 replay buffer 是 structured output 专属机制，不为通用功能开销）。
- `Sampler` ↔ `StructuredOutputManager`：通过 `SamplingMetadata.guided_bitmask` 字段间接耦合（[sampling_metadata.py:386-388](d:\design\MindIE-LLM\mindie_llm\text_generator\utils\sampling_metadata.py)），不直接 import；`GuidedDecodingLogitsHandler` 懒导入 `structured_output_bitmask` 1 个函数（[pta_handlers.py:120](d:\design\MindIE-LLM\mindie_llm\text_generator\samplers\logits_handlers\pta_handlers.py)），即 sampler 不持有 manager 引用。
- `Generator` / `LlmEngine`：在 `d:\design\MindIE-LLM\mindie_llm\text_generator\generator.py` 全树 grep `structured_output|StructuredOutput` **0 命中**（即 `Generator` 不直接感知 structured output，全部下沉到 `PluginManager`）；C++ `LlmEngine` 同样在 `src/engine/llm_engine.cpp` 0 命中。

### 3) 配置 / IPC 共享数据结构

- `response_format` proto wire 编号：`d:\design\MindIE-LLM\` 全树 grep `response_format = 34` 共 **1 命中**（[model_execute_data.proto:170](d:\design\MindIE-LLM\proto\model_execute_data.proto)）；驼峰版本 `responseFormat = 21` 在 [prefillAndDecodeCommunication.proto:87](d:\design\MindIE-LLM\src\server\endpoint\grpc_wrapper\prefillAndDecodeCommunication.proto) **1 命中**——见 §6 末尾 CONTRADICTION。
- `predicted_token_ids` proto wire 编号 35：`d:\design\MindIE-LLM\` 全树 grep `predicted_token_ids` 共 **6 命中**（proto 1 + Python 解码 4 + C++ 写入 1），仅与 structured output 关联（PD replay 用途）。
- `enable_structured_output` / `guided_decoding_backend` 配置字段：`mindie_llm/` 全树 grep `enable_structured_output` 共 **2 命中** ([plugin_manager.py:125, 1047](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py))；`guided_decoding_backend` **2 命中** ([config.py:205, plugin_manager.py:1075](d:\design\MindIE-LLM\mindie_llm\text_generator\utils\config.py))。`d:\design\MindIE-LLM\src\` C++ 树 grep `guided_decoding` **0 命中**（C++ 侧不感知后端选择）。
- 部署 yaml/json：`d:\design\MindIE-LLM\` 全树 grep `structured_output|response_format|guided_decoding` 在 `*.yaml`/`*.json` 类型文件 **0 命中**（除了 `.gitignore` 提到日志路径）；即用户启用结构化输出**不需要**改动 `config.json`，与用户文档 [docs:35-36](d:\design\MindIE-LLM\docs\zh\user_guide\feature\structured_output.md) "无需在 config.json 中为本特性单独增加插件配置" 一致。

### 4) 测试覆盖反查

- 主目录 `d:\design\MindIE-LLM\tests\pythontest\cpu\text_generator\plugins\structured_output\` 共 **3 个测试文件**：
  - [test_structured_output_manager.py](d:\design\MindIE-LLM\tests\pythontest\cpu\text_generator\plugins\structured_output\test_structured_output_manager.py)（1294 行，覆盖 `StructuredOutputConfig` / `GrammarBackend` / `StructuredOutputManager` 全部 API；含 `RealTokenizerForXgrammar` 真实 xgrammar 集成测试，gating by `XGRAMMAR_AVAILABLE`）。
  - [test_structured_output_grammar.py](d:\design\MindIE-LLM\tests\pythontest\cpu\text_generator\plugins\structured_output\test_structured_output_grammar.py)（覆盖 `StructuredOutputType`、`StructuredOutputRequest.from_response_format` 全部分支：none / json_object / json_schema with name / direct / 无 name / 空 name 等）。
  - [test_structured_output_bitmask.py](d:\design\MindIE-LLM\tests\pythontest\cpu\text_generator\plugins\structured_output\test_structured_output_bitmask.py)（覆盖 `apply_token_bitmask_inplace_npu` 三种 vocab 大小分支 + 异常 reraise）。
- 集成测试 [test_plugin_manager_structured.py](d:\design\MindIE-LLM\tests\pythontest\cpu\text_generator\plugins\test_plugin_manager_structured.py)：用 Mock 不依赖 NPU，验证 `PluginManager` 的 `_init_structured_output_manager` / `preprocess` / `postprocess` 路径。
- NPU 端 [test_pta_handlers.py:88-137](d:\design\MindIE-LLM\tests\pythontest\npu\text_generator\logits_handlers\test_pta_handlers.py)：`TestGuidedDecodingLogitsHandler` 5 个用例（bitmask=None / 导入成功 / 导入失败 / apply 异常 / 懒导入首次/已尝试），覆盖 sampler-side 全部失败路径。
- `test_batch_context.py` / `test_plugin_manager.py` / `test_generator.py` 也命中 `structured_output|response_format`（grep 4 文件），但都是间接覆盖（DictContext 字段、kwargs 透传等）。

### 5) Doc / config 反查

- 用户指南：`d:\design\MindIE-LLM\docs\` 全树 grep `structured_output|StructuredOutput|response_format|guided_decoding` 仅 **1 命中文件** [docs/zh/user_guide/feature/structured_output.md](d:\design\MindIE-LLM\docs\zh\user_guide\feature\structured_output.md)（214 行，含 json_object / json_schema 请求/响应样例 + xgrammar 后端说明）。**英文版不存在**——`docs/en/` 树 grep 0 命中。
- README / CHANGELOG：`d:\design\MindIE-LLM\` 顶层 `README*` / `CHANGELOG*` 文件 grep `structured_output|response_format` **0 命中**（即特性未在主 README 露出）。
- examples：`d:\design\MindIE-LLM\mindie_llm\examples\` 树 grep `structured_output|response_format|guided_decoding` **0 命中**（即 examples/ 没有结构化输出示例脚本）。

## See also

- [mindie/entities/PluginManager.md](../entities/PluginManager.md) — `_init_structured_output_manager` 与 `compute_structured_output_accepted` 在 PluginManager 层的位置，以及 "structured output 不走 importlib 循环" 的差异说明。
- [mindie/entities/Generator.md](../entities/Generator.md) — `Generator` 不直接持有 structured output 引用（grep 0 命中）。
- [mindie/topics/speculative.md](speculative.md) — 与 mtp 互斥的 anchor `infer_param.cpp:216-225`（已在 speculative.md §第 107 行引用）。
- [mindie/topics/connector.md](connector.md) — `input_metadata_builder.py` 的 PD 链路如何把 `response_format` / `predicted_token_ids` 从 protobuf 传到 Python `InputMetadata`。
- [mindie/topics/request-lifecycle.md](request-lifecycle.md) — request 进入 sampler 之前的整体生命周期。
- [comparison/dimensions.md §dim-structured](../../comparison/dimensions.md) — 三家结构化输出实现对照（已用本页 anchor 增强 MindIE cell）。
