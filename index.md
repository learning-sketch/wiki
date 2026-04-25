---
type: index
project: cross
status: verified
confidence: high
verified_against: 2026-04-18
sources: []
related:
  - README.md
  - AGENTS.md
  - log.md
  - source-versions.md
  - comparison/index.md
  - mindie/index.md
  - vllm/index.md
  - sglang/index.md
---

# Global Index

> 全局目录。按"项目"和"主题"两个维度组织。
> 项目内详细索引见 `<project>\index.md`。

---

## 项目

| 项目 | Overview | Index | 主代码根 |
|---|---|---|---|
| **MindIE-LLM** | [overview](mindie/overview.md) | [index](mindie/index.md) | [d:\design\MindIE-LLM\mindie_llm\](d:\design\MindIE-LLM\mindie_llm) |
| **vLLM** | [overview](vllm/overview.md) | [index](vllm/index.md) | [d:\design\vllm\vllm\](d:\design\vllm\vllm) |
| **SGLang** | [overview](sglang/overview.md) | [index](sglang/index.md) | [d:\design\sglang\python\sglang\srt\](d:\design\sglang\python\sglang\srt) |

---

## 跨项目对比

- [comparison/index.md](comparison/index.md)
- [comparison/dimensions.md](comparison/dimensions.md) — 24 个对比维度

按主题对照（`comparison/topics/`，待 ingest）：

| 主题 | 状态 |
|---|---|
| 整体架构 | **DONE** → [comparison/topics/engine-architecture.md](comparison/topics/engine-architecture.md) |
| **Engine 抽象** | **DONE** → [comparison/topics/engine-architecture.md](comparison/topics/engine-architecture.md)（13 子维度，§dim-overall / §dim-engine 双维度深度对比） |
| **Executor / Worker** | **DONE** → [comparison/topics/executor-worker.md](comparison/topics/executor-worker.md)（11 子维度：Executor 抽象 / Worker 类层次 / 进程模型 / RPC 调用机制 / forward+sample 拆分 / 多 worker 并行 / 启动序列 / 错误处理 / 综合 cheat sheet / anchor-driven cross-check / 与 PD 优化的关联）|
| **Multi-process IPC** | **DONE** → [comparison/topics/multiproc-ipc.md](comparison/topics/multiproc-ipc.md)（9 子维度：进程边界数 / 协议栈对照 / 序列化机制 / 零拷贝路径 / ready+death 检测 / ZMQ socket 类型 / 综合 cheat sheet / anchor-driven cross-check / 与 PD 优化的关联） |
| **Scheduler** | **DONE** → [comparison/topics/scheduler.md](comparison/topics/scheduler.md) |
| **Sync-Schedule（默认无 overlap 路径）** | **DONE** → [comparison/topics/sync-schedule.md](comparison/topics/sync-schedule.md) |
| **Async-Schedule（CPU/GPU overlap 路径）** | **DONE** → [comparison/topics/async-schedule.md](comparison/topics/async-schedule.md) |
| **KV Cache** | **DONE** → [comparison/topics/kv-cache.md](comparison/topics/kv-cache.md) |
| **Chunked Prefill** | **DONE** → [comparison/topics/chunked-prefill.md](comparison/topics/chunked-prefill.md) |
| **CP / SP** | **DONE** → [comparison/topics/cp-sp.md](comparison/topics/cp-sp.md) |
| **FlashComm** | **DONE** → [comparison/topics/flashcomm.md](comparison/topics/flashcomm.md) |
| **Distributed Parallel (TP/PP/DP/EP)** | **DONE** → [comparison/topics/distributed.md](comparison/topics/distributed.md) |
| **PD 分离** | **DONE** → [comparison/topics/pd-disaggregation.md](comparison/topics/pd-disaggregation.md) |
| KV 传输 | **覆盖在 PD 分离对比 §3** → [comparison/topics/pd-disaggregation.md](comparison/topics/pd-disaggregation.md) |
| **Speculative Decoding** | **DONE** → [comparison/topics/speculative-decoding.md](comparison/topics/speculative-decoding.md) |
| **Prefix Cache** | **DONE** → [comparison/topics/prefix-cache.md](comparison/topics/prefix-cache.md)（11 子维度，**hash table vs trie 三家两派**：MindIE C++ `unordered_map`+Python plugin 双段 / vLLM Python `BlockHashToBlockMap` 单段 / SGLang `RadixCache` trie + 8+ 实现工厂分支） |
| Compilation / Graph capture | TODO（MindIE 已有 [aclgraph-pp.md](mindie/topics/aclgraph-pp.md)） |
| Sampling | TODO |
| LoRA | TODO |
| 量化 | TODO |
| MoE | TODO |
| 多模态 | TODO |
| 服务化层 | TODO |
| 硬件后端 | TODO |
| Continuous batching | TODO |

---

## 跨项目主题快速跳转（按关键词）

> ingest 时把关键词指向具体页，下面会逐渐填满。

| 关键词 | MindIE | vLLM | SGLang |
|---|---|---|---|
| 生成层入口 / Engine | [mindie/modules/text_generator.md](mindie/modules/text_generator.md) | [vllm/modules/engine.md](vllm/modules/engine.md) | [sglang/modules/entrypoints.md](sglang/modules/entrypoints.md) |
| Executor / 调度进程 | [mindie/modules/runtime_model_runner.md](mindie/modules/runtime_model_runner.md) | [vllm/modules/executor.md](vllm/modules/executor.md) | [sglang/modules/managers.md](sglang/modules/managers.md) |
| 主对象（顶层 class） | [mindie/entities/Generator.md](mindie/entities/Generator.md) | [vllm/entities/LLMEngine.md](vllm/entities/LLMEngine.md) / [AsyncLLM.md](vllm/entities/AsyncLLM.md) | [sglang/entities/Engine.md](sglang/entities/Engine.md) |
| 模型执行器 / Worker | [mindie/entities/ModelRunner.md](mindie/entities/ModelRunner.md) | [vllm/entities/GPUWorker.md](vllm/entities/GPUWorker.md) + [GPUModelRunner.md](vllm/entities/GPUModelRunner.md) | [sglang/entities/TpModelWorker.md](sglang/entities/TpModelWorker.md) |
| **Scheduler** | [mindie/entities/BatchScheduler.md](mindie/entities/BatchScheduler.md) (C++) | [vllm/entities/Scheduler.md](vllm/entities/Scheduler.md) | [sglang/entities/Scheduler.md](sglang/entities/Scheduler.md) |
| **KV Cache** | [mindie/topics/kv-cache.md](mindie/topics/kv-cache.md) (C++ + Python) | [vllm/entities/KVCacheManager.md](vllm/entities/KVCacheManager.md) | [sglang/modules/mem_cache.md](sglang/modules/mem_cache.md) (62 .py) |
| Request lifecycle | [mindie/topics/request-lifecycle.md](mindie/topics/request-lifecycle.md) | [vllm/topics/request-lifecycle.md](vllm/topics/request-lifecycle.md) | [sglang/topics/request-lifecycle.md](sglang/topics/request-lifecycle.md) |
| Multi-process IPC / Pipeline | [mindie/topics/request-lifecycle.md](mindie/topics/request-lifecycle.md)（含 PD 链路） | [vllm/topics/multiproc-ipc.md](vllm/topics/multiproc-ipc.md) | [sglang/topics/manager-pipeline.md](sglang/topics/manager-pipeline.md) |
| **Pipeline Parallel** | [mindie/topics/aclgraph-pp.md](mindie/topics/aclgraph-pp.md) | TODO | TODO |
| **PD 分离** | [mindie/topics/request-lifecycle.md](mindie/topics/request-lifecycle.md) §PD 分离链路 | [comparison/topics/pd-disaggregation.md](comparison/topics/pd-disaggregation.md) §1 vLLM 分支（KVConnector_V1 + entrypoints/serve/disagg） | [comparison/topics/pd-disaggregation.md](comparison/topics/pd-disaggregation.md) §1 SGLang 分支（disaggregation/ + scheduler 两 mixin） |

---

## 操作入口

- 新增 ingest：跟 LLM 说 `ingest <project> <topic>`
- 提问：直接问
- 健康检查：`lint`
- 上游代码增量：`increment` / `update`（详 [AGENTS.md §12](AGENTS.md)）
- 详细规则：[AGENTS.md](AGENTS.md)
- 三仓代码版本 pin：[source-versions.md](source-versions.md)
- 历史：[log.md](log.md) + [log-archive/](log-archive)
