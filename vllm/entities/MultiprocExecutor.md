---
type: entity
project: vllm
status: verified
confidence: high
verified_against: 2026-04-17
sources:
  - d:\design\vllm\vllm\v1\executor\multiproc_executor.py
  - d:\design\vllm\vllm\v1\executor\abstract.py
related:
  - vllm/modules/executor.md
  - vllm/entities/EngineCore.md
  - vllm/topics/multiproc-ipc.md
  - vllm/topics/request-lifecycle.md
---

# `MultiprocExecutor` (and `WorkerProc`, `WorkerProcHandle`, `FutureWrapper`)

## Summary
`MultiprocExecutor` 是 vLLM v1 的多进程 worker 编排器（[multiproc_executor.py:101](d:\design\vllm\vllm\v1\executor\multiproc_executor.py)），**用 `multiprocessing` spawn 出 N=world_size 个 worker 子进程**，每个跑 `WorkerProc.worker_main`。主→worker 用 1 个共享内存广播队列（`MessageQueue`）下发 `SchedulerOutput`，worker→主用每 worker 一个 `MessageQueue` 回送 `ModelRunnerOutput`。生命周期管理用 `multiprocessing.Pipe` 做 ready 信号 + death 信号双通道。

## Sources
- 全文：[d:\design\vllm\vllm\v1\executor\multiproc_executor.py](d:\design\vllm\vllm\v1\executor\multiproc_executor.py)（1048 行）
- 父类：[d:\design\vllm\vllm\v1\executor\abstract.py:37](d:\design\vllm\vllm\v1\executor\abstract.py)
- 共享内存队列：[vllm/distributed/device_communicators/shm_broadcast.py](d:\design\vllm\vllm\distributed\device_communicators\shm_broadcast.py)（`MessageQueue`, `Handle`）

## 类层次

```mermaid
classDiagram
    class Executor {
        <<abstract>>
        +collective_rpc(...)
    }
    class MultiprocExecutor {
        +supports_pp = True
        +rpc_broadcast_mq : MessageQueue
        +response_mqs : list[MessageQueue]
        +workers : list[WorkerProcHandle]
        +futures_queue : deque[FutureWrapper]
        +_init_executor()
        +collective_rpc(...)
        +execute_model(...)
        +sample_tokens(...)
        +shutdown()
    }
    class WorkerProc {
        +rank
        +worker : WorkerWrapperBase
        +rpc_broadcast_mq : MessageQueue
        +worker_response_mq : MessageQueue
        +async_output_queue : queue.Queue
        +worker_busy_loop()
        +async_output_busy_loop()
        +monitor_death_pipe(...)
    }
    class FutureWrapper {
        +get_response : Callable
        +aggregate : Callable
        +_wait_for_response()
    }
    class UnreadyWorkerProcHandle {
        +proc : BaseProcess
        +ready_pipe : Connection
        +death_writer : Connection
    }
    class WorkerProcHandle {
        +proc, rank
        +worker_response_mq
        +peer_worker_response_mqs
        +death_writer
    }
    Executor <|-- MultiprocExecutor
    MultiprocExecutor o-- WorkerProcHandle
    MultiprocExecutor o-- FutureWrapper
    UnreadyWorkerProcHandle ..> WorkerProcHandle : promotes after READY
```

## `_init_executor` 启动流程（[multiproc_executor.py:108-256](d:\design\vllm\vllm\v1\executor\multiproc_executor.py)）

> **入口**：被 `Executor.__init__` 调用 ([abstract.py:109](d:\design\vllm\vllm\v1\executor\abstract.py))

1. **注册 finalizer** 保证退出时一定 shutdown ([multiproc_executor.py:111](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
2. **算 parallel sizes**：`tp_size * pp_size * pcp_size == world_size` ([multiproc_executor.py:115-121](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))；`_get_parallel_sizes` 在 [multiproc_executor.py:257-268](d:\design\vllm\vllm\v1\executor\multiproc_executor.py)
3. `set_multiprocessing_worker_envs()` — 设 `OMP_NUM_THREADS=1` 防止 CPU 抢占 ([multiproc_executor.py:1018-1047](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
4. 算 distributed init method：`tcp://<loopback>:<open_port>` ([multiproc_executor.py:126-128](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
5. **DP 主节点**才创建 `rpc_broadcast_mq` 并 `export_handle()`（[multiproc_executor.py:133-155](d:\design\vllm\vllm\v1\executor\multiproc_executor.py)）
6. 创建 `shared_worker_lock` ([multiproc_executor.py:158](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
7. **循环 spawn workers**：每个 local_rank 一个 `WorkerProc.make_worker_process(...)` ([multiproc_executor.py:172-200](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))。CPU 平台还套一层 OMP manager ([multiproc_executor.py:175-188](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
8. **fork 模式**特殊处理 inherited fds，用于让后续 worker 关闭已 inherit 但属于其它 worker 的 pipe fd ([multiproc_executor.py:165-203](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
9. `WorkerProc.wait_for_ready(unready_workers)` 阻塞等所有 worker 通过 ready_pipe 发送 `READY` 字符串 ([multiproc_executor.py:209](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
10. 启动 worker 健康监控线程 `start_worker_monitor()` ([multiproc_executor.py:212-213](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
11. 装配 `response_mqs`：本地 worker 用 `worker_response_mq`，远端用 `peer_worker_response_mqs` ([multiproc_executor.py:215-228](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
12. **MQ 双向 wait_until_ready**（**有顺序依赖，颠倒会死锁**）：
    - 先 `rpc_broadcast_mq.wait_until_ready()` ([multiproc_executor.py:234-235](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
    - 再 `for response_mq in self.response_mqs: response_mq.wait_until_ready()` ([multiproc_executor.py:236-238](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
13. `futures_queue = deque[FutureWrapper]()` ([multiproc_executor.py:240](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
14. 失败回滚：`_ensure_worker_termination` ([multiproc_executor.py:245-253](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
15. `output_rank = _get_output_rank()` — TP=0 + PP rank=-1（最后一个 PP stage 的 TP 0） ([multiproc_executor.py:255, 489-503](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))

## `collective_rpc` 核心 RPC（[multiproc_executor.py:348-412](d:\design\vllm\vllm\v1\executor\multiproc_executor.py)）

签名：`collective_rpc(method, timeout, args, kwargs, non_block, unique_reply_rank, kv_output_aggregator)` → `Any`

```mermaid
sequenceDiagram
    participant Caller as EngineCore
    participant Exec as MultiprocExecutor
    participant MQ as rpc_broadcast_mq
    participant W as WorkerProc(N个)
    participant RMQ as response_mqs

    Caller->>Exec: collective_rpc("execute_model", args, non_block=True)
    Note over Exec: 检查 is_failed，<br/>kv_output_aggregator 决定 aggregate 函数
    Exec->>MQ: enqueue((method, args, kwargs, output_rank))
    MQ-->>W: 广播
    W->>W: worker_busy_loop dequeue → func(*args)
    W-->>RMQ: enqueue((SUCCESS|FAILURE, output))
    Exec-->>Caller: FutureWrapper（appendleft 到 futures_queue）
    Caller->>Caller: future.result()
    Note over Caller: result() 内 drain 队列前面的 future
    RMQ-->>Exec: dequeue 取 responses
    Exec-->>Caller: aggregate(responses) 或 responses[0]
```

要点：

- **路径选择**（[multiproc_executor.py:369-376](d:\design\vllm\vllm\v1\executor\multiproc_executor.py)）：
  - 有 `kv_output_aggregator`（PD 分离场景）：`output_rank=None`（收所有 ranks 的输出），`aggregate=partial(...)` 做聚合
  - 否则：`output_rank=unique_reply_rank`，`aggregate=identity`
- **方法序列化**（[multiproc_executor.py:378-381](d:\design\vllm\vllm\v1\executor\multiproc_executor.py)）：字符串方法名直接发；callable 用 `cloudpickle.dumps(method, protocol=HIGHEST_PROTOCOL)` 序列化（worker 端 [multiproc_executor.py:961-964](d:\design\vllm\vllm\v1\executor\multiproc_executor.py) 用 `cloudpickle.loads` 还原 + `partial(..., self.worker)`）
- **Future 队列保证顺序**（[multiproc_executor.py:68-99](d:\design\vllm\vllm\v1\executor\multiproc_executor.py)）：`FutureWrapper.result()` 会先 drain 队列里排在它前面的所有 future，避免乱序拿到响应
- 超时控制：`deadline = time.monotonic() + timeout` → 每次 `mq.dequeue(timeout=...)` 用剩余时间 ([multiproc_executor.py:366-367, 391-393](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
- 任意 worker 返 `FAILURE` 立刻 raise `RuntimeError` ([multiproc_executor.py:398-402](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
- 必须在 leader 节点（DP 内）调，follower 调会断言 ([multiproc_executor.py:360-362](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))

## 高层 API

| 方法 | 行号 | 实现 |
|---|---|---|
| `execute_model(scheduler_output, non_block)` | [315-325](d:\design\vllm\vllm\v1\executor\multiproc_executor.py) | `collective_rpc("execute_model", ..., unique_reply_rank=output_rank, timeout=VLLM_EXECUTE_MODEL_TIMEOUT_SECONDS, kv_output_aggregator=...)` |
| `sample_tokens(grammar_output, non_block)` | [327-337](d:\design\vllm\vllm\v1\executor\multiproc_executor.py) | 同上风格 |
| `execute_dummy_batch()` | [339-340](d:\design\vllm\vllm\v1\executor\multiproc_executor.py) | warmup 用 |
| `take_draft_token_ids()` | [342-346](d:\design\vllm\vllm\v1\executor\multiproc_executor.py) | spec decode 取 draft，**只从 output_rank 取**，注释明示是 OPTIMIZATION |
| `check_health()` | [479-481](d:\design\vllm\vllm\v1\executor\multiproc_executor.py) | `collective_rpc("check_health", timeout=10)` |
| `max_concurrent_batches` (cached_property) | [483-487](d:\design\vllm\vllm\v1\executor\multiproc_executor.py) | `pp_size` if pp>1 else (`2` if `async_scheduling` else `1`) |
| `supports_async_scheduling()` (classmethod) | [505-507](d:\design\vllm\vllm\v1\executor\multiproc_executor.py) | 返 True |
| `register_failure_callback(callback)` | [309-313](d:\design\vllm\vllm\v1\executor\multiproc_executor.py) | 失败已发生则立即调，否则存起来 |

## Worker 监控（[multiproc_executor.py:276-307](d:\design\vllm\vllm\v1\executor\multiproc_executor.py)）

后台线程 `monitor_workers()`：

- `multiprocessing.connection.wait([h.proc.sentinel for h in workers])` 阻塞等任一 worker 进程结束 ([multiproc_executor.py:284-285](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
- 任意 worker 死 → `is_failed = True` + 记录死掉的进程名 + 触发 `shutdown()` + 调 `failure_callback` ([multiproc_executor.py:286-299](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
- `failure_callback` 由 `EngineCoreProc` 注册成 "把 EXECUTOR_FAILED put 到 input_queue"（[engine/core.py:823-825, 117-118](d:\design\vllm\vllm\v1\engine\core.py)）

## 优雅关闭（[multiproc_executor.py:414-477](d:\design\vllm\vllm\v1\executor\multiproc_executor.py)）

`_ensure_worker_termination` 三段式：

1. 等 4s 让 worker 自然退出（关 death_writer 后 worker 端 death_pipe.recv() 会 EOFError） ([multiproc_executor.py:435](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
2. 不行 → `proc.terminate()`（SIGTERM）等 4s ([multiproc_executor.py:439-442](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
3. 还不行 → `proc.kill()`（SIGKILL） ([multiproc_executor.py:443-448](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))

`shutdown()` 序列：

1. 标 `shutting_down = True`（防重入） ([multiproc_executor.py:452-454](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
2. 关每个 worker 的 `death_writer` → 触发 worker 端 EOFError → graceful exit ([multiproc_executor.py:457-461](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
3. `_ensure_worker_termination` ([multiproc_executor.py:463](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
4. 关每个 `worker_response_mq` ([multiproc_executor.py:465-469](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
5. 关 `rpc_broadcast_mq` ([multiproc_executor.py:471-473](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
6. 关所有 `response_mqs` ([multiproc_executor.py:474-477](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))

## `WorkerProc` 子进程内（[multiproc_executor.py:548-1015](d:\design\vllm\vllm\v1\executor\multiproc_executor.py)）

### `make_worker_process` 静态工厂（[multiproc_executor.py:652-703](d:\design\vllm\vllm\v1\executor\multiproc_executor.py)）

- 创建 ready_pipe（duplex=False，子→父） ([multiproc_executor.py:665](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
- 创建 death_pipe（duplex=False，父→子） ([multiproc_executor.py:667](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
- `context.Process(target=WorkerProc.worker_main, kwargs=..., name=f"VllmWorker-{rank}", daemon=True)` ([multiproc_executor.py:685-690](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
- NUMA binding：`numa_utils.configure_subprocess(...)` 上下文内启动 ([multiproc_executor.py:692-696](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
- 父端关 child 端，保持 death_writer 在父进程开着 — 父退出 → death_pipe 的 reader 在 child 触发 EOFError ([multiproc_executor.py:698-703](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))

### `worker_main` 子进程入口（[multiproc_executor.py:800-904](d:\design\vllm\vllm\v1\executor\multiproc_executor.py)）

1. 安装 SIGTERM/SIGINT signal handler，`shutdown_requested.set() + raise SystemExit` ([multiproc_executor.py:810-821](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
2. 关 inherited fds（防止 fork 模式残留） ([multiproc_executor.py:831-835](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
3. 初始化 tracer ([multiproc_executor.py:840-844](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
4. 实例化 `WorkerProc(*args, **kwargs)` — 在 `__init__` 内：load_model + 设进程名 + spec/async output thread + 算 block_size + `_init_message_queues` ([multiproc_executor.py:587-650](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
5. 启 `monitor_death_pipe` 线程：父进程退出 → recv EOFError → shutdown 所有 MQ ([multiproc_executor.py:775-798](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
6. **送 READY** 给父进程：`ready_writer.send({"status": "READY", "handle": ..., "peer_response_handles": ...})` ([multiproc_executor.py:854-860](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
7. 双 MQ wait_until_ready（必须与 Executor 端**同序**） ([multiproc_executor.py:864-866](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
8. 进入 `worker.worker_busy_loop()` ([multiproc_executor.py:870](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))
9. 异常处理：失败时 `shutdown_requested.set()` 防止 `__del__` 时再次抛异常引发 zmq 异常 ([multiproc_executor.py:872-895](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))

### `worker_busy_loop` （[multiproc_executor.py:953-979](d:\design\vllm\vllm\v1\executor\multiproc_executor.py)）

```python
while True:
    method, args, kwargs, output_rank = self.rpc_broadcast_mq.dequeue(indefinite=True)
    try:
        if isinstance(method, str):
            func = getattr(self.worker, method)
        elif isinstance(method, bytes):
            func = partial(cloudpickle.loads(method), self.worker)
        output = func(*args, **kwargs)
    except Exception as e:
        if hasattr(e, "add_note"): e.add_note(traceback.format_exc())
        if output_rank is None or self.rank == output_rank:
            self.handle_output(e)   # 异常作 FAILURE 回送
        continue
    if output_rank is None or self.rank == output_rank:
        self.handle_output(output)
```

要点：

- `output_rank=None` → **所有 rank 都回送**（KV connector 聚合场景）
- `output_rank=int` → 只该 rank 回送（默认场景，省带宽）
- 字节串方法走 `cloudpickle.loads`，让前端能传任意 callable

### Async output（[multiproc_executor.py:925-951](d:\design\vllm\vllm\v1\executor\multiproc_executor.py)）

- 当 `scheduler_config.async_scheduling=True` 时，构造时启 `async_output_copy_thread`
- `handle_output` 把 output put 到 `async_output_queue`，由 thread 取出后 `enqueue_output` 到 MQ
- 线程内 `current_platform.set_device(...)` 确保 cuda context 与主线程一致（避免新建 cuda context 占内存） ([multiproc_executor.py:935-947](d:\design\vllm\vllm\v1\executor\multiproc_executor.py))

### `_init_message_queues` （[multiproc_executor.py:555-585](d:\design\vllm\vllm\v1\executor\multiproc_executor.py)）

- 单节点（`nnodes_within_dp == 1`）：直接用 `MessageQueue.create_from_handle(input_shm_handle, rank)` 创建本地接收端
- 多节点：通过 `get_inner_dp_world_group().create_mq_broadcaster(...)` 跨节点 MQ；用 `create_single_reader_mq_broadcasters` 给所有 ranks 暴露给 driver worker 的 `peer_response_handles`

## `setup_proc_title_and_log_prefix` （[multiproc_executor.py:981-1015](d:\design\vllm\vllm\v1\executor\multiproc_executor.py)）

进程名按并行模式拼：`Worker_DP{r}_PP{r}_PCP{r}_TP{r}_DCP{r}_EP{r}`，方便 `ps` / 日志区分。

## Notes / Caveats
> [!todo] VERIFY: `MessageQueue` 内部实现（共享内存环形 buffer 的具体协议） — 在 [vllm/distributed/device_communicators/shm_broadcast.py](d:\design\vllm\vllm\distributed\device_communicators\shm_broadcast.py)，待 ingest。
> [!todo] VERIFY: 多节点 DP 时 `create_mq_broadcaster` / `create_single_reader_mq_broadcasters` 的协议（涉及 `inner_dp_world_group`）。
> [!warning] CONTRADICTION: [multiproc_executor.py:600](d:\design\vllm\vllm\v1\executor\multiproc_executor.py) 注释 `# TODO: move \`init_worker\` to executor level as a collective rpc call` 表明当前 init_worker 在 worker 进程构造时就跑（而非通过 RPC），后续可能重构。
> [!todo] VERIFY: `_get_output_rank` 公式 `world_size - tp_size * pcp_size` 在 PP=1 + PCP>1 场景下是否正确（[multiproc_executor.py:489-503](d:\design\vllm\vllm\v1\executor\multiproc_executor.py)）。

## See also
- [modules/executor.md](../modules/executor.md)
- [entities/EngineCore.md](EngineCore.md)
- [topics/multiproc-ipc.md](../topics/multiproc-ipc.md) — 详解 MQ + Pipe + Signal 三类 IPC
- [topics/request-lifecycle.md](../topics/request-lifecycle.md) — 看 collective_rpc 在端到端中的位置
