---
type: topic
project: sglang
status: draft
confidence: high
verified_against: 2026-04-19
sources:
  - d:\design\sglang\python\sglang\srt\disaggregation\utils.py:L304-L309
  - d:\design\sglang\python\sglang\srt\disaggregation\utils.py:L342-L431
  - d:\design\sglang\python\sglang\srt\disaggregation\base\conn.py
  - d:\design\sglang\python\sglang\srt\disaggregation\common\conn.py
  - d:\design\sglang\python\sglang\srt\disaggregation\mooncake\conn.py
  - d:\design\sglang\python\sglang\srt\disaggregation\nixl\conn.py
  - d:\design\sglang\python\sglang\srt\disaggregation\mori\conn.py
  - d:\design\sglang\python\sglang\srt\disaggregation\ascend\conn.py
  - d:\design\sglang\python\sglang\srt\disaggregation\ascend\transfer_engine.py
  - d:\design\sglang\python\sglang\srt\disaggregation\fake\conn.py
  - d:\design\sglang\python\sglang\srt\disaggregation\decode.py:L1171-L1370
  - d:\design\sglang\python\sglang\srt\disaggregation\prefill.py:L355-L800
  - d:\design\sglang\python\sglang\srt\disaggregation\encode_server.py
  - d:\design\sglang\python\sglang\srt\disaggregation\encode_receiver.py
  - d:\design\sglang\python\sglang\srt\disaggregation\encode_grpc_server.py
  - d:\design\sglang\python\sglang\srt\managers\scheduler.py:L317-L329
  - d:\design\sglang\python\sglang\srt\managers\scheduler.py:L1051-L1182
  - d:\design\sglang\python\sglang\srt\managers\scheduler.py:L3628-L3654
  - d:\design\sglang\python\sglang\srt\managers\disagg_service.py
  - d:\design\sglang\python\sglang\srt\managers\scheduler_pp_mixin.py:L148, L324
  - d:\design\sglang\python\sglang\srt\server_args.py:L160, L212, L706-L718, L3537-L3603, L6138-L6215, L6510-L6523
related:
  - sglang/modules/disaggregation.md
  - sglang/modules/multimodal.md
  - sglang/modules/multiplex.md
  - sglang/modules/managers.md
  - sglang/entities/Scheduler.md
  - comparison/topics/pd-disaggregation.md
  - comparison/dimensions.md
---

# PD 分离架构（SGLang 内部）

## Summary

SGLang 的 PD 分离子系统由 **5 个 KV 传输 backend**（[`TransferBackend` 枚举 5 项](d:\design\sglang\python\sglang\srt\disaggregation\utils.py)：`MOONCAKE` / `MORI` / `NIXL` / `ASCEND` / `FAKE`）、**2 个 `Scheduler` mixin**（[`SchedulerDisaggregationDecodeMixin`](d:\design\sglang\python\sglang\srt\disaggregation\decode.py) + [`SchedulerDisaggregationPrefillMixin`](d:\design\sglang\python\sglang\srt\disaggregation\prefill.py)，构成 [`Scheduler` 11 父类](d:\design\sglang\python\sglang\srt\managers\scheduler.py:317-329) 中的 2 席）、**EPD 多模态三段架构**（[`encode_server.py`](d:\design\sglang\python\sglang\srt\disaggregation\encode_server.py) / [`encode_receiver.py`](d:\design\sglang\python\sglang\srt\disaggregation\encode_receiver.py) / [`encode_grpc_server.py`](d:\design\sglang\python\sglang\srt\disaggregation\encode_grpc_server.py)）和 **与 PD-Multiplex 显式互斥的 [server_args 校验](d:\design\sglang\python\sglang\srt\server_args.py:6510-6523)** 4 个核心维度组成。本页是 SGLang **内部**深度参考，跨项目三方对照见 [`comparison/topics/pd-disaggregation.md`](../../comparison/topics/pd-disaggregation.md)（14 子维度），不在此重复。

## Sources

核心锚点（详尽列表见 frontmatter `sources:`）：

- **枚举 + 工厂**：[`utils.py:304-309`](d:\design\sglang\python\sglang\srt\disaggregation\utils.py)（`TransferBackend` 5 项）+ [L312-317](d:\design\sglang\python\sglang\srt\disaggregation\utils.py)（`KVClassType` 5 项）+ [L342-431](d:\design\sglang\python\sglang\srt\disaggregation\utils.py)（`get_kv_class` 5 路 if/elif，FAKE 缺 `BOOTSTRAP_SERVER`）
- **抽象基类**：[`base/conn.py`](d:\design\sglang\python\sglang\srt\disaggregation\base\conn.py)（`KVArgs` L15 / `KVPoll` L42 / `BaseKVManager` L50 / `BaseKVSender` L68 / `BaseKVReceiver` L113 / `BaseKVBootstrapServer` L172）+ [`common/conn.py`](d:\design\sglang\python\sglang\srt\disaggregation\common\conn.py)（`CommonKVManager` L88 / `CommonKVBootstrapServer` L692）
- **5 backend `conn.py`**：[mooncake](d:\design\sglang\python\sglang\srt\disaggregation\mooncake\conn.py) / [nixl](d:\design\sglang\python\sglang\srt\disaggregation\nixl\conn.py) / [mori](d:\design\sglang\python\sglang\srt\disaggregation\mori\conn.py) / [ascend](d:\design\sglang\python\sglang\srt\disaggregation\ascend\conn.py) / [fake](d:\design\sglang\python\sglang\srt\disaggregation\fake\conn.py)
- **2 mixin**：[`decode.py:1171-1370`](d:\design\sglang\python\sglang\srt\disaggregation\decode.py)（6 方法）+ [`prefill.py:355-800`](d:\design\sglang\python\sglang\srt\disaggregation\prefill.py)（9 方法）
- **EPD 三件套**：[`encode_server.py`](d:\design\sglang\python\sglang\srt\disaggregation\encode_server.py) + [`encode_receiver.py`](d:\design\sglang\python\sglang\srt\disaggregation\encode_receiver.py) + [`encode_grpc_server.py`](d:\design\sglang\python\sglang\srt\disaggregation\encode_grpc_server.py)
- **Scheduler 接入**：[`scheduler.py:317-329`](d:\design\sglang\python\sglang\srt\managers\scheduler.py)（11 mixin MRO）+ [`L1051-1182`](d:\design\sglang\python\sglang\srt\managers\scheduler.py)（`init_disaggregation`：DECODE L1073-1125 / PREFILL L1127-1167 / EPD L1170-1181）+ [`L3628-3654`](d:\design\sglang\python\sglang\srt\managers\scheduler.py)（`dispatch_event_loop` 9 路）+ [`disagg_service.py:14-44`](d:\design\sglang\python\sglang\srt\managers\disagg_service.py)
- **CLI / 校验**：[`server_args.py:706-718`](d:\design\sglang\python\sglang\srt\server_args.py)（字段定义）+ [`L6138-6215`](d:\design\sglang\python\sglang\srt\server_args.py)（argparse）+ [`L160` choices](d:\design\sglang\python\sglang\srt\server_args.py) + [`L6510-6523`](d:\design\sglang\python\sglang\srt\server_args.py)（PD-Mux 互斥）

## Architecture / Data flow

```mermaid
flowchart TB
    SA["ServerArgs<br/>--disaggregation-mode {null,prefill,decode}<br/>--disaggregation-transfer-backend {mooncake,nixl,mori,ascend,fake}<br/>--language-only / --encoder-* (EPD)"]
    SA --> SCH["Scheduler.__init__ → init_disaggregation()<br/>scheduler.py:1051-1182"]

    SCH --> BR{"disaggregation_mode"}
    BR -->|PREFILL| PREP["L1127-1167<br/>MetadataBuffers + PrefillBootstrapQueue<br/>+ disagg_prefill_inflight_queue"]
    BR -->|DECODE| DECP["L1073-1125<br/>MetadataBuffers + DecodeTransferQueue<br/>+ DecodePreallocQueue"]
    BR -->|NULL+language_only| EPDP["L1170-1181<br/>create_mm_receiver()"]

    PREP --> GK["get_kv_class(backend, KVClassType)<br/>utils.py:342-431"]
    DECP --> GK
    GK --> B1["MooncakeKV*"]
    GK --> B2["NixlKV*"]
    GK --> B3["MoriKV*"]
    GK --> B4["AscendKV* (extends Mooncake)"]
    GK --> B5["FakeKV* (no BootstrapServer)"]

    B1 -.extends.-> CKM["CommonKVManager / CommonKVBootstrapServer<br/>common/conn.py L88, L692"]
    B2 -.-> CKM
    B3 -.-> CKM
    B5 -.extends.-> BASE["Base* (base/conn.py)"]

    SA -->|PREFILL only| DSV["start_disagg_service<br/>disagg_service.py:14-44"]

    SCH --> DEL["dispatch_event_loop<br/>scheduler.py:3628-3654<br/>(9 路：mode × pp × overlap)"]

    EPDP --> EPDR["MMReceiverGrpc / MMReceiverHTTP<br/>encode_receiver.py:1207, 1048"]
    EPDR -.HTTP/gRPC.-> EPDS["MMEncoder (encode_server.py:172)"]
    EPDS -.import.-> MM["multimodal.processors.qwen_vl<br/>.preprocess_video"]
```

**关键点**

- **触发位置**：[`Scheduler.__init__ L452`](d:\design\sglang\python\sglang\srt\managers\scheduler.py) 调 `self.init_disaggregation()`，进入 [L1051-1182](d:\design\sglang\python\sglang\srt\managers\scheduler.py)，按 `DisaggregationMode(...)` + `TransferBackend(...)` 两枚举分发。
- **请求级耦合**：每个 `Req` 上挂 [`Req.disagg_kv_sender: Optional[BaseKVSender]`](d:\design\sglang\python\sglang\srt\managers\schedule_batch.py:855)（type annotation [L53](d:\design\sglang\python\sglang\srt\managers\schedule_batch.py)），是「调度层 ↔ PD 包」的唯一显式类型接口。
- **Bootstrap 仅 PREFILL 拉起**：[`start_disagg_service` L21](d:\design\sglang\python\sglang\srt\managers\disagg_service.py) 仅在 `DisaggregationMode.PREFILL` 时实例化 `kv_bootstrap_server_class`；DECODE 端仅做 receiver 注册；ASCEND 额外加 `memfabric_hybrid.create_config_store`（仅 `node_rank==0`，[L30-42](d:\design\sglang\python\sglang\srt\managers\disagg_service.py)）。
- **Event-loop 9 路分发**：[`dispatch_event_loop` L3628-3654](d:\design\sglang\python\sglang\srt\managers\scheduler.py) 按 `disaggregation_mode` × `pp_size > 1` × `enable_overlap` 三维分派；PP 路径走 [`scheduler_pp_mixin.py:148, 324`](d:\design\sglang\python\sglang\srt\managers\scheduler_pp_mixin.py)（PP 版本**不在** `disaggregation/` 包内）。

## 5 个 Backend 矩阵

枚举源：[`TransferBackend` L304-309`](d:\design\sglang\python\sglang\srt\disaggregation\utils.py)；CLI 白名单：[`DISAGG_TRANSFER_BACKEND_CHOICES = ["mooncake","nixl","ascend","fake","mori"]` L160](d:\design\sglang\python\sglang\srt\server_args.py)；运行时分发：[`get_kv_class` L342-431](d:\design\sglang\python\sglang\srt\disaggregation\utils.py)。

| Backend | 实现 / 关键类 | 协议 · 平台 · 状态 |
|---|---|---|
| **mooncake** | [`mooncake/conn.py`](d:\design\sglang\python\sglang\srt\disaggregation\mooncake\conn.py)：`MooncakeKVManager(CommonKVManager)` L187 / `…Sender` L1634 / `…Receiver` L1740 / `…BootstrapServer(CommonKVBootstrapServer)` L1927 | `mooncake-transfer-engine` + RDMA/IB（`--disaggregation-ib-device` 注入；自动探测 [L3580-3587](d:\design\sglang\python\sglang\srt\server_args.py)）+ `batch_transfer_sync` + ZMQ 多部分消息。**NVIDIA + IB；默认 backend ✅**（[L707](d:\design\sglang\python\sglang\srt\server_args.py)） |
| **nixl** | [`nixl/conn.py`](d:\design\sglang\python\sglang\srt\disaggregation\nixl\conn.py)：`NixlKVManager(CommonKVManager)` L170 / Sender / Receiver / BootstrapServer | NVIDIA `nixl_agent` + 插件；VRAM/DRAM 注册 + `initialize_xfer` / `transfer`。**NVIDIA / NIXL 覆盖硬件 ✅** |
| **mori** | [`mori/conn.py`](d:\design\sglang\python\sglang\srt\disaggregation\mori\conn.py)：`MoriKVManager(CommonKVManager)` L176 / Sender / Receiver / BootstrapServer | AMD `mori-io` python binding；`TransferInfo` 管线（与 mooncake 类似）。**AMD ROCm ✅** |
| **ascend** | [`ascend/conn.py`](d:\design\sglang\python\sglang\srt\disaggregation\ascend\conn.py) + [`transfer_engine.py`](d:\design\sglang\python\sglang\srt\disaggregation\ascend\transfer_engine.py)：`AscendKVManager(MooncakeKVManager)` [L21](d:\design\sglang\python\sglang\srt\disaggregation\ascend\conn.py)，Sender/Receiver/Bootstrap **全子类化 Mooncake** | `AscendTransferEngine` + `memfabric_hybrid.create_config_store`（仅 `node_rank==0` 拉起，[`disagg_service.py:30-42`](d:\design\sglang\python\sglang\srt\managers\disagg_service.py)）。**Ascend NPU ✅** |
| **fake** | [`fake/conn.py`](d:\design\sglang\python\sglang\srt\disaggregation\fake\conn.py)：`FakeKVManager(BaseKVManager)` L21 / `FakeKVSender(BaseKVSender)` L35 / `FakeKVReceiver`；**无 `FakeKVBootstrapServer`** | warmup 占位；`poll()` 立即 `WaitingForInput → Success`。**测试 ⚠️** [`L3543`](d:\design\sglang\python\sglang\srt\server_args.py) 断言 prefill 禁止 fake |

> synthesis: **5 backend 非平行抽象**——形成 **3 层继承树**：`Base*`（接口，[base/conn.py](d:\design\sglang\python\sglang\srt\disaggregation\base\conn.py)）→ `Common*`（ZMQ + bootstrap 拓扑，[common/conn.py:88, 692](d:\design\sglang\python\sglang\srt\disaggregation\common\conn.py)）→ 具体 backend。`mooncake` / `nixl` / `mori` 走 `Common*`；**`ascend` 是唯一**「backend 复用 backend」case：直接继承 `MooncakeKVManager`，替换底层 `TransferEngine` 为 [`AscendTransferEngine`](d:\design\sglang\python\sglang\srt\disaggregation\ascend\transfer_engine.py)（[L22-29](d:\design\sglang\python\sglang\srt\disaggregation\ascend\conn.py)），复用 `batch_register` API；`fake` 抄近路直接落 `Base*` 跳过 `Common*`。

> synthesis: **`get_kv_class` 中 FAKE 缺 `BOOTSTRAP_SERVER` 映射**（[utils.py:415-428](d:\design\sglang\python\sglang\srt\disaggregation\utils.py)，`class_mapping` 仅 4 键），返回 `None`。这与 [`server_args.py:3541-3544`](d:\design\sglang\python\sglang\srt\server_args.py) 的「prefill 禁止 fake」断言形成两层防护：第一层启动期 `AssertionError`，第二层若被绕过则 [`disagg_service.py:23-29`](d:\design\sglang\python\sglang\srt\managers\disagg_service.py) 触发 `TypeError: 'NoneType' object is not callable`。

## 2 个 Scheduler Mixin

[`Scheduler` 多重继承](d:\design\sglang\python\sglang\srt\managers\scheduler.py) 共 **11 个 mixin**（[L317-329](d:\design\sglang\python\sglang\srt\managers\scheduler.py)），其中 PD 占 2 席：`SchedulerDisaggregationDecodeMixin`、`SchedulerDisaggregationPrefillMixin`。Mixin 用 `self: Scheduler` type-hint，但本身不是 `Scheduler` 子类——是经典 mixin 模式。

### 2.1 `SchedulerDisaggregationPrefillMixin`（[`prefill.py:355`](d:\design\sglang\python\sglang\srt\disaggregation\prefill.py)）

类内 **9 方法**（按出现顺序），均以 `self: Scheduler` type-hint：

| 方法 | 行 | 角色 |
|---|---|---|
| `maybe_prefetch_staging_for_batch` | [L360](d:\design\sglang\python\sglang\srt\disaggregation\prefill.py) | mooncake `SGLANG_DISAGG_STAGING_BUFFER` 提前 prefetch |
| `get_next_disagg_prefill_batch_to_run` | [L371](d:\design\sglang\python\sglang\srt\disaggregation\prefill.py) | 调度下一 prefill batch |
| `event_loop_normal_disagg_prefill` | [L389](d:\design\sglang\python\sglang\srt\disaggregation\prefill.py) | 主循环（非 overlap） |
| `event_loop_overlap_disagg_prefill` | [L423](d:\design\sglang\python\sglang\srt\disaggregation\prefill.py) | 主循环（overlap schedule） |
| `process_batch_result_disagg_prefill` | [L468](d:\design\sglang\python\sglang\srt\disaggregation\prefill.py) | 替代 `process_batch_result`；含 `inflight_queue.append` |
| `process_disagg_prefill_inflight_queue` | [L589](d:\design\sglang\python\sglang\srt\disaggregation\prefill.py) | 每 tick 轮询 sender `KVPoll`，完成则 finalize |
| `get_transferred_rids` | [L704](d:\design\sglang\python\sglang\srt\disaggregation\prefill.py) | 返回已完成传输的 rid（给 tokenizer manager） |
| `process_prefill_chunk` | [L722](d:\design\sglang\python\sglang\srt\disaggregation\prefill.py) | chunked prefill 分块发送 |
| `send_kv_chunk` | [L750](d:\design\sglang\python\sglang\srt\disaggregation\prefill.py) | 单 chunk KV 发送（调 `Req.disagg_kv_sender.send`） |

**Scheduler 调用钩子**：主循环 `event_loop_normal/overlap_disagg_prefill` 由 [`dispatch_event_loop` L3641-3647](d:\design\sglang\python\sglang\srt\managers\scheduler.py) 在 PREFILL 模式下调用；`process_disagg_prefill_inflight_queue` 每 tick 扫 [`disagg_prefill_inflight_queue: List[Req]`](d:\design\sglang\python\sglang\srt\managers\scheduler.py:1167)。

请求生命周期（[`prefill.py:1-18` docstring](d:\design\sglang\python\sglang\srt\disaggregation\prefill.py)）：**Bootstrap Queue → Waiting Queue → Inflight Queue**；`PrefillBootstrapQueue` ([L87](d:\design\sglang\python\sglang\srt\disaggregation\prefill.py)) 的 `pop_bootstrapped` ([L270](d:\design\sglang\python\sglang\srt\disaggregation\prefill.py)) 由 `BaseKVSender.poll()` 驱动；[`KVPoll` 5 状态 L42-47](d:\design\sglang\python\sglang\srt\disaggregation\base\conn.py) = `Failed=0` / `Bootstrapping=1` / `WaitingForInput=2` / `Transferring=3` / `Success=4`。

### 2.2 `SchedulerDisaggregationDecodeMixin`（[`decode.py:1171`](d:\design\sglang\python\sglang\srt\disaggregation\decode.py)）

类内 **6 方法**：

| 方法 | 行 | 角色 |
|---|---|---|
| `event_loop_normal_disagg_decode` | [L1174](d:\design\sglang\python\sglang\srt\disaggregation\decode.py) | 主循环（非 overlap） |
| `event_loop_overlap_disagg_decode` | [L1201](d:\design\sglang\python\sglang\srt\disaggregation\decode.py) | 主循环（overlap：`result_queue` + 上批 result 滞后处理） |
| `_run_batch_prebuilt` | [L1238](d:\design\sglang\python\sglang\srt\disaggregation\decode.py) | 跑「假完成的 prefill batch」 |
| `get_next_disagg_decode_batch_to_run` | [L1249](d:\design\sglang\python\sglang\srt\disaggregation\decode.py) | 处理 prebuilt batch + 调度下一 decode batch |
| `get_new_prebuilt_batch` | [L1281](d:\design\sglang\python\sglang\srt\disaggregation\decode.py) | 把已 transfer 完的 req 组成 `PrebuiltExtendBatch`（跳 forward 仅填 metadata） |
| `process_decode_queue` | [L1333](d:\design\sglang\python\sglang\srt\disaggregation\decode.py) | 推进 `prealloc_queue → transfer_queue → waiting_queue` 三段 |

**Scheduler 调用钩子**：[`dispatch_event_loop` L3648-3654](d:\design\sglang\python\sglang\srt\managers\scheduler.py) 按 `pp_size > 1` / `enable_overlap` 三选一（`event_loop_pp_disagg_decode` 来自 [`scheduler_pp_mixin.py:324`](d:\design\sglang\python\sglang\srt\managers\scheduler_pp_mixin.py)，**不在本 mixin** 内）。

请求生命周期（[`decode.py:1-19` docstring](d:\design\sglang\python\sglang\srt\disaggregation\decode.py)）：**4 段队列** PreallocQueue → TransferQueue → WaitingQueue → RunningBatch。

> synthesis: **2 mixin 不对称**——prefill 9 方法（含 chunk send + inflight 轮询 + 替换 `process_batch_result`），decode 6 方法（仅 event_loop + prebuilt batch + 推进队列）。原因：prefill 是 KV 数据生产者 + 对端通知者，decode 是消费者 + 等待者——生产端工作更多，消费端只需 poll receiver 状态机。

> synthesis: **`enable_overlap` 在 PD 模式下语义不同**——非 PD 时 overlap = forward 与 sample/output 流水线交错；PD-decode overlap（[`decode.py:1201-1236`](d:\design\sglang\python\sglang\srt\disaggregation\decode.py)）是「当前 batch forward + 上一批 result 处理」滞后一拍。这与 PD-Multiplex 的「同 GPU prefill/decode SM 并发」属完全不同的并发层（详见 [`sglang/modules/multiplex.md`](../modules/multiplex.md)）。

## EPD（Encode-Prefill-Decode）三段架构

EPD 是 SGLang 在 PD 之外的**额外切分**：多模态模型的 ViT/VL encoder 独立成单独服务，与文本 prefill/decode 节点用 ZMQ/gRPC/HTTP 通信。**vLLM 与 MindIE 都没有**这一设计点。

### 3 个 EPD 文件

| 文件 | 关键类 / 函数 |
|---|---|
| [`encode_server.py`](d:\design\sglang\python\sglang\srt\disaggregation\encode_server.py)（编码器服务） | `MMEncoder` L172 / `EncoderProfiler` L1239 / `launch_encoder` L1342 / `launch_server` L1351 |
| [`encode_receiver.py`](d:\design\sglang\python\sglang\srt\disaggregation\encode_receiver.py)（接收器） | `MMReceiverBase` L597 / `MMReceiverHTTP` L1048 / `MMReceiverGrpc` L1207 / `create_mm_receiver` L1388 / `WaitingImageRequest` L385 |
| [`encode_grpc_server.py`](d:\design\sglang\python\sglang\srt\disaggregation\encode_grpc_server.py)（gRPC handler） | 使用 `MMEncoder` L25；按 `encoder_transfer_backend` 选 3 种回程（详下表） |

### 单向依赖：encode → multimodal

[`encode_server.py:40`](d:\design\sglang\python\sglang\srt\disaggregation\encode_server.py) `from sglang.srt.multimodal.processors.qwen_vl import preprocess_video`，并在 [L515](d:\design\sglang\python\sglang\srt\disaggregation\encode_server.py) `await preprocess_video(...)`。**单向**：`disaggregation/encode_*.py` 是 `multimodal/` 的消费者；详见 [`sglang/modules/multimodal.md`](../modules/multimodal.md)。

### 3 种 encoder 回程后端

[`ENCODER_TRANSFER_BACKEND_CHOICES = ["zmq_to_scheduler", "zmq_to_tokenizer", "mooncake"]`](d:\design\sglang\python\sglang\srt\server_args.py:212)，默认 `zmq_to_scheduler`（[`server_args.py:718`](d:\design\sglang\python\sglang\srt\server_args.py)）。

| 模式 | 回程数据流 | 锚点 |
|---|---|---|
| `zmq_to_scheduler` | encoder → ZMQ → scheduler | [`encode_server.py:1422, 1446`](d:\design\sglang\python\sglang\srt\disaggregation\encode_server.py) + [`encode_receiver.py:1309`](d:\design\sglang\python\sglang\srt\disaggregation\encode_receiver.py) + [`scheduler.py:1170-1181`](d:\design\sglang\python\sglang\srt\managers\scheduler.py) |
| `zmq_to_tokenizer` | encoder → ZMQ → tokenizer manager | [`encode_server.py:1466`](d:\design\sglang\python\sglang\srt\disaggregation\encode_server.py) + [`encode_receiver.py:754`](d:\design\sglang\python\sglang\srt\disaggregation\encode_receiver.py) |
| `mooncake` | encoder → Mooncake RDMA → scheduler | [`encode_server.py:273, 1048, 1436`](d:\design\sglang\python\sglang\srt\disaggregation\encode_server.py) + [`encode_receiver.py:612, 777`](d:\design\sglang\python\sglang\srt\disaggregation\encode_receiver.py) |

### EPD 触发条件（区别于纯 PD）

[`Scheduler.init_disaggregation` L1170-1181`](d:\design\sglang\python\sglang\srt\managers\scheduler.py)：

```python
if (
    self.server_args.language_only
    and self.server_args.encoder_transfer_backend == "zmq_to_scheduler"
):
    self.mm_receiver = create_mm_receiver(...)
```

即 **EPD 触发 ≠ `disaggregation_mode != "null"`**——而是 `--language-only`（只装文本部分）+ `--encoder-urls <encoder server urls>`（[`server_args.py:6203-6209`](d:\design\sglang\python\sglang\srt\server_args.py)）。EPD 与纯 PD（`disaggregation_mode in {prefill, decode}`）**正交且可叠加**：3 段架构是 encoder + (prefill+decode) 或 encoder + prefill + decode。

> synthesis: **EPD backend 池 ≠ PD 的 5 backend**。PD = `[mooncake,nixl,ascend,fake,mori]`（5 项）；EPD = `[zmq_to_scheduler, zmq_to_tokenizer, mooncake]`（3 项）。**唯一交集**是 `mooncake`，且 PD-mooncake 走 KV cache buffer 注册，EPD-mooncake 走 multimodal embedding tensor 注册——同名不同路径。EPD 还有模型族白名单限制：[`server_args.py:3592-3604`](d:\design\sglang\python\sglang\srt\server_args.py) 显式限定仅 11 个 Qwen + Kimi 系 multimodal 架构（其它 VLM 如 LLaVA 当前会报错）。

## PD-Disagg vs PD-Multiplex 互斥

SGLang 有 **2 个名字含「PD」的子系统**，且**互斥**：

| 子系统 | 模块 | 含义 | wiki 页 |
|---|---|---|---|
| **PD-Disaggregation** | `srt/disaggregation/` | **跨节点**：prefill 与 decode 跑在**不同进程/不同节点**，KV cache 通过 RDMA/IB 跨节点传输 | 本页 + [`sglang/modules/disaggregation.md`](../modules/disaggregation.md) |
| **PD-Multiplexing (PD-Mux)** | `srt/multiplex/` | **同节点同 GPU**：prefill 与 decode 跑在**同一 GPU 上**，通过 CUDA green context + SM 分区 + 多 stream 并发 | [`sglang/modules/multiplex.md`](../modules/multiplex.md) |

显式互斥校验在 [`server_args.py:6510-6523`](d:\design\sglang\python\sglang\srt\server_args.py)：

```python
if self.enable_pdmux:
    assert self.pp_size == 1, "PD-Multiplexing is only supported with pipeline parallelism disabled (pp_size=1)."
    assert self.chunked_prefill_size == -1, "PD-Multiplexing is not compatible with chunked prefill."
    assert self.disaggregation_mode == "null", "PD-Multiplexing is not compatible with disaggregation mode."
    assert self.disable_overlap_schedule, "PD-Multiplexing is not compatible with overlap schedule."
```

**4 个互斥前提**（任一不满足直接 `AssertionError` 启动失败）：
1. `pp_size == 1`（PP 关闭）
2. `chunked_prefill_size == -1`（chunked prefill 关闭）
3. `disaggregation_mode == "null"`（PD 分离关闭，**本互斥与本页相关**）
4. `disable_overlap_schedule == True`（overlap schedule 关闭）

> synthesis: **互斥本质 = 调度循环结构不兼容**——PD-Disagg 的 `event_loop_normal_disagg_*` 把 KV 传输状态机塞进 event loop；PD-Mux 的 [`event_loop_pdmux`](d:\design\sglang\python\sglang\srt\multiplex\multiplexing_mixin.py) 把 prefill/decode SM 切换塞进 event loop。两者在 [`dispatch_event_loop`](d:\design\sglang\python\sglang\srt\managers\scheduler.py) 是**互斥分支**：`enable_pdmux` 仅在 `disaggregation_mode == NULL` 路径下被检查。`Scheduler` 11 mixin 中 PD-Disagg 占 2 席、PD-Mux 占 1 席，全部静态加进 MRO，运行期通过 `dispatch_*` 选择——SGLang 统一 mixin 模式（见 [`sglang/entities/Scheduler.md`](../entities/Scheduler.md)）。

## CLI 参数全表

字段定义集中在 [`ServerArgs` L706-718`](d:\design\sglang\python\sglang\srt\server_args.py)，argparse 在 [L6138-6215](d:\design\sglang\python\sglang\srt\server_args.py)。

### PD 主字段

| 字段 | 默认 | 说明（锚点统一指 [server_args.py](d:\design\sglang\python\sglang\srt\server_args.py)） |
|---|---|---|
| `--disaggregation-mode` | `"null"` | `null`/`prefill`/`decode`（field L706, argparse L6139） |
| `--disaggregation-transfer-backend` | `"mooncake"` | choices = `[mooncake,nixl,ascend,fake,mori]`（field L707 / argparse L6146 / choices L160；与 `TransferBackend` 枚举一一对应） |
| `--disaggregation-bootstrap-port` | `8998` | prefill bootstrap HTTP 端口（field L708 / argparse L6153） |
| `--disaggregation-ib-device` | `None` | IB 设备名（可逗号分隔多个）；mooncake 时可自动探测（field L709 / argparse L6159 / 校验 L3580-3587） |
| `--disaggregation-decode-enable-offload-kvcache` | `False` | decode KV cache 异步 offload（field L710 / argparse L6167） |
| `--num-reserved-decode-tokens` | `512` | decode 新 req 入 batch 预留 token 数（field L711 / argparse L6172） |
| `--disaggregation-decode-polling-interval` | `1` | decode 轮询 receiver 间隔（field L713 / argparse L6178） |

### EPD 字段

| 字段 | 默认 | 说明 |
|---|---|---|
| `--encoder-only` | `False` | encoder-only 服务（与 `--language-only` / `--disaggregation-mode != null` 互斥；argparse L6186 / 校验 L3568-3573） |
| `--language-only` | `False` | VLM 仅装语言模型（field L717 / argparse L6191 / 校验 L3575-3578） |
| `--encoder-transfer-backend` | `"zmq_to_scheduler"` | choices = `[zmq_to_scheduler, zmq_to_tokenizer, mooncake]`（field L718 / argparse L6196 / choices L212） |
| `--encoder-urls` | `[]` | encoder server URL 列表（argparse L6203） |
| `--enable-adaptive-dispatch-to-encoder` | `False` | 多图走 encoder，单图本地处理（argparse L6210） |

### 第三方扩展点

[`add_disagg_transfer_backend_choices(choices)` L258-259](d:\design\sglang\python\sglang\srt\server_args.py) 允许外部扩展 `DISAGG_TRANSFER_BACKEND_CHOICES`，但 **`TransferBackend` 枚举是闭集 5 项**——外部 backend 名传进来会在 [`utils.py:431`](d:\design\sglang\python\sglang\srt\disaggregation\utils.py) `raise ValueError`。扩展 choices 只让 argparse 通过，运行时仍需 patch `get_kv_class`。

### 关联校验

| 校验 | 内容与锚点 |
|---|---|
| `disaggregation_mode` 取值 | 必须 ∈ `{"null","prefill","decode"}`（[L890-892](d:\design\sglang\python\sglang\srt\server_args.py)） |
| decode 强制 chunk cache | [L3537-3539](d:\design\sglang\python\sglang\srt\server_args.py)：decode → `disable_radix_cache=True` |
| prefill 禁止 fake | [L3541-3544](d:\design\sglang\python\sglang\srt\server_args.py)：`prefill + fake` → `AssertionError` |
| `SGLANG_DISAGG_STAGING_BUFFER` 仅 mooncake | env 启用时强制 backend = mooncake（[L3552-3561](d:\design\sglang\python\sglang\srt\server_args.py)） |
| chunked prefill | `size > 0` 且非 decode 时需 `% page_size == 0`（[L6505-6508](d:\design\sglang\python\sglang\srt\server_args.py)） |
| **PD-Mux ↔ PD-Disagg 互斥** | [L6510-6523](d:\design\sglang\python\sglang\srt\server_args.py)（详见 §「PD-Disagg vs PD-Multiplex 互斥」） |
| EPD 模型族白名单 | 11 个 Qwen / Kimi 架构（[L3592-3604](d:\design\sglang\python\sglang\srt\server_args.py)） |

## Notes / Caveats

> [!warning] CONTRADICTION（命名陷阱）：**SGLang `srt/disaggregation/` ≠ vLLM `kv_transfer/kv_connector/`**——前者把「PD 角色判定 + KV 传输 + bootstrap 服务 + scheduler mixin + EPD encoder 分离」**全打包**；后者仅「KV 传输 backend」，PD 角色靠 [`vllm/entrypoints/serve/disagg/`](d:\design\vllm\vllm\entrypoints\serve\disagg) 前端 + scheduler `_update_waiting_for_remote_kv` 分摊。MindIE 又是另一种切分：**独立 `connector/` 子进程仅做 KV transfer**（[`mindie/topics/connector.md`](../../mindie/topics/connector.md)）。三家「PD 子模块」内容范围都不同，对比时不可只按目录名套等价（详见 [`comparison/topics/pd-disaggregation.md`](../../comparison/topics/pd-disaggregation.md)）。

> [!warning] CONTRADICTION（命名陷阱 2）：**`srt/multiplex/` ≠ HTTP 多路复用 ≠ 请求多租户**——见 [`sglang/modules/multiplex.md` Summary](../modules/multiplex.md) 同名警告。本页关心的是 **PD-Mux 与 PD-Disagg 互斥**这一具体语义。

> [!todo] VERIFY: PP + PD 复合循环 [`scheduler_pp_mixin.py:148, 324`](d:\design\sglang\python\sglang\srt\managers\scheduler_pp_mixin.py)（`event_loop_pp_disagg_prefill` / `event_loop_pp_disagg_decode`）**不在 `disaggregation/` 包内** 而在 `scheduler_pp_mixin.py`，跨包定义；wiki 中尚未单独整理 PP+PD 协同。后续若 ingest `topics/scheduler-mixins.md` 应特别注明。

> [!todo] VERIFY: `--disaggregation-decode-polling-interval` 默认 `1`（[`server_args.py:713`](d:\design\sglang\python\sglang\srt\server_args.py)），未追到 `decode.py` 中具体生效点（应在 `process_decode_queue` 或 `DecodeTransferQueue` 内）。

> [!todo] VERIFY: 第三方 backend 扩展能力——`add_disagg_transfer_backend_choices` 与闭集 `TransferBackend` / `get_kv_class` 之间的鸿沟，是否有 examples/docs 演示完整扩展路径？需在 `docs/` / `tests/` 反查。

> synthesis: 本页 4 维度的对偶关系——**5 backend 选 1（运行时枚举）** + **2 mixin 同时挂载（静态 MRO）** + **EPD 与 PD 正交叠加（独立 server_args 字段）** + **PD-Mux 与 PD-Disagg 强互斥（启动期断言）**。「枚举/MRO/正交/互斥」混合模式相比 vLLM「单一 connector 接口 + 14 实现」抽象层级更多但拼接更灵活。

## See also

- [sglang/modules/disaggregation.md](../modules/disaggregation.md) — 模块页（28 文件清单 + 5 backend + 类继承图，本页的代码侧底座）
- [sglang/modules/multimodal.md](../modules/multimodal.md) — EPD `encode_server.py` `from sglang.srt.multimodal.processors.qwen_vl import preprocess_video` 的被依赖方
- [sglang/modules/multiplex.md](../modules/multiplex.md) — PD-Multiplex（互斥的另一边）
- [sglang/modules/managers.md](../modules/managers.md) — `disagg_service.py` 所属模块
- [sglang/entities/Scheduler.md](../entities/Scheduler.md) — `init_disaggregation` ([L1051-L1182](d:\design\sglang\python\sglang\srt\managers\scheduler.py)) 与 11 mixin 含本页 2 个的整体视图
- [comparison/topics/pd-disaggregation.md](../../comparison/topics/pd-disaggregation.md) — **三家 PD 14 子维度跨项目对比**（对照页，本页是其 SGLang 行展开）
- [comparison/dimensions.md](../../comparison/dimensions.md) — 维度索引（`§dim-pd` / `§dim-kv-transfer`）
- [vllm/topics/kv-connector.md](../../vllm/topics/kv-connector.md) — vLLM 14 backend 的 connector 抽象（对照参考）
- [mindie/topics/connector.md](../../mindie/topics/connector.md) — MindIE 独立 connector 子进程实现（对照参考）
