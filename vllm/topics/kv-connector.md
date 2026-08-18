---
type: topic
project: vllm
status: stale
confidence: medium
verified_against: 2026-08-18 (vLLM d29dc3ab, increment from 5f7fab88)
sources:
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\__init__.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\factory.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\multi_connector.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\connector.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\__init__.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\base_scheduler.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\base_worker.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\pull_scheduler.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\pull_worker.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\push_scheduler.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\push_worker.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\scheduler.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\worker.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\utils.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\metadata.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\stats.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\mooncake\mooncake_connector.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\mooncake\store\connector.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\mooncake\mooncake_utils.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\moriio\moriio_connector.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\moriio\moriio_engine.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\moriio\moriio_common.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\hf3fs\hf3fs_connector.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\hf3fs\hf3fs_client.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\hf3fs\hf3fs_metadata_server.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\lmcache_connector.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\lmcache_mp_connector.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\lmcache_integration\__init__.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\lmcache_integration\vllm_v1_adapter.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\offloading_connector.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\offloading\scheduler.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\offloading\worker.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\offloading\common.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\offloading\config.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\offloading\events.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\offloading\canonical_mapping.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\flexkv_connector.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\simple_cpu_offload_connector.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\decode_bench_connector.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\example_connector.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\example_hidden_states_connector.py
  - d:\design\vllm\vllm\config\kv_transfer.py
  - d:\design\vllm\vllm\v1\core\sched\scheduler.py
  - d:\design\vllm\vllm\v1\worker\kv_connector_model_runner_mixin.py
  - d:\design\vllm\vllm\v1\outputs.py
  - d:\design\vllm\vllm\v1\kv_offload\abstract.py
  - d:\design\vllm\vllm\v1\kv_offload\factory.py
  - d:\design\vllm\vllm\v1\kv_offload\spec.py
  - d:\design\vllm\vllm\v1\kv_offload\mediums.py
  - d:\design\vllm\vllm\v1\kv_offload\reuse_manager.py
  - d:\design\vllm\vllm\v1\kv_offload\worker\worker.py
  - d:\design\vllm\vllm\v1\kv_offload\worker\cpu_gpu.py
  - d:\design\vllm\vllm\v1\kv_offload\cpu\manager.py
  - d:\design\vllm\vllm\v1\kv_offload\cpu\spec.py
  - d:\design\vllm\vllm\entrypoints\scale_out\token_in_token_out\api_router.py
  - d:\design\vllm\vllm\entrypoints\scale_out\token_in_token_out\serving.py
  - d:\design\vllm\vllm\entrypoints\scale_out\token_in_token_out\protocol.py
related:
  - vllm/index.md
  - vllm/entities/Scheduler.md
  - vllm/entities/KVCacheManager.md
  - vllm/entities/EngineCore.md
  - vllm/topics/request-lifecycle.md
  - comparison/topics/pd-disaggregation.md
  - comparison/dimensions.md
  - sglang/modules/mem_cache.md
---

# vLLM KV Connector / KV Offload 子系统

> [!warning] STALE NOTICE (2026-08-18)：本页正文基于 vLLM 5f7fab88。本期增量（→ d29dc3ab）`vllm/distributed/kv_transfer/` 有 54 文件 +17333/-6313 的大 churn：**P2pNcclConnector 整体删除**、NIXL 重构为 base/pull/push 分层（新增 push 模式）、Mooncake 新增 `store/` 子树（`MooncakeStoreConnector`）、offloading 大幅膨胀、HTTP 前端 `entrypoints/serve/disagg/` 迁至 `entrypoints/scale_out/`。**注册 connector 数 14 → 16**。骨架级修正见 [§Increment 2026-08-18](#increment-2026-08-18-vllm-5f7fab88--d29dc3ab)；§3.2/§3.3/§3.5 等 backend 内部行号未逐一重核（见 §Notes VERIFY）。

## Summary

vLLM 用 **`KVConnectorBase_V1` 抽象类**（现 [base.py:171](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py)）把"远端 / 异构 KV 传输"做成 **scheduler ↔ worker 双角色钩子** —— scheduler 端通过 `get_num_new_matched_tokens` / `build_connector_meta` / `request_finished` 控制语义，worker 端通过 `register_kv_caches` / `start_load_kv` / `wait_for_save` / `get_finished` 执行实际拷贝。`KVConnectorFactory`~~注册了 **14 个 v1 connector**~~ **RESOLVED 2026-08-18**：现注册 **16 个 v1 connector**（[factory.py:152-260](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\factory.py)）——P2pNcclConnector 已删除，新增 `NixlPullConnector` / `NixlPushConnector` / `MooncakeStoreConnector`，详见 §Increment.2。`v1/kv_offload/`（[abstract.py](d:\design\vllm\vllm\v1\kv_offload\abstract.py) + [factory.py](d:\design\vllm\vllm\v1\kv_offload\factory.py) + [spec.py](d:\design\vllm\vllm\v1\kv_offload\spec.py) + cpu/ + worker/）是被 `OffloadingConnector` 复用的**底层块管理层**（`OffloadingManager` + `OffloadingHandler` + `LoadStoreSpec` + `CanonicalKVCaches`），与 connector 形成"协议 + 块管理"两层结构。

> synthesis: 与 [comparison/topics/pd-disaggregation.md §3](../../comparison/topics/pd-disaggregation.md) 对照，vLLM 是三家中**唯一把"PD 分离 + KV offload + 多级 cache"统一到同一个 connector 抽象**的项目，代价是抽象很深（base 660 行 + 13 backend），优势是 `MultiConnector` 可以"同一进程同时跑多个 storage layer connector"，例如 `mooncake_store + LMCache` 二级 cache。

## Sources

| 类别 | 路径 |
|---|---|
| 抽象基类 | [v1/base.py](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py)（660+ 行） |
| 工厂 / 注册表 | [kv_connector/factory.py](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\factory.py) |
| 配置 dataclass | [config/kv_transfer.py:23-122](d:\design\vllm\vllm\config\kv_transfer.py)（`KVTransferConfig`） |
| Scheduler 集成 | [v1/core/sched/scheduler.py](d:\design\vllm\vllm\v1\core\sched\scheduler.py)（搜 `self.connector`，46 处命中） |
| Worker 集成 mixin | [v1/worker/kv_connector_model_runner_mixin.py](d:\design\vllm\vllm\v1\worker\kv_connector_model_runner_mixin.py)（284 行） |
| Output dataclass | [v1/outputs.py:127-153](d:\design\vllm\vllm\v1\outputs.py)（`KVConnectorOutput`） |
| KV offload 块管理层 | [v1/kv_offload/](d:\design\vllm\vllm\v1\kv_offload)（13 文件） |
| HTTP 服务前端 | [entrypoints/serve/disagg/](d:\design\vllm\vllm\entrypoints\serve\disagg)（3 文件） |
| 14 个 connector 实现 | [v1/](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1)（详见 §3） |

## Architecture / Data flow

```mermaid
flowchart TB
    subgraph SchedProc [Scheduler 进程]
        Sched["Scheduler\nself.connector: KVConnectorBase_V1\n(role=SCHEDULER)"]
        SchedAPI["get_num_new_matched_tokens()\nupdate_state_after_alloc()\nbuild_connector_meta()\nrequest_finished()\nupdate_connector_output()\ntake_events()"]
        Sched --- SchedAPI
    end
    subgraph WorkerProc [Worker 进程 x N]
        Mixin["KVConnectorModelRunnerMixin\n(in execute_model)"]
        WConn["worker connector\n(role=WORKER)"]
        WorkerAPI["register_kv_caches()\nbind_connector_metadata()\nstart_load_kv()\nwait_for_layer_load()\nsave_kv_layer()\nwait_for_save()\nget_finished()\nget_block_ids_with_load_errors()"]
        Mixin --> WConn
        WConn --- WorkerAPI
    end
    Output["KVConnectorOutput\nfinished_sending / finished_recving\nkv_connector_stats / kv_cache_events\ninvalid_block_ids / kv_connector_worker_meta"]
    Sched -->|"build_connector_meta(scheduler_output)\n→ KVConnectorMetadata"| Mixin
    Mixin -->|"_get_kv_connector_output\nKVConnectorOutput"| Output
    Output -->|"_update_from_kv_xfer_finished"| Sched

    subgraph Backends [14 个注册的 v1 backend]
        B1[NIXL]
        B2[Mooncake]
        B3[MoRIIO]
        B4[LMCache×3]
        B5[HF3FS]
        B6[P2P-NCCL]
        B7[Offloading + SimpleCPUOffload]
        B8[FlexKV]
        B9[Multi]
        B10[Example×2 + DecodeBench]
    end
    WConn -.-> Backends

    subgraph OffloadLayer [v1/kv_offload/ 块管理层]
        OM["OffloadingManager\n(SCHEDULER 端)"]
        OH["OffloadingHandler\n(WORKER 端)"]
        OS["OffloadingSpec\nFactory"]
    end
    B7 ==> OffloadLayer

    subgraph Frontend [HTTP 前端 (Disaggregated Everything)]
        Disagg["entrypoints/serve/disagg/\nServingTokens + /inference/v1/generate\n+ /abort_requests"]
    end
    Disagg -->|kv_transfer_params 透传| Sched
```

锚点：
- 抽象基类与 13 个非 abstract 方法 + 8 个 abstract 方法：[base.py:170-662](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py)
- Scheduler 持 `self.connector` 字段：[scheduler.py:120](d:\design\vllm\vllm\v1\core\sched\scheduler.py)；通过 [factory.py:42-82](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\factory.py) `KVConnectorFactory.create_connector(role=SCHEDULER)` 创建：[scheduler.py:127-131](d:\design\vllm\vllm\v1\core\sched\scheduler.py)
- Worker 端通过 `KVConnectorModelRunnerMixin._get_kv_connector_output` context manager 包住整个 `execute_model`：[kv_connector_model_runner_mixin.py:84-119](d:\design\vllm\vllm\v1\worker\kv_connector_model_runner_mixin.py)
- 14 backend 注册表：[factory.py:149-228](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\factory.py)

---

## 1. KVConnectorBase_V1 抽象类

[base.py:170-662](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py)。`__init__(vllm_config, role, kv_cache_config=None)` 强制注入 `KVConnectorRole`（[base.py:123-128](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py)）：

```python
class KVConnectorRole(enum.Enum):
    SCHEDULER = 0  # Connector running in the scheduler process
    WORKER = 1     # Connector running in the worker process
```

由 `KVConnectorFactory` 显式区分两次构造（[factory.py:69-82](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\factory.py)：`# v1 connector is explicitly separated into two roles`），所以 **每个 backend 通常分 `XxxScheduler` + `XxxWorker` 两个内部实现类**（NIXL / Mooncake / MoRIIO / Offloading 都遵循该模式，详见 §3）。

### 1.1 核心方法（双视角）

> [!warning] STALE (2026-08-18)：base.py 本期 +87 行，下表行号为 5f7fab88 快照，方法集合本身经抽查未见删减。已在 d29dc3ab 重核的新行号：`KVConnectorBase_V1` 类 L171 / `register_kv_caches` L272 / `start_load_kv` L314 / `wait_for_save` L368 / `get_finished` L378 / `get_block_ids_with_load_errors` L396 / `get_num_new_matched_tokens` L475 / `build_connector_meta` L536 / `request_finished` L568 / `get_finished_count` L651；辅助 ABC：`SupportsHMA` L85 / `KVConnectorRole` L124 / `KVConnectorHandshakeMetadata` L132 / `KVConnectorMetadata` L141 / `KVConnectorWorkerMetadata` L150（[base.py](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py)）。其余行号偏移 +10~+20 未逐一核。

| 方法 | 行号 | role | 必须 override | 语义 |
|---|---|---|---|---|
| `register_kv_caches(kv_caches)` | [257](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py) | WORKER | 否（默认 no-op） | 注册 vLLM 的 paged KV buffer，供 connector 后续 RDMA 注册 / pin |
| `register_cross_layers_kv_cache(kv_cache, attn_backend)` | [267](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py) | WORKER | 否 | 单 tensor 含全 layer（`prefer_cross_layer_blocks=True` 时使用，可加速跨层 RDMA） |
| `bind_connector_metadata(metadata)` / `clear_connector_metadata()` | [217-235](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py) | WORKER | 否 | 每次 forward 前 set / 后 clear，由 mixin 自动调 |
| `set_host_xfer_buffer_ops(copy_operation)` | [284](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py) | WORKER | 否 | 注入 xPU 的 host↔device copy op（`CopyBlocksOp` typedef [base.py:70-79](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py)） |
| `handle_preemptions(metadata)` | [291](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py) | WORKER | 否 | 抢占 / 块覆写**之前**的 hook（OffloadingConnector 用） |
| `start_load_kv(forward_context)` | [299](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py) | WORKER | **是** | 启动异步 load（forward 前 fire） |
| `wait_for_layer_load(layer_name)` | [317](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py) | WORKER | **是** | layer-by-layer 流水时阻塞等单层 load |
| `save_kv_layer(layer_name, kv_layer, attn_metadata)` | [331](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py) | WORKER | **是** | layer-by-layer 流水时启动单层 save |
| `wait_for_save()` | [353](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py) | WORKER | **是** | forward context 退出前阻塞等所有 save 完成 |
| `get_finished(finished_req_ids)` | [363](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py) | WORKER | 否（默认 None,None） | 返 `(finished_sending, finished_recving)` 两个 set |
| `get_block_ids_with_load_errors()` | [381](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py) | WORKER | 否（默认空集） | **块级失败粒度**（vLLM 独有），见 §6 |
| `shutdown()` | [401](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py) | both | 否 | worker / scheduler 关闭时清理 |
| `get_handshake_metadata()` / `set_xfer_handshake_metadata(metadata)` | [423, 624](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py) | both | 否 | P/D 之间 out-of-band 握手元数据交换 |
| `get_num_new_matched_tokens(request, num_computed_tokens)` | [449](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py) | SCHEDULER | **是** | **PD 入口**，返 `(count_or_None, async_load: bool)` |
| `update_state_after_alloc(request, blocks, num_external_tokens)` | [484](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py) | SCHEDULER | **是** | scheduler 分配块后通知 connector |
| `build_connector_meta(scheduler_output)` | [505](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py) | SCHEDULER | **是** | 每 step 构造发往 worker 的 metadata（**resets connector state**） |
| `update_connector_output(connector_output)` | [520](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py) | SCHEDULER | 否 | 每 step 收到 worker 端 output |
| `request_finished(request, block_ids)` | [530](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py) | SCHEDULER | 否（默认 `False, None`） | 返 `(async_save: bool, kv_transfer_params)`，True 时 connector **接管块释放** |
| `take_events()` | [551](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py) | SCHEDULER | 否 | 返 KV cache 事件流 |
| `get_required_kvcache_layout()` | [560](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py) | classmethod | 否 | 声明 connector 需要的 KV layout（`HND` / `NHD`） |
| `requires_piecewise_for_cudagraph(extra_config)` | [579](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py) | classmethod | 否 | layerwise op 不能 capture 进 CUDA graph，需 `PIECEWISE` 模式 |
| `get_finished_count()` | [601](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py) | both | 否 | 覆盖 `KVOutputAggregator` 的 world_size |
| `build_kv_connector_stats(data=None)` | [613](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py) | classmethod | 否 | 自定义 stats 聚合逻辑 |
| `build_prom_metrics(...)` | [635](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py) | classmethod | 否 | Prometheus 指标 |
| `reset_cache()` | [650](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py) | both | 否 | reset internal cache |
| `request_finished_all_groups(request, block_ids)` | [91-113](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py)（`SupportsHMA` 接口） | SCHEDULER | **是 (HMA)** | HMA（Hybrid Memory Allocator）多 KV 组场景的等价 `request_finished` |

### 1.2 5 种辅助 ABC

| 类 | 行号 | 用途 |
|---|---|---|
| `SupportsHMA(ABC)` | [84-113](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py) | 标记 connector 支持 hybrid memory allocator（Mamba / SWA 多 KV 组） |
| `KVConnectorRole(enum.Enum)` | [123-128](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py) | `SCHEDULER=0` / `WORKER=1` |
| `KVConnectorHandshakeMetadata(ABC)` | [131-137](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py) | P/D 之间 out-of-band 握手 payload（如 NIXL `NixlHandshakePayload`） |
| `KVConnectorMetadata(ABC)` | [140-146](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py) | scheduler → worker 单步 metadata |
| `KVConnectorWorkerMetadata(ABC)` + `aggregate()` | [149-167](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py) | worker → scheduler 单步 metadata，**多 worker 时聚合** |

辅助函数：`supports_hma(connector)`（[base.py:116-120](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py)）；factory 在 HMA enabled 但 connector 不支持时直接 `raise ValueError`（[factory.py:57-62](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\factory.py)）。

### 1.3 `KVConnectorOutput` dataclass

[v1/outputs.py:127-153](d:\design\vllm\vllm\v1\outputs.py)：worker → scheduler 单步回传，`is_empty()` 短路判定空 output（[outputs.py:145-153](d:\design\vllm\vllm\v1\outputs.py)）：

```python
@dataclass
class KVConnectorOutput:
    finished_sending: set[str] | None = None
    finished_recving: set[str] | None = None
    kv_connector_stats: KVConnectorStats | None = None
    kv_cache_events: KVConnectorKVEvents | None = None
    kv_connector_worker_meta: KVConnectorWorkerMetadata | None = None
    invalid_block_ids: set[int] = field(default_factory=set)
    expected_finished_count: int = 0
```

> [!todo] VERIFY: vLLM 没有 SGLang 那种独立的 `KVPoll` 5 态机 enum——`d:\design\vllm` 全树 grep `KVPoll` **0 命中**。vLLM 用 `RequestStatus.WAITING_FOR_REMOTE_KVS` + `KVConnectorOutput.{finished_sending, finished_recving, invalid_block_ids}` 三字段表达"传输中 / 完成 / 失败"，状态承载分散在 scheduler 与 connector 之间。

---

## 2. KVConnectorRole 与 Scheduler / Worker 集成

### 2.1 Scheduler 端集成（[scheduler.py](d:\design\vllm\vllm\v1\core\sched\scheduler.py) 共 46 处 `self.connector` 引用）

| 阶段 | 锚点 | 行为 |
|---|---|---|
| 构造 | [scheduler.py:117-137](d:\design\vllm\vllm\v1\core\sched\scheduler.py) | `KVConnectorFactory.create_connector(role=KVConnectorRole.SCHEDULER, kv_cache_config=...)` + 解析 `kv_load_failure_policy ∈ {"recompute", "fail"}` |
| WAITING 阶段查 prefix | [scheduler.py:616-619](d:\design\vllm\vllm\v1\core\sched\scheduler.py) | 对每个 candidate req 调 `connector.get_num_new_matched_tokens(req, num_computed_tokens)` |
| 分配后通知 | [scheduler.py:778-779](d:\design\vllm\vllm\v1\core\sched\scheduler.py) | `connector.update_state_after_alloc(req, blocks, num_external_tokens)` |
| 进入异步等待 | [scheduler.py:798](d:\design\vllm\vllm\v1\core\sched\scheduler.py) | `request.status = RequestStatus.WAITING_FOR_REMOTE_KVS` |
| 每 step build metadata | [scheduler.py:941-942, 957-959](d:\design\vllm\vllm\v1\core\sched\scheduler.py) | `connector.build_connector_meta(scheduler_output)` 注入 `scheduler_output.kv_connector_metadata` |
| 收 worker output | [scheduler.py:1508, 2103-2130](d:\design\vllm\vllm\v1\core\sched\scheduler.py) | `_update_from_kv_xfer_finished(kv_connector_output)`：解析 `finished_recving` / `finished_sending` |
| 提升 blocked req | [scheduler.py:1810-1815, 2070-2102](d:\design\vllm\vllm\v1\core\sched\scheduler.py) | `_try_promote_blocked_waiting_request(req)` 把 `WAITING_FOR_REMOTE_KVS` 提升回 `WAITING` |
| 失败块处理 | [scheduler.py:2046-2057](d:\design\vllm\vllm\v1\core\sched\scheduler.py) | `failed_recving_kv_req_ids` 集合 + `kv_cache_manager.cache_blocks(req, num_computed_tokens)` 兜底缓存有效前缀 |
| Req 完成时 | [scheduler.py:2014-2034](d:\design\vllm\vllm\v1\core\sched\scheduler.py) | 单 KV 组调 `connector.request_finished(req, block_ids[0])`；多组（HMA）调 `request_finished_all_groups(req, block_ids)` |
| Take events / shutdown | [scheduler.py:1514-1515, 1995-1996](d:\design\vllm\vllm\v1\core\sched\scheduler.py) | `take_events()` / `shutdown()` |

scheduler 的 PD 状态机扩展：

```
NORMAL                            +PD                                +Other
─────────                         ──────────                         ──────────
WAITING                          + WAITING_FOR_REMOTE_KVS  ←新       + WAITING_FOR_STRUCTURED_OUTPUT_GRAMMAR
RUNNING                                                              + WAITING_FOR_STREAMING_REQ
PREEMPTED
FINISHED_*
```

`finished_recving_kv_req_ids: set[str]`（[scheduler.py:182](d:\design\vllm\vllm\v1\core\sched\scheduler.py)）+ `failed_recving_kv_req_ids: set[str]`（[scheduler.py:183](d:\design\vllm\vllm\v1\core\sched\scheduler.py)）配合 `WAITING_FOR_REMOTE_KVS` 状态实现"**异步 D-pull → blocked → promote**"流程。

### 2.2 Worker 端集成（[kv_connector_model_runner_mixin.py](d:\design\vllm\vllm\v1\worker\kv_connector_model_runner_mixin.py)）

`KVConnectorModelRunnerMixin._get_kv_connector_output(scheduler_output)` 是个 context manager，包住整个 `execute_model`（[kv_connector_model_runner_mixin.py:84-119](d:\design\vllm\vllm\v1\worker\kv_connector_model_runner_mixin.py)）：

```python
# 简化伪码（行号 92-119）
kv_connector = get_kv_transfer_group()
kv_connector.bind_connector_metadata(scheduler_output.kv_connector_metadata)
kv_connector.start_load_kv(get_forward_context())  # 异步 fire
try:
    yield output  # forward 期间各 attention layer 调 wait_for_layer_load / save_kv_layer
finally:
    if wait_for_save and not defer_finalize:
        kv_connector.wait_for_save()
    output.finished_sending, output.finished_recving = kv_connector.get_finished(scheduler_output.finished_req_ids)
    output.invalid_block_ids = kv_connector.get_block_ids_with_load_errors()
    output.kv_connector_stats = kv_connector.get_kv_connector_stats()
    output.kv_cache_events = kv_connector.get_kv_connector_kv_cache_events()
    output.kv_connector_worker_meta = kv_connector.build_connector_worker_meta()
    if not defer_finalize:
        kv_connector.clear_connector_metadata()
```

特殊 path：

- `kv_connector_no_forward(scheduler_output, vllm_config)`（[kv_connector_model_runner_mixin.py:37-55](d:\design\vllm\vllm\v1\worker\kv_connector_model_runner_mixin.py)）：**即使没 model 工作也要跑 connector 的 send/recv**（用于纯传输 batch）
- `finalize_kv_connector()`（[kv_connector_model_runner_mixin.py:71-79](d:\design\vllm\vllm\v1\worker\kv_connector_model_runner_mixin.py)）：spec decoding 的 draft forward 完才能 finalize（`defer_finalize=True` 路径）
- `use_uniform_kv_cache(...)` / `allocate_uniform_kv_caches(...)`（[kv_connector_model_runner_mixin.py:121-283](d:\design\vllm\vllm\v1\worker\kv_connector_model_runner_mixin.py)）：当 connector `prefer_cross_layer_blocks=True` 时，**所有 layer 共享同一 `(num_layers, ...)` 大 tensor**，便于一次 RDMA 跨层

### 2.3 KVTransferConfig dataclass

[config/kv_transfer.py:23-122](d:\design\vllm\vllm\config\kv_transfer.py)：

| 字段 | 默认 | 含义 |
|---|---|---|
| `kv_connector` | `None` | connector 名（必须在 factory 注册或给 `kv_connector_module_path`） |
| `engine_id` | uuid | 自动生成，作为 P/D 路由的 engine 标识 |
| `kv_buffer_device` | `current_platform.device_type` | KV buffer 在 cuda / cpu / xpu 哪一侧 |
| `kv_role` | `None` | `"kv_producer"` / `"kv_consumer"` / `"kv_both"`（[kv_transfer.py:11-13](d:\design\vllm\vllm\config\kv_transfer.py)） |
| `kv_rank` / `kv_parallel_size` | 0 / 1 | "Currently only 1P1D is supported"（[kv_transfer.py:46-48](d:\design\vllm\vllm\config\kv_transfer.py) 注释） |
| `kv_ip` / `kv_port` | `127.0.0.1:14579` | 通用 IP/port |
| `kv_connector_extra_config: dict` | `{}` | 各 backend 自定义参数（如 NIXL `backends`，Multi `connectors`，Offloading `cpu_bytes_to_use`） |
| `kv_connector_module_path` | `None` | 外部 Python 模块路径，可注入第三方 connector |
| `enable_permute_local_kv` | `False` | 实验：HND→NHD permute |
| `kv_load_failure_policy` | `"fail"` | `"recompute"`：失败时重算；`"fail"`：直接失败 |

> [!warning] CONTRADICTION: `kv_rank` / `kv_parallel_size` 注释说 "Currently only 1P1D is supported"（[kv_transfer.py:46-48](d:\design\vllm\vllm\config\kv_transfer.py)），但 examples/disaggregated_serving/disagg_proxy_demo.py 注释说 demo XpYd（"X prefill, Y decode"，[examples/online_serving/disaggregated_serving/README.md:7](d:\design\vllm\examples\online_serving\disaggregated_serving\README.md)）—— XpYd 通过**前端代理**层实现（多个独立的 1P1D 进程对），不是 connector 层。

---

## 3. ~~14~~ 16 个 v1 backend 实现矩阵

> [!warning] STALE (2026-08-18)：本表为 5f7fab88 快照。d29dc3ab 注册表变动：**删** `P2pNcclConnector`（p2p/ 目录整体移除，-1436 行）；**增** `NixlPullConnector` / `NixlPushConnector`（`NixlConnector` 降级为 `NixlPullConnector` 的向后兼容别名，[nixl/connector.py:12, 390](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\connector.py)）与 `MooncakeStoreConnector`（[mooncake/store/connector.py](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\mooncake\store\connector.py)）。16 个注册名单见 §Increment.2。

### 3.1 完整注册表（来自 [factory.py:149-228](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\factory.py)）

| # | 名称 | 路径 | 类型 | 同步性 | 后端依赖 | 主要 IPC |
|---|---|---|---|---|---|---|
| 1 | `NixlConnector` | [nixl/](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl) | **目录式 (4 文件 + meta + stats + utils)** | 异步 D-pull | NVIDIA NIXL native lib | RDMA + ZMQ ROUTER 握手 |
| 2 | `MooncakeConnector` | [mooncake/](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\mooncake) | 目录式（2 文件） | 异步 D-pull | Mooncake C++ engine (`mooncake.engine.TransferEngine`) | ZMQ + bootstrap HTTP server |
| 3 | `MoRIIOConnector` | [moriio/](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\moriio) | 目录式（3 文件） | 异步 P-write | mori native lib（AMD） | ZMQ + handshake |
| 4 | `LMCacheConnectorV1` | [lmcache_connector.py](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\lmcache_connector.py) | 文件式 wrapper | 取决于子实现 | LMCache python pkg（含 `use_native` 二选一，见 §3.4） | LMCache 内部 |
| 5 | `LMCacheMPConnector` | [lmcache_mp_connector.py](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\lmcache_mp_connector.py) | 文件式（1132 行） | 异步多进程 | LMCache MP adapter | ZMQ + 共享内存 |
| 6 | `HF3FSKVConnector` | [hf3fs/](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\hf3fs) | **目录式 (3 文件 + utils + metadata server)** | 异步 store-fetch | `hf3fs_fuse.io` FUSE binding | 文件系统 + 独立 metadata server |
| 7 | ~~`P2pNcclConnector`~~ **RESOLVED 2026-08-18**：p2p/ 目录整体删除（p2p_nccl_connector.py / p2p_nccl_engine.py / tensor_memory_pool.py 均已不在 d29dc3ab） | — | — | — | — | — |
| 8 | `OffloadingConnector` | [offloading_connector.py](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\offloading_connector.py) + [offloading/](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\offloading) | 文件 + 目录（4 子文件） | 异步 store-fetch | `v1/kv_offload/` 块管理层 + `OffloadingSpecFactory` | 进程内 thread pool |
| 9 | `MultiConnector` | [multi_connector.py](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\multi_connector.py) | wrapper | 取决于子 connector | 任意子集 | 委派 |
| 10 | `FlexKVConnectorV1` | [flexkv_connector.py](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\flexkv_connector.py) | thin wrapper | 异步（scheduler 端管理） | `flexkv.integration.vllm.vllm_v1_adapter` | FlexKV server |
| 11 | `SimpleCPUOffloadConnector` | [simple_cpu_offload_connector.py](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\simple_cpu_offload_connector.py) | 文件式 | 异步 | `vllm.v1.simple_kv_offload.*`（自带最小 manager / worker / metadata） | 进程内 |
| 12 | `DecodeBenchConnector` | [decode_bench_connector.py](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\decode_bench_connector.py) | 文件式（385 行） | 同步（直接 `torch.fill` 假数据） | 无外部依赖 | benchmark 用 |
| 13 | `ExampleConnector` | [example_connector.py](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\example_connector.py) | 文件式 | safetensors 文件 IO | safetensors | 文件系统 |
| 14 | `ExampleHiddenStatesConnector` | [example_hidden_states_connector.py](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\example_hidden_states_connector.py) | 文件式 | 文件 IO | — | 教学样例 |

> synthesis: **目录式 vs 文件式**：4 个目录式 backend（NIXL / Mooncake / MoRIIO / HF3FS / Offloading）都把 scheduler / worker / metadata / utils 拆成多个文件；这些是**与外部 native lib 强绑定 + 实现复杂度最高**的 backend。10 个文件式 backend 多是 wrapper / 教学 / benchmark，逻辑较简单。

> synthesis (与 [comparison/topics/pd-disaggregation.md §3](../../comparison/topics/pd-disaggregation.md) 校准): pd-disaggregation 称 "13+ 实现"——本页清点后 **factory 注册 14 个 + lmcache_integration 是被 LMCacheConnectorV1 内部 lazy-import 的 helper 模块**（含 `vllm_v1_adapter` 与 `multi_process_adapter`，[lmcache_integration/__init__.py:5-11](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\lmcache_integration\__init__.py)），不算独立 connector，所以**真实独立 connector 数 = 14**（本页将 pd-disaggregation 的 "13+" 锁定为 14）。

### 3.2 NixlConnector — 异步 D-pull 范式

> [!warning] STALE (2026-08-18)：NIXL 子树已重构为 **base / pull / push 三层**（新增 8 文件：`base_scheduler.py` 510 行 / `base_worker.py` 2619 行 / `pull_scheduler.py` / `pull_worker.py` / `push_scheduler.py` 792 行 / `push_worker.py` / `tp_mapping.py` / `__init__.py`；原 `scheduler.py` -504 行、`worker.py` -2362 行）。本节 D-pull 叙事对 `NixlPullConnector`（= 现 `NixlConnector` 别名）仍近似成立，但行号全部失效；push 模式为全新语义（见 §Increment.3）。

[nixl/connector.py:56-284](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\connector.py) 是 thin facade：

- `NixlConnector(KVConnectorBase_V1, SupportsHMA)`（[connector.py:56](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\connector.py)）—— 唯一同时支持 HMA 的官方 backend（除 SimpleCPUOffload 外）
- 按 role 分别构造 `NixlConnectorScheduler` / `NixlConnectorWorker`（[connector.py:99-108](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\connector.py)）
- `get_required_kvcache_layout()` 强制返 `"HND"`（除 MLA）以利 RDMA 传输（[connector.py:113-130](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\connector.py)）

**Scheduler 端核心字段**（[nixl/scheduler.py:52-120](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\scheduler.py)）：

```python
# scheduler.py:62-66
self.side_channel_host = envs.VLLM_NIXL_SIDE_CHANNEL_HOST
self.side_channel_port = envs.VLLM_NIXL_SIDE_CHANNEL_PORT + dp_idx
# scheduler.py:92-105
self._nixl_handshake_listener_t: threading.Thread | None = None
self._reqs_need_recv: dict[ReqId, tuple[Request, BlockIds]] = {}
self._reqs_need_save: dict[ReqId, Request] = {}
self._reqs_need_send: dict[ReqId, float] = {}  # 带 expire_time
self._reqs_in_batch: set[ReqId] = set()
self._reqs_not_processed: set[ReqId] = set()
```

`get_num_new_matched_tokens` 入口（[nixl/scheduler.py:264-302](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\scheduler.py)）：

```python
params = request.kv_transfer_params
if params is not None and params.get("do_remote_prefill"):
    # Remote prefill: get all prompt blocks from remote.
    token_ids = request.prompt_token_ids or []
    actual = self._mamba_prefill_token_count(len(token_ids))  # Mamba 减 1
    count = actual - num_computed_tokens
    if count > 0:
        return count, True  # ← async=True 触发 WAITING_FOR_REMOTE_KVS
...
return 0, False
```

握手：当 `set_xfer_handshake_metadata` 被调时启动 `_nixl_handshake_listener` 后台线程（[nixl/scheduler.py:155-229](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\scheduler.py)），用 `zmq.ROUTER` socket 暴露 `NixlHandshakePayload`（msgpack 编码）；D 端通过该 socket 拿到 P 的 NIXL agent metadata 后才能 `_read_blocks`。

**Worker 端**（[nixl/worker.py:81-2280](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\worker.py)）：

- `start_load_kv(metadata)` 注释明示 "Start loading by triggering non-blocking nixl_xfer"（[worker.py:1849](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\worker.py)）
- `_read_blocks_for_req(req_id, meta)`（[worker.py:1904](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\worker.py)）→ `_read_blocks(...)`（[worker.py:1988-2049](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\worker.py)）→ 完成后 `nixl_wrapper.send_notif(agent_name, notif_msg=notif_id)`（[worker.py:2049](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\worker.py)）通知 P 端释放
- 注释 "D pulls the whole kv cache from corresponding [P]"（[worker.py:1276](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\worker.py)）

**关键 cross-language binding**（[nixl/utils.py:21-58](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\utils.py)）：

```python
if not current_platform.is_rocm():
    from nixl._api import nixl_agent as NixlWrapper
    from nixl._bindings import nixlXferTelemetry
else:
    from rixl._api import nixl_agent as NixlWrapper      # ROCm fallback
    from rixl._bindings import nixlXferTelemetry
```

环境变量副作用：[nixl/utils.py:22-35](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\utils.py) 自动设 `UCX_RCACHE_MAX_UNRELEASED=1024` 防内存泄漏（链接 issue #24264）。

支持 KV buffer 设备矩阵（[utils.py:62-75](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\utils.py)）：

```python
_NIXL_SUPPORTED_DEVICE = {
    "cuda": ("cuda", "cpu"),
    "tpu": ("cpu",),
    "xpu": ("cpu", "xpu"),
    "cpu": ("cpu",),
}
```

### 3.3 MooncakeConnector — D-pull 的另一形态

> [!warning] STALE (2026-08-18)：mooncake_connector.py 本期 ±769 行 churn，且 mooncake/ 新增 `rdma_utils.py` / `stats.py` / **`store/` 子树 7 文件（约 4100 行）**——后者是独立注册的 `MooncakeStoreConnector`（Mooncake 分布式 KV store 作为 storage backend，与本节 P/D 直传的 `MooncakeConnector` 不同物），见 §Increment.4。下文类行号未重核。

[mooncake/mooncake_connector.py](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\mooncake\mooncake_connector.py) 共 1693 行。关键类层次：

| 类 | 行号 | 角色 |
|---|---|---|
| `TransferRegion` | [75](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\mooncake\mooncake_connector.py) | 注册的 KV region（含 base_addr / block_len） |
| `MooncakeXferMetadata` | [247](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\mooncake\mooncake_connector.py) | 拆分 / 路由 metadata |
| `MooncakeXferResponseStatus(IntEnum)` | [260](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\mooncake\mooncake_connector.py) | 响应状态码 enum |
| `PullReqMeta` | [280-289](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\mooncake\mooncake_connector.py) | D 端拉取请求 dataclass，含 `expire_time` 与 `pull_tasks_count`（"Designed for one D pairing to multiple P"） |
| `SendBlockMeta` | [292-301](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\mooncake\mooncake_connector.py) | P 端发送 dataclass，含 `need_send / sent / sending` 三计数 |
| `MooncakeConnectorMetadata` | [304-330](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\mooncake\mooncake_connector.py) | `reqs_to_recv: dict[EngineId, dict[ReqId, PullReqMeta]]` + `reqs_to_send` + `reqs_not_processed` |
| `MooncakeConnector(KVConnectorBase_V1)` | [333](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\mooncake\mooncake_connector.py) | facade |
| `MooncakeConnectorScheduler` | [445](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\mooncake\mooncake_connector.py) | scheduler 端 |
| `MooncakeConnectorWorker` | [641](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\mooncake\mooncake_connector.py) | worker 端 |

**关键 cross-language binding**（[mooncake_connector.py:55-63](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\mooncake\mooncake_connector.py)）：

```python
try:
    from mooncake.engine import TransferEngine
except ImportError:
    logger.warning("Please install mooncake by following the instructions at "
        "https://github.com/kvcache-ai/Mooncake/blob/main/doc/en/build.md ...")
    TransferEngine = None
```

`mooncake.engine.TransferEngine` 是 **Mooncake C++ engine** 的 Python binding。

### 3.4 LMCache 三变体

| Connector | 触发条件 | 行号 |
|---|---|---|
| `LMCacheConnectorV1` 走 **vendored adapter** | `kv_connector_extra_config["use_native"] = True` | [lmcache_connector.py:96-103](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\lmcache_connector.py) → import `vllm.distributed.kv_transfer.kv_connector.v1.lmcache_integration.vllm_v1_adapter` |
| `LMCacheConnectorV1` 走 **upstream LMCache pkg** | `use_native = False`（默认） | [lmcache_connector.py:104-111](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\lmcache_connector.py) → import `lmcache.integration.vllm.vllm_v1_adapter.LMCacheConnectorV1Impl` |
| `LMCacheMPConnector` | 多进程 LMCache 场景 | [lmcache_mp_connector.py:25-31](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\lmcache_mp_connector.py) → `lmcache.integration.vllm.vllm_multi_process_adapter` |

vendored adapter 内部 7 个核心类（[lmcache_integration/vllm_v1_adapter.py](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\lmcache_integration\vllm_v1_adapter.py)）：`LoadSpec` / `SaveSpec` / `DisaggSpec` / `RequestTracker` / `ReqMeta` / `LMCacheConnectorMetadata` / `LMCacheConnectorV1Impl`（行 71 / 81 / 89 / 121 / 249 / 556 / 570）。

### 3.5 OffloadingConnector — 与 v1/kv_offload/ 的协作

[offloading_connector.py:44-178](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\offloading_connector.py)：

- `prefer_cross_layer_blocks = True`（[offloading_connector.py:46-47](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\offloading_connector.py)）—— 触发 mixin 的 uniform layout path
- 用 `OffloadingSpecFactory.create_spec(vllm_config, kv_cache_config)` 拿 spec（[offloading_connector.py:58](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\offloading_connector.py)）
- `wait_for_save()` → `connector_worker.prepare_store_kv(metadata)`（[offloading_connector.py:105-108](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\offloading_connector.py)）
- `start_load_kv(...)` → `connector_worker.start_kv_transfers(metadata)`（[offloading_connector.py:88-91](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\offloading_connector.py)）
- `handle_preemptions(metadata)`（[offloading_connector.py:83-86](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\offloading_connector.py)）—— 唯一在 base 标 default no-op 但实际用上的 backend

`OffloadingConnectorScheduler` 持核心状态（[offloading/scheduler.py:112-130](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\offloading\scheduler.py)）：

```python
self.config = SchedulerOffloadConfig.from_spec(spec)
self.manager: OffloadingManager = spec.get_manager()
self._req_status: dict[ReqId, RequestOffloadState] = {}
self._reqs_to_load: dict[ReqId, TransferSpec] = {}
self._blocks_being_loaded: set[OffloadKey] | None = (set() if prefix_caching else None)
self._reqs_being_stored = defaultdict[ReqId, set[OffloadKey]](set)
self._reqs_being_loaded = defaultdict[ReqId, set[OffloadKey]](set)
```

`get_num_new_matched_tokens` 用 `manager.touch(offload_keys)` + `manager.lookup(offload_keys[start_block_idx:])`（[offloading/scheduler.py:176-184](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\offloading\scheduler.py)）—— **store-fetch 异步语义**：lookup 返 `None` 让 scheduler 稍后重试。

### 3.6 MultiConnector — storage layer wrapper（非 P+D 角色合并）

[multi_connector.py:126-582](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\multi_connector.py)。**关键语义注释**（[multi_connector.py:130-133](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\multi_connector.py)）：

```
The current logic is:
- Load KV from the first connector that advertises available tokens from
  get_num_new_matched_tokens(), based on the order in the config.
- Save to all connectors.
```

核心字段（[multi_connector.py:161-177](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\multi_connector.py)）：

```python
self._connectors: list[KVConnectorBase_V1] = []
self._ktc_kv_transfer_config = []
# req_id → 选中的 connector 索引（用于 load）
self._requests_to_connector: dict[str, int] = {}
# req_id → 还有几个 connector 没完成 async save（防 race）
self._extra_async_saves: dict[str, int] = {}
```

`get_num_new_matched_tokens` 实现"first hit"语义（[multi_connector.py:350-369](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\multi_connector.py)）：第一个返回 `toks > 0` 的 connector 被选中，其余 connector 仍被遍历但忽略；任一 connector 返 `None` 立即 return `(None, False)` 让 scheduler 重试。

`request_finished` 多 async save 计数（[multi_connector.py:439-464](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\multi_connector.py)）：

```python
async_saves = 0
for c in self._connectors:
    async_save, txfer_params = c.request_finished(request, blocks)
    if async_save:
        async_saves += 1
    ...
if async_saves > 1:
    self._extra_async_saves[request.request_id] = async_saves - 1  # 多余的等待计数
```

`get_finished` 匹配地"消化"额外计数（[multi_connector.py:280-305](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\multi_connector.py)）：每个 connector 报 finished_sending 时只有当 `_extra_async_saves[req_id]` 减到 0 才把 req 加进聚合 set。

`MultiKVConnectorMetadata` 含 `extra_async_saves: dict[str, int]`（[multi_connector.py:43-46](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\multi_connector.py)）随 metadata 传给 worker。

`MultiKVConnectorWorkerMetadata.aggregate(other)`（[multi_connector.py:53-66](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\multi_connector.py)）：多 worker 聚合时按 connector 索引位置成对调子 connector 的 `aggregate`。

> [!warning] CONTRADICTION: `MultiConnector` 名称容易让人误以为是 P+D **角色合并**（同一进程同时承担 P/D），但源码注释明示是 **多 storage layer 同时存 KV** 的 wrapper。pd-disaggregation §2 已记录 "通过 `kv_role=both` + `MultiConnector`，间接支持" 同节点 P+D，但实际**实现路径仍是 `kv_role=kv_both`**（[kv_transfer.py:11-13](d:\design\vllm\vllm\config\kv_transfer.py)），`MultiConnector` 只负责 storage layer 多路复用。

### 3.7 其它 backend 简表

| Backend | scheduler 状态 | 备注 |
|---|---|---|
| `MoRIIOConnector` | `_reqs_in_batch` / 与 `do_remote_prefill` / `do_remote_decode` 同 NIXL pattern（[moriio_connector.py](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\moriio\moriio_connector.py)；factory 中 `mori.io.{BackendType, IOEngine, IOEngineConfig}` import [moriio_connector.py:71-76](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\moriio\moriio_connector.py)） | 与 NIXL 互为 GPU 厂商替代（NVIDIA / AMD） |
| `HF3FSKVConnector` | 含 `AsyncOperationManager`（[hf3fs_connector.py:101-468](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\hf3fs\hf3fs_connector.py)），`HF3FSKVConnector(KVConnectorBase_V1)` 在行 469；FUSE binding `from hf3fs_fuse.io import deregister_fd`（[hf3fs_connector.py:75](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\hf3fs\hf3fs_connector.py)），import 失败时 fallback 到 `Hf3fsClient` mock（[hf3fs_connector.py:82-88](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\hf3fs\hf3fs_connector.py)） | 独立 metadata server 进程 [hf3fs_metadata_server.py](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\hf3fs\hf3fs_metadata_server.py) |
| ~~`P2pNcclConnector`~~ | **RESOLVED 2026-08-18**：p2p/ 目录（p2p_nccl_connector.py / p2p_nccl_engine.py / tensor_memory_pool.py）在 d29dc3ab 已整体删除 | — |
| `FlexKVConnectorV1` | `start_load_kv` 是 no-op；KV transfer 由 **scheduler 端 `build_connector_meta`/`update_connector_output`** 全权管理（[flexkv_connector.py:82-90](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\flexkv_connector.py)）注释明示 "similar to how NIXL operates" | 依赖 `flexkv.integration.vllm.vllm_v1_adapter.FlexKVConnectorV1Impl`（[flexkv_connector.py:65-72](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\flexkv_connector.py)） |
| `SimpleCPUOffloadConnector` | `(KVConnectorBase_V1, SupportsHMA)`（[simple_cpu_offload_connector.py:45](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\simple_cpu_offload_connector.py)），自带极简 `vllm.v1.simple_kv_offload.*` | 默认 8GB CPU buffer（[simple_cpu_offload_connector.py:42](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\simple_cpu_offload_connector.py)） |
| `DecodeBenchConnector` | "Emulates a prefill-decode disaggregated setting by filling the KV cache with dummy values"（[decode_bench_connector.py:6-9](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\decode_bench_connector.py)），`fill_mean` / `fill_std` 配置正态分布填充 | 纯 benchmark，无外部依赖 |
| `Example*Connector` | safetensors 文件 IO 形式的教学样例 | — |

---

## 4. v1/kv_offload/ — Worker 端块管理层

与 connector 是**两层**结构：connector 定义"协议"，offload 定义"块管理 + handler"。被 `OffloadingConnector` 直接复用，也是其它 backend 设计的参考。

### 4.1 抽象层

**`OffloadingManager(ABC)`**（[abstract.py:87-184](d:\design\vllm\vllm\v1\kv_offload\abstract.py)）—— scheduler 端块管理：

| 方法 | 行号 | 语义 |
|---|---|---|
| `lookup(keys) -> int \| None` | [89-103](d:\design\vllm\vllm\v1\kv_offload\abstract.py) | 找 prefix 中已 offloaded 的最大长度；返 `None` 让 scheduler 重试 |
| `prepare_load(keys) -> LoadStoreSpec` | [106](d:\design\vllm\vllm\v1\kv_offload\abstract.py) | 标块为不可 evict + 返 spec |
| `touch(keys)` | [122](d:\design\vllm\vllm\v1\kv_offload\abstract.py) | LRU 用，与 prepare_load 分离允许 GPU prefix hit 也更新 LRU |
| `complete_load(keys)` | [132](d:\design\vllm\vllm\v1\kv_offload\abstract.py) | 解 evict 锁 |
| `prepare_store(keys) -> PrepareStoreOutput \| None` | [142](d:\design\vllm\vllm\v1\kv_offload\abstract.py) | 返 `keys_to_store / store_spec / evicted_keys` |
| `complete_store(keys, success)` | [159](d:\design\vllm\vllm\v1\kv_offload\abstract.py) | 标 store 完成，failed 时清掉未 store 的 |
| `take_events()` | [172](d:\design\vllm\vllm\v1\kv_offload\abstract.py) | 返 `OffloadingEvent` |

**`LoadStoreSpec(ABC)`** + 子类（[abstract.py:56-69](d:\design\vllm\vllm\v1\kv_offload\abstract.py), [mediums.py:11-71](d:\design\vllm\vllm\v1\kv_offload\mediums.py)）：

| 类 | 行号 | medium |
|---|---|---|
| `BlockIDsLoadStoreSpec` | [mediums.py:11-20](d:\design\vllm\vllm\v1\kv_offload\mediums.py) | 抽象，持 `np.ndarray` 块 IDs |
| `GPULoadStoreSpec` | [mediums.py:23-60](d:\design\vllm\vllm\v1\kv_offload\mediums.py) | `"GPU"`，含 `group_sizes` 与 `block_indices`（用于"offloaded block 比 GPU block 大"时跳过首块前缀） |
| `CPULoadStoreSpec` | [mediums.py:63-70](d:\design\vllm\vllm\v1\kv_offload\mediums.py) | `"CPU"` |

**`OffloadKey = NewType("OffloadKey", bytes)`**（[abstract.py:38-53](d:\design\vllm\vllm\v1\kv_offload\abstract.py)）：`block_hash + group_idx.to_bytes(4, "big")` 编码到一个 bytes，避免 tuple GC 开销。

### 4.2 OffloadingSpec + Factory

**`OffloadingSpec(ABC)`**（[spec.py:71-142](d:\design\vllm\vllm\v1\kv_offload\spec.py)）：

- `__init__` 校验 `gpu_block_size % hash_block_size == 0`（[spec.py:95-101](d:\design\vllm\vllm\v1\kv_offload\spec.py)）—— 与 prefix caching 对齐
- `block_size_factor = offloaded_block_size // gpu_block_size`（[spec.py:104-118](d:\design\vllm\vllm\v1\kv_offload\spec.py)）—— offloaded block 必须是 GPU block 的整数倍
- `get_manager() -> OffloadingManager`（abstract）
- `get_handlers(kv_caches) -> Iterator[(src_type, dst_type, OffloadingHandler)]`（abstract）

**`CanonicalKVCaches`** dataclass（[spec.py:51-68](d:\design\vllm\vllm\v1\kv_offload\spec.py)）：把 attention 后端不同 layout（如 FlashAttention 的 `(2, num_blocks, ...)`）规范化成 `(num_blocks, page_size_in_bytes)` int8 tensor + 每组 layer 的 ref。

**`OffloadingSpecFactory`**（[factory.py:17-58](d:\design\vllm\vllm\v1\kv_offload\factory.py)）：注册表 + lazy import。**默认只注册 `CPUOffloadingSpec`**（[factory.py:55-58](d:\design\vllm\vllm\v1\kv_offload\factory.py)），其它 spec 通过 `kv_connector_extra_config["spec_module_path"]` 动态加载（[factory.py:43-49](d:\design\vllm\vllm\v1\kv_offload\factory.py)）。

### 4.3 CPUOffloadingSpec — 唯一内置实现

[cpu/spec.py:17-101](d:\design\vllm\vllm\v1\kv_offload\cpu\spec.py)：

- `cpu_bytes_to_use` 必须显式指定（[cpu/spec.py:21-25](d:\design\vllm\vllm\v1\kv_offload\cpu\spec.py)）
- `eviction_policy: "lru" | "arc"`（[cpu/spec.py:54](d:\design\vllm\vllm\v1\kv_offload\cpu\spec.py)）
- `get_manager()` 返 `CPUOffloadingManager`（[cpu/spec.py:56-67](d:\design\vllm\vllm\v1\kv_offload\cpu\spec.py)）；可选包一层 `FilterReusedOffloadingManager`（[reuse_manager.py:22-115](d:\design\vllm\vllm\v1\kv_offload\reuse_manager.py)）—— **重用次数 < threshold 不 offload**（默认 0 不过滤；>=2 才生效，[cpu/spec.py:69-81](d:\design\vllm\vllm\v1\kv_offload\cpu\spec.py)）
- `get_handlers()` yield 双向 handler：`GPULoadStoreSpec → CPULoadStoreSpec` 与 `CPULoadStoreSpec → GPULoadStoreSpec`（[cpu/spec.py:99-101](d:\design\vllm\vllm\v1\kv_offload\cpu\spec.py)）

`CPUOffloadingManager`（[cpu/manager.py:24-...](d:\design\vllm\vllm\v1\kv_offload\cpu\manager.py)）—— 共享 ref-count + event emit + block pool 管理，cache policy 通过 `_CACHE_POLICIES = {"lru": LRUCachePolicy, "arc": ARCCachePolicy}` 注入（[cpu/manager.py:18-21](d:\design\vllm\vllm\v1\kv_offload\cpu\manager.py)）。

### 4.4 Worker 端 handler

**`OffloadingHandler(ABC)`** + **`OffloadingWorker`**（[worker/worker.py:26-178](d:\design\vllm\vllm\v1\kv_offload\worker\worker.py)）：

```python
TransferSpec = tuple[LoadStoreSpec, LoadStoreSpec]    # (src, dst)
TransferType = tuple[str, str]                         # ("GPU", "CPU")

@dataclass
class TransferResult:
    job_id: int
    success: bool
    transfer_size: int | None = None
    transfer_time: float | None = None
    transfer_type: TransferType | None = None
```

`OffloadingHandler` 三抽象方法：`transfer_async(job_id, spec)` / `get_finished()` / `wait(job_ids)`（[worker/worker.py:39-70](d:\design\vllm\vllm\v1\kv_offload\worker\worker.py)）。

**`CpuGpuOffloadingHandlers`** 实现（[worker/cpu_gpu.py:25-...](d:\design\vllm\vllm\v1\kv_offload\worker\cpu_gpu.py)）：

- `Transfer` dataclass 持 `cuda.Stream + start_event + end_event + num_bytes`（[worker/cpu_gpu.py:25-31](d:\design\vllm\vllm\v1\kv_offload\worker\cpu_gpu.py)）
- 用 `vllm._custom_ops` 的自定义 kernel + `is_pin_memory_available()` 决定是否 pin host buffer（[worker/cpu_gpu.py:10-12](d:\design\vllm\vllm\v1\kv_offload\worker\cpu_gpu.py)）
- 双方向 handler：`gpu_to_cpu_handler` / `cpu_to_gpu_handler`

---

## 5. ~~entrypoints/serve/disagg/~~ entrypoints/scale_out/ — HTTP 前端

> [!warning] STALE (2026-08-18) **RESOLVED 2026-08-18**：`vllm/entrypoints/serve/disagg/` 目录在 d29dc3ab **已不存在**。"tokens-in/tokens-out"前端迁至 [vllm/entrypoints/scale_out/token_in_token_out/](d:\design\vllm\vllm\entrypoints\scale_out\token_in_token_out)（api_router.py / serving.py / protocol.py / mm_serde.py，`ServingTokens` 与 `/inference/v1/generate` 端点在此），`scale_out/` 下另有 `derender/`、`render/`、`factories.py`。下文 3 文件表按旧路径书写，类名/端点语义经 grep 确认仍在新路径（`ServingTokens` 命中 scale_out/token_in_token_out/serving.py）。

3 文件构成"**Disaggregated Everything**"前端：

| 文件 | 类 / 端点 | 锚点 |
|---|---|---|
| [api_router.py](d:\design\vllm\vllm\entrypoints\serve\disagg\api_router.py) | `POST /inference/v1/generate`（流式 SSE） + `POST /abort_requests`（仅 `tokens_only` 模式） | [api_router.py:46-105](d:\design\vllm\vllm\entrypoints\serve\disagg\api_router.py) |
| [serving.py](d:\design\vllm\vllm\entrypoints\serve\disagg\serving.py) | `ServingTokens(OpenAIServing)` + `serve_tokens(request, raw_request)` | [serving.py:46-393](d:\design\vllm\vllm\entrypoints\serve\disagg\serving.py) |
| [protocol.py](d:\design\vllm\vllm\entrypoints\serve\disagg\protocol.py) | `GenerateRequest` / `GenerateResponse` / `MultiModalFeatures` / `PlaceholderRangeInfo` | [protocol.py:1-162](d:\design\vllm\vllm\entrypoints\serve\disagg\protocol.py) |

`GenerateRequest` 关键字段（[protocol.py:55-114](d:\design\vllm\vllm\entrypoints\serve\disagg\protocol.py)）：
- `token_ids: list[int]`（**tokens-in，不再 detokenize**）
- `features: MultiModalFeatures | None`（`mm_hashes` + `mm_placeholders`，[protocol.py:32-52](d:\design\vllm\vllm\entrypoints\serve\disagg\protocol.py)）
- `kv_transfer_params: dict[str, Any] | None`（[protocol.py:105-108](d:\design\vllm\vllm\entrypoints\serve\disagg\protocol.py)）—— **PD 路由信息透传通道**

`/abort_requests` 端点注释（[api_router.py:84-87](d:\design\vllm\vllm\entrypoints\serve\disagg\api_router.py)）："To be used in a Disaggregated Everything setup"。该端点仅在 `app.state.args.tokens_only=True` 才挂载。

> synthesis: 这个前端的设计哲学 = "**外部代理 / 路由器决定 P/D，前端 vLLM 只做 tokens-in tokens-out + kv_transfer_params 透传**"。examples 里的 `disagg_proxy_demo.py`（XpYd）/ `mooncake_connector_proxy.py` / `moriio_toy_proxy_server.py` 都是这种**外部代理**实例（详见 §跨子系统引用 第 5 类）。

---

## 6. 关键命名陷阱与 contradictions

### 6.1 `do_remote_prefill` / `do_remote_decode` 命名（已在 pd-disaggregation §Notes [!warning] CONTRADICTION NEW 标注）

`request.kv_transfer_params` 字段意义（[nixl/scheduler.py:282-302](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\scheduler.py), [mooncake_connector.py:498, 547](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\mooncake\mooncake_connector.py)）：

| 字段 | 出现在哪一端 req 上 | 真实含义 |
|---|---|---|
| `do_remote_prefill: True` | **D 端 req** | "我（D）的 prefill 在远端 P 做了，本地需要 pull 全部 prompt blocks" |
| `do_remote_decode: True` | **P 端 req** | "我（P）的 decode 在远端 D 做，需要把 KV 准备好让 D 来 pull" |
| `remote_engine_id` / `remote_request_id` / `remote_host` / `remote_port` / `remote_block_ids` / `remote_bootstrap_addr` | D 端 req（搭配 `do_remote_prefill`） | 路由信息：去哪个 P 拉 |
| `transfer_id` | P/D 共享 | KV transfer 协调 ID（Mooncake 用，[mooncake_connector.py:71, 319](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\mooncake\mooncake_connector.py)） |

> [!warning] CONTRADICTION (确认): `do_remote_prefill` 字面像"本地做 remote prefill"，但实际是"**让本地等远端 prefill 完后拉 KV**"——以"意图视角"读名字会反向理解。源码注释（NIXL [scheduler.py:282-302](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\scheduler.py)）只说语义不解释命名。Mooncake `request_finished` 中"P 端检查 `if do_remote_prefill: ... params['do_remote_prefill'] = False`"（[mooncake_connector.py:608-617](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\mooncake\mooncake_connector.py)）也证实该字段是"D 端的 expectation 标记"。

### 6.2 块级失败粒度（vLLM 独有，与 MindIE / SGLang 对比）

`get_block_ids_with_load_errors() -> set[int]`（[base.py:381-399](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py)）—— 注释明示：

```
- Async loading: failed blocks may be reported in any forward pass up to and
  including the pass where the request ID is returned by `get_finished()`.
- Sync loading: failed blocks should be reported in the forward pass in which
  they are detected.
```

scheduler `_update_requests_with_invalid_blocks` 配合（[scheduler.py:2132-...](d:\design\vllm\vllm\v1\core\sched\scheduler.py)）+ `KVConnectorOutput.invalid_block_ids`（[outputs.py:135-137](d:\design\vllm\vllm\v1\outputs.py)）+ `kv_load_failure_policy: "recompute" | "fail"`（[kv_transfer.py:70-73](d:\design\vllm\vllm\config\kv_transfer.py)）。

NIXL worker 端实现：在 `recv_complete` 失败时 `self._invalid_block_ids.update(meta.local_block_ids[0])`（[worker.py:1842-1843](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\worker.py)）。

> synthesis: 与 SGLang 的 `KVPoll.Failed` + req 级 abort 相比，**vLLM 的块级粒度允许"部分块失败时只重算失败的几块，已成功的块继续保留"**——尤其在 `kv_load_failure_policy=recompute` 时把"重传开销"压到最小。

### 6.3 `prefer_cross_layer_blocks` + `requires_piecewise_for_cudagraph`

[base.py:175-181, 579-599](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py)：

- `prefer_cross_layer_blocks: bool` —— True 时一个块同时含全 layer KV，便于"一次 RDMA 跨层"。NIXL 在 FLASH_ATTN/FLASHINFER/TRITON_ATTN + HND layout + 显式 `enable_cross_layers_blocks=true` 时 True（[nixl/connector.py:58-85](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\connector.py)），Offloading 永远 True（[offloading_connector.py:46-47](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\offloading_connector.py)），MultiConnector 取所有子 connector AND（[multi_connector.py:179-183](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\multi_connector.py)）。
- `requires_piecewise_for_cudagraph(extra_config) -> bool`（classmethod）—— layerwise op (`wait_for_layer_load` / `save_kv_layer`) 不能 capture 进 CUDA graph，需 PIECEWISE 模式让 Python code 在 graph piece 之间执行。LMCache `use_layerwise=True` 时 True（[lmcache_connector.py:73-81](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\lmcache_connector.py)），MultiConnector 取所有子 connector OR（[multi_connector.py:136-149](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\multi_connector.py)）。

---

## 7. 跨子系统引用（[AGENTS.md §5 step 3](../../AGENTS.md) — 5 类 grep 结果）

### 7.1 跨语言绑定（外部 native lib / FUSE / NCCL binding）

| Backend | 绑定 | 锚点 |
|---|---|---|
| NIXL | `from nixl._api import nixl_agent as NixlWrapper` + `from nixl._bindings import nixlXferTelemetry`；ROCm fallback `from rixl._api/_bindings`（[nixl/utils.py:38-42](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\utils.py)） + `from nixl._api import nixl_agent_config`（[utils.py:53-55](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\utils.py)）。**全仓库 grep `from nixl` / `import nixl` 命中 4 文件**：`v1/nixl/utils.py`、`model_executor/layers/fused_moe/nixl_ep_prepare_finalize.py`（**注意：MoE 也用 NIXL，与 KV connector 共享 NIXL agent**）、`device_communicators/all2all.py`、`tests/v1/kv_connector/unit/test_nixl_connector.py` |
| Mooncake | `from mooncake.engine import TransferEngine`（[mooncake_connector.py:55-63](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\mooncake\mooncake_connector.py)）。**全仓库 grep `mooncake.engine` 命中 1 文件**（仅 mooncake_connector.py），但 grep 'TransferEngine' 命中 16 个文件包括 `weight_transfer/` 子系统 —— **vLLM 的 weight transfer 也定义了同名 `TransferEngine` 但是独立类**（[distributed/weight_transfer/base.py](d:\design\vllm\vllm\distributed\weight_transfer\base.py)），与 KV connector 的 mooncake `TransferEngine` 同名但无关 |
| MoRIIO | `from mori.io import BackendType, IOEngine, IOEngineConfig`（[moriio_connector.py:71-76](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\moriio\moriio_connector.py)）；含 `MoRIIOWrapper` / `MoRIIOWriter` Python wrapper 类（[moriio_engine.py](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\moriio\moriio_engine.py)） |
| HF3FS | `from hf3fs_fuse.io import deregister_fd`（[hf3fs_connector.py:75](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\hf3fs\hf3fs_connector.py)）—— **FUSE 文件系统 binding**；import 失败时 fallback 到 `Hf3fsClient` mock 实现（[utils/hf3fs_mock_client.py](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\hf3fs\utils\hf3fs_mock_client.py)）保证 unit test 能跑 |
| LMCache | `from lmcache.integration.vllm.vllm_v1_adapter import LMCacheConnectorV1Impl`（默认 path，[lmcache_connector.py:107-109](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\lmcache_connector.py)）；MP 版 `from lmcache.integration.vllm.vllm_multi_process_adapter import LMCacheMPSchedulerAdapter, LMCacheMPWorkerAdapter`（[lmcache_mp_connector.py:25-31](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\lmcache_mp_connector.py)） |
| FlexKV | `from flexkv.integration.vllm.vllm_v1_adapter import FlexKVConnectorV1Impl`（[flexkv_connector.py:65-72](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\flexkv_connector.py)）（lazy in `__init__`，import 失败给"see github 安装说明"友好错误） |
| ~~P2P-NCCL~~ | **RESOLVED 2026-08-18**：p2p/ 目录已随 `P2pNcclConnector` 删除 |

> synthesis: vLLM 的"connector 多 backend"实质是 **"统一 ABC + 14 个独立 native lib lazy import"** —— 任意一个 backend 不存在不影响其它 backend 启动，import 失败 fallback 到 mock 或 raise 友好错误。这是与 SGLang `disaggregation/{nixl,mooncake,mori,ascend}/conn.py` "同位平铺"风格的最大差异。

### 7.2 协作伙伴跨子系统引用

| 符号 | 在 `d:\design\vllm\` 全仓库 grep 命中数 / 文件 |
|---|---|
| `KVConnectorBase_V1` | 在 `vllm/distributed/kv_transfer/kv_connector/v1/` 全部 14 个 connector 文件命中（每个 backend 都 import）；scheduler.py 1 处（[scheduler.py:23](d:\design\vllm\vllm\v1\core\sched\scheduler.py)） |
| `KVConnectorRole` | 在 14 个 connector + factory.py + scheduler.py + multi_connector.py 命中（**确认每个 backend 都按 role 分支构造**） |
| `Scheduler` (`self.connector`) | `scheduler.py` 46 处 self.connector 引用 |
| `KVCacheManager` | `scheduler.py` 调 `self.kv_cache_manager.cache_blocks` / `.free` 在 PD 失败兜底路径（[scheduler.py:2051, 2055](d:\design\vllm\vllm\v1\core\sched\scheduler.py)） |
| `EngineCore` | EngineCore 不直接引用 connector（connector 是 Scheduler 字段） |
| `KVConnectorOutput` | 在 v1/outputs.py 定义（[outputs.py:127-153](d:\design\vllm\vllm\v1\outputs.py)）+ scheduler.py 用 + kv_connector_model_runner_mixin.py 写出 + 14 个 backend get_finished 路径回填 |
| `OffloadingManager` | `v1/kv_offload/abstract.py`（[abstract.py:87](d:\design\vllm\vllm\v1\kv_offload\abstract.py)）+ `cpu/manager.py:24` 子类 + `offloading_connector.py` 间接通过 `OffloadingSpecFactory.create_spec` 拿 |
| `OffloadingHandler` | `v1/kv_offload/worker/worker.py:26`（ABC）+ `cpu_gpu.py` 子类 |
| `MultiConnector` | 注册 1 处（factory.py），实际只在 `multi_connector.py` 与 tests 出现 |

### 7.3 配置 / IPC 共享数据结构

| 关键字 | 命中范围 |
|---|---|
| `KVTransferConfig` 字段名 | `vllm/config/kv_transfer.py:23-122` 定义；engine 路径 `vllm/v1/engine/{core.py, utils.py}` + executor `vllm/v1/executor/ray_executor.py` + worker `vllm/v1/worker/worker_base.py` 多处读 `vllm_config.kv_transfer_config` |
| `kv_connector_extra_config` | 各 backend 用作"per-backend 自定义参数"通道（NIXL `backends`/`enable_cross_layers_blocks`、Multi `connectors`、Offloading `cpu_bytes_to_use`/`spec_name`/`spec_module_path`/`block_size`、LMCache `use_native`/`use_layerwise`、HF3FS `mount_path` 等） |
| `kv_transfer_params: dict` | 透传字段，跨 frontend (`entrypoints/serve/disagg/protocol.py:105-108`) → engine req → scheduler → connector 的 4 段链路（**字段名集合: `do_remote_prefill / do_remote_decode / remote_engine_id / remote_request_id / remote_host / remote_port / remote_block_ids / remote_bootstrap_addr / transfer_id / _p_side_truncated`**） |
| `KVConnectorMetadata` | 抽象 base + 各 backend 子类（`NixlConnectorMetadata` / `MooncakeConnectorMetadata` / `MoRIIOConnectorMetadata` / `OffloadingConnectorMetadata` / `MultiKVConnectorMetadata` / `LMCacheConnectorMetadata` / `HF3FSConnectorMetadata` / `P2pNcclConnectorMetadata` / `SimpleCPUOffloadMetadata` 等） |
| `KVConnectorOutput` 字段 | scheduler 端 `_update_from_kv_xfer_finished` 解析 `finished_recving / finished_sending / invalid_block_ids`；mixin 写 `kv_connector_stats / kv_cache_events / kv_connector_worker_meta` |

### 7.4 测试覆盖反查

`d:\design\vllm\tests\v1\kv_connector\` 下 **2 个子目录**：

| 目录 | 文件数 | 关键文件 |
|---|---|---|
| `unit/` | **26 个 .py**（每个 backend 都有专用 test + 5 个跨 connector 集成测试） | `test_kv_connector_lifecycle.py`、`test_remote_prefill_lifecycle.py`、`test_remote_decode_lifecycle.py`、`test_multi_connector.py`、`test_kv_load_failure_recovery.py`、`test_invalid_blocks_correctness.py`、`test_cache_pollution_prevention.py`、`test_error_propagation.py`、`test_output_aggregator.py`、`test_kv_cache_layout.py`、`test_backwards_compatibility.py`、`test_scheduler_kv_connector_override.py`、`test_config.py` + 14 个 backend 一对一 test + `unit/offloading_connector/` 5 个子文件 |
| `nixl_integration/` | 9 个 .sh + 1 .py | `run_accuracy_test.sh` / `run_xpu_disagg_accuracy_test.sh` / `run_tpu_disagg_accuracy_test.sh` / `run_multi_connector_accuracy_test.sh` / `spec_decode_acceptance_test.sh` / `config_sweep_accuracy_test.sh` / `toy_proxy_server.py` 等 — **NIXL 有最全的 multi-platform integration test (xpu/tpu/multi-connector/spec-decode/config sweep)**，其它 backend 无对等覆盖 |

`extract_hidden_states_integration/` 1 个 test —— `ExampleHiddenStatesConnector` 的集成测试。

`d:\design\vllm\tests\v1\kv_offload\` 5 个 .py：`test_worker.py` / `test_cpu_offloading.py` / `test_shared_offload_region.py` / `test_cpu_gpu.py` / `test_cpu_manager.py` —— 验证 `v1/kv_offload/` 块管理层独立可测。

> synthesis: **测试覆盖密度**：每个 backend 都有专用 unit test + 跨 backend 的 lifecycle/failure/correctness 测试 + NIXL 独占 9 套 integration shell（多平台 + multi-connector + spec decode + config sweep）—— 表明 NIXL 是 vLLM 团队**主推的 PD backend**。

### 7.5 doc / examples 反查

`d:\design\vllm\docs\` `kv_connector` 命中 **8 文件**：

| 文档 | 主题 |
|---|---|
| [docs/features/disagg_prefill.md](d:\design\vllm\docs\features\disagg_prefill.md) | PD 分离总览 |
| [docs/features/nixl_connector_usage.md](d:\design\vllm\docs\features\nixl_connector_usage.md) | NIXL 使用 |
| [docs/features/nixl_connector_compatibility.md](d:\design\vllm\docs\features\nixl_connector_compatibility.md) | NIXL 平台兼容矩阵 |
| [docs/features/mooncake_connector_usage.md](d:\design\vllm\docs\features\mooncake_connector_usage.md) | Mooncake 使用 |
| [docs/features/disagg_encoder.md](d:\design\vllm\docs\features\disagg_encoder.md) | 编码器分离 |
| [docs/design/p2p_nccl_connector.md](d:\design\vllm\docs\design\p2p_nccl_connector.md) | P2P NCCL 设计 |
| [docs/serving/expert_parallel_deployment.md](d:\design\vllm\docs\serving\expert_parallel_deployment.md) | EP 部署（提到 kv_connector） |
| [docs/mkdocs/hooks/generate_metrics.py](d:\design\vllm\docs\mkdocs\hooks\generate_metrics.py) | metrics 生成钩子 |

`d:\design\vllm\examples\online_serving\disaggregated_serving/` 6 个文件：

| 文件 | 用途 |
|---|---|
| [disagg_proxy_demo.py](d:\design\vllm\examples\online_serving\disaggregated_serving\disagg_proxy_demo.py) | XpYd 代理（"X prefill, Y decode"） |
| [kv_events.sh](d:\design\vllm\examples\online_serving\disaggregated_serving\kv_events.sh) | KV cache event 演示 |
| [mooncake_connector/run_mooncake_connector.sh](d:\design\vllm\examples\online_serving\disaggregated_serving\mooncake_connector\run_mooncake_connector.sh) | Mooncake 端到端启动 |
| [mooncake_connector/mooncake_connector_proxy.py](d:\design\vllm\examples\online_serving\disaggregated_serving\mooncake_connector\mooncake_connector_proxy.py) | Mooncake 代理 |
| [moriio_toy_proxy_server.py](d:\design\vllm\examples\online_serving\disaggregated_serving\moriio_toy_proxy_server.py) | MoRIIO 代理 |
| [README.md](d:\design\vllm\examples\online_serving\disaggregated_serving\README.md) | 入口说明 |

`d:\design\vllm\examples\others\lmcache\` 9 个文件 —— LMCache 独立 examples 树（含 `disagg_prefill_lmcache_v1/`、`cpu_offload_lmcache.py`、`kv_cache_sharing_lmcache_v1.py`、`disagg_proxy_server.py` 等）。

> synthesis: **三层反查交叉验证**："14 backend 都有 connector 实现" + "其中 5 个有专用 docs/features 文档（NIXL × 2 + Mooncake + P2P + Disagg overview）+ 4 个有 examples 代理脚本（NIXL/Mooncake/MoRIIO/LMCache）" + "全部 14 个有 unit test，NIXL 还有 9 套 integration"。Backend 成熟度梯队明显：**NIXL > Mooncake ≈ LMCache > MoRIIO ≈ P2P > Offloading > FlexKV/HF3FS/Simple > Example/DecodeBench**。

---

## 8. 与跨项目对比的差异（synthesis）

| 维度 | vLLM | MindIE | SGLang |
|---|---|---|---|
| 抽象方式 | **`KVConnectorBase_V1` 单 ABC + role enum + 14 backend lazy import** | LLMDataDist 一等公民 + `mempool/mooncake_mempool.py` 旁路 | `BaseKVManager` + `BaseKVSender` + `BaseKVReceiver` + `BaseKVBootstrapServer` 4 ABC |
| Backend 数 | **14（v1 注册表）** | 2（LLMDataDist 主 + Mooncake mempool 旁路） | 5（mooncake/nixl/mori/ascend/fake） |
| 进程拓扑 | scheduler / worker 单进程内拆 SCHEDULER/WORKER 两个 connector 实例 | C++ Executor + Python `Generator` + 独立 `connector/` 进程 | scheduler 进程内独立 disagg event loop |
| 失败粒度 | **块级 `get_block_ids_with_load_errors`** | req 级 LinkResult 4 态 | req 级 KVPoll.Failed |
| 多 storage layer 复用 | **`MultiConnector` + `_extra_async_saves` 多 connector 同时 async save 同 req** | 单一栈，不复用 | HiCache 多层（device→host→storage）但与 disagg backend 解耦 |
| 块管理层是否独立 | **是**（`v1/kv_offload/` 与 connector 分层） | 否（C++ block_manager 与 LLMDataDist 紧耦合） | 否（KVCache pool 在 mem_cache，与 disagg conn 解耦） |
| 服务前端 | **`entrypoints/serve/disagg/`**（tokens-only + abort 端点） | 独立 `connector/main.py` 进程 | 复用普通 OpenAI server，靠请求 `bootstrap_room` 路由 |

详见 [comparison/topics/pd-disaggregation.md](../../comparison/topics/pd-disaggregation.md)（已含 14 维度对比表）+ [comparison/dimensions.md §dim-kv-transfer](../../comparison/dimensions.md)（本页是 vLLM 列的深度链接目标）。

---

## Increment 2026-08-18 (vLLM 5f7fab88 → d29dc3ab)

> 摸底命令：`git -C <vllm> diff 5f7fab88..HEAD --stat -- vllm/distributed/kv_transfer`（54 文件，+17333/-6313）。本节行号已在 d29dc3ab 实地核对。

### Inc.1 base.py / mixin / output 骨架漂移（语义未变，位置变）

- `KVConnectorBase_V1` 现 [base.py:171](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py)（+87 行 diff）；关键方法新行号见 §1.1 顶部 banner。
- `KVConnectorOutput` 从 outputs.py:127-153 移至 [v1/outputs.py:264](d:\design\vllm\vllm\v1\outputs.py)。
- `KVConnectorModelRunnerMixin`：`kv_connector_no_forward` 现 L36、新增 **`maybe_get_kv_connector_output`**（L51）、`_get_kv_connector_output` 现 L78（[kv_connector_model_runner_mixin.py](d:\design\vllm\vllm\v1\worker\kv_connector_model_runner_mixin.py)）。
- `KVTransferConfig` 仍在 [config/kv_transfer.py:23](d:\design\vllm\vllm\config\kv_transfer.py)，`kv_load_failure_policy: Literal["recompute","fail"] = "fail"` 现 L69。
- 新文件 [ssm_conv_transfer_utils.py](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\ssm_conv_transfer_utils.py)（±210 行）：Mamba/SSM conv state 的传输辅助（未细读）。

### Inc.2 注册表：14 → 16 个 connector

d29dc3ab 的 `KVConnectorFactory` 注册名单（[factory.py:152-260](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\factory.py)，按注册顺序）：`ExampleConnector` / `ExampleHiddenStatesConnector` / `LMCacheConnectorV1` / `LMCacheMPConnector` / `NixlConnector` / **`NixlPullConnector`（新）** / **`NixlPushConnector`（新）** / `MultiConnector` / `MoRIIOConnector` / `OffloadingConnector` / `DecodeBenchConnector` / `MooncakeConnector` / **`MooncakeStoreConnector`（新）** / `FlexKVConnectorV1` / `SimpleCPUOffloadConnector` / `HF3FSKVConnector`。**删除**：`P2pNcclConnector`（p2p/ 目录 3 文件 -1436 行整体移除）。

### Inc.3 NIXL：单一 D-pull → base / pull / push 三层

- 类层次重排：`NixlBaseConnector(KVConnectorBase_V1, SupportsHMA)`（[nixl/connector.py:79](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\connector.py)）→ `NixlPullConnector`（L326，"Pull-based (READ)"）+ `NixlPushConnector`（L354，"Push-based (WRITE)"）；`NixlConnector` 是 `NixlPullConnector` 的**向后兼容别名**（[connector.py:12, 390](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\connector.py)）。
- 新文件：`base_scheduler.py`（`NixlBaseConnectorScheduler`，510 行）/ `base_worker.py`（`NixlBaseConnectorWorker`，**2619 行**，原 worker.py 主体迁入）/ `pull_scheduler.py` + `pull_worker.py` / `push_scheduler.py` + `push_worker.py`（792 行）/ `tp_mapping.py`（异构 TP 映射，142 行）/ [`nixl/__init__.py`](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\__init__.py)（统一导出 16 个符号）。
- **push 模式语义**（[push_worker.py:1-32](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\push_worker.py) module docstring 直接观察）：专属 `nixl-push-writer` 线程独占所有 push NIXL ops——D 端发 `PUSH_REG` 注册，P 端把 finished blocks 与 D 注册配对后发 WRITE（`make_prepped_xfer`/`transfer`）；engine 主线程经 `_reg_send_inbox` / `_finished_blocks_inbox` / `_pending_completion_notifs` 三队列喂 writer，event-driven 唤醒 + 仅在有未配对块时自轮询。synthesis: 这补上了旧页 §8 对比表里"vLLM 只有 D-pull"的空缺——vLLM 现在 pull（D 读）/ push（P 写）双范式，与 SGLang mooncake（P write）/ MindIE LLMDataDist（pull）的对照关系需要在 comparison 页更新（本组不动 comparison 页，留给主 agent）。
- 另注意 [connector.py:128](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\connector.py)：`kv_role='kv_both'` 用于 NixlConnector 已标 deprecated。

### Inc.4 Mooncake：新增 store/ 子树（`MooncakeStoreConnector`）

[mooncake/store/](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\mooncake\store) 7 文件（connector.py 372 / coordinator.py 412 / data.py 472 / metrics.py 189 / protocol.py 40 / scheduler.py 500 / worker.py **2113** 行）——把 Mooncake 分布式 KV store 作为 **storage backend**（类似 LMCache 定位），与既有 P/D 直传的 `MooncakeConnector` 并列注册。mooncake/ 一级另新增 `rdma_utils.py`（46 行）与 `stats.py`（146 行）。未细读，见 VERIFY。

### Inc.5 Offloading 子系统膨胀

`offloading/scheduler.py` +1705 行；新增 `config.py`（222 行，`SchedulerOffloadConfig` 独立成文件）/ `events.py`（413 行）/ `canonical_mapping.py`（456 行）/ `metrics.py` +517。`offloading_connector.py` ±80。旧页 §3.5 的行号与字段清单需整体重核（见 VERIFY）。

### Inc.6 HTTP 前端迁移

`entrypoints/serve/disagg/` → [entrypoints/scale_out/token_in_token_out/](d:\design\vllm\vllm\entrypoints\scale_out\token_in_token_out)（详见 §5 banner）。`entrypoints/serve/` 现存子目录为 dev / elastic_ep / engine / exception_handling / fault_tolerance / instrumentator / lora / middleware / profile / sagemaker / tokenize / utils（实地 ls）。

## Notes / Caveats

> [!todo] VERIFY (increment 2026-08-18): NIXL base/pull/push 拆分后，旧页 §3.2 的 scheduler 端字段（`_reqs_need_recv` 等）现落在 `base_scheduler.py` 还是 `pull_scheduler.py` 未核对；`tp_mapping.py`（异构 TP）的语义未读；Mooncake store/ 子树 4100 行、offloading 增量 ~3300 行均未细读；`example_hidden_states_connector.py`（±541）与 `kv_connector/utils.py`（±794）大改未分析。

> [!todo] VERIFY: NIXL `_nixl_handshake_listener` 后台线程在 `set_xfer_handshake_metadata` 仅被 scheduler 端调用，但对应的 D 端获取 metadata 的代码路径需追到 `worker.py` 的 `_background_nixl_handshake`（[worker.py:1872-1875](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\nixl\worker.py)）—— D 端 worker 是否通过 ZMQ REQ 主动 pull 还是 P 端 push？源码注释 "Initiate handshake with remote engine to exchange metadata" 暗示 pull。

> [!todo] VERIFY: `OffloadingSpecFactory` 默认只注册 `CPUOffloadingSpec`，其它 spec 必须通过 `kv_connector_extra_config["spec_module_path"]` 动态加载（[factory.py:43-49](d:\design\vllm\vllm\v1\kv_offload\factory.py)）。**是否存在第三方 spec（如 NVMe / GDS / Mooncake store）实例**待与社区/上游 PR 校对。

> [!todo] VERIFY: `LMCacheConnectorV1` 的 `use_native=True` 路径用 vendored `lmcache_integration.vllm_v1_adapter`，`use_native=False` 走 upstream `lmcache.integration.vllm.vllm_v1_adapter`（[lmcache_connector.py:96-111](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\lmcache_connector.py)）。两套实现的语义差异（vendored 是 snapshot，upstream 是 latest）—— **生产环境应该选哪个？**未在文档明示。

> [!warning] CONTRADICTION (resolved C2 from pd-disaggregation): vLLM `MultiConnector` 名称易误解。**实际是"多 storage layer 同时存 KV"的 wrapper，不是 P+D 角色合并**（[multi_connector.py:130-133](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\multi_connector.py) 注释）。同节点 P+D 由 `kv_role=kv_both` + 前端代理实现，与 `MultiConnector` 无关。

> [!warning] CONTRADICTION: `KVTransferConfig.kv_rank/kv_parallel_size` 注释说 "Currently only 1P1D is supported"（[kv_transfer.py:46-48](d:\design\vllm\vllm\config\kv_transfer.py)），但 examples 中 `disagg_proxy_demo.py` 演示 XpYd（X prefill, Y decode）。**XpYd 是 vLLM 进程外的"代理 + 多个独立 1P1D 进程对"实现**，不是 connector 层的多对多。

> [!warning] CONTRADICTION (命名陷阱): `request.kv_transfer_params["do_remote_prefill"]=True` 出现在 **D 端 req** 上意为"我（D）需要去远端 P 拉 prefill"——字面意义反向。该命名在 NIXL/Mooncake/MoRIIO 三个 backend 都使用（共享同一字段名）。详见 §6.1。

---

## See also

- [vllm/entities/Scheduler.md](../entities/Scheduler.md)（含 `self.connector` / `WAITING_FOR_REMOTE_KVS` 路径）
- [vllm/entities/KVCacheManager.md](../entities/KVCacheManager.md)（PD 失败时 `cache_blocks` 兜底）
- [vllm/topics/request-lifecycle.md](request-lifecycle.md)
- [comparison/topics/pd-disaggregation.md](../../comparison/topics/pd-disaggregation.md)（§3 KV 传输栈 vLLM 列与本页互链）
- [comparison/dimensions.md §dim-kv-transfer](../../comparison/dimensions.md)
- [sglang/modules/mem_cache.md](../../sglang/modules/mem_cache.md)（HiCache 7 backend 与本页 14 backend 对照）
