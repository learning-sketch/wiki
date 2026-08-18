---
type: entity
project: vllm
status: verified
confidence: high
verified_against: 2026-08-18 (vLLM d29dc3ab, increment from 5f7fab88)
sources:
  - d:\design\vllm\vllm\v1\engine\output_processor.py
  - d:\design\vllm\vllm\v1\engine\detokenizer.py
  - d:\design\vllm\vllm\v1\engine\logprobs.py
  - d:\design\vllm\vllm\v1\engine\parallel_sampling.py
related:
  - vllm/entities/EngineCore.md
  - vllm/entities/LLMEngine.md
  - vllm/entities/AsyncLLM.md
  - vllm/topics/request-lifecycle.md
  - vllm/modules/engine.md
---

# `OutputProcessor` (and `RequestState` / `RequestOutputCollector`)

## Summary

`OutputProcessor` 将 [`EngineCoreOutput`](d:\design\vllm\vllm\v1\engine\__init__.py) 批次转为面向用户的 [`RequestOutput`](d:\design\vllm\vllm\outputs.py) / [`PoolingRequestOutput`](d:\design\vllm\vllm\outputs.py)，并在 `process_outputs` 的**唯一全批次循环**中完成统计、增量 detokenize、logprobs 与完成/中止处理（[output_processor.py:598-724](d:\design\vllm\vllm\v1\engine\output_processor.py)）。`AsyncLLM` 路径下每请求挂一个 `RequestOutputCollector`，由后台 output handler 调用 `put`、[`generate`](d:\design\vllm\vllm\v1\engine\async_llm.py) 协程 `get`/`get_nowait` 消费（分工见下文）；`LLMEngine` 路径无 queue，则把结果收集到返回列表（[output_processor.py:692-697](d:\design\vllm\vllm\v1\engine\output_processor.py)）。

## Sources

- 全文：[d:\design\vllm\vllm\v1\engine\output_processor.py](d:\design\vllm\vllm\v1\engine\output_processor.py)（849 行）
- 全文：[d:\design\vllm\vllm\v1\engine\detokenizer.py](d:\design\vllm\vllm\v1\engine\detokenizer.py)（362 行）
- 全文：[d:\design\vllm\vllm\v1\engine\logprobs.py](d:\design\vllm\vllm\v1\engine\logprobs.py)（352 行）
- 全文：[d:\design\vllm\vllm\v1\engine\parallel_sampling.py](d:\design\vllm\vllm\v1\engine\parallel_sampling.py)（150 行）
- 聚合引用：[d:\design\vllm\vllm\v1\engine\llm_engine.py](d:\design\vllm\vllm\v1\engine\llm_engine.py)
- 聚合引用：[d:\design\vllm\vllm\v1\engine\async_llm.py:433-745](d:\design\vllm\vllm\v1\engine\async_llm.py)

## 类层次（mermaid classDiagram）

```mermaid
classDiagram
    class RequestOutputCollector {
        +aggregate: bool
        +request_id: str
        +output
        +ready: asyncio.Event
        +put(output)
        +get() async
        +get_nowait()
        +close()
    }
    class StreamingUpdate {
        +prompt: str
        +prompt_token_ids: list
        +arrival_time: float
        +final: bool
    }
    class RequestState {
        +request_id: str
        +external_req_id: str
        +parent_req: ParentRequest
        +detokenizer: IncrementalDetokenizer
        +logprobs_processor: LogprobsProcessor
        +queue: RequestOutputCollector
        +is_prefilling: bool
        +streaming_input: bool
        +routed_experts_chunks: list
        +sampling_mask_chunks: list
        +make_request_output(...)
        +apply_streaming_update(...)
    }
    class OutputProcessor {
        +request_states: dict
        +parent_requests: dict
        +external_req_ids: defaultdict
        +lora_states: LoRARequestStates
        +add_request(...)
        +process_outputs(...)
        +abort_requests(...)
        +propagate_error(...)
        +update_scheduler_stats(...)
    }
    class ParentRequest {
        +child_requests: set
        +get_outputs(child_id, completion)
    }
    class IncrementalDetokenizer {
        +update(new_token_ids, stop_terminated)
        +get_next_output_text(finished, delta)
    }
    class LogprobsProcessor {
        +update_from_output(engine_core_output)
    }
    OutputProcessor "1" --> "*" RequestState : request_states
    RequestState --> "0..1" RequestOutputCollector : queue
    RequestState --> ParentRequest : parent_req
    RequestState --> IncrementalDetokenizer
    RequestState --> LogprobsProcessor
    RequestState ..> StreamingUpdate : input_chunk_queue
    ParentRequest ..> RequestState : fan-in n>1
```

## OutputProcessor 主 API

| API | 行号 | 作用 |
|-----|------|------|
| `__init__(tokenizer, log_stats=, stream_interval=, tracing_enabled=)` | [441-456](d:\design\vllm\vllm\v1\engine\output_processor.py) | 初始化 per-request 表、`parent_requests`、`external_req_ids` 映射与 LoRA 统计状态 |
| `get_num_unfinished_requests` / `has_unfinished_requests` | [458-462](d:\design\vllm\vllm\v1\engine\output_processor.py) | 未完成请求计数 |
| `propagate_error(e)` | [464-469](d:\design\vllm\vllm\v1\engine\output_processor.py) | 向所有未完成请求的 `queue` 注入异常（AsyncLLM output_handler 失败路径） |
| `abort_requests(request_ids, internal)` | [471-532](d:\design\vllm\vllm\v1\engine\output_processor.py) | 按外部/内部 ID 解析、移除状态、必要时对子请求递归中止，并产出 `FinishReason.ABORT` 的最终输出 |
| `add_request(request, prompt, parent_req=, request_index=, queue=)` | [534-563](d:\design\vllm\vllm\v1\engine\output_processor.py) | 新建或更新（流式输入）`RequestState`，登记父子与 external→internal 映射 |
| `_update_streaming_request_state` | [565-596](d:\design\vllm\vllm\v1\engine\output_processor.py) | 流式输入：排队 `StreamingUpdate` 或收尾（`STREAM_FINISHED`） |
| `process_outputs(engine_core_outputs, ...)` | [598-724](d:\design\vllm\vllm\v1\engine\output_processor.py) | 核心：统计 → detokenize/logprobs → 组装输出 → 清理；返回 `OutputProcessorOutput` |
| `_finish_request` | [726-738](d:\design\vllm\vllm\v1\engine\output_processor.py) | 从各映射中移除已完成请求 |
| `update_scheduler_stats` | [740-741](d:\design\vllm\vllm\v1\engine\output_processor.py) | 转发 LoRA 调度统计 |
| `do_tracing` / `_update_stats_from_output` / `_update_stats_from_finished` | [743-849](d:\design\vllm\vllm\v1\engine\output_processor.py) | 可观测性与迭代统计 |

## RequestState 字段表与状态机

- **未完成（生成中）**：仍在 `OutputProcessor.request_states` 中（[452](d:\design\vllm\vllm\v1\engine\output_processor.py)）；首次 decode 前 `is_prefilling` 为 True（[174](d:\design\vllm\vllm\v1\engine\output_processor.py)），在收到第一条含 `EngineCore` 输出后置 False，并顺带从 `prefill_stats` 记录 `num_cached_tokens` / `num_cache_creation_tokens`（[651-659](d:\design\vllm\vllm\v1\engine\output_processor.py)）。
- **finished**：`finish_reason is not None` 时 `_finish_request` 移除状态（[700-708](d:\design\vllm\vllm\v1\engine\output_processor.py)、[726-728](d:\design\vllm\vllm\v1\engine\output_processor.py)）；若 detokenizer 判停而 core 仍标记未 finish，则返回 `reqs_to_abort` 供上层中止 core（[709-712](d:\design\vllm\vllm\v1\engine\output_processor.py)）。
- **aborted**：`abort_requests` 中弹出状态并构造 `FinishReason.ABORT` 输出（[503-524](d:\design\vllm\vllm\v1\engine\output_processor.py)）。
- **流式输入**：`streaming_input` 与 `input_chunk_queue` 控制是否在 finish 后合并下一段 prompt（[189-193](d:\design\vllm\vllm\v1\engine\output_processor.py)、[701-706](d:\design\vllm\vllm\v1\engine\output_processor.py)）；未完成多段时强制 `request_output.finished = False`（[689-690](d:\design\vllm\vllm\v1\engine\output_processor.py)）。

构造子与依赖见 `RequestState.__init__` 与 `from_new_request`（[131-277](d:\design\vllm\vllm\v1\engine\output_processor.py)）：无 `sampling_params` 时为 pooling，仅 `pooling_params.output_kind`（[245-253](d:\design\vllm\vllm\v1\engine\output_processor.py)）。

## RequestOutputCollector

- **设计**：`asyncio.Event` `ready` + 单槽 `output`（[56-62](d:\design\vllm\vllm\v1\engine\output_processor.py)）；`put` 非阻塞（[64-78](d:\design\vllm\vllm\v1\engine\output_processor.py)），`get` 在 `output` 为空前等待（[80-88](d:\design\vllm\vllm\v1\engine\output_processor.py)）。
- **与 AsyncLLM.generate 的分工**：`AsyncLLM._add_request` 把同一 `queue` 交给 `OutputProcessor.add_request`（[async_llm.py:433](d:\design\vllm\vllm\v1\engine\async_llm.py)）；后台 `output_handler` 调用 `process_outputs` 后断言同步路径不返回列表（`assert not processed_outputs.request_outputs`，[async_llm.py:704-709](d:\design\vllm\vllm\v1\engine\async_llm.py)）；`generate` 从 `q` 拉取并 `yield`（[async_llm.py:601-614](d:\design\vllm\vllm\v1\engine\async_llm.py)）。消费侧细节以 [`AsyncLLM.md`](AsyncLLM.md) 为准。
- **synthesis: 为何不是无界 `asyncio.Queue`**：源码仅保证「单槽 + 可合并」——`DELTA` 时在 `put` 内对已存在的 `RequestOutput.add(..., aggregate=True)`（[66-74](d:\design\vllm\vllm\v1\engine\output_processor.py)），类文档说明生产者快于消费者时合并流式增量（[48-54](d:\design\vllm\vllm\v1\engine\output_processor.py)），从而避免为每个 step 堆积独立消息。

## process_outputs 完整流程

```mermaid
sequenceDiagram
    participant EC as EngineCoreOutput batch
    participant OP as OutputProcessor
    participant RS as RequestState
    participant DT as IncrementalDetokenizer
    participant LP as LogprobsProcessor
    participant Q as RequestOutputCollector
    EC->>OP: for each EngineCoreOutput
    OP->>RS: lookup request_states
    alt missing 已 abort
        OP-->>OP: continue
    else
        OP->>OP: _update_stats_from_output
        alt pooling_output is None 生成
            OP->>DT: update(new_token_ids, ...)
            DT-->>OP: stop_string?
            OP->>LP: update_from_output
        end
        OP->>RS: make_request_output
        alt queue is not None
            RS->>Q: put RequestOutput
        else
            OP-->>OP: append request_outputs
        end
        alt finish_reason
            OP->>OP: streaming chunk / _finish_request
        end
    end
```

分节（锚点均在 [output_processor.py:598-724](d:\design\vllm\vllm\v1\engine\output_processor.py)）：

1. **统计**：`_update_stats_from_output`（[635-638](d:\design\vllm\vllm\v1\engine\output_processor.py)）。
2. **增量 detokenize**：若 `pooling_output is None`，`detokenizer.update`（[661-674](d:\design\vllm\vllm\v1\engine\output_processor.py)）；停词命中则覆盖 `finish_reason`/`stop_reason`（[672-674](d:\design\vllm\vllm\v1\engine\output_processor.py)）。
3. **logprobs**：`logprobs_processor.update_from_output`（[676-678](d:\design\vllm\vllm\v1\engine\output_processor.py)）。
4. **组装与输出**：`make_request_output`（[680-697](d:\design\vllm\vllm\v1\engine\output_processor.py)）。
5. **finish / 流式下一段**：`finish_reason is not None` 分支（[699-719](d:\design\vllm\vllm\v1\engine\output_processor.py)）。

## ParentRequest（n>1 / best_of）协作

- 子请求 ID 形如 `{index}_{parent_id}`（[parallel_sampling.py:92-94](d:\design\vllm\vllm\v1\engine\parallel_sampling.py)）；`ParentRequest.get_outputs` 在流式模式下每步返回当前子 completion（[parallel_sampling.py:115-119](d:\design\vllm\vllm\v1\engine\parallel_sampling.py)），`FINAL_ONLY` 时聚齐 `n` 份后一次性返回（[parallel_sampling.py:121-123](d:\design\vllm\vllm\v1\engine\parallel_sampling.py)）。
- `RequestState.make_request_output` 在 `parent_req is not None` 时委托 `get_outputs` 并改用父级 `external_req_id`（[329-335](d:\design\vllm\vllm\v1\engine\output_processor.py)）。
- `OutputProcessor.add_request` 登记 `parent_requests[parent_req.request_id]`（[559-560](d:\design\vllm\vllm\v1\engine\output_processor.py)）；`abort_requests` 可通过父 ID 递归中止子请求（[525-531](d:\design\vllm\vllm\v1\engine\output_processor.py)）。
- 统计侧 `ParentRequest.observe_finished_request` 在 `_update_stats_from_finished` 末尾调用（[output_processor.py:847-849](d:\design\vllm\vllm\v1\engine\output_processor.py)）。

## Detokenizer / Logprobs 协作要点

- **Detokenizer**：`IncrementalDetokenizer.from_new_request` 在 `detokenize=False` 时退回空操作实现（[detokenizer.py:57-59](d:\design\vllm\vllm\v1\engine\detokenizer.py)）；Fast/Slow 路径见 [detokenizer.py:61-67](d:\design\vllm\vllm\v1\engine\detokenizer.py)（Fast 路径类型判定已由 `PreTrainedTokenizerFast` 改为 `TokenizersBackend`，见增量小节）。`_new_completion_output` 中 `get_next_output_text` 与 `DELTA` 下 token 切片（[output_processor.py:402-410](d:\design\vllm\vllm\v1\engine\output_processor.py)）。**`stream_interval > 1`** 时按 detokenizer 计数以降低输出频率（[295-316](d:\design\vllm\vllm\v1\engine\output_processor.py)）。
- **Logprobs**：`LogprobsProcessor.update_from_output` 聚合采样与 prompt logprobs（[logprobs.py:348-352](d:\design\vllm\vllm\v1\engine\logprobs.py)）；`DELTA` 模式下 `pop_prompt_logprobs` 在组装 `RequestOutput` 时清空 prompt 侧缓存（[output_processor.py:370-374](d:\design\vllm\vllm\v1\engine\output_processor.py)）。

## RequestOutput vs PoolingRequestOutput 分发

- **Pooling**：`pooling_output is not None` 时跳过 detokenize/logprobs 更新（[661-678](d:\design\vllm\vllm\v1\engine\output_processor.py)），`make_request_output` 走 `PoolingRequestOutput` 分支（[320-325](d:\design\vllm\vllm\v1\engine\output_processor.py)、[359-368](d:\design\vllm\vllm\v1\engine\output_processor.py)）。
- **生成**：构造 `CompletionOutput` 与 `RequestOutput`（[327-389](d:\design\vllm\vllm\v1\engine\output_processor.py)、[391-432](d:\design\vllm\vllm\v1\engine\output_processor.py)）。

## structured output / reasoning 相关说明

`OutputProcessor` 与 `RequestState` **未**出现 `structured` / `reasoning` 字段或分支（在 [output_processor.py](d:\design\vllm\vllm\v1\engine\output_processor.py) 内 grep 0 处，2026-08-18 复核仍成立）。`reasoning_ended` 由 `AsyncLLM.add_request` 写入 `EngineCoreRequest`（[async_llm.py:387-388](d:\design\vllm\vllm\v1\engine\async_llm.py)），属输入/调度侧；本层仍按普通 token 流做 detokenize 与 logprobs。

## abort / finish / error propagation 路径

- **Abort**：见 `abort_requests`（[471-532](d:\design\vllm\vllm\v1\engine\output_processor.py)）；生成路径用 `pooling_output=None`，中止 pooling 请求时用占位 `EMPTY_CPU_TENSOR` 进入 pooling 分支（[44](d:\design\vllm\vllm\v1\engine\output_processor.py)、[510-522](d:\design\vllm\vllm\v1\engine\output_processor.py)）。
- **流式输入结束**：`STREAM_FINISHED` 入队（[577](d:\design\vllm\vllm\v1\engine\output_processor.py)）。
- **全局错误**：`propagate_error`（[464-469](d:\design\vllm\vllm\v1\engine\output_processor.py)），由 `async_llm.output_handler` except 路径调用（[async_llm.py:743-745](d:\design\vllm\vllm\v1\engine\async_llm.py)）。

## hidden state（§9）

| 状态 | 位置 | 说明 |
|------|------|------|
| `request_states` | [452](d:\design\vllm\vllm\v1\engine\output_processor.py) | 每内部请求 ID 一条 `RequestState` |
| `parent_requests` / `external_req_ids` | [453-454](d:\design\vllm\vllm\v1\engine\output_processor.py) | 并行采样与外部 ID 映射 |
| `routed_experts_chunks` / `sampling_mask_chunks` | [181-183](d:\design\vllm\vllm\v1\engine\output_processor.py) | 本期新增：跨 step 累积 routed experts / sampling mask，finish 时合并（见增量小节） |
| Detokenizer 缓冲 | [detokenizer.py](d:\design\vllm\vllm\v1\engine\detokenizer.py)（如 `output_text`、`token_ids`、Fast 路径 `DecodeStream`） | 增量解码与停词缓冲 |
| Logprobs 累积 | [logprobs.py](d:\design\vllm\vllm\v1\engine\logprobs.py) | `logprobs` / `prompt_logprobs` / `cumulative_logprob` |
| `RequestOutputCollector.output` | [59-60](d:\design\vllm\vllm\v1\engine\output_processor.py) | 单槽 + Event，与 AsyncLLM 协程交接 |

## §5 step 3 hidden cross-reference grep 结果

| # | 类别 | 结果 |
|---|------|------|
| 1 | 跨语言绑定 | `OutputProcessor`：在 `d:\design\vllm\csrc\` 全树 grep **0** 命中 |
| 2 | 协作伙伴跨子系统引用 | `OutputProcessor` / `RequestOutputCollector` / `IncrementalDetokenizer` / `LogprobsProcessor` 联合模式：在 `d:\design\vllm\vllm\entrypoints\` 全树 grep **0** 命中；在 `d:\design\vllm\tests\` 全树 grep **51** 命中（分布于 11 个文件）；在 `d:\design\vllm\benchmarks\` 全树 grep **0** 命中；在 `d:\design\vllm\examples\` 全树 grep **0** 命中 |
| 3 | 配置 / IPC 共享数据结构 | `EngineCoreOutputs`：在 `d:\design\vllm\` 下 `*.py` grep **62** 命中；`EngineCoreOutput`：**79** 命中；`PoolingRequestOutput`：**109** 命中；`SchedulerStats`：**39** 命中（均为跨模块工程内引用总量，2026-04-18 统计口径） |
| 4 | 测试覆盖反查 | [`test_output_processor.py`](d:\design\vllm\tests\v1\engine\test_output_processor.py) 存在；**不存在** `tests/v1/engine/test_detokenizer.py`（在 `d:\design\vllm\tests\` glob `test_detokenizer*.py` **0** 文件）。Detokenizer 相关见 [`tests\tokenizers_\test_detokenize.py`](d:\design\vllm\tests\tokenizers_\test_detokenize.py)、[`tests\v1\engine\test_fast_incdec_prefix_err.py`](d:\design\vllm\tests\v1\engine\test_fast_incdec_prefix_err.py)、[`tests\detokenizer\`](d:\design\vllm\tests\detokenizer) 等 |
| 5 | doc / config 反查 | `OutputProcessor`：在 `d:\design\vllm\docs\` 全树 grep **0** 命中；`detokeniz`（大小写不敏感）：**5** 命中（4 个文件）；`logprob`（大小写不敏感）：**36** 命中（12 个文件）；`output process` / `OutputProcessing`（大小写不敏感）：**7** 命中（5 个文件）。设计向概述见 [arch_overview.md](d:\design\vllm\docs\design\arch_overview.md)、[model_runner_v2.md](d:\design\vllm\docs\design\model_runner_v2.md) |

## Increment 2026-08-18 (vLLM 5f7fab88 → d29dc3ab)

本组四文件本期改动较小（output_processor.py +45/-8、detokenizer.py +57、logprobs.py 1 行、parallel_sampling.py 无变化），output_processor.py 812 → 849 行。类层次与主循环结构不变，全部行号锚点已按 HEAD 重校。功能性增量：

- **routed experts 改为跨 step 累积**（上游 commit `c7560af424`，"Replace shared-memory routed experts with ModelRunnerOutput transfer"）：旧版 `make_request_output` 直接透传单 step 的 `routed_experts` 参数；现改为 `RequestState.routed_experts_chunks` 列表逐 step 追加（[output_processor.py:181-182](d:\design\vllm\vllm\v1\engine\output_processor.py)、[646-649](d:\design\vllm\vllm\v1\engine\output_processor.py)），finish 时 `np.concatenate` 一次性写入 `CompletionOutput.routed_experts`（[417-420](d:\design\vllm\vllm\v1\engine\output_processor.py)）。
- **Mask Replay / sampling_mask**（上游 commit `50ba4bc6b2`）：`EngineCoreOutput.new_sampling_mask`（[engine/__init__.py:225](d:\design\vllm\vllm\v1\engine\__init__.py)）在生成分支累积到 `sampling_mask_chunks`（[664-667](d:\design\vllm\vllm\v1\engine\output_processor.py)），finish 时 `SamplingMaskLists.merge` 合并为 `CompletionOutput.sampling_mask`（[412-415](d:\design\vllm\vllm\v1\engine\output_processor.py)）。
- **per-request `stream_interval`**（上游 commit `8040ef2426`）：`SamplingParams.stream_interval`（[sampling_params.py:318](d:\design\vllm\vllm\sampling_params.py)）可按请求覆盖引擎级值，`from_new_request` 中取两者较大值 clamp（[output_processor.py:230-232](d:\design\vllm\vllm\v1\engine\output_processor.py)）。
- **`num_cache_creation_tokens`**（上游 commit `a287eb163f`）：prefill 首包从 `prefill_stats` 同时记录 `num_cached_tokens` 与新增的 `num_cache_creation_tokens`（[651-659](d:\design\vllm\vllm\v1\engine\output_processor.py)），随 `RequestOutput` 返回（[387](d:\design\vllm\vllm\v1\engine\output_processor.py)）。
- **EC (encoder cache) transfer params**（上游 commit `8df14cfc8c`）：`make_request_output` 新增 `ec_transfer_params` 参数（[286](d:\design\vllm\vllm\v1\engine\output_processor.py)），与 `kv_transfer_params` 并列穿透到 `RequestOutput`（[385](d:\design\vllm\vllm\v1\engine\output_processor.py)、[645](d:\design\vllm\vllm\v1\engine\output_processor.py)），对应 `EngineCoreOutput.ec_transfer_params`（[engine/__init__.py:209](d:\design\vllm\vllm\v1\engine\__init__.py)）。
- **Detokenizer 两处细节**：(a) Fast 路径类型判定由 `PreTrainedTokenizerFast` 改为 `TokenizersBackend`，且 `DecodeStream` 改为经 `tokenizers.decoders` 模块属性查找以兼容 fastokens shim 的运行时替换（[detokenizer.py:61](d:\design\vllm\vllm\v1\engine\detokenizer.py)、[184-187](d:\design\vllm\vllm\v1\engine\detokenizer.py)、[244-246](d:\design\vllm\vllm\v1\engine\detokenizer.py)）；(b) `check_stop_strings` 重写为"在新文本中最早完成的停词优先"，修正投机解码单 step 追加多 token 时多个停词同时命中的选择语义（[detokenizer.py:310-362](d:\design\vllm\vllm\v1\engine\detokenizer.py)）。
- **Logprobs 单行改动**：`from_new_request` 改读 `sampling_params.num_logprobs`（原 `.logprobs`，属 SamplingParams 字段重命名跟进，[logprobs.py:50](d:\design\vllm\vllm\v1\engine\logprobs.py)）。

synthesis: 本页核心论断（唯一全批次循环、单槽 collector、streaming input 状态机、ParentRequest fan-in、abort/finish 路径）在本期 4273 commits 后全部仍然成立；本文件组的变化均为"随 `RequestOutput` 携带更多信息"型增量（routed experts / sampling mask / cache creation tokens / EC transfer params），未触及处理管线结构。本期 vLLM 新增的 `vllm/v1/worker/gpu/` 新一代 model runner 不影响本页——`OutputProcessor` 仅消费 `EngineCoreOutput`，与 worker 实现解耦。

## Notes / Caveats

> [!todo] VERIFY: **单批次循环约束** —— `process_outputs` 注释要求 V1 仅此一处遍历整批 `EngineCoreOutput`（[616-623](d:\design\vllm\vllm\v1\engine\output_processor.py)）。需 verify 全代码库无第二处全批次遍历（grep `for output in engine_core_outputs` 等模式）。

> ~~[!todo] VERIFY: **AsyncLLM 分块** —— `VLLM_V1_OUTPUT_PROC_CHUNK_SIZE` 仅切片循环，不改变单切片内处理语义；分块大小默认值与运维语义需对照 envs.py。~~ **RESOLVED 2026-08-18**：默认值 **128**（[envs.py:169](d:\design\vllm\vllm\envs.py)、[envs.py:1431-1432](d:\design\vllm\vllm\envs.py)）；分块循环在 `output_handler` 内仅做切片、每片独立调用 `process_outputs`，语义不变（[async_llm.py:684](d:\design\vllm\vllm\v1\engine\async_llm.py)、[698-709](d:\design\vllm\vllm\v1\engine\async_llm.py)）。

## See also

- [vllm/entities/EngineCore.md](EngineCore.md)
- [vllm/entities/LLMEngine.md](LLMEngine.md)
- [vllm/entities/AsyncLLM.md](AsyncLLM.md)
- [vllm/topics/request-lifecycle.md](../topics/request-lifecycle.md)
- [vllm/modules/engine.md](../modules/engine.md)
