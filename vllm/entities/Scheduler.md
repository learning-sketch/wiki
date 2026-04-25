---
type: entity
project: vllm
status: verified
confidence: high
verified_against: 2026-04-17
sources:
  - d:\design\vllm\vllm\v1\core\sched\scheduler.py
  - d:\design\vllm\vllm\v1\core\sched\interface.py
  - d:\design\vllm\vllm\v1\core\sched\async_scheduler.py
  - d:\design\vllm\vllm\v1\core\sched\request_queue.py
related:
  - vllm/entities/EngineCore.md
  - vllm/topics/request-lifecycle.md
  - comparison/topics/scheduler.md
---

# `Scheduler` (and `AsyncScheduler`, `SchedulerInterface`, `RequestQueue`)

## Summary
`Scheduler` 是 vLLM v1 的**核心调度器**，由 `EngineCore` 在初始化时实例化（[v1/engine/core.py:145-152](d:\design\vllm\vllm\v1\engine\core.py)）。调度的核心抽象——**没有 prefill/decode 阶段之分**，每个请求只有 `num_computed_tokens` 与 `num_tokens_with_spec` 两个数（schedule.py:349-358 注释明示），调度的目标是让前者追上后者。这让 chunked prefill / prefix caching / spec decoding / continuous batching 全都用同一套代码处理。子类 `AsyncScheduler` 在它之上把"放占位 token + 等异步结果"拆开做 GPU/CPU overlap。

## Sources
- 接口定义：[d:\design\vllm\vllm\v1\core\sched\interface.py](d:\design\vllm\vllm\v1\core\sched\interface.py)（244 行）
- 主实现：[d:\design\vllm\vllm\v1\core\sched\scheduler.py](d:\design\vllm\vllm\v1\core\sched\scheduler.py)（约 2270 行）
- 异步实现：[d:\design\vllm\vllm\v1\core\sched\async_scheduler.py](d:\design\vllm\vllm\v1\core\sched\async_scheduler.py)（61 行）
- 队列：[d:\design\vllm\vllm\v1\core\sched\request_queue.py](d:\design\vllm\vllm\v1\core\sched\request_queue.py)（209 行）
- 输出 dataclass：[d:\design\vllm\vllm\v1\core\sched\output.py](d:\design\vllm\vllm\v1\core\sched\output.py)
- 工具：[d:\design\vllm\vllm\v1\core\sched\utils.py](d:\design\vllm\vllm\v1\core\sched\utils.py)

## 类层次

```mermaid
classDiagram
    class SchedulerInterface {
        <<abstract>>
        +schedule() SchedulerOutput
        +get_grammar_bitmask(scheduler_output) GrammarOutput
        +update_from_output(scheduler_output, model_runner_output) dict
        +update_draft_token_ids(draft_token_ids)
        +add_request(request)
        +finish_requests(request_ids, finished_status)
        +get_num_unfinished_requests() int
        +has_finished_requests() bool
        +pause_state PauseState
        +reset_prefix_cache(...)
        +reset_encoder_cache()
        +get_request_counts() Tuple
        +make_stats() SchedulerStats
        +shutdown()
    }
    class Scheduler {
        +vllm_config
        +scheduler_config
        +cache_config
        +kv_cache_manager : KVCacheManager
        +encoder_cache_manager : EncoderCacheManager
        +structured_output_manager
        +waiting : RequestQueue
        +running : list[Request]
        +requests : dict[str, Request]
        +finished_req_ids : set
        +connector : KVConnectorBase_V1
        +ec_connector
        +max_num_running_reqs : int
        +max_num_scheduled_tokens : int
        +max_model_len : int
        +policy : SchedulingPolicy
        +schedule() SchedulerOutput
        +update_from_output(...) dict
        +add_request(request)
        +finish_requests(request_ids, status)
        +reset_prefix_cache(...)
    }
    class AsyncScheduler {
        +_spec_token_placeholders : list[int]
        +_update_after_schedule(scheduler_output)
        +_update_request_with_output(request, new_token_ids)
    }
    class RequestQueue {
        <<abstract>>
        +add_request(request)
        +pop_request()
        +peek_request()
        +prepend_request(request)
        +remove_request(request)
    }
    class FCFSRequestQueue
    class PriorityRequestQueue
    class SchedulingPolicy {
        <<enumeration>>
        FCFS
        PRIORITY
    }
    SchedulerInterface <|-- Scheduler
    Scheduler <|-- AsyncScheduler
    RequestQueue <|-- FCFSRequestQueue
    RequestQueue <|-- PriorityRequestQueue
```

## `SchedulerInterface` 抽象 （[interface.py:36-243](d:\design\vllm\vllm\v1\core\sched\interface.py)）

接口定义 9 个抽象方法 + 6 个具体方法。摘要：

| 方法 | 行号 | 角色 |
|---|---|---|
| `schedule()` | [50-74](d:\design\vllm\vllm\v1\core\sched\interface.py) | **核心**：选 batch + 算 token 数（"a dictionary of {req_id: num_tokens}"） |
| `update_from_output(scheduler_output, model_runner_output)` | [82-100](d:\design\vllm\vllm\v1\core\sched\interface.py) | 把 worker 输出回写到 scheduler 状态 |
| `update_draft_token_ids(draft_token_ids)` | [102-110](d:\design\vllm\vllm\v1\core\sched\interface.py) | spec decode draft token 更新 |
| `update_draft_token_ids_in_output(...)` | [112-124](d:\design\vllm\vllm\v1\core\sched\interface.py) | structured output + spec 的 deferred sampling 用 |
| `add_request(request)` | [126-133](d:\design\vllm\vllm\v1\core\sched\interface.py) | 入队 |
| `finish_requests(request_ids, finished_status)` | [135-157](d:\design\vllm\vllm\v1\core\sched\interface.py) | abort / 检测 stop string 时调 |
| `get_num_unfinished_requests()` | [159-162](d:\design\vllm\vllm\v1\core\sched\interface.py) | 状态查询 |
| `has_finished_requests()` | [169-182](d:\design\vllm\vllm\v1\core\sched\interface.py) | DP attention 用 |
| `pause_state` (property + setter) | [189-197](d:\design\vllm\vllm\v1\core\sched\interface.py) | UNPAUSED / PAUSED_NEW / PAUSED_ALL 三状态 |
| `reset_prefix_cache(reset_running_requests, reset_connector)` | [199-213](d:\design\vllm\vllm\v1\core\sched\interface.py) | 模型权重 live-update 时调 |
| `reset_encoder_cache()` | [215-222](d:\design\vllm\vllm\v1\core\sched\interface.py) | 多模态权重更新时调 |
| `get_request_counts()` | [224-227](d:\design\vllm\vllm\v1\core\sched\interface.py) | `(num_running_reqs, num_waiting_reqs)` |
| `make_stats()` | [229-235](d:\design\vllm\vllm\v1\core\sched\interface.py) | 每步生成 SchedulerStats |
| `get_grammar_bitmask(scheduler_output)` | [76-80](d:\design\vllm\vllm\v1\core\sched\interface.py) | structured output bitmask |
| `get_kv_connector()` | [242-243](d:\design\vllm\vllm\v1\core\sched\interface.py) | 默认返 None，PD 分离时返 KVConnectorBase_V1 |

`PauseState` 枚举 ([interface.py:22-33](d:\design\vllm\vllm\v1\core\sched\interface.py))：

| 值 | 语义 |
|---|---|
| UNPAUSED | 正常调度 |
| PAUSED_NEW | 不接新请求，已 RUNNING 的继续跑 |
| PAUSED_ALL | 全停 |

## `Scheduler.__init__` （[scheduler.py:67-296](d:\design\vllm\vllm\v1\core\sched\scheduler.py)）

构造时绑入：

| 字段 | 来源 | 行号 |
|---|---|---|
| `vllm_config / scheduler_config / cache_config / parallel_config / lora_config / kv_cache_config / kv_events_config / observability_config` | 入参 | [78-86](d:\design\vllm\vllm\v1\core\sched\scheduler.py) |
| `kv_metrics_collector: KVCacheMetricsCollector \| None` | 受 `observability_config.kv_cache_metrics` 控制 | [87-91](d:\design\vllm\vllm\v1\core\sched\scheduler.py) |
| `structured_output_manager` | 入参 | [92](d:\design\vllm\vllm\v1\core\sched\scheduler.py) |
| `is_encoder_decoder` | 模型配置 | [93](d:\design\vllm\vllm\v1\core\sched\scheduler.py) |
| `finished_req_ids_dict: dict[client_idx, set[req_id]] \| None` | 仅 `include_finished_set=True` 时 | [99-101](d:\design\vllm\vllm\v1\core\sched\scheduler.py) |
| `prev_step_scheduled_req_ids: set[str]` | 跨 step 跟踪上一 step 调过的 req | [102](d:\design\vllm\vllm\v1\core\sched\scheduler.py) |
| `max_num_running_reqs = scheduler_config.max_num_seqs` | 配置 | [105](d:\design\vllm\vllm\v1\core\sched\scheduler.py) |
| `max_num_scheduled_tokens` | `max_num_scheduled_tokens` 或 `max_num_batched_tokens` | [106-110](d:\design\vllm\vllm\v1\core\sched\scheduler.py) |
| `max_model_len` | 模型配置 | [111](d:\design\vllm\vllm\v1\core\sched\scheduler.py) |
| `enable_kv_cache_events` | KV events 配置 | [112-115](d:\design\vllm\vllm\v1\core\sched\scheduler.py) |
| `connector: KVConnectorBase_V1 \| None` | 由 `KVConnectorFactory` 创建（PD 分离） | [120 之后](d:\design\vllm\vllm\v1\core\sched\scheduler.py) |
| `kv_cache_manager : KVCacheManager` | 内部创建 | （详见 v1/core/kv_cache_manager.py） |
| `encoder_cache_manager` | 多模态时创建 | （[scheduler.py:89, 35-38](d:\design\vllm\vllm\v1\core\sched\scheduler.py) import） |
| `waiting : RequestQueue` | 通过 `create_request_queue(policy)` 创建 | （request_queue.py:201-208） |
| `running : list[Request]` | 空 list 起步 | |
| `requests : dict[str, Request]` | rid → Request 全表 | |

## `Scheduler.schedule()` 主算法 （[scheduler.py:348-955](d:\design\vllm\vllm\v1\core\sched\scheduler.py)）

**关键设计**（[scheduler.py:349-358](d:\design\vllm\vllm\v1\core\sched\scheduler.py) 注释）：

> There's no "decoding phase" nor "prefill phase" in the scheduler. Each request just has `num_computed_tokens` and `num_tokens_with_spec`. ... `num_tokens_with_spec = len(prompt_token_ids) + len(output_token_ids) + len(spec_token_ids)`. At each step, the scheduler tries to assign tokens to the requests so that each request's `num_computed_tokens` can catch up its `num_tokens_with_spec`. This is general enough to cover **chunked prefills, prefix caching, speculative decoding, and the "jump decoding" optimization in the future**.

主循环结构：

```mermaid
flowchart TB
    Start["schedule() entry"] --> Init["初始化 token_budget = max_num_scheduled_tokens"]
    Init --> KVStep["kv_cache_manager.new_step_starts()"]
    KVStep --> RunLoop["遍历 self.running"]
    RunLoop --> RunCheck{"token_budget > 0?"}
    RunCheck -- yes --> CalcTokens["算 num_new_tokens<br/>= num_tokens_with_spec - num_computed_tokens"]
    CalcTokens --> ChunkLimit["min(num_new_tokens, long_prefill_token_threshold)"]
    ChunkLimit --> BudgetLimit["min(num_new_tokens, token_budget)"]
    BudgetLimit --> EncCheck{"has encoder inputs?"}
    EncCheck -- yes --> SchedEnc["_try_schedule_encoder_inputs"]
    EncCheck -- no --> Mamba{"need_mamba_block_aligned_split?"}
    SchedEnc --> Mamba
    Mamba -- yes --> MambaSplit["_mamba_block_aligned_split"]
    Mamba -- no --> ZeroCheck
    MambaSplit --> ZeroCheck{"num_new_tokens == 0?"}
    ZeroCheck -- yes --> Skip["req_index += 1; continue<br/>(skip lower-priority)"]
    ZeroCheck -- no --> Alloc["kv_cache_manager.allocate_slots"]
    Alloc --> AllocOK{"new_blocks?"}
    AllocOK -- yes --> Sched["scheduled_running_reqs.append<br/>token_budget -= num_new_tokens"]
    AllocOK -- no --> Preempt["pop running, 抢占<br/>(PRIORITY: max priority+arrival_time)<br/>(FCFS: 队尾)"]
    Preempt --> Alloc
    Sched --> RunLoop
    Skip --> RunLoop
    RunCheck -- no --> WaitPhase["进入 waiting 队列阶段<br/>仅 if not preempted_reqs and UNPAUSED"]
    WaitPhase --> WaitLoop["遍历 waiting + skipped_waiting"]
```

**RUNNING 阶段**（[scheduler.py:383-561](d:\design\vllm\vllm\v1\core\sched\scheduler.py)）：

- **不严格 FCFS**：注释 [scheduler.py:454-457](d:\design\vllm\vllm\v1\core\sched\scheduler.py)："Here, by doing `continue` instead of `break`, we do not strictly follow the FCFS scheduling policy and allow the lower-priority requests to be scheduled."
- 抢占在两种 policy 下行为不同：
  - PRIORITY ([scheduler.py:475-498](d:\design\vllm\vllm\v1\core\sched\scheduler.py))：`max(self.running, key=lambda r: (r.priority, r.arrival_time))` 选择被抢占的；如果该 req 已经在本步被调度，要回退 token_budget 与 encoder budget
  - FCFS ([scheduler.py:499-500](d:\design\vllm\vllm\v1\core\sched\scheduler.py))：`self.running.pop()` 直接砍队尾
- `_preempt_request(preempted_req, scheduled_timestamp)` ([scheduler.py:502, 961-982](d:\design\vllm\vllm\v1\core\sched\scheduler.py))

**WAITING 阶段**（[scheduler.py:563-955](d:\design\vllm\vllm\v1\core\sched\scheduler.py)）：

- 仅当本步**没发生抢占**且 `pause_state == UNPAUSED` 才进 ([scheduler.py:564](d:\design\vllm\vllm\v1\core\sched\scheduler.py))
- 同时遍历 `self.waiting` 与 `self.skipped_waiting`（被跳过的低优先级队列）
- 每个 waiting req 都尝试 `_try_promote_blocked_waiting_request`（[scheduler.py:2070-2102](d:\design\vllm\vllm\v1\core\sched\scheduler.py)）— 处理 PD 分离场景下"等远端 KV"的状态

## `update_from_output` （[scheduler.py:1299-1562](d:\design\vllm\vllm\v1\core\sched\scheduler.py)）

把 worker 的 `ModelRunnerOutput` 回写到 scheduler 状态：

- 累加 `num_computed_tokens`
- 处理 finished requests（按 client_idx 分组成 `dict[int, EngineCoreOutputs]` 返回）
- 更新 `kv_connector` 的状态（如果启了 PD）
- 处理 ec connector 的状态

返回的 `dict[int, EngineCoreOutputs]` 由 `EngineCore.step` 转入 `output_queue` 给 `EngineCoreProc.process_output_sockets` 发给 client（详见 [vllm/topics/request-lifecycle.md](../topics/request-lifecycle.md)）。

## `RequestQueue` 抽象 与两种实现 （[request_queue.py](d:\design\vllm\vllm\v1\core\sched\request_queue.py)）

`SchedulingPolicy` 枚举 ([request_queue.py:13-17](d:\design\vllm\vllm\v1\core\sched\request_queue.py))：`FCFS` / `PRIORITY`。

`RequestQueue(ABC)` ([request_queue.py:20-72](d:\design\vllm\vllm\v1\core\sched\request_queue.py)) 8 个抽象方法：`add_request` / `pop_request` / `peek_request` / `prepend_request` / `prepend_requests` / `remove_request` / `remove_requests` / `__bool__` / `__len__` / `__iter__`。

| 实现 | 行号 | 数据结构 | 复杂度 |
|---|---|---|---|
| `FCFSRequestQueue` | [75-128](d:\design\vllm\vllm\v1\core\sched\request_queue.py) | `deque` 子类 | add: O(1), pop: O(1), remove: O(N) |
| `PriorityRequestQueue` | [131-198](d:\design\vllm\vllm\v1\core\sched\request_queue.py) | `heapq` (`list` + heap invariant) | add: O(log N), pop: O(log N), remove: O(N) + heapify O(N) |

工厂：`create_request_queue(policy: SchedulingPolicy) -> RequestQueue` ([request_queue.py:201-208](d:\design\vllm\vllm\v1\core\sched\request_queue.py))。

> [!todo] VERIFY: `prepend_request` 在 `PriorityRequestQueue` 里被解释为 "fall back to add_request"（[request_queue.py:160-165](d:\design\vllm\vllm\v1\core\sched\request_queue.py)），意味着抢占恢复时**优先级队列下高 priority 旧请求不一定回到队首**——这是个语义陷阱。

## `AsyncScheduler` （[async_scheduler.py:12-60](d:\design\vllm\vllm\v1\core\sched\async_scheduler.py)）

继承 `Scheduler`，覆盖两个方法：

### `_update_after_schedule(scheduler_output)` （[async_scheduler.py:18-35](d:\design\vllm\vllm\v1\core\sched\async_scheduler.py)）

每次 `schedule` 完成后调用。**关键**：

- 提前给每个被调度的 req 加 `num_output_placeholders += 1 + cur_num_spec_tokens`（[async_scheduler.py:32](d:\design\vllm\vllm\v1\core\sched\async_scheduler.py)）
- 把 `request.spec_token_ids` 设为 placeholder list `[-1] * num_spec_tokens`（[async_scheduler.py:35](d:\design\vllm\vllm\v1\core\sched\async_scheduler.py)）
- 实际 spec token 由 worker 进程在执行时 in-place 更新

> synthesis: 这个 placeholder 机制让 scheduler 可以**在 worker 还没产出本 step token 时**，就开始为下一 step 计算 budget，从而 GPU 与 CPU overlap。

### `_update_request_with_output(request, new_token_ids)` （[async_scheduler.py:37-60](d:\design\vllm\vllm\v1\core\sched\async_scheduler.py)）

- 抢占场景的清理：`discard_latest_async_tokens=True` 时丢掉最新 async token（[async_scheduler.py:40-44](d:\design\vllm\vllm\v1\core\sched\async_scheduler.py)）
- 减 `num_output_placeholders -= len(new_token_ids)`（[async_scheduler.py:52](d:\design\vllm\vllm\v1\core\sched\async_scheduler.py)）
- 调 `kv_cache_manager.cache_blocks` 真正缓存

## 与 EngineCore 的协作链

```mermaid
sequenceDiagram
    participant EC as EngineCore
    participant Sched as Scheduler
    participant KVM as KVCacheManager
    participant Exec as Executor

    EC->>Sched: schedule()
    Sched->>Sched: 遍历 running, 算 token_budget
    Sched->>KVM: allocate_slots(req, num_new_tokens)
    KVM-->>Sched: new_blocks 或 None
    alt None
        Sched->>Sched: preempt 队尾 (FCFS) 或 max-priority (PRIORITY)
        Sched->>KVM: 重试 allocate_slots
    end
    Sched->>Sched: WAITING 阶段（仅未抢占时）
    Sched-->>EC: SchedulerOutput
    EC->>Exec: execute_model(scheduler_output, non_block=True)
    Exec-->>EC: Future[ModelRunnerOutput]
    EC->>Sched: get_grammar_bitmask(scheduler_output)
    Sched-->>EC: GrammarOutput | None
    EC->>EC: future.result()
    EC->>Sched: update_from_output(scheduler_output, model_output)
    Sched-->>EC: dict[client_idx, EngineCoreOutputs]
```

锚点：[v1/engine/core.py:404-433](d:\design\vllm\vllm\v1\engine\core.py)（`EngineCore.step` 调 `schedule` / `execute_model` / `get_grammar_bitmask` / `update_from_output`）。

## 与 KV connector / PD 分离的关系

调度器层面感知 PD 分离的 3 个钩子：

1. `connector: KVConnectorBase_V1 | None` — 由 `KVConnectorFactory` 创建（[scheduler.py:120 上下](d:\design\vllm\vllm\v1\core\sched\scheduler.py)），暴露给 `EngineCore`（`EngineCore.scheduler.get_kv_connector()`）
2. `_try_promote_blocked_waiting_request` ([scheduler.py:2070-2102](d:\design\vllm\vllm\v1\core\sched\scheduler.py)) — 处理"等远端 KV 完成"状态的 waiting req
3. `_update_waiting_for_remote_kv` ([scheduler.py:2036-2069](d:\design\vllm\vllm\v1\core\sched\scheduler.py)) — 把 NIXL/connector 推回的 KV 完成事件应用到 waiting req

## Notes / Caveats
> [!todo] VERIFY: `get_grammar_bitmask` 与 structured output 的具体实现（在 [v1/structured_output/](d:\design\vllm\vllm\v1\structured_output)）。
> [!todo] VERIFY: `_handle_invalid_blocks` ([scheduler.py:2235-...](d:\design\vllm\vllm\v1\core\sched\scheduler.py)) 与 `_update_requests_with_invalid_blocks` ([scheduler.py:2132-2234](d:\design\vllm\vllm\v1\core\sched\scheduler.py)) 处理什么场景。
> [!warning] CONTRADICTION: `Scheduler.running` 是 `list[Request]`（线性结构），抢占时 `pop()` / `remove()` 都是 O(N)；而 `waiting` 是 `RequestQueue`。这种异构设计在大 batch_size 下可能成为瓶颈，需要 ingest performance 分析时关注。

## See also
- [entities/EngineCore.md](EngineCore.md)
- [topics/request-lifecycle.md](../topics/request-lifecycle.md)
- [comparison/topics/scheduler.md](../../comparison/topics/scheduler.md)
- [modules/engine.md](../modules/engine.md)
