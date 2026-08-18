---
type: entity
project: vllm
status: verified
confidence: high
verified_against: 2026-08-18 (vLLM d29dc3ab, increment from 5f7fab88)
sources:
  - d:\design\vllm\vllm\v1\engine\async_llm.py
  - d:\design\vllm\vllm\v1\engine\output_processor.py:47-109
  - d:\design\vllm\vllm\v1\engine\llm_engine.py:48-296
  - d:\design\vllm\vllm\v1\engine\core_client.py:89-139
  - d:\design\vllm\vllm\engine\async_llm_engine.py
  - d:\design\vllm\vllm\engine\protocol.py:41-56
  - d:\design\vllm\vllm\entrypoints\openai\api_server.py:102-180
related:
  - vllm/entities/EngineCore.md
  - vllm/entities/EngineCoreClient.md
  - vllm/entities/LLMEngine.md
  - vllm/entities/OutputProcessor.md
  - vllm/topics/request-lifecycle.md
  - vllm/topics/multiproc-ipc.md
  - vllm/modules/engine.md
---

# `AsyncLLM` (v1)

## Summary

`AsyncLLM` 实现 [`EngineClient`](d:\design\vllm\vllm\engine\protocol.py) 协议，在**前端进程**内组合 [`InputProcessor`](d:\design\vllm\vllm\v1\engine\async_llm.py)（[138](d:\design\vllm\vllm\v1\engine\async_llm.py)）、[`OutputProcessor`](d:\design\vllm\vllm\v1\engine\async_llm.py)（[141-146](d:\design\vllm\vllm\v1\engine\async_llm.py)）与 [`EngineCoreClient.make_async_mp_client`](d:\design\vllm\vllm\v1\engine\async_llm.py)（[149-156](d:\design\vllm\vllm\v1\engine\async_llm.py)），并用常驻 **`asyncio.create_task` 的 `output_handler`** 循环 [`await engine_core.get_output_async()`](d:\design\vllm\vllm\v1\engine\async_llm.py)（[690](d:\design\vllm\vllm\v1\engine\async_llm.py)）→ [`output_processor.process_outputs`](d:\design\vllm\vllm\v1\engine\async_llm.py)（[705-707](d:\design\vllm\vllm\v1\engine\async_llm.py)），把结果写入每请求的 [`RequestOutputCollector`](d:\design\vllm\vllm\v1\engine\async_llm.py)（[400](d:\design\vllm\vllm\v1\engine\async_llm.py)）。对外主入口为 [`generate`](d:\design\vllm\vllm\v1\engine\async_llm.py)（[550-663](d:\design\vllm\vllm\v1\engine\async_llm.py)）/ [`encode`](d:\design\vllm\vllm\v1\engine\async_llm.py)（[843-924](d:\design\vllm\vllm\v1\engine\async_llm.py)）异步生成器与 [`add_request`](d:\design\vllm\vllm\v1\engine\async_llm.py)（[283-422](d:\design\vllm\vllm\v1\engine\async_llm.py)）。

## Sources

- 全文：[d:\design\vllm\vllm\v1\engine\async_llm.py](d:\design\vllm\vllm\v1\engine\async_llm.py)（1166 行，增量前约 1067 行）
- `RequestOutputCollector`：[d:\design\vllm\vllm\v1\engine\output_processor.py:47-109](d:\design\vllm\vllm\v1\engine\output_processor.py)
- 同步对照 `LLMEngine`：[d:\design\vllm\vllm\v1\engine\llm_engine.py](d:\design\vllm\vllm\v1\engine\llm_engine.py)
- 公开别名：`AsyncLLMEngine` → `AsyncLLM`：[d:\design\vllm\vllm\engine\async_llm_engine.py](d:\design\vllm\vllm\engine\async_llm_engine.py)
- OpenAI 入口装配：[d:\design\vllm\vllm\entrypoints\openai\api_server.py:102-180](d:\design\vllm\vllm\entrypoints\openai\api_server.py)

## 类层次 + 字段表

- **继承**：`class AsyncLLM(EngineClient)`（[72](d:\design\vllm\vllm\v1\engine\async_llm.py)）
- **公开别名**：`AsyncLLMEngine = AsyncLLM`（[4-7](d:\design\vllm\vllm\engine\async_llm_engine.py)）
- **本期新增**：模块级异常 `InputStreamError`（流式输入 generator 抛错的包装，[60-69](d:\design\vllm\vllm\v1\engine\async_llm.py)）

```mermaid
classDiagram
    class AsyncLLM {
        +VllmConfig vllm_config
        +ModelConfig model_config
        +ObservabilityConfig observability_config
        +BaseRenderer renderer
        +InputProcessor input_processor
        +OutputProcessor output_processor
        +EngineCoreClient engine_core
        +StatLoggerManager logger_manager
        +asyncio.Task output_handler
        +asyncio.Lock _elastic_ep_lock
        +list _logger_ref
        +Profiler profiler
    }
    class EngineClient {
        <<abstract>>
    }
    EngineClient <|-- AsyncLLM
```

| 字段 / 属性 | 类型（源码） | 说明 |
|---|---|---|
| `vllm_config` / `model_config` / `observability_config` | 配置 | [async_llm.py:112-115](d:\design\vllm\vllm\v1\engine\async_llm.py) |
| `_elastic_ep_lock` | `asyncio.Lock` | **本期新增**：串行化 `scale_elastic_ep`（[async_llm.py:113, 1055](d:\design\vllm\vllm\v1\engine\async_llm.py)） |
| `renderer` | renderer | [async_llm.py:135](d:\design\vllm\vllm\v1\engine\async_llm.py) |
| `input_processor` | `InputProcessor` | [async_llm.py:138](d:\design\vllm\vllm\v1\engine\async_llm.py) |
| `output_processor` | `OutputProcessor` | [async_llm.py:141-146](d:\design\vllm\vllm\v1\engine\async_llm.py) |
| `engine_core` | `EngineCoreClient` | `EngineCoreClient.make_async_mp_client` [149-156](d:\design\vllm\vllm\v1\engine\async_llm.py) |
| `logger_manager` | `StatLoggerManager \| None` | [async_llm.py:159-169](d:\design\vllm\vllm\v1\engine\async_llm.py)；本期新增 stat logger 插件加载 `load_stat_logger_plugin_factories`（[123-133](d:\design\vllm\vllm\v1\engine\async_llm.py)） |
| `output_handler` | `asyncio.Task \| None` | [async_llm.py:173](d:\design\vllm\vllm\v1\engine\async_llm.py)、`_run_output_handler` [665-747](d:\design\vllm\vllm\v1\engine\async_llm.py) |
| `_logger_ref` | `list`（惰性创建） | 供 `output_handler` 间接引用 logger，避免环引用；`scale_elastic_ep` 会更新 [676-680, 1084-1085](d:\design\vllm\vllm\v1\engine\async_llm.py) |
| `profiler` | `torch.profiler.profile \| None` | [async_llm.py:181-203](d:\design\vllm\vllm\v1\engine\async_llm.py)；本期改由 `vllm_config.profiler_config` 驱动（原为环境变量） |
| `_supported_tasks` | 惰性缓存 | `get_supported_tasks` [276-281](d:\design\vllm\vllm\v1\engine\async_llm.py) |
| `tokenizer` / `get_tokenizer` | 委托 `renderer` | [async_llm.py:926-931](d:\design\vllm\vllm\v1\engine\async_llm.py) |

**async vs sync（`LLMEngine`）**：`LLMEngine` 用 `EngineCoreClient.make_client(..., asyncio_mode=False)`（[llm_engine.py:105-111](d:\design\vllm\vllm\v1\engine\llm_engine.py)），无 `output_handler` 任务；`AsyncLLM` 用 `make_async_mp_client`（[async_llm.py:149-156](d:\design\vllm\vllm\v1\engine\async_llm.py)）并启动前述异步拉取循环。`LLMEngine.add_request` 为同步、返回 `str` request id（[llm_engine.py:218-296](d:\design\vllm\vllm\v1\engine\llm_engine.py)）；`AsyncLLM.add_request` 为 `async def`，返回 `RequestOutputCollector`（[async_llm.py:283-422](d:\design\vllm\vllm\v1\engine\async_llm.py)）。

## 主 API

| API | 行号 | sync/async | 作用 |
|---|---|---|---|
| `__init__` | [75-203](d:\design\vllm\vllm\v1\engine\async_llm.py) | sync（内联 `try: asyncio.get_running_loop()` 时可能**同步**启动 `_run_output_handler`） | 组装 processor / `EngineCoreClient` / 统计 / profiler；[173-179](d:\design\vllm\vllm\v1\engine\async_llm.py) |
| `from_vllm_config` | [205-232](d:\design\vllm\vllm\v1\engine\async_llm.py) | sync `@classmethod` | 选 `Executor.get_class` 并构造 |
| `from_engine_args` | [234-257](d:\design\vllm\vllm\v1\engine\async_llm.py) | sync `@classmethod` | 从 `AsyncEngineArgs` 建配置 |
| `shutdown` | [262-274](d:\design\vllm\vllm\v1\engine\async_llm.py) | sync | **本期新增 `timeout` 参数**（透传给 `engine_core.shutdown(timeout=)`，配合引擎侧优雅 drain）；顺序 `shutdown_prometheus`、`renderer.shutdown` [264-267](d:\design\vllm\vllm\v1\engine\async_llm.py)、`engine_core.shutdown` [269-270](d:\design\vllm\vllm\v1\engine\async_llm.py)、`cancel_task_threadsafe(output_handler)` [272-274](d:\design\vllm\vllm\v1\engine\async_llm.py) |
| `get_supported_tasks` | [276-281](d:\design\vllm\vllm\v1\engine\async_llm.py) | **async** | 缓存 `_supported_tasks` |
| `add_request` | [283-422](d:\design\vllm\vllm\v1\engine\async_llm.py) | **async** | 输入处理、`n>1` 扇出、返回 `RequestOutputCollector`；流式输入走 `_add_streaming_input_request` [441-527](d:\design\vllm\vllm\v1\engine\async_llm.py)；本期新增 `session_id` / `reasoning_parser_kwargs` 参数与 `kv_sharing_fast_prefill`+`prompt_logprobs` 校验（[309-318](d:\design\vllm\vllm\v1\engine\async_llm.py)） |
| `generate` | [550-663](d:\design\vllm\vllm\v1\engine\async_llm.py) | **async generator** | 迭代 collector 直至 `finished` |
| `encode` | [843-924](d:\design\vllm\vllm\v1\engine\async_llm.py) | **async generator** | pooling 流式输出 |
| `abort` | [749-761](d:\design\vllm\vllm\v1\engine\async_llm.py) | **async** | `output_processor` + `engine_core.abort_requests_async` |
| `notify_kv_transfer_request_rejected` | [763-788](d:\design\vllm\vllm\v1\engine\async_llm.py) | **async** | **本期新增**：构造 `abort_immediately=True` 的假请求，让 KV connector 的 `request_finished` 钩子释放 P 节点上被 pin 的 prefill block（NIXL pre-admission rejection） |
| `pause_generation` / `resume_generation` / `is_paused` | [790-841](d:\design\vllm\vllm\v1\engine\async_llm.py) | **async** | 委托 `engine_core` 调度暂停；本期签名改为 `mode: PauseMode`（abort/wait/keep）+ `clear_cache`，`wait_for_inflight_requests` 参数 deprecated（[815-823](d:\design\vllm\vllm\v1\engine\async_llm.py)） |
| `check_health` / `do_log_stats` / `is_tracing_enabled` | [933-943](d:\design\vllm\vllm\v1\engine\async_llm.py) | **async** | 健康与统计 |
| `start_profile` / `stop_profile` | [945-956](d:\design\vllm\vllm\v1\engine\async_llm.py) | **async** | `engine_core.profile_async` + 可选 CPU profiler |
| `reset_mm_cache` / `reset_prefix_cache` / `reset_encoder_cache` | [957-970](d:\design\vllm\vllm\v1\engine\async_llm.py) | **async** | 运维 |
| `sleep` / `wake_up` / `is_sleeping` | [971-993](d:\design\vllm\vllm\v1\engine\async_llm.py) | **async** | 委托 core；本期 `sleep` 新增 `mode: PauseMode` 参数；新增 `checkpoint_prepare` / `checkpoint_restore`（[985-990](d:\design\vllm\vllm\v1\engine\async_llm.py)） |
| LoRA：`add_lora` / `remove_lora` / `list_loras` / `pin_lora` | [994-1009](d:\design\vllm\vllm\v1\engine\async_llm.py) | **async** | 委托 core |
| `collective_rpc` | [1010-1022](d:\design\vllm\vllm\v1\engine\async_llm.py) | **async** | 分布式 RPC |
| `wait_for_requests_to_drain` / `scale_elastic_ep` | [1024-1094](d:\design\vllm\vllm\v1\engine\async_llm.py) | **async** | DP / 弹性 EP；本期 EEP 改两段式 `prepare_elastic_ep`（[1069](d:\design\vllm\vllm\v1\engine\async_llm.py)）→ `commit_elastic_ep`（[1092](d:\design\vllm\vllm\v1\engine\async_llm.py)），并以 `_elastic_ep_lock` 串行化（[1055](d:\design\vllm\vllm\v1\engine\async_llm.py)） |
| `handle_fault` / `get_status` | [1096-1103](d:\design\vllm\vllm\v1\engine\async_llm.py) | **async** | **本期新增**：fault tolerance 指令与状态查询（委托 `engine_core`） |
| `is_running` / `is_stopped` / `errored` / `dead_error` | [1105-1120](d:\design\vllm\vllm\v1\engine\async_llm.py) | sync `@property` | 状态；`errored` 综合 `engine_core.resources.engine_dead` 与 `is_running` [1114-1116](d:\design\vllm\vllm\v1\engine\async_llm.py) |
| RL 权重族：`init_weight_transfer_engine` / `start_weight_update` / `start_draft_weight_update` / `update_weights` / `finish_weight_update` / `update_weight_version` / `get_weight_version` | [1122-1166](d:\design\vllm\vllm\v1\engine\async_llm.py) | **async** | RL 权重更新；本期扩展出 start/finish 生命周期与 weight version 标签（经 `engine_core.set_weight_version_async`，[1160-1166](d:\design\vllm\vllm\v1\engine\async_llm.py)） |
| `_run_output_handler` | [665-747](d:\design\vllm\vllm\v1\engine\async_llm.py) | sync（内部 `async def output_handler` + **`asyncio.create_task`**） | 后台拉取 core 输出 |

## 异步运行模型

1. **启动时机**：`__init__` [173-179](d:\design\vllm\vllm\v1\engine\async_llm.py) 若已有 running loop 则调用 `_run_output_handler` [665-747](d:\design\vllm\vllm\v1\engine\async_llm.py)；否则延迟到首次 `add_request` [394-397](d:\design\vllm\vllm\v1\engine\async_llm.py)（注释说明允许在 **无 event loop** 下构造，便于 OpenAI server 捕获启动失败）。
2. **`output_handler` 协程**：死循环 `await engine_core.get_output_async()` [690](d:\design\vllm\vllm\v1\engine\async_llm.py)，按 `VLLM_V1_OUTPUT_PROC_CHUNK_SIZE` [684](d:\design\vllm\vllm\v1\engine\async_llm.py) 分块调用 `output_processor.process_outputs` [705-707](d:\design\vllm\vllm\v1\engine\async_llm.py)，块间 `await asyncio.sleep(0)` [721-723](d:\design\vllm\vllm\v1\engine\async_llm.py)；异常时 `output_processor.propagate_error` [743-745](d:\design\vllm\vllm\v1\engine\async_llm.py)。本期新增块内两步：(a) 处理 `mm_cache_miss_hashes`——engine 报多模态 P0/P1 cache drift 时在前端 shadow cache `invalidate` 对应 hash（[711-719](d:\design\vllm\vllm\v1\engine\async_llm.py)）；(b) stop-string 触发的 `reqs_to_abort` 改在分块循环内立即 `abort_requests_async`（[725-729](d:\design\vllm\vllm\v1\engine\async_llm.py)）。
3. **派发至请求**：`OutputProcessor.process_outputs` 将 `RequestOutput` 写入对应 `RequestOutputCollector.put` [output_processor.py:64-78](d:\design\vllm\vllm\v1\engine\output_processor.py)（非 `asyncio.Queue`，而是 **`asyncio.Event` + 单槽** `output` [output_processor.py:56-62](d:\design\vllm\vllm\v1\engine\output_processor.py)）。
4. **消费者**：`generate` 中 `q.get_nowait() or await q.get()` [605-607](d:\design\vllm\vllm\v1\engine\async_llm.py)（优先非阻塞取，减少负载下任务切换）。

```mermaid
sequenceDiagram
    participant Caller as 调用方 (generate)
    participant AL as AsyncLLM
    participant OP as OutputProcessor
    participant EC as EngineCoreClient
    participant OH as output_handler task
    participant Q as RequestOutputCollector

    Caller->>AL: await add_request(...)
    AL->>OP: add_request(...) + queue
    AL->>EC: await add_request_async(...)
    loop 后台
        OH->>EC: await get_output_async()
        EC-->>OH: EngineCoreOutputs
        OH->>OP: process_outputs(...)
        OP->>Q: put(RequestOutput)
    end
    Caller->>Q: get_nowait / await get
    Q-->>Caller: RequestOutput
```

## `generate()` async generator 完整流程

1. `await add_request(...)` [586-599](d:\design\vllm\vllm\v1\engine\async_llm.py) 得到 `RequestOutputCollector` `q`。
2. `while not finished` [603-614](d:\design\vllm\vllm\v1\engine\async_llm.py)：`out = q.get_nowait() or await q.get()`，`yield out` [613-614](d:\design\vllm\vllm\v1\engine\async_llm.py)（`STREAM_FINISHED` 不 yield）。
3. 取消 / 断开：`asyncio.CancelledError` / `GeneratorExit` → `await abort(q.request_id, internal=True)` [619-624](d:\design\vllm\vllm\v1\engine\async_llm.py)。
4. `EngineDeadError` [627-630](d:\design\vllm\vllm\v1\engine\async_llm.py)、`VLLMClientError`（本期取代原 `ValueError` 分支，[633-636](d:\design\vllm\vllm\v1\engine\async_llm.py)）、`InputStreamError` [639-644](d:\design\vllm\vllm\v1\engine\async_llm.py)、其他 `Exception` → `EngineGenerateError` [647-660](d:\design\vllm\vllm\v1\engine\async_llm.py)。
5. `finally: q.close()` [661-663](d:\design\vllm\vllm\v1\engine\async_llm.py)。

## `abort` / `shutdown` 流程

- **`abort`**：`output_processor.abort_requests` [757](d:\design\vllm\vllm\v1\engine\async_llm.py) 后 `await engine_core.abort_requests_async` [758](d:\design\vllm\vllm\v1\engine\async_llm.py)。
- **`shutdown`**：顺序 Prometheus → renderer → `engine_core.shutdown(timeout=timeout)` [262-270](d:\design\vllm\vllm\v1\engine\async_llm.py)，再 `cancel_task_threadsafe(handler)` [272-274](d:\design\vllm\vllm\v1\engine\async_llm.py)。析构 `__del__` 调 `shutdown()` [259-260](d:\design\vllm\vllm\v1\engine\async_llm.py)。

## hidden state（§9）

| 类别 | 成员 / 模式 | 锚点 |
|---|---|---|
| **asyncio.Task（真异步）** | `output_handler` | `asyncio.create_task(output_handler())` [747](d:\design\vllm\vllm\v1\engine\async_llm.py) |
| | 流式输入 `handle_inputs` | `asyncio.create_task(handle_inputs())` [526](d:\design\vllm\vllm\v1\engine\async_llm.py)，挂 `queue._input_stream_task` [517, 526](d:\design\vllm\vllm\v1\engine\async_llm.py) |
| **asyncio.Event** | `RequestOutputCollector.ready` | [output_processor.py:60](d:\design\vllm\vllm\v1\engine\output_processor.py) |
| **asyncio.Queue** | **本类路径无**每请求 `asyncio.Queue`；跨进程输出队列在 `EngineCoreProc` / `EngineCoreClient` 侧（见 `EngineCore` entity） | synthesis: 与 `RequestOutputCollector` 设计 [output_processor.py:47-54](d:\design\vllm\vllm\v1\engine\output_processor.py) 对照 |
| **Lock / Semaphore** | ~~`AsyncLLM` 源文件内**无**~~ **RESOLVED 2026-08-18**：本期新增 `_elastic_ep_lock = asyncio.Lock()`（[async_llm.py:113](d:\design\vllm\vllm\v1\engine\async_llm.py)），仅用于 `scale_elastic_ep` 互斥（[1055](d:\design\vllm\vllm\v1\engine\async_llm.py)）；请求热路径仍无锁 | 基于 grep（2026-08-18），`async_llm.py` 内 `Lock` 仅此一处、无 `Semaphore` |
| **可变间接引用** | `_logger_ref` 列表避免 `output_handler` 闭包直接持有 `self` | [async_llm.py:676-680](d:\design\vllm\vllm\v1\engine\async_llm.py) |

## 与 `LLMEngine` 的对比表

| 维度 | `LLMEngine` (sync) | `AsyncLLM` (async) |
|---|---|---|
| `EngineCoreClient` 工厂 | `make_client(..., asyncio_mode=False)` [llm_engine.py:105-111](d:\design\vllm\vllm\v1\engine\llm_engine.py) | `make_async_mp_client` [async_llm.py:149-156](d:\design\vllm\vllm\v1\engine\async_llm.py) |
| 前台输出泵 | 无 `output_handler` 任务 | `_run_output_handler` + `get_output_async` 循环 [665-747](d:\design\vllm\vllm\v1\engine\async_llm.py) |
| 添加请求 | `add_request` 同步，返回 `str` [llm_engine.py:218-296](d:\design\vllm\vllm\v1\engine\llm_engine.py) | `add_request` async，返回 `RequestOutputCollector` [async_llm.py:283-422](d:\design\vllm\vllm\v1\engine\async_llm.py) |
| 流式输入 | 本文件未提供 `AsyncGenerator` 分支 | `_add_streaming_input_request` [441-527](d:\design\vllm\vllm\v1\engine\async_llm.py) |

## Increment 2026-08-18 (vLLM 5f7fab88 → d29dc3ab)

本文件本期 +159/-59，约 1067 → 1166 行；类结构与异步运行模型（output_handler 单任务 + 单槽 collector）**未变**，全部行号锚点已重校。新增/变化要点（均已 inline 标注，此处汇总）：

- **输入处理异步化**：raw prompt 走 `input_processor.process_inputs_async`（不阻塞 event loop，[async_llm.py:369-384](d:\design\vllm\vllm\v1\engine\async_llm.py)）；已渲染的 `EngineInput` dict 仍走同步 `process_inputs`（[354-368](d:\design\vllm\vllm\v1\engine\async_llm.py)）。直接传 `EngineCoreRequest` 已标 deprecated（v0.18 移除，[339-344](d:\design\vllm\vllm\v1\engine\async_llm.py)）。
- **多模态 cache drift 恢复**：`output_handler` 消费 `EngineCoreOutput.mm_cache_miss_hashes` 并 invalidate P0 shadow cache（[711-719](d:\design\vllm\vllm\v1\engine\async_llm.py)），与 engine 侧 `_handle_mm_cache_miss` 配对（见 [EngineCore.md](EngineCore.md)）。
- **PD/KV transfer 前端钩子**：`notify_kv_transfer_request_rejected`（[763-788](d:\design\vllm\vllm\v1\engine\async_llm.py)）。
- **暂停/睡眠 API 对齐引擎两阶段 pause**：`pause_generation(mode=...)` / `sleep(level, mode)`（[790-833, 971-978](d:\design\vllm\vllm\v1\engine\async_llm.py)）；新增 `checkpoint_prepare/restore`（[985-990](d:\design\vllm\vllm\v1\engine\async_llm.py)）。
- **Fault tolerance 前端入口**：`handle_fault(FaultToleranceRequest)` / `get_status`（[1096-1103](d:\design\vllm\vllm\v1\engine\async_llm.py)，上游 commit `0b0bd2b5f6`）。
- **RL 权重更新扩展**：weight update 生命周期方法族 + weight version 标签（[1122-1166](d:\design\vllm\vllm\v1\engine\async_llm.py)，上游 commit `9069a57139`）。
- **EEP 两段式**：`prepare_elastic_ep` → drain → `commit_elastic_ep`，加 `_elastic_ep_lock`（[1051-1094](d:\design\vllm\vllm\v1\engine\async_llm.py)，上游 commit `35efdf6b34`）。
- synthesis: 本页 §5 hidden cross-reference 表的 grep 计数为 2026-04-18 结果，本期未重跑（引用关系结论——AsyncLLM 由 OpenAI api_server 经 `build_async_engine_client_from_engine_args` 构造并作为 `EngineClient` 注入——已按 HEAD 重验，见 [api_server.py:133-175](d:\design\vllm\vllm\entrypoints\openai\api_server.py)、[protocol.py:41](d:\design\vllm\vllm\engine\protocol.py)）。

## §5 step 3 hidden cross-reference grep 结果

> 计数快照日期 2026-04-18（增量 2026-08-18 未重跑计数，结构性结论已重验，见上文 Increment 小节）。

| # | 类别 | 强论断 |
|---:|---|---|
| 1 | 跨语言绑定 | `AsyncLLM`：在 `d:\design\vllm\csrc\` 全 C++/CUDA 树 grep **0** 命中。 |
| 2 | 协作伙伴跨子系统 | `AsyncLLM`：在 `d:\design\vllm\vllm\entrypoints\openai\` 全树 grep **6** 命中（`server_utils.py` 1 + `api_server.py` 5）；`d:\design\vllm\vllm\entrypoints\api_server.py`（legacy）grep **0** 命中（该文件使用 `AsyncLLMEngine` 旧模块路径，见 [api_server.py:23](d:\design\vllm\vllm\entrypoints\api_server.py)）；`d:\design\vllm\vllm\entrypoints\cli\` 对上述三符号 grep **0** 命中；`d:\design\vllm\vllm\benchmarks\` grep **0** 命中；`d:\design\vllm\tests\` 中 `AsyncLLM` grep **141** 命中（跨多测试文件累加 per-file 计数）。`AsyncEngineClient`：在 `d:\design\vllm\` 全仓库 `*.py` grep **0** 命中（代码库无此符号；前端协议为 `EngineClient` [protocol.py:41](d:\design\vllm\vllm\engine\protocol.py)）。`OpenAIServing`：在 `d:\design\vllm\vllm\entrypoints\openai\` 全树 grep **91** 命中；`tests/` 中 **106** 命中；`benchmarks/` **0**。synthesis: **V1 OpenAI 服务**通过 `build_async_engine_client_from_engine_args` 构造并 `yield` `AsyncLLM` [api_server.py:133-175](d:\design\vllm\vllm\entrypoints\openai\api_server.py)，serving 层将其实例作为 `EngineClient` 注入各 `OpenAIServing*`。 |
| 3 | 配置 / IPC 共享数据结构 | `RequestStream`：在 `d:\design\vllm\` 全仓库 `*.py` grep **0** 命中。`AsyncEngineDeadError`：grep **0** 命中；异常类型为 `EngineDeadError` [`v1/engine/exceptions.py`](d:\design\vllm\vllm\v1\engine\exceptions.py)。`output_queue`：在 `d:\design\vllm\` 全仓库 `*.py` grep **32** 命中，分布于 `core.py`、`core_client.py`、`multiproc_executor.py`；**`async_llm.py` 自身 0 命中**（前台 per-request 为 `RequestOutputCollector` + `Event`，非 `asyncio.Queue`）。`AsyncMicroBatcher`：grep **0** 命中。 |
| 4 | 测试覆盖反查 | `AsyncLLM`：在 `d:\design\vllm\tests\` 全树 **141** 命中；专项 `tests/v1/engine/test_async_llm.py`、`tests/v1/e2e/general/test_streaming_input.py`、`tests/v1/shutdown/*.py` 等。 |
| 5 | doc / config 反查 | `AsyncLLM`：在 `d:\design\vllm\docs\` 全树 **23** 命中（含 `custom_logitsprocs.md`、`io_processor_plugins.md`、`metrics.md` 等）；在 `d:\design\vllm\examples\` **29** 命中（如 `async_llm_streaming.py`、`pause_resume.py`）。 |

## Notes / Caveats

> ~~[!todo] VERIFY: `start_engine_loop` 出现在 `from_vllm_config`/`from_engine_args` 参数与 `__init__` 签名中，但当前 `__init__` 正文**未读取**该参数；是否由 `EngineCoreClient` 侧消费需对照 [`core_client.py`](d:\design\vllm\vllm\v1\engine\core_client.py) 单独验证。~~
>
> **RESOLVED 2026-04-18（2026-08-18 重验仍成立）**：`start_engine_loop` 在 [async_llm.py:83](d:\design\vllm\vllm\v1\engine\async_llm.py) 及工厂 [async_llm.py:209-223, 238-254](d:\design\vllm\vllm\v1\engine\async_llm.py) 传入，但 `__init__` 正文 [async_llm.py:109-203](d:\design\vllm\vllm\v1\engine\async_llm.py) **未引用**；`core_client.py` 全文 grep `start_engine_loop` **0 命中**（`make_async_mp_client` 见 [core_client.py:114-139](d:\design\vllm\vllm\v1\engine\core_client.py)）。**未被 EngineCoreClient 消费** —— dead parameter / 未接线参数。

> [!warning] CONTRADICTION: [docs/design/arch_overview.md](d:\design\vllm\docs\design\arch_overview.md) 仍描述「`AsyncLLMEngine` 包装 `LLMEngine`」并指向旧路径（**verify 2026-04-18 仍为旧表述**）；当前实现中 `AsyncLLMEngine` 仅为 `AsyncLLM` 别名（[async_llm_engine.py:4-7](d:\design\vllm\vllm\engine\async_llm_engine.py)，2026-08-18 重验仍为别名），与 v1 `LLMEngine` 为并列前端而非包装关系（见上文对比表）。待上游 docs 修订后改 RESOLVED。

## See also

- [vllm/entities/EngineCore.md](EngineCore.md)
- [vllm/entities/EngineCoreClient.md](EngineCoreClient.md)
- [vllm/entities/LLMEngine.md](LLMEngine.md)
- [vllm/entities/OutputProcessor.md](OutputProcessor.md)（同批 P1 ingest）
- [vllm/topics/request-lifecycle.md](../topics/request-lifecycle.md)
- [vllm/topics/multiproc-ipc.md](../topics/multiproc-ipc.md)
- [vllm/modules/engine.md](../modules/engine.md)
