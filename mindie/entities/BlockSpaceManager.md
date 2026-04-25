---
type: entity
project: mindie
status: verified
confidence: high
verified_against: 2026-04-18
sources:
  - d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h
  - d:\design\MindIE-LLM\src\block_manager\self_attn_block_manager.h
  - d:\design\MindIE-LLM\src\block_manager\self_attn_block_manager.cpp
  - d:\design\MindIE-LLM\src\block_manager\lwd_self_attn_block_manager.h
  - d:\design\MindIE-LLM\src\block_manager\lwd_self_attn_block_manager.cpp
  - d:\design\MindIE-LLM\src\block_manager\request_single_block_manager.h
  - d:\design\MindIE-LLM\src\block_manager\request_single_block_manager.cpp
  - d:\design\MindIE-LLM\src\block_manager\block_obj.h
  - d:\design\MindIE-LLM\src\block_manager\block_table.h
  - d:\design\MindIE-LLM\src\block_manager\block_table.cpp
  - d:\design\MindIE-LLM\src\scheduler\scheduler.h
  - d:\design\MindIE-LLM\src\scheduler\scheduler.cpp
  - d:\design\MindIE-LLM\src\scheduler\policy\policy_helper.cpp
  - d:\design\MindIE-LLM\src\include\config\config_info.h
  - d:\design\MindIE-LLM\src\engine\llm_engine.cpp
related:
  - mindie/entities/BatchScheduler.md
  - mindie/entities/LlmEngine.md
  - mindie/topics/prefix-cache.md
  - comparison/topics/kv-cache.md
  - comparison/topics/pd-disaggregation.md
  - comparison/topics/distributed.md
  - vllm/entities/KVCacheManager.md
---

# `BlockSpaceManager` (MindIE C++ KV block 抽象)

## Summary

`BlockSpaceManager` 是 MindIE-LLM C++ 侧 **KV cache 物理块（block）空间管理**的抽象基类：为自回归序列分配/追加 NPU（及可选 CPU）上的块、支持 **swap**、**prefix cache 统计**、**多 rank（SP/CP）块视图**，并在 **layerwise 边云 PD** 场景下通过扩展方法暴露 **云端** 侧的块元数据。工厂为 `BlockManagerFactory::CreateBlockSpaceManager`（[block_manager_interface.h:191-195](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h)），类型枚举 `BlockManagerType`（[block_manager_interface.h:26-32](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h)）。

> [!warning] CONTRADICTION（与 wiki 早期"5 个实现"表述）：`BlockManagerType` **枚举有 5 项**，但 `CreateBlockSpaceManager` 的 `switch` **仅实例化 2 种**（`SELFATTNBLOCKMANAGER`、`LWDSELFATTNBLOCKMANAGER`），其余枚举值在工厂中 **未实现分支**（[self_attn_block_manager.cpp:66-74](d:\design\MindIE-LLM\src\block_manager\self_attn_block_manager.cpp)）。源码中另有 **`RequestSingleBlockManager`** 具体类（[request_single_block_manager.h:23](d:\design\MindIE-LLM\src\block_manager\request_single_block_manager.h)），检索 **未见** `CompositeBlockManager` / `RequestSlidingWindowBlockManager` 类名。因此：**"5 个实现"应理解为"5 个枚举槽位 / 规划中的类型"更准确；以源码为准，当前工厂可创建的实现为 2 种，另 1 个独立实现类未接工厂。** → 已同步更新 [comparison/topics/kv-cache.md](../../comparison/topics/kv-cache.md) 与 [dimensions.md §dim-kv](../../comparison/dimensions.md)。

## Sources

### `block_manager/` 目录（Glob：共 34 个 `.h`/`.cpp`）

`self_attn_block_manager.{h,cpp}`、`lwd_self_attn_block_manager.{h,cpp}`、`request_single_block_manager.{h,cpp}`、`block_table.{h,cpp}`、`block_obj.h`、`block_allocator.h`、`cpu_npu_block_allocator.{h,cpp}`、`device_aware_block_allocator.h`、`hashless_block_*`、`prefix_cache_block*`、`ref_counter*`、`block_tracker.*`、`lru_evictor.*`、`evictor.h`、`copy_on_write_tracker.*`、`hit_rate_calculator.*`、`obj_pool.h` 等，根目录：[d:\design\MindIE-LLM\src\block_manager\](d:\design\MindIE-LLM\src\block_manager)。

### 接口与类型别名

- 抽象类与配置：[block_manager_interface.h](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h)
- `BlockSpaceManagerSPtr`：[block_manager_interface.h:188](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h)

## 接口定义（`BlockSpaceManager` 全部纯虚方法表）

基类中 **除默认构造/虚析构外，下列方法均为 `= 0` 纯虚**（无默认实现）（[block_manager_interface.h:91-186](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h)）。**不存在**名为 `GetBlockTable` 的虚方法；块表为 `SelfAttnBlockManager` 内部 `seqId2BlockTable_`（见下文）。

| 方法签名 | 行号 | 语义摘要 |
|----------|------|----------|
| `AllocStatus CanAllocate(const SequenceGroupSPtr &seqGroup) const` | [97](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 能否为 waiting 序列分配块 |
| `bool Allocate(const SequenceGroupSPtr &seqGroup)` | [99](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 分配块并建立 per-seq 状态 |
| `bool CanAppendSlot(const SequenceGroupSPtr &seqGroup) const` | [101](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | decode 追加 slot 前是否有足够 NPU 块 |
| `std::vector<std::pair<BlockId, BlockId>> AppendSlot(const SequenceSPtr &seq)` | [103](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 追加 slot，返回 COW 等 block 对 |
| `bool CanAppendSlotNew(const SequenceGroupSPtr &seqGroup) const` | [106](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | SP 专用 CanAppend |
| `void AppendSlotNew(const SequenceGroupSPtr &seqGroup)` | [108](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | SP 专用 Append |
| `void AppendTokenToLatestRank(SequenceId seqId, const std::vector<TokenId>& tokens)` | [110](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 向最近 rank 追加 token |
| `void Fork(SequenceSPtr &parentSeq, SequenceSPtr &childSeq)` | [112](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 父子序列共享块（fork） |
| `bool CanSwapOut(const SequenceGroupSPtr &seqGroup)` | [114](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 是否可换出到 host |
| `std::vector<std::pair<PhysicalBlockId, PhysicalBlockId>> SwapOut(const SequenceGroupSPtr &seqGroup)` | [116](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 执行 SwapOut，返回物理块映射 |
| `AllocStatus CanSwapIn(const SequenceGroupSPtr &seqGroup, size_t numLookheadSlots)` | [118](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 是否可换回 NPU |
| `std::vector<std::pair<PhysicalBlockId, PhysicalBlockId>> SwapIn(const SequenceGroupSPtr &seqGroup)` | [120](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 执行 SwapIn |
| `void Free(SequenceId seqId)` | [122](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 释放某序列占用的块 |
| `std::vector<BlockIds> GetBlockIds(SequenceId seqId) const` | [124](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 查询块 ID 列表 |
| `void GetRankedBlockIds(SequenceId seqId, std::vector<RankedBlockId> &rankedBlockIds) const` | [127](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 带 rank 的块 ID |
| `void GetRankedBlockIds(SequenceId seqId, std::vector<std::vector<BlockId>> &rankedBlockIds) const` | [129](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 按 rank 分组的块 ID |
| `std::vector<std::vector<HashValue>> GetRankedHashValues(SequenceId seqId) const` | [131](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | prefix cache 相关 hash |
| `std::vector<HashValue> GetSeqHashValues(SequenceId seqId) const` | [133](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 序列级 hash |
| `std::vector<size_t> GetTokenCountPerRank(SequenceId seqId) const` | [135](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 各 rank token 计数 |
| `size_t GetLatestAppendedRankId(SequenceId seqId) const` | [137](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 最近追加的 rank |
| `size_t GetAppendedBlockRankId(SequenceId seqId) const` | [139](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 追加块所在 rank |
| `bool IsAppendBlock(SequenceId seqId)` | [141](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 是否为 append 块语义 |
| `size_t GetNumFreeNpuBlocks() const` | [143](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 空闲 NPU 块数 |
| `size_t GetNumFreeCpuBlocks() const` | [145](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 空闲 CPU 块数 |
| `size_t GetTotalNpuBlocks() const` | [147](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | NPU 块总数 |
| `void AccessAllblocksInSeq(const SequenceSPtr &seq, float accessTime)` | [149](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 访问时间戳（LRU 等） |
| `std::vector<BlockId> GetCommonComputedBlockIds(const std::vector<SequenceSPtr> &seqs)` | [151](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 多序列公共已算块 |
| `std::vector<size_t> GetAllrankComputedBlockNum(const std::vector<SequenceSPtr> &seqs)` | [153](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 各 rank 已算块数 |
| `std::vector<BlockId> GetRemoteComputedBlockIds(...)` | [155-156](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 分布式/远程已算块 |
| `std::vector<size_t> GetAllRankRemoteComputedBlockIds(...)` | [158-159](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 远程已算块数 |
| `void MarkBlocksAsComputed()` | [161](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 标记本轮块为已计算 |
| `float GetPrefixCacheHitRate() const` | [163](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | prefix cache 命中率 |
| `bool ResetPrefixCache() const` | [165](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 重置 prefix cache |
| `size_t GetNumCachedTokens(const SequenceSPtr &seq)` | [167](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | cached token 数 |
| `size_t GetSeqNumCachedTokens(const SequenceSPtr &seq)` | [169](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 序列级 cached token |
| `void ReplaceTrailingPlaceHolder(...)` | [171-172](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 占位符替换（spec 解码等） |
| `size_t GetLocalDPRank() const` | [174](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 本地 DP rank |
| `void LwdInitCloudBlockManager(const BlockManagerConfig &lwdCloudConfig, size_t localDPRank)` | [176](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | layerwise：初始化云端子 manager |
| `void LwdGetCloudRankedBlockIds(SequenceId seqId, std::vector<std::vector<BlockId>> &rankedBlockIds) const` | [178-179](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | layerwise：云端块视图 |
| `size_t LwdGetCloudLatestAppendedRankId(SequenceId seqId) const` | [181](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 云端 latest rank |
| `size_t LwdGetCloudAppendedBlockRankId(SequenceId seqId) const` | [183](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 云端 append rank |
| `std::vector<size_t> LwdGetCloudTokenCountPerRank(SequenceId seqId) const` | [185](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 云端 per-rank token 数 |

**工厂**：`BlockManagerFactory::CreateBlockSpaceManager(BlockManagerType type, const BlockManagerConfig &config, size_t localDPRank = 0)`（[block_manager_interface.h:191-195](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h)，实现 [self_attn_block_manager.cpp:66-74](d:\design\MindIE-LLM\src\block_manager\self_attn_block_manager.cpp)）。

**配置载体 `BlockManagerConfig`**（[block_manager_interface.h:55-89](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h)）：含 `cacheBlockSize`、`cpuBlockNum`、`npuBlockNum`、`reservedBlockNum`、`speculativeSlots`、`enableCaching`、`rankSize`、`hostSize`、`enableKvPool`、`allocationMode`、`cacheType`、`subManagers`（composite 预留）、`requestBlockWindowSize` 等。

## "5 个实现"对照表（源码真相）

| 层级 | 名称 | 源码锚点 | 角色（一句话） |
|------|------|----------|----------------|
| 枚举 | `BlockManagerType` 五项 | [block_manager_interface.h:26-32](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | `SELFATTN` / `LWDSELFATTN` / `COMPOSITE` / `REQUESTSINGLE` / `REQUESTSLIDINGWINDOW` |
| 工厂可创建 | `SelfAttnBlockManager` | [self_attn_block_manager.h:35](d:\design\MindIE-LLM\src\block_manager\self_attn_block_manager.h)，[self_attn_block_manager.cpp:70](d:\design\MindIE-LLM\src\block_manager\self_attn_block_manager.cpp) | 标准自注意力 KV：prefix cache 可选、`BlockTable`+`CpuNpuBlockAllocator` |
| 工厂可创建 | `LwdSelfAttnBlockManager` | [lwd_self_attn_block_manager.h:35](d:\design\MindIE-LLM\src\block_manager\lwd_self_attn_block_manager.h)，[self_attn_block_manager.cpp:71-72](d:\design\MindIE-LLM\src\block_manager\self_attn_block_manager.cpp) | **继承** `SelfAttnBlockManager`，内含 **`SelfAttnBlockManager` 型 `lwdCloudBlockManager_`**，双边同步 Allocate/Append/Free 等（[lwd_self_attn_block_manager.cpp:29-90](d:\design\MindIE-LLM\src\block_manager\lwd_self_attn_block_manager.cpp)） |
| 独立类（未接工厂） | `RequestSingleBlockManager` | [request_single_block_manager.h:23](d:\design\MindIE-LLM\src\block_manager\request_single_block_manager.h) | **请求级单块** 复用；Swap 等标注为当前不支持（[request_single_block_manager.h:40-46](d:\design\MindIE-LLM\src\block_manager\request_single_block_manager.h)）；单元测试直接构造 |
| 未见类定义 | `COMPOSITEBLOCKMANAGER` / `REQUESTSLIDINGWINDOWBLOCKMANAGER` | 枚举 [block_manager_interface.h:29-31](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h)；`subManagers` 注释 [block_manager_interface.h:82-84](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) | 仓库内 **无** 对应 `class` 名检索结果；工厂 **未** `case` |

### 实现差异：重写哪些虚方法

- **`SelfAttnBlockManager`**：实现接口全部纯虚；对 layerwise 云端接口给出 **空默认**（`LwdInitCloudBlockManager` 等为空体，`LwdGetCloud*` 返回 0/空）（[self_attn_block_manager.h:123-132](d:\design\MindIE-LLM\src\block_manager\self_attn_block_manager.h)）。
- **`LwdSelfAttnBlockManager`**：重写 `CanAllocate`/`Allocate`/`CanAppendSlot`/`CanAppendSlotNew`/`AppendSlotNew`/`Free`/`AccessAllblocksInSeq` 及全部 `LwdGet*`（[lwd_self_attn_block_manager.h:39-61](d:\design\MindIE-LLM\src\block_manager\lwd_self_attn_block_manager.h)），逻辑上 **双边调用** 父类与 `lwdCloudBlockManager_`（[lwd_self_attn_block_manager.cpp:41-90](d:\design\MindIE-LLM\src\block_manager\lwd_self_attn_block_manager.cpp)）。
- **`RequestSingleBlockManager`**：完整 `override` 列表见头文件（[request_single_block_manager.h:27-96](d:\design\MindIE-LLM\src\block_manager\request_single_block_manager.h)），内部 **`seqId -> RequestId -> 单 BlockObj`**（[request_single_block_manager.h:98-123](d:\design\MindIE-LLM\src\block_manager\request_single_block_manager.h)），与 `SelfAttnBlockManager` 的 `BlockTable` 路径不同。

## 核心数据结构

### `Block` 概念

源码中块对象为 **`BlockObj`** 抽象类（非 `class Block`）（[block_obj.h:26-76](d:\design\MindIE-LLM\src\block_manager\block_obj.h)）：`GetBlockId`、`AppendTokenIds`、hash、`IsComputed`、`LastAccessed` 等。

### `BlockTable`

`BlockTable`（[block_table.h:25-119](d:\design\MindIE-LLM\src\block_manager\block_table.h)）持有 `blockObjs_`（第一维为 **rank**）、`blockIds_`、`blockSize_`、`blockAllocator_` 等；提供 `Allocate`/`AppendTokenIds`/`Fork`/`Free` 等。

### `SequenceId -> BlockTable` 映射

`SelfAttnBlockManager`：`std::unordered_map<SequenceId, BlockTable> seqId2BlockTable_`（[self_attn_block_manager.h:160](d:\design\MindIE-LLM\src\block_manager\self_attn_block_manager.h)）。**无** `SequenceBlockMap` 类型名。

### `BlockSpaceManagerSPtr`

`using BlockSpaceManagerSPtr = std::shared_ptr<BlockSpaceManager>`（[block_manager_interface.h:188](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h)）。

## 关键 API（与调度/PD 强相关）

接口 **无** `RegisterPrefixCache`；prefix 通过 `enableCaching` 配置 allocator（`SelfAttnBlockManager` 构造 [self_attn_block_manager.cpp:44-50](d:\design\MindIE-LLM\src\block_manager\self_attn_block_manager.cpp)）及 `GetPrefixCacheHitRate`/`ResetPrefixCache`（[block_manager_interface.h:163-165](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h)）。

调度侧集中封装见 `PolicyHelper`（[policy_helper.cpp](d:\design\MindIE-LLM\src\scheduler\policy\policy_helper.cpp)）：`Allocate`、`CanAppendSlot`/`CanAppendSlotNew`、`AppendSlot`/`AppendSlotNew`、`CanSwapOut`/`SwapOut`、`CanSwapIn`/`SwapIn`、`Fork`、`Free` 等（grep 锚点如 `blockManager_->Allocate` [policy_helper.cpp:132](d:\design\MindIE-LLM\src\scheduler\policy\policy_helper.cpp) 等）。

## 与 `Scheduler` 的协作

- **成员**：`BlockSpaceManagerSPtr blockManager_`（[scheduler.h:247](d:\design\MindIE-LLM\src\scheduler\scheduler.h)）。
- **构造时创建**：`layerwiseDisaggregated && sp*cp>1` 时创建 `LWDSELFATTNBLOCKMANAGER` 并 `LwdInitCloudBlockManager`（[scheduler.cpp:101-115](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)）；否则 `CreateBlockManagerFromSchedulerConfig`（多数路径为 `SELFATTNBLOCKMANAGER`）（[scheduler.cpp:39-118](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)）。
- **调度决策**：`PolicyHelper` 在 prefill/decode 路径调用 `CanAllocate`/`Allocate`、`CanAppend*`、`Append*`（见上）。
- **Preempt / swap**：`CanSwapOut`/`SwapOut`、`CanSwapIn`/`SwapIn`（policy_helper）。
- **KV 释放**：`ReleaseKvPulledBlocks` 内 `blockManager_->Free(seqId)`（[scheduler.cpp:387](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)）。
- **指标**：`GetNumFreeNpuBlocks`/`GetNumFreeCpuBlocks`、`MarkBlocksAsComputed`（如 [scheduler.cpp:217-218, 342](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)）。
- **元数据填充**：`GetRankedBlockIds`、`GetLatestAppendedRankId`、`LwdGetCloud*` 等（[scheduler.cpp:979-1053](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp) 段）。

## Layerwise SelfAttn 独有路径

- **类注释**：边云特性专用，**暂不支持** prefix caching / copy-on-write 等（与基类能力对比的 **产品说明** 层表述）（[lwd_self_attn_block_manager.h:26-34](d:\design\MindIE-LLM\src\block_manager\lwd_self_attn_block_manager.h)）。
- **云端子管理器**：`LwdInitCloudBlockManager` 内 `std::make_shared<SelfAttnBlockManager>(lwdCloudConfig, ...)`（[lwd_self_attn_block_manager.cpp:29-32](d:\design\MindIE-LLM\src\block_manager\lwd_self_attn_block_manager.cpp)）。
- **与普通 SelfAttn 差异**：同一序列在 **边、云两套** `SelfAttnBlockManager` 上同时 `Allocate`/`Append`/`Free`（[lwd_self_attn_block_manager.cpp:53-90](d:\design\MindIE-LLM\src\block_manager\lwd_self_attn_block_manager.cpp)）；`CanAllocate` 取两边 **最悲观** `AllocStatus`（[lwd_self_attn_block_manager.cpp:53-62](d:\design\MindIE-LLM\src\block_manager\lwd_self_attn_block_manager.cpp)）。
- **与 layerwise PD 关联**：`Scheduler` 在 `layerwiseDisaggregated && spSize*cpSize>1` 选用该类型并初始化云端块数配置（见 scheduler 构造锚点）；云端块数来自 `SchedulerConfig::lwdCloudNpuBlockNum`（见配置依赖）。

## PD 分离接入

- **Swap 与 host-device**：接口层 `SwapIn`/`SwapOut` 返回 `PhysicalBlockId` 对（[block_manager_interface.h:114-120](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h)）；调度通过 `PolicyHelper` 调用。
- **`ScheduleTransfer`**：[scheduler.cpp:397-441](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp) — P 角色仅 `ReleaseKvPulledBlocks` 内 `Free`；D 角色用 `blockManager_->GetNumFreeNpuBlocks()` 计算 `freeTokenNum` 与 `maxTransferTokens`（[scheduler.cpp:404-411](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)），再经 `KVTransferSchedulePolicy`（构造持有 `blockManager_`，[pdds_policy.cpp:67-77](d:\design\MindIE-LLM\src\scheduler\policy\pdds_policy.cpp)）选型；**不在此函数内直接 SwapIn**。
- **引擎循环**：`LlmEngine::ScheduleExecTransfer` 调 `ScheduleTransfer`（[llm_engine.cpp:698-711](d:\design\MindIE-LLM\src\engine\llm_engine.cpp)）。
- **Mooncake / LLMDataDist**：当前 MindIE-LLM C++ 树内 **未** 检索到 `Mooncake`/`LLMDataDist` 与 `BlockSpaceManager` 的交叉引用（与 [SeparateDeploymentEngine.md](SeparateDeploymentEngine.md) 一致——它们在 Python 端是独立 concern）。

## 跨项目对照（synthesis）

> 非 MindIE 源码推导，仅结构对照。

| 系统 | 对应概念 | 抽象层次 | 锚点 |
|---|---|---|---|
| **MindIE** | `BlockSpaceManager` 单一多方法纯虚表 | C++ 编译期纯虚 + 共享指针工厂 | [block_manager_interface.h:91-186](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) |
| **vLLM** | `KVCacheManager` 类族 + `BlockPool` + `BlockHashToBlockMap` | Python ABC + dispatcher | [vllm/entities/KVCacheManager.md](../../vllm/entities/KVCacheManager.md) |
| **SGLang** | `BasePrefixCache` + `BaseTokenToKVPoolAllocator` 双层 | Python ABC + RadixCache 等多实现 | [sglang/python/sglang/srt/mem_cache/base_prefix_cache.py](d:\design\sglang\python\sglang\srt\mem_cache\base_prefix_cache.py) |

> synthesis: **MindIE 把"分配 + swap + prefix 统计 + 分布式块视图 + layerwise 扩展" 5 类语义全部塞进单一抽象**，方法数高（~40 个虚函数）；vLLM/SGLang 拆成多个抽象（KVCache vs Prefix vs Pool）。MindIE 的 swap/PD 与 C++ `Scheduler`/`PolicyHelper` **强耦合**——这是 C++ 编译期抽象的双刃剑。

## 配置依赖（`SchedulerConfig` / `BlockManagerConfig`）

- `SchedulerConfig` 块相关字段：`cpuBlockNum`、`npuBlockNum`、`lwdCloudNpuBlockNum`、`cacheBlockSize`、`kvCacheDescs`（[config_info.h:442-472](d:\design\MindIE-LLM\src\include\config\config_info.h)）；`layerwiseDisaggregated`（[config_info.h:515](d:\design\MindIE-LLM\src\include\config\config_info.h) 等）。
- `Scheduler` 构造将 `cacheBlockSize`、`cpuBlockNum`、`npuBlockNum` 填入 `BlockManagerConfig`（[scheduler.cpp:89-99](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)）；layerwise 云端使用 `lwdCloudNpuBlockNum`（[scheduler.cpp:104-115](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)）。
- `CreateBlockManagerFromSchedulerConfig`：`kvCacheDescs` 多项或 `sp*cp>1` 且多 desc 会 **抛错**（[scheduler.cpp:41-79](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)）。

## Notes / Caveats

> [!warning] CONTRADICTION: 类名是 **`BlockSpaceManager`**，头文件保护宏仍为 `INTERFACE_H`（[block_manager_interface.h:13-14](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h)），与文件名（`block_manager_interface.h`）略有历史痕迹。
> [!warning] CONTRADICTION: 之前 wiki "5 实现"表述 vs 实际工厂仅产 2 种 + 1 独立未接工厂（详 §"5 个实现"对照表）。
> [!todo] VERIFY: `Composite` / `RequestSlidingWindow` 枚举值的设计意图（dead enum / 未实现 / 计划项？）—— 需确认产品 roadmap。
> [!todo] VERIFY: `SequenceBlockMap` 类型名是否在更深的实现细节里（仅当前 grep 范围未见）。

## See also

- [BatchScheduler.md](BatchScheduler.md)（`blockManager_` 持有者）
- [LlmEngine.md](LlmEngine.md)（`ScheduleExecTransfer` 主循环步骤）
- [SeparateDeploymentEngine.md](SeparateDeploymentEngine.md)（PD Python 端，与本类协作模式）
- [topics/prefix-cache.md](../topics/prefix-cache.md)（`enableCaching=true` 时的 hash table + LRU 实现 + PD 远端 KV pool 命中通路）
- [comparison/topics/kv-cache.md](../../comparison/topics/kv-cache.md)
- [comparison/topics/pd-disaggregation.md](../../comparison/topics/pd-disaggregation.md)
- [vllm/entities/KVCacheManager.md](../../vllm/entities/KVCacheManager.md)（vLLM 对应物）
