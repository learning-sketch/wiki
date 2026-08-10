---
type: topic
project: sglang
status: verified
confidence: high
verified_against: 2026-08-10
sources:
  - d:\design\sglang\python\sglang\srt\managers\scheduler.py:L516-L555
  - d:\design\sglang\python\sglang\srt\mem_cache\kv_cache_builder.py:L133-L284
  - d:\design\sglang\python\sglang\srt\mem_cache\registry.py:L1-L243
  - d:\design\sglang\python\sglang\srt\mem_cache\base_prefix_cache.py:L230-L413
  - d:\design\sglang\python\sglang\srt\mem_cache\cache_init_params.py
  - d:\design\sglang\python\sglang\srt\mem_cache\allocator\
  - d:\design\sglang\python\sglang\srt\mem_cache\radix_cache.py:L279
  - d:\design\sglang\python\sglang\srt\mem_cache\radix_cache_cpp.py:L35
  - d:\design\sglang\python\sglang\srt\mem_cache\hiradix_cache.py:L76
  - d:\design\sglang\python\sglang\srt\mem_cache\pure_swa_radix_cache.py:L24
  - d:\design\sglang\python\sglang\srt\mem_cache\chunk_cache.py:L35-L142
  - d:\design\sglang\python\sglang\srt\mem_cache\unified_radix_cache.py:L133
  - d:\design\sglang\python\sglang\srt\mem_cache\unified_cache\
  - d:\design\sglang\python\sglang\srt\mem_cache\swa_radix_cache.py:L343
  - d:\design\sglang\python\sglang\srt\mem_cache\mamba_radix_cache.py:L442
  - d:\design\sglang\python\sglang\srt\mem_cache\storage\lmcache\lmc_radix_cache.py:L86
  - d:\design\sglang\python\sglang\srt\mem_cache\storage\flexkv\flexkv_radix_cache.py:L72
  - d:\design\sglang\python\sglang\srt\mem_cache\storage\backend_factory.py:L197-L247
  - d:\design\sglang\python\sglang\srt\mem_cache\hicache_storage.py:L150-L361
  - d:\design\sglang\python\sglang\srt\mem_cache\multimodal_cache.py:L11-L76
  - d:\design\sglang\python\sglang\srt\mem_cache\memory_pool.py:L256
  - d:\design\sglang\python\sglang\srt\mem_cache\evict_policy.py:L10-L49
  - d:\design\sglang\python\sglang\srt\session\streaming_session.py:L130
  - d:\design\sglang\python\sglang\srt\environ.py:L451
  - d:\design\sglang\python\sglang\srt\environ.py:L981
  - d:\design\sglang\python\sglang\srt\environ.py:L1019
  - d:\design\sglang\python\sglang\srt\server_args.py
related:
  - sglang/modules/mem_cache.md
  - sglang/modules/kv_canary.md
  - sglang/modules/session.md
  - sglang/modules/multimodal.md
  - sglang/entities/Scheduler.md
  - sglang/entities/TpModelWorker.md
  - sglang/topics/pd-disaggregation.md
  - comparison/topics/kv-cache.md
  - comparison/topics/prefix-cache.md
---

# KV Cache 体系（SGLang 内部）

## Summary

synthesis: 相对 pin `34fef07a`，HEAD `06f32bab` 把 KV/prefix cache **工厂从 `Scheduler.init_cache_with_memory_pool` 抽到** [`kv_cache_builder.build_kv_cache`](d:\design\sglang\python\sglang\srt\mem_cache\kv_cache_builder.py) + [`registry.create_tree_cache`](d:\design\sglang\python\sglang\srt\mem_cache\registry.py)。四层正交抽象仍在：(1) **tree_cache 工厂链**（`default_radix_cache_factory` + 可插拔 `--radix-cache-backend`）；(2) **`BaseTokenToKVPoolAllocator`**（现拆到 [`allocator/`](d:\design\sglang\python\sglang\srt\mem_cache\allocator) 包）与 `ReqToTokenPool` / 各 `*TokenToKVPool`；(3) **HiCache** device→host→storage（plain `HiRadixCache` 或 `UnifiedRadixCache.init_hicache`）；(4) **多模态嵌入** `MultiModalStaticCache`（经 [`mm_schedule.init_mm_embedding_cache`](d:\design\sglang\python\sglang\srt\managers\mm_schedule.py)）。重大漂移：`HiMambaRadixCache` / `session_aware_cache.py` / `unified_cache_components/` **已删除**；hybrid SWA/SSM + hierarchical 默认走 **`UnifiedRadixCache`**；`StreamingSession` 迁到 [`session/`](d:\design\sglang\python\sglang\srt\session)；新增 **FlexKV** 与 **9** 个 HiCache storage 注册名。

## Sources

- Scheduler 入口：[scheduler.py:L516-L555](d:\design\sglang\python\sglang\srt\managers\scheduler.py)（`build_kv_cache` → `self.tree_cache`；可选 `canary_manager.attach_radix_cache`）
- Builder：[kv_cache_builder.py:L133-L284](d:\design\sglang\python\sglang\srt\mem_cache\kv_cache_builder.py)
- 工厂 / 注册表：[registry.py](d:\design\sglang\python\sglang\srt\mem_cache\registry.py)（`TreeCacheBuildContext` / `default_radix_cache_factory` / `create_tree_cache`）
- 抽象：[base_prefix_cache.py:L230+](d:\design\sglang\python\sglang\srt\mem_cache\base_prefix_cache.py)、[cache_init_params.py](d:\design\sglang\python\sglang\srt\mem_cache\cache_init_params.py)、[allocator/](d:\design\sglang\python\sglang\srt\mem_cache\allocator)
- 生产路径实现：见 §工厂矩阵
- Legacy 仍在树内但**不在默认工厂链**：[swa_radix_cache.py:L343](d:\design\sglang\python\sglang\srt\mem_cache\swa_radix_cache.py)、[mamba_radix_cache.py:L442](d:\design\sglang\python\sglang\srt\mem_cache\mamba_radix_cache.py)
- Unified：[unified_radix_cache.py](d:\design\sglang\python\sglang\srt\mem_cache\unified_radix_cache.py) + [unified_cache/](d:\design\sglang\python\sglang\srt\mem_cache\unified_cache)
- HiCache storage：[hicache_storage.py](d:\design\sglang\python\sglang\srt\mem_cache\hicache_storage.py) + [storage/backend_factory.py:L197-L247](d:\design\sglang\python\sglang\srt\mem_cache\storage\backend_factory.py)
- Session wrap：[streaming_session.py:L130](d:\design\sglang\python\sglang\srt\session\streaming_session.py)
- 多模态：[multimodal_cache.py](d:\design\sglang\python\sglang\srt\mem_cache\multimodal_cache.py)、[mm_schedule.py:L21](d:\design\sglang\python\sglang\srt\managers\mm_schedule.py)
- Env：[environ.py:L451/L981/L1019](d:\design\sglang\python\sglang\srt\environ.py)

## Architecture

```mermaid
flowchart TD
    Sched["Scheduler.__init__<br/>scheduler.py:516"]
    Build["kv_cache_builder.build_kv_cache"]
    Pool["tp_worker.get_memory_pool()<br/>ReqToTokenPool + Allocator"]
    Params["CacheInitParams"]
    Create["registry.create_tree_cache"]
    Sched --> Build
    Build --> Pool & Params --> Create

    subgraph Factory["default_radix_cache_factory / registered backend"]
      direction TB
      B["--radix-cache-backend"] --> Reg[registered factory e.g. flexkv]
      D1[disable_radix + chunked] --> Chunk[ChunkCache / SWAChunk* / PureSWAChunk*]
      D2[CPP_RADIX_TREE env] --> Cpp[RadixCacheCpp]
      D3[UNIFIED env / mlx / hybrid / DSA+Hi] --> Uni[UnifiedRadixCache]
      D4[pure SWA full_tokens=0] --> PureSWA[PureSWARadixCache]
      D5[hierarchical plain] --> Hi[HiRadixCache]
      D6[enable_lmcache] --> LMC[LMCRadixCache]
      D7[enable_flexkv] --> Flex[FlexKVRadixCache]
      D8[default] --> RC[RadixCache]
    end

    Create --> Factory
    Factory -->|enable_streaming_session<br/>and not supports_streaming| SS[StreamingSession wrap]
    Uni -->|always embeds| SS2[StreamingSession inner]
    Hi --> Host[HostKVCache + HiCacheController]
    Uni -->|init_hicache| Host
    Host --> Store[(HiCacheStorage<br/>9 registered names)]
    Build -->|init_mm_embedding_cache| MM[MultiModalStaticCache]
    Sched -->|optional| Canary[kv_canary.attach_radix_cache]
```

> synthesis: **工厂外置**是本次最大架构变化——Scheduler 只持 `tree_cache` 句柄；选型逻辑集中在 `registry.py`，便于 `--radix-cache-backend` 插件化。Unified 已成为 **hybrid + hierarchical/DSA** 的汇合点，旧 `SWARadixCache`/`MambaRadixCache`/`HiMambaRadixCache` 不再出现在默认链上。

## 调用链（Scheduler → tree_cache）

1. [`Scheduler`](d:\design\sglang\python\sglang\srt\managers\scheduler.py) 在模型 worker 初始化后调用 [`build_kv_cache(...)`](d:\design\sglang\python\sglang\srt\mem_cache\kv_cache_builder.py)（[scheduler.py:516-550](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）。
2. Builder 判定 `is_hybrid_swa` / `is_hybrid_ssm` / `is_dsa`，并从 `tp_worker.get_memory_pool()` 取 pool（[kv_cache_builder.py:158-178](d:\design\sglang\python\sglang\srt\mem_cache\kv_cache_builder.py)）。
3. 组装 [`CacheInitParams`](d:\design\sglang\python\sglang\srt\mem_cache\cache_init_params.py)（含 `eviction_policy`、`enable_session_radix_cache`、PP/TP/CP groups 等）（[kv_cache_builder.py:212-242](d:\design\sglang\python\sglang\srt\mem_cache\kv_cache_builder.py)）。
4. [`create_tree_cache(TreeCacheBuildContext)`](d:\design\sglang\python\sglang\srt\mem_cache\registry.py)（[kv_cache_builder.py:244-260](d:\design\sglang\python\sglang\srt\mem_cache\kv_cache_builder.py) + [registry.py:196-243](d:\design\sglang\python\sglang\srt\mem_cache\registry.py)）。
5. 可选：hierarchical draft 注册、`init_mm_embedding_cache`（[kv_cache_builder.py:263-272](d:\design\sglang\python\sglang\srt\mem_cache\kv_cache_builder.py)）。
6. 返回后 Scheduler 赋值 `self.tree_cache`；若存在 `canary_manager` 则 `attach_radix_cache`（[scheduler.py:548-555](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）。

## 工厂矩阵（`default_radix_cache_factory`）

优先级自顶向下（[registry.py:79-160](d:\design\sglang\python\sglang\srt\mem_cache\registry.py)）。另：`--radix-cache-backend=<name>` 走注册表，**跳过**本链（[registry.py:198-208](d:\design\sglang\python\sglang\srt\mem_cache\registry.py)）。

| # | 条件 | 构造类 | 锚点 |
|---|---|---|---|
| 0 | `--radix-cache-backend` 已设 | 注册工厂（内置仅见 `"flexkv"`） | [registry.py:198-208](d:\design\sglang\python\sglang\srt\mem_cache\registry.py) |
| 1 | `disable_radix_cache` ∧ `effective_chunked_prefill_size!=None` | `ChunkCache` / `PureSWAChunkCache` / `SWAChunkCache` | [registry.py:84-95](d:\design\sglang\python\sglang\srt\mem_cache\registry.py)；类 [chunk_cache.py:35/115/142](d:\design\sglang\python\sglang\srt\mem_cache\chunk_cache.py) |
| 2 | env `SGLANG_EXPERIMENTAL_CPP_RADIX_TREE` | `RadixCacheCpp` | [registry.py:97-102](d:\design\sglang\python\sglang\srt\mem_cache\registry.py)；[radix_cache_cpp.py:35](d:\design\sglang\python\sglang\srt\mem_cache\radix_cache_cpp.py)；C++ [cpp_radix_tree/](d:\design\sglang\python\sglang\srt\mem_cache\cpp_radix_tree) |
| 3 | env `SGLANG_ENABLE_UNIFIED_RADIX_TREE` ∨ `use_mlx()` | `UnifiedRadixCache`（可 `init_hicache`） | [registry.py:104-105](d:\design\sglang\python\sglang\srt\mem_cache\registry.py) → [`_create_unified_radix_cache`](d:\design\sglang\python\sglang\srt\mem_cache\registry.py):163-193 |
| 4a | `is_hybrid_swa` ∧ `full_tokens_per_layer==0` | `PureSWARadixCache` | [registry.py:107-111](d:\design\sglang\python\sglang\srt\mem_cache\registry.py)；[pure_swa_radix_cache.py:24](d:\design\sglang\python\sglang\srt\mem_cache\pure_swa_radix_cache.py) |
| 4b/5 | `is_hybrid_swa` / `is_hybrid_ssm` | `UnifiedRadixCache` | [registry.py:112-115](d:\design\sglang\python\sglang\srt\mem_cache\registry.py) |
| 6a | `enable_hierarchical_cache` ∧ (hybrid_ssm ∨ hybrid_swa ∨ DSA) | `UnifiedRadixCache` + `init_hicache` | [registry.py:117-121](d:\design\sglang\python\sglang\srt\mem_cache\registry.py) |
| 6b | `enable_hierarchical_cache`（plain） | `HiRadixCache` | [registry.py:122-129](d:\design\sglang\python\sglang\srt\mem_cache\registry.py)；[hiradix_cache.py:76](d:\design\sglang\python\sglang\srt\mem_cache\hiradix_cache.py) |
| 7 | `--enable-lmcache` | `LMCRadixCache` | [registry.py:131-142](d:\design\sglang\python\sglang\srt\mem_cache\registry.py)；[lmc_radix_cache.py:86](d:\design\sglang\python\sglang\srt\mem_cache\storage\lmcache\lmc_radix_cache.py) |
| 8 | `--enable-flexkv` | `FlexKVRadixCache` | [registry.py:144-156](d:\design\sglang\python\sglang\srt\mem_cache\registry.py)；[flexkv_radix_cache.py:72](d:\design\sglang\python\sglang\srt\mem_cache\storage\flexkv\flexkv_radix_cache.py) |
| 9 | else | `RadixCache` | [registry.py:158-160](d:\design\sglang\python\sglang\srt\mem_cache\registry.py)；[radix_cache.py:279](d:\design\sglang\python\sglang\srt\mem_cache\radix_cache.py) |

**后处理（非分支）**（[registry.py:213-231](d:\design\sglang\python\sglang\srt\mem_cache\registry.py)）：

- `--enable-session-radix-cache` 要求实现已是 `UnifiedRadixCache`（否则 `ValueError`）。
- `--enable-streaming-session` 且 `not cache.supports_streaming_session()` → 外裹 [`StreamingSession`](d:\design\sglang\python\sglang\srt\session\streaming_session.py)（[L130](d:\design\sglang\python\sglang\srt\session\streaming_session.py)）。`UnifiedRadixCache` 在构造时**内嵌** `StreamingSession`（[unified_radix_cache.py:196-201](d:\design\sglang\python\sglang\srt\mem_cache\unified_radix_cache.py)）并 `supports_streaming_session()→True`（[L2109](d:\design\sglang\python\sglang\srt\mem_cache\unified_radix_cache.py)），避免双重包装。

> [!warning] CONTRADICTION（相对旧 wiki）: 旧页写「10 分支含 `HiMambaRadixCache` / `SWARadixCache` / `MambaRadixCache` + `SessionAwareCache`」。**RESOLVED 2026-08-10**：`HiMambaRadixCache`/`session_aware_cache.py` 源文件已删除；`SWARadixCache`/`MambaRadixCache` 类仍在树内但**全仓库无构造点**（工厂改走 Unified / PureSWA）。权威链见上表。

## `BasePrefixCache` 契约

[`BasePrefixCache(ABC, PrefixCacheTrait)`](d:\design\sglang\python\sglang\srt\mem_cache\base_prefix_cache.py) @ [L230](d:\design\sglang\python\sglang\srt\mem_cache\base_prefix_cache.py)。

- 抽象方法：`reset` / `match_prefix` / `cache_finished_req` / `cache_unfinished_req` / `evict` / `update` `inc_lock_ref` / `dec_lock_ref`（[L265-L327](d:\design\sglang\python\sglang\srt\mem_cache\base_prefix_cache.py)）。
- Capability 探针默认 False：`supports_swa` / `supports_mamba` / `supports_streaming_session` / `is_chunk_cache`（[L377-L413](d:\design\sglang\python\sglang\srt\mem_cache\base_prefix_cache.py)）。
- 参数化 dataclass：`MatchPrefixParams` / `EvictParams` / `MatchResult` 等（[L49+](d:\design\sglang\python\sglang\srt\mem_cache\base_prefix_cache.py)）。

**直接子类（生产+legacy）**：`ChunkCache`、`RadixCache`、`RadixCacheCpp`、`SWARadixCache`（legacy）、`MambaRadixCache`（legacy）、`UnifiedRadixCache`、`StreamingSession`。  
**间接**：`SWAChunkCache`/`PureSWAChunkCache` ← Chunk；`HiRadixCache`/`PureSWARadixCache`/`LMCRadixCache`/`FlexKVRadixCache` ← Radix。

## UnifiedRadixCache + `unified_cache/`

- 包分层见 [unified_cache/__init__.py](d:\design\sglang\python\sglang\srt\mem_cache\unified_cache\__init__.py)：`UnifiedTreeCore` + `components/{full,swa,mamba}_component.py` + facade [`UnifiedRadixCache`](d:\design\sglang\python\sglang\srt\mem_cache\unified_radix_cache.py)（类 @ [L133](d:\design\sglang\python\sglang\srt\mem_cache\unified_radix_cache.py)）。
- `_create_unified_radix_cache` 组装 components：始终 `FULL`；hybrid SWA → +`SWA`；hybrid SSM → +`MAMBA`（[registry.py:172-177](d:\design\sglang\python\sglang\srt\mem_cache\registry.py)）。MLX 可覆盖 MAMBA component（[L179-186](d:\design\sglang\python\sglang\srt\mem_cache\registry.py)）。
- hierarchical 时 `cache.init_hicache(server_args, params)` 并注册 layer transfer counter（[registry.py:188-192](d:\design\sglang\python\sglang\srt\mem_cache\registry.py)）。
- `--enable-session-radix-cache` 的会话 radix 跟踪在 [`unified_cache/session_ref_tracker.py`](d:\design\sglang\python\sglang\srt\mem_cache\unified_cache\session_ref_tracker.py)（与 `StreamingSession` 正交）。

> synthesis: Unified 从「env 实验开关」升级为 **hybrid/DSA/hierarchical 默认汇合实现**；`SGLANG_ENABLE_UNIFIED_RADIX_TREE` 仍可强制走 Unified（默认 False，[environ.py:1019](d:\design\sglang\python\sglang\srt\environ.py)），但 hybrid 路径不再依赖该 env。

## KV Pool / Allocator（与 prefix 解耦）

- Pool 仍由 `tp_worker.get_memory_pool()` 提供（[kv_cache_builder.py:178](d:\design\sglang\python\sglang\srt\mem_cache\kv_cache_builder.py)）。
- 旧单体 `allocator.py` **已删除**，改为包 [`mem_cache/allocator/`](d:\design\sglang\python\sglang\srt\mem_cache\allocator)：
  - `BaseTokenToKVPoolAllocator` — [allocator/base.py:27](d:\design\sglang\python\sglang\srt\mem_cache\allocator\base.py)
  - `TokenToKVPoolAllocator` — [allocator/token.py:28](d:\design\sglang\python\sglang\srt\mem_cache\allocator\token.py)
  - `PagedTokenToKVPoolAllocator` — [allocator/paged.py:105](d:\design\sglang\python\sglang\srt\mem_cache\allocator\paged.py)
  - 另有 `SWA*` / `HiSparse*` / `MambaSlotAllocator` 等专用实现（[allocator/swa.py](d:\design\sglang\python\sglang\srt\mem_cache\allocator\swa.py)、[hisparse.py](d:\design\sglang\python\sglang\srt\mem_cache\allocator\hisparse.py)、[mamba.py](d:\design\sglang\python\sglang\srt\mem_cache\allocator\mamba.py)）。
- Device pool 多态仍在巨大的 [`memory_pool.py`](d:\design\sglang\python\sglang\srt\mem_cache\memory_pool.py)：`ReqToTokenPool` @ [L256](d:\design\sglang\python\sglang\srt\mem_cache\memory_pool.py)、`MHATokenToKVPool` @ [L1740](d:\design\sglang\python\sglang\srt\mem_cache\memory_pool.py)、`MLATokenToKVPool` @ [L3910](d:\design\sglang\python\sglang\srt\mem_cache\memory_pool.py)、`DSATokenToKVPool` @ [L4326](d:\design\sglang\python\sglang\srt\mem_cache\memory_pool.py)、`HybridLinearKVPool` @ [L3555](d:\design\sglang\python\sglang\srt\mem_cache\memory_pool.py) 等。

## HiCache 存储后端

`HiCacheStorage(ABC)` @ [hicache_storage.py:150](d:\design\sglang\python\sglang\srt\mem_cache\hicache_storage.py)；内置 `HiCacheFile` @ [L361](d:\design\sglang\python\sglang\srt\mem_cache\hicache_storage.py)。

[`StorageBackendFactory`](d:\design\sglang\python\sglang\srt\mem_cache\storage\backend_factory.py) 注册名（[L197-L247](d:\design\sglang\python\sglang\srt\mem_cache\storage\backend_factory.py)）共 **9**：

| name | 类 |
|---|---|
| `file` | `HiCacheFile` |
| `nixl` | `HiCacheNixl` |
| `mooncake` | `MooncakeStore` |
| `hf3fs` | `HiCacheHF3FS` |
| `aibrix` | `AibrixKVCacheStorage` |
| `eic` | `EICStorage` |
| `simm` | `HiCacheSiMM` |
| `mori` | `UMBPStore`（`storage/umbp/`） |
| `shm` | `HiCacheShm` |

> synthesis: 相对旧 wiki「7 backends」，现注册表增 `file` / `shm` / `mori`（umbp），仍不含 lmcache/flexkv——后两者是 **prefix-cache 后端**（走 registry 工厂），不是 `HiCacheStorage` 插件。

## Eviction

[`evict_policy.py`](d:\design\sglang\python\sglang\srt\mem_cache\evict_policy.py)：`EvictionStrategy` + `LRU/LFU/FIFO/MRU/FILO/Priority/SLRU`（[L10-L49](d:\design\sglang\python\sglang\srt\mem_cache\evict_policy.py)）。经 `CacheInitParams.eviction_policy` ← `--radix-eviction-policy`（builder [L231](d:\design\sglang\python\sglang\srt\mem_cache\kv_cache_builder.py)）注入；主要被 `RadixCache` 家族消费。

## 多模态 Cache（平行子图）

- `MultiModalStaticCache` @ [multimodal_cache.py:76](d:\design\sglang\python\sglang\srt\mem_cache\multimodal_cache.py)
- 初始化：`build_kv_cache` 末尾 `init_mm_embedding_cache(SGLANG_VLM_CACHE_SIZE_MB * 1MiB)`（[kv_cache_builder.py:271-272](d:\design\sglang\python\sglang\srt\mem_cache\kv_cache_builder.py)；env 默认 100，[environ.py:981](d:\design\sglang\python\sglang\srt\environ.py)）；实现位于 [mm_schedule.py:21](d:\design\sglang\python\sglang\srt\managers\mm_schedule.py)（`mm_utils` 再导出）。
- EPD 全局嵌入：`EmbeddingCacheController` + Mooncake embedding store（见 [modules/multimodal.md](../modules/multimodal.md) / PD topic）——与 process 内 static cache **并行**。

## CLI / env 触发表

| 触发器 | 类型 | 默认 | 作用 |
|---|---|---|---|
| `--disable-radix-cache` | CLI | False | + chunked → Chunk*；多模态 Transformers 后端可强制 disable（[kv_cache_builder.py:181-188](d:\design\sglang\python\sglang\srt\mem_cache\kv_cache_builder.py)） |
| `SGLANG_EXPERIMENTAL_CPP_RADIX_TREE` | env | False | `RadixCacheCpp`（[environ.py:451](d:\design\sglang\python\sglang\srt\environ.py)） |
| `SGLANG_ENABLE_UNIFIED_RADIX_TREE` | env | False | 强制 Unified（[environ.py:1019](d:\design\sglang\python\sglang\srt\environ.py)）；hybrid 亦可不依赖此 env |
| `--enable-hierarchical-cache` | CLI | False | plain→`HiRadixCache`；hybrid/DSA→Unified+`init_hicache` |
| `--enable-lmcache` | CLI | False | `LMCRadixCache` |
| `--enable-flexkv` / `--radix-cache-backend` | CLI | — | FlexKV / 插件工厂 |
| `--enable-streaming-session` | CLI | False | 外裹或内嵌 `StreamingSession` |
| `--enable-session-radix-cache` | CLI | False | 要求 Unified + session_ref_tracker |
| `SGLANG_VLM_CACHE_SIZE_MB` | env | 100 | MM embedding static cache |

Decode PD 额外约束：`--disaggregation-decode-enable-radix-cache` 与 hybrid SWA/SSM **互斥**（[kv_cache_builder.py:193-206](d:\design\sglang\python\sglang\srt\mem_cache\kv_cache_builder.py)）。

## 使用方调用清单 / 跨子系统引用

### 1. 跨语言绑定
- `RadixCacheCpp` → C++ 树 [`mem_cache/cpp_radix_tree/`](d:\design\sglang\python\sglang\srt\mem_cache\cpp_radix_tree)（`tree_v2*.cpp/h` + pybind `radix_tree.py`）；工厂仅 env 开启（[registry.py:97-102](d:\design\sglang\python\sglang\srt\mem_cache\registry.py)）。
- `HiMambaRadixCache` pybind：在 `d:\design\sglang\python\sglang\srt\` 全树 grep 0 命中（类已删除）。

### 2. 协作伙伴跨子系统
| 伙伴 | 关系 | 证据 |
|---|---|---|
| `Scheduler` | 持有 `tree_cache`；session_controller / schedule_batch 消费 | [scheduler.py:516-550](d:\design\sglang\python\sglang\srt\managers\scheduler.py)、[L1137](d:\design\sglang\python\sglang\srt\managers\scheduler.py) |
| `TpModelWorker` / `ModelRunner` | 提供 memory pool；HiCache layer counter | builder + [registry.py:126-128](d:\design\sglang\python\sglang\srt\mem_cache\registry.py) |
| `kv_canary` | `attach_radix_cache(tree_cache)` | [scheduler.py:553-555](d:\design\sglang\python\sglang\srt\managers\scheduler.py)；详 [modules/kv_canary.md](../modules/kv_canary.md) |
| `session.StreamingSession` / `SessionController` | wrap 或内嵌；session API | [registry.py:223-231](d:\design\sglang\python\sglang\srt\mem_cache\registry.py)；[modules/session.md](../modules/session.md) |
| `platforms` | Unified/MLX 路径、`current_platform` | [registry.py:104](d:\design\sglang\python\sglang\srt\mem_cache\registry.py)、[platforms.md](../modules/platforms.md) |
| Mooncake / NIXL / … | HiCache storage | [backend_factory.py:197+](d:\design\sglang\python\sglang\srt\mem_cache\storage\backend_factory.py) |

### 3. 配置 / IPC 共享字段
- `enable_hierarchical_cache` / `disable_radix_cache` / `radix_cache_backend` / `enable_lmcache` / `enable_flexkv` / `enable_streaming_session` / `enable_session_radix_cache`：在 `server_args` + `arg_groups`（PD/HiSparse hooks）多处读写（srt 全树各数十命中）。
- PD decode radix：`disaggregation_decode_enable_radix_cache` 校验见 builder。

### 4. 测试覆盖反查
- `tests/` 顶层：在仓库对 `RadixCache|HiRadixCache|UnifiedRadixCache` 的独立 test 目录命名扫描弱；**模块内**存在 storage 单测：[`storage/simm/test_simm.py`](d:\design\sglang\python\sglang\srt\mem_cache\storage\simm\test_simm.py)、[`storage/mooncake_store/test_mooncake_store.py`](d:\design\sglang\python\sglang\srt\mem_cache\storage\mooncake_store\test_mooncake_store.py)、[`storage/aibrix_kvcache/unit_test.py`](d:\design\sglang\python\sglang\srt\mem_cache\storage\aibrix_kvcache\unit_test.py)。
- 顶层 `test/` / `tests/` 以 `*radix*`/`*hicache*` 文件名：本环境 grep 命中有限 → 主体验证依赖 CI 集成测（[!todo] VERIFY 完整 test tree 布局）。

### 5. doc / config 反查
- 产品文档：[`docs/docs/advanced_features/hicache_design.mdx`](d:\design\sglang\docs\docs\advanced_features\hicache_design.mdx)、[`session_radix_cache.mdx`](d:\design\sglang\docs\docs\advanced_features\session_radix_cache.mdx)、[`hicache_storage_runtime_attach_detach.mdx`](d:\design\sglang\docs\docs\advanced_features\hicache_storage_runtime_attach_detach.mdx)、[`pd_disaggregation.mdx`](d:\design\sglang\docs\docs\advanced_features\pd_disaggregation.mdx) 等。

## Notes / Caveats

> ~~[!todo] VERIFY: pin 从 `34fef07a` → `06f32bab` 后本页未深 verify~~ **RESOLVED 2026-08-10**：本轮 re-ingest 已按 `kv_cache_builder`/`registry` 重写。

> [!warning] CONTRADICTION: [modules/mem_cache.md](../modules/mem_cache.md) 仍写 62 文件、`init_cache_with_memory_pool`、7 HiCache 后端、`allocator.py` 单体——相对 HEAD 全部 stale。**本页为 prefix/KV 工厂维度最新事实源**；mem_cache module 页待单独 re-ingest（已 `status: stale`）。

> [!todo] VERIFY: `SWARadixCache` / `MambaRadixCache` 是否仍被测试/外部插件直接构造（生产工厂 0 命中）；若仅历史遗留可在下一轮标注 deprecated。

> [!todo] VERIFY: FlexKV 与 hierarchical/lmcache 的互斥断言完整集合（`server_args` 内多处 assert，未逐条展开）。

> [!todo] VERIFY: `UnifiedTreeCore` C++/python backend 切换（`SGLANG_UNIFIED_RADIX_TREE_CORE_BACKEND`）行为与性能边界。

## See also

- [sglang/modules/mem_cache.md](../modules/mem_cache.md) — 模块全景（**stale@06f32bab**，116 `.py`）
- [sglang/modules/kv_canary.md](../modules/kv_canary.md) — KV 完整性 canary 与 radix walk
- [sglang/modules/session.md](../modules/session.md) — `StreamingSession` / `SessionController`
- [sglang/modules/multimodal.md](../modules/multimodal.md) — MM embedding cache / EPD
- [sglang/entities/Scheduler.md](../entities/Scheduler.md) — `build_kv_cache` 调用点
- [sglang/entities/TpModelWorker.md](../entities/TpModelWorker.md) — memory pool 来源
- [sglang/topics/pd-disaggregation.md](pd-disaggregation.md) — decode radix / KV transfer 交叉
- [comparison/topics/kv-cache.md](../../comparison/topics/kv-cache.md)
- [comparison/topics/prefix-cache.md](../../comparison/topics/prefix-cache.md)
