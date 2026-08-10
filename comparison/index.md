---
type: index
project: cross
status: verified
confidence: high
verified_against: 2026-04-18
sources: []
related:
  - ../index.md
  - ../AGENTS.md
  - dimensions.md
  - topics/scheduler.md
  - topics/sync-schedule.md
  - topics/async-schedule.md
  - topics/kv-cache.md
  - topics/chunked-prefill.md
  - topics/cp-sp.md
  - topics/flashcomm.md
  - topics/pd-disaggregation.md
  - topics/distributed.md
  - topics/speculative-decoding.md
  - topics/engine-architecture.md
  - topics/executor-worker.md
  - topics/multiproc-ipc.md
  - topics/prefix-cache.md
  - topics/moe.md
  - topics/scheduler-architecture.md
---

# Cross-project Comparison Index

> 三项目（MindIE / vLLM / SGLang）的横向对比入口。
>
> 维度清单见 [dimensions.md](dimensions.md)，按主题的三方对照页待 ingest 时填到 `topics/` 下。

---

## 工作约定（摘自 [AGENTS.md §8](../AGENTS.md)）

1. 每个 `topics/<topic>.md` 必须有一张三方对照表，表头固定为 `维度 | MindIE | vLLM | SGLang`。
2. 每个 cell 必须指向具体页面（实证 anchor），未实现写 `N/A (verified <日期>)`，禁止臆测。
3. 不写"X 比 Y 好"，只写"X 用 A 方案，Y 用 B 方案"。

---

## 待建对比页清单

按优先级排序（高 → 低）。粗体是与你当前 PD 分离优化最直接相关的。

| 主题 | wiki 页 | 状态 | 涉及维度 |
|---|---|---|---|
| **PD 分离 / Disaggregated Serving** | [topics/pd-disaggregation.md](topics/pd-disaggregation.md) | **DONE** | §dim-pd, §dim-kv-transfer |
| **KV Cache 管理** | [topics/kv-cache.md](topics/kv-cache.md) | **DONE** | §dim-kv |
| **Chunked Prefill / Splitfuse** | [topics/chunked-prefill.md](topics/chunked-prefill.md) | **DONE** | §dim-batching |
| **CP / SP（Context / Sequence Parallel）** | [topics/cp-sp.md](topics/cp-sp.md) | **DONE** | §dim-cp-sp |
| **FlashComm（TP 通信优化）** | [topics/flashcomm.md](topics/flashcomm.md) | **DONE** | §dim-flashcomm |
| Continuous batching | （已合并到 chunked-prefill） | — | §dim-batching |
| **Engine 架构** | [topics/engine-architecture.md](topics/engine-architecture.md) | **DONE** | §dim-overall, §dim-engine, §dim-executor |
| **Executor / Worker 模型** | [topics/executor-worker.md](topics/executor-worker.md) | **DONE** | §dim-executor |
| **Scheduler 策略** | [topics/scheduler.md](topics/scheduler.md) | **DONE** | §dim-scheduler |
| **Sync-Schedule（默认无 overlap 路径）** | [topics/sync-schedule.md](topics/sync-schedule.md) | **DONE** | §dim-sync-schedule |
| **Async-Schedule（CPU/GPU overlap 路径）** | [topics/async-schedule.md](topics/async-schedule.md) | **DONE** | §dim-async-schedule |
| **分布式并行（TP/PP/DP/EP）** | [topics/distributed.md](topics/distributed.md) | **DONE** | §dim-distributed |
| **Speculative Decoding** | [topics/speculative-decoding.md](topics/speculative-decoding.md) | **DONE** | §dim-spec |
| **Prefix Cache** | [topics/prefix-cache.md](topics/prefix-cache.md) | **DONE** | §dim-prefix-cache |
| Compilation / Graph capture | `topics/compilation.md` | TODO | §dim-compile |
| Sampling | `topics/sampling.md` | TODO | §dim-sampling |
| LoRA / Multi-LoRA | `topics/lora.md` | TODO | §dim-lora |
| 量化 | `topics/quantization.md` | TODO | §dim-quant |
| **MoE** | [topics/moe.md](topics/moe.md) | **DONE** | §dim-moe |
| 多模态 | `topics/multimodal.md` | TODO | §dim-multimodal |
| 服务化 / API 协议 | `topics/serving-api.md` | TODO | §dim-serving |
| 硬件后端 | `topics/hardware-backends.md` | TODO | §dim-hardware |
| **Scheduler 架构分解（mixin / 单类 / C++Python 切分）** | [topics/scheduler-architecture.md](topics/scheduler-architecture.md) | **DONE** | §dim-scheduler |
| KV 传输协议（Mooncake/NIXL/MORI） | `topics/kv-transfer-protocols.md` | TODO | §dim-kv-transfer |
| **Multi-process IPC** | [topics/multiproc-ipc.md](topics/multiproc-ipc.md) | **DONE** | §dim-ipc（新增）、§dim-overall |

---

## 项目入口快速跳转

- MindIE：N/A（`mindie/` wiki 已删除 2026-08-10；对比页 MindIE 列仍保留源码锚点）
- [vllm/overview.md](../vllm/overview.md) | [vllm/index.md](../vllm/index.md)
- [sglang/overview.md](../sglang/overview.md) | [sglang/index.md](../sglang/index.md)

## 已建对比页

| 主题 | 页 | 简介 |
|---|---|---|
| Scheduler | [topics/scheduler.md](topics/scheduler.md) | 三方调度器对比：进程位置 / Schedule 主循环 / 队列 / 策略模块化 / overlap / PD 支持 / 抢占 / Pause 状态机 |
| KV Cache | [topics/kv-cache.md](topics/kv-cache.md) | 三方 KV cache 体系：抽象层数 / Prefix 数据结构 / Eviction 策略 / 多 attention 类型 / HiCache / 量化 / PD 原生支持 / 稀疏 |
| Chunked Prefill | [topics/chunked-prefill.md](topics/chunked-prefill.md) | Splitfuse vs `long_prefill_token_threshold` vs `PrefillDelayer`，含 LoRA 互斥 / async 互斥等约束对比 |
| CP / SP | [topics/cp-sp.md](topics/cp-sp.md) | ATTN_CP/SP vs PCP/DCP vs zigzag CP，三家切分策略与 attention impl 协议对比 |
| FlashComm | [topics/flashcomm.md](topics/flashcomm.md) | MindIE 专有命名 + vLLM torch.compile pass + SGLang LayerCommunicator 三种 TP 通信优化路径 |
| PD-Disaggregation | [topics/pd-disaggregation.md](topics/pd-disaggregation.md) | 14 个子维度：架构 / 角色 / 传输栈 / Bootstrap / Scheduler 集成 / 队列 / 状态机 / pull-vs-push / HMA / PrefixCache+PD / 失败处理 / 服务化前端 / Chunked+Spec / 优化建议 |
| Sync-Schedule | [topics/sync-schedule.md](topics/sync-schedule.md) | 三家 sync 调度路径对比：默认值 / 配置开关 / in-flight batch 上限 / 主循环 / 与 spec/grammar/PD/PP 兼容性 / `Schedule(needSync)` 命名陷阱（≠ "sync schedule"） |
| Async-Schedule | [topics/async-schedule.md](topics/async-schedule.md) | 三家 async 调度路径对比：MindIE C++ AsyncExecute callback vs vLLM placeholder 状态级异步 vs SGLang result_queue 三段流水；spec+grammar 三家协议 + `is_disable_overlap_for_batch` per-batch 决策（仅 SGLang） |
| Distributed Parallel | [topics/distributed.md](topics/distributed.md) | TP/PP/DP/EP 四轴对比：进程组体系 / Device communicators / PP 完成度（MindIE 草稿、SGLang 最完整）/ DP 双语义（DP-batch vs DP-attention，vLLM 缺后者）/ EPLB（MindIE 缺）+ Elastic-EP（MindIE 缺）/ scheduler 显式 rank 维度数（6/3-5/2） |
| Speculative Decoding | [topics/speculative-decoding.md](topics/speculative-decoding.md) | 三家投机解码对比：算法目录（MindIE plugin / vLLM SpecDecodeBaseProposer / SGLang SpeculativeAlgorithm Enum）+ draft 模型集成（双 ModelRunner / proposer 单 model / 独立 TpModelWorker）+ verify（Python 贪婪 / RejectionSampler 三模式 triton / verify_tree_greedy CUDA kernel）+ scheduler 占位（PLACEHOLDER_TOKEN -1 写序列 / num_output_placeholders int 计数 / spec_info dataclass）+ 兼容性矩阵 + anchor-driven cross-check |
| Engine Architecture | [topics/engine-architecture.md](topics/engine-architecture.md) | 三家顶层引擎架构对比：**MindIE Python+C++ 双层** vs **vLLM 三层 Python 洋葱 + 4 实现 Executor 抽象** vs **SGLang 严格 3 进程 ZMQ pipeline**；13 子维度（进程拓扑 / 顶层入口 / sync-async API 拆分 / Engine↔Scheduler 边界 / Engine↔Worker 边界 / Tokenize-Detokenize 进程位置 / 跨语言 / 启动序列 / 关闭语义 / PD 关系 / 综合 cheat sheet / anchor-driven cross-check / 与 PD 优化的关联）+ §13 综合 cheat sheet + 5 条 PD 优化借鉴 synthesis |
| Executor / Worker | [topics/executor-worker.md](topics/executor-worker.md) | 三家 executor / worker 边界深度对比：**vLLM 唯一显式 `Executor` 抽象**（4 实现）+ Worker 独立进程 + MessageQueue 字符串 RPC vs **MindIE 无 Executor**（Generator 直挂 PluginManager.forward_thread 同进程线程）vs **SGLang 无 Executor**（Scheduler 直持 TpModelWorker 同进程对象）；11 子维度（Executor 抽象 / Worker 类层次 / 进程模型 / RPC 调用机制 / forward+sample 拆分原因 / 多 worker 并行 / 启动序列 / 错误处理三段式 / 综合 cheat sheet / anchor-driven cross-check / 与 PD 优化的关联）+ §11 综合 cheat sheet（13 行）+ 6 条 PD 优化借鉴 synthesis |
| Multi-process IPC | [topics/multiproc-ipc.md](topics/multiproc-ipc.md) | 三家 IPC 协议栈深度对比：**vLLM 5+1 类机制**（ZMQ + MessageQueue 共享内存 + Pipe + queue.Queue + POSIX signal + TensorIPC）vs **SGLang 1 类**（ZMQ pickle 全 IPC）vs **MindIE 1 类**（共享内存 + protobuf，仅 connector 子进程）；9 子维度（进程边界数 / 协议栈对照 / 序列化机制 / 零拷贝路径 / ready+death 检测 / ZMQ socket 类型 / 综合 cheat sheet / anchor-driven cross-check / 与 PD 优化的关联）+ 6 条 PD 优化借鉴 synthesis（含 vLLM MessageQueue / cloudpickle / TensorIPC 三大可借鉴点） |
| Prefix Cache | [topics/prefix-cache.md](topics/prefix-cache.md) | 三家 prefix cache 深度对比：**hash table vs trie 三家两派**（MindIE C++ `unordered_map` + Python plugin 双段 / vLLM Python `BlockHashToBlockMap` 单段 / SGLang `RadixCache` trie + 8+ 实现工厂分支）；11 子维度（数据结构与抽象 / hash 算法 / 命中查询 API / eviction 策略 / hybrid 模型 / 多层存储 HiCache / 配置开关与默认 / 兼容性矩阵 / 跨语言绑定 / anchor-driven cross-check / 与 PD 优化的关联）+ §11 综合 cheat sheet（17 行）+ 6 处 anchor cross-check N/A 强论断 + 6 条 PD 优化借鉴 synthesis（含 prefetch 异步 / delay_cache_blocks / write-back ack / 可配置 eviction） |
| **MoE** | [topics/moe.md](topics/moe.md) | 三家 MoE 体系深度对比：**SGLang 三层正交**（FusedMoE/DeepEPMoE × 7 dispatcher × 11 runner）vs **vLLM 双轴正交 + oracle 子包**（68 .py fused_moe / 9 .py eplb / 3 .py elastic_ep / 13 .cu csrc/moe / 10 All2AllBackend Literal）vs **MindIE 单枚举二合一 + dispatch+FFN+combine 单算子**（6 .py fused_moe + 6 .cpp）；13 子维度（impl 抽象层 / a2a backend / runner backend / EP / EPLB / Elastic EP / TBO+SBO / DeepEP+Mooncake+Mori+NIXL+FlashInfer+MC2 集成 / C++/CUDA kernel / topk router / expert recorder / 量化 / 模型族）+ 5 synthesis（DeepSeek-V3 三家共同灯塔 + vLLM `routed_experts_capturer.py` 显式 attribution to SGLang URL — 稀有跨项目代码继承显式案例 + 三家「枚举先于实现」dead-branch 模式）+ 13 处 anchor cross-check N/A 强论断 |
| Scheduler Architecture | [topics/scheduler-architecture.md](topics/scheduler-architecture.md) | **架构分解**视角（区别于 scheduler.md 的策略视角）：**MindIE 双语言双层**（C++ LlmEngine + C++ BatchScheduler + 15 Policy .h + Python Generator+PluginManager，跨 pybind 紧耦合）vs **vLLM 单类双层**（EngineCore + Scheduler 各 0 mixin + AsyncScheduler 子类 + 6 EngineCoreClient 分支；但 Worker 层 `GPUModelRunner` 有 3 mixin）vs **SGLang 1 类 + 11 mixin**（5 external 跟随子系统目录 + 6 internal in `srt/managers/`+ `dispatch_event_loop` 8 mode + `process_batch_result` 6-way 共享分派）；9 子维度（顶层架构 / mixin 模块化 / 跨语言 / 多 mode dispatch / 共享分派点 / 可选 feature 加载 / 测试粒度 / 热更新 / cheat sheet）+ 5 synthesis（含「vLLM Scheduler 0 mixin vs Worker 3 mixin」项目内不同立场 + 「MindIE SwitchRole 一等公民设计 vLLM/SGLang 缺失」可借鉴点）|
