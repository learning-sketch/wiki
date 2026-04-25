---
type: topic
project: mindie
status: verified
confidence: high
verified_against: 2026-04-18
sources:
  - d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\prefix_cache\prefix_cache_plugin.py
  - d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\prefix_cache\prefix_cache_preprocess.py
  - d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\prefix_cache\__init__.py
  - d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_utils.py
  - d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py
  - d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager_lwd.py
  - d:\design\MindIE-LLM\mindie_llm\text_generator\generator.py
  - d:\design\MindIE-LLM\mindie_llm\text_generator\utils\input_metadata.py
  - d:\design\MindIE-LLM\mindie_llm\connector\common\input_metadata_builder.py
  - d:\design\MindIE-LLM\src\block_manager\prefix_cache_block.h
  - d:\design\MindIE-LLM\src\block_manager\prefix_cache_block.cpp
  - d:\design\MindIE-LLM\src\block_manager\prefix_cache_block_allocator.h
  - d:\design\MindIE-LLM\src\block_manager\prefix_cache_block_allocator.cpp
  - d:\design\MindIE-LLM\src\block_manager\self_attn_block_manager.cpp
  - d:\design\MindIE-LLM\src\block_manager\cpu_npu_block_allocator.cpp
  - d:\design\MindIE-LLM\src\scheduler\scheduler.cpp
  - d:\design\MindIE-LLM\src\scheduler\policy\policy_helper.cpp
  - d:\design\MindIE-LLM\src\engine\construct_execute_request.cpp
  - d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h
  - d:\design\MindIE-LLM\src\include\config\config_info.h
  - d:\design\MindIE-LLM\src\config_manager\schedule_config.cpp
  - d:\design\MindIE-LLM\src\llm_manager_v2\impl\llm_manager_impl.cpp
  - d:\design\MindIE-LLM\proto\model_execute_data.proto
  - d:\design\MindIE-LLM\docs\zh\user_guide\feature\prefix_cache.md
  - d:\design\MindIE-LLM\docs\zh\user_guide\feature\kv_cache_pool.md
related:
  - mindie/entities/BlockSpaceManager.md
  - mindie/entities/PluginManager.md
  - mindie/entities/Generator.md
  - mindie/topics/kv-cache.md
  - mindie/topics/connector.md
  - comparison/topics/prefix-cache.md
  - comparison/topics/kv-cache.md
  - comparison/dimensions.md
---

# Prefix Cache（MindIE-LLM）

## Summary

MindIE-LLM 的 **Prefix Cache** 是 **C++ 块管理器 + Python 插件 + 可选 KV Cache 池化** 三段协作的 token 块级前缀复用机制：

- **匹配 / 复用** 在 C++ 侧由 `PrefixCacheBlockAllocator` 用 **`unordered_map<HashValue, BlockId>`**（**hash table**，**不是** trie / radix tree）+ LRU `Evictor` 完成，块的前缀 hash 由 `PrefixCachingBlockObj::PrefixHash()` 在块**写满且无占位符**时增量计算（[prefix_cache_block.cpp:90-121](d:\design\MindIE-LLM\src\block_manager\prefix_cache_block.cpp)，[prefix_cache_block_allocator.h:116](d:\design\MindIE-LLM\src\block_manager\prefix_cache_block_allocator.h)）。
- **调度路径** `Scheduler::CollectComputedBlocksInfo` 调 `BlockSpaceManager::GetCommonComputedBlockIds` 得到本地命中块；若 `enableKvPool` 则再调 `GetRemoteComputedBlockIds` 把池化命中（**MemPool LookUp**）拼到 `remote_computed_blocks`，两者通过 protobuf `computed_block_lens` / `remote_computed_block_lens` 字段（[model_execute_data.proto:145, 163](d:\design\MindIE-LLM\proto\model_execute_data.proto)）下发到 Python 端。
- **Python 插件 `PrefixCachePlugin`** 在 prefill 时把命中块裁出 input_ids（`PrefixCachePreprocess.update_infer_input`），并在写回阶段通过 `MemPool.put`（同步 / 异步线程）把新算的 KV 块上传到池化后端；插件本身**不** maintain 任何 prefix tree，只做"输入裁剪 + 命中率统计 + KV 池读写"。

> synthesis: MindIE 的 prefix cache 是**两个一半**的设计——**块级 hash 复用**留在 C++ 与所有 KV 块管理共生（`enableCaching=true` 切换 `BlockAllocatorType::PREFIXCACHING`，[self_attn_block_manager.cpp:45](d:\design\MindIE-LLM\src\block_manager\self_attn_block_manager.cpp)），**KV pool 跨实例 / 跨 session 复用**才挂在 Python plugin 路径上。这与 vLLM 把整个 prefix cache 集成进 `KVCacheManager` / SGLang 用 `RadixCache` 都不同。

## Sources

| 资源 | 锚点 |
|------|------|
| Python 插件主体 | [prefix_cache_plugin.py](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\prefix_cache\prefix_cache_plugin.py)（L1-409） |
| Python 输入预处理 | [prefix_cache_preprocess.py](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\prefix_cache\prefix_cache_preprocess.py)（L1-242） |
| C++ 块对象（前缀 hash） | [prefix_cache_block.h](d:\design\MindIE-LLM\src\block_manager\prefix_cache_block.h)，[prefix_cache_block.cpp](d:\design\MindIE-LLM\src\block_manager\prefix_cache_block.cpp) |
| C++ 块分配器（hash table + LRU） | [prefix_cache_block_allocator.h](d:\design\MindIE-LLM\src\block_manager\prefix_cache_block_allocator.h)，[prefix_cache_block_allocator.cpp](d:\design\MindIE-LLM\src\block_manager\prefix_cache_block_allocator.cpp) |
| C++ 工厂选择 / pybind 池化集成 | [self_attn_block_manager.cpp:43-63](d:\design\MindIE-LLM\src\block_manager\self_attn_block_manager.cpp)，[cpu_npu_block_allocator.cpp:39-59](d:\design\MindIE-LLM\src\block_manager\cpu_npu_block_allocator.cpp) |
| 调度器查命中块 | [scheduler.cpp:1242-1274](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)，[policy_helper.cpp:60-87](d:\design\MindIE-LLM\src\scheduler\policy\policy_helper.cpp) |
| Mark computed | [scheduler.cpp:342](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)，[self_attn_block_manager.cpp:319](d:\design\MindIE-LLM\src\block_manager\self_attn_block_manager.cpp) |
| 协议字段（C++→Python） | [model_execute_data.proto:145, 163](d:\design\MindIE-LLM\proto\model_execute_data.proto)，[construct_execute_request.cpp:90-105](d:\design\MindIE-LLM\src\engine\construct_execute_request.cpp)，[input_metadata_builder.py:823-860](d:\design\MindIE-LLM\mindie_llm\connector\common\input_metadata_builder.py) |
| 配置入口 | [config_info.h:219, 383, 502](d:\design\MindIE-LLM\src\include\config\config_info.h)，[schedule_config.cpp:38-42, 188-198](d:\design\MindIE-LLM\src\config_manager\schedule_config.cpp)，[llm_manager_impl.cpp:687-689](d:\design\MindIE-LLM\src\llm_manager_v2\impl\llm_manager_impl.cpp) |
| 校验器 | [plugin_utils.py:9, 81-89, 127-134](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_utils.py) |
| Plugin 编排 | [plugin_manager.py:201-202, 245-252, 317-321, 495-502, 803-807, 888-902](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py)，[plugin_manager_lwd.py:95-104, 221-228](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager_lwd.py) |
| 产品文档 | [docs/zh/user_guide/feature/prefix_cache.md](d:\design\MindIE-LLM\docs\zh\user_guide\feature\prefix_cache.md)，[docs/zh/user_guide/feature/kv_cache_pool.md](d:\design\MindIE-LLM\docs\zh\user_guide\feature\kv_cache_pool.md) |

## Architecture / Data flow

```
                                +---------------------------+
   ScheduleConfig.enablePrefix  |   Server / llm_manager_v2 |
   ←  plugin_params="prefix_cache"  GetPluginEnable("prefix_cache")
                                +-------------+-------------+
                                              |
                            engineConfig.enablePrefixCache=true
                                              |
                                              v
   +----------+ enablePrefixCache  +----------------------+
   | Scheduler|------------------> | BlockManagerConfig   |
   +----+-----+                    | .enableCaching=true  |
        |                          +----------+-----------+
        |                                     |
        |                                     v
        |               +-----------------------------------------+
        |               | SelfAttnBlockManager                    |
        |               | allocatorType = PREFIXCACHING           |
        |               | → CpuNpuBlockAllocator                  |
        |               |   ├─ NPU: PrefixCacheBlockAllocator     |
        |               |   └─ CPU: PrefixCacheBlockAllocator     |
        |               | (hash table cachedBlocks_ + LRU evictor)|
        |               +-----------------------------------------+
        |                                     ^                  ^
        |   每轮 schedule 末尾                 |                  | (可选 enableKvPool)
        +---blockManager_->MarkBlocksAsComputed                  |
        |                                                        v
        |   PolicyHelper.GetNumComputeNewUnCached…              MemPool (Python)
        |       ↳ blockManager_->GetSeqNumCachedTokens          ↳ pybind from C++
        |                                                          self_attn_block_manager.cpp
        |   GenerateSequenceGroupMetadata
        |       ↳ GetCommonComputedBlockIds   →  computedLens_
        |       ↳ GetRemoteComputedBlockIds   →  remoteComputedLens_   (查 MemPool)
        v
   protobuf  computed_block_lens / remote_computed_block_lens
        |
        v
   Python  input_metadata.computed_blocks / remote_computed_blocks
        |
        v
   PluginManager.model_inputs_update_manager
        ↳ PrefixCachePlugin.model_inputs_update
              ↳ get_prefix_kvcache_from_mempool   (MemPool.get)
              ↳ PrefixCachePreprocess.update_infer_input
                    （把 input_ids/positions/slots 砍掉缓存部分，转 PA prefill）

   写回（本轮 prefill 完成后）：
        SYNC_WRITE  →  PrefixCachePlugin.put_prefix_kvcache_to_mempool
        ASYNC_WRITE → put_input_queue → _put_prefix_kvcache_thread → put_task_queue
                       →  wait_event(pipe_key) →  m_store.put(prefix_keys, kvcache_tensors)
```

## C++ 块层：hash table + LRU，**不是** trie

### 数据结构

`PrefixCacheBlockAllocator` 内部**就一个 hash 映射 + 一个 LRU evictor**：

| 字段 | 类型 | 作用 | 锚点 |
|---|---|---|---|
| `cachedBlocks_` | `std::unordered_map<HashValue, BlockId>` | "前缀 hash → 已 promote 块 ID" 主索引 | [prefix_cache_block_allocator.h:116](d:\design\MindIE-LLM\src\block_manager\prefix_cache_block_allocator.h) |
| `freeBlockIndices_` | `std::deque<BlockId>` | 空闲块队列 | [prefix_cache_block_allocator.h:107](d:\design\MindIE-LLM\src\block_manager\prefix_cache_block_allocator.h) |
| `evictor_` | `EvictorPtr`（`MakeEvictor(EvictionPolicy::LRU)`） | 引用计数归零的已缓存块进 LRU 池供再用 / 老化驱逐 | [prefix_cache_block_allocator.cpp:38-41, 257-258](d:\design\MindIE-LLM\src\block_manager\prefix_cache_block_allocator.cpp) |
| `refCounter_` | `RefCounterProtocolSPtr` | 块共享计数；`Decrease==0` 后块归 LRU 而非立即 free | [prefix_cache_block_allocator.cpp:32-35, 234-261](d:\design\MindIE-LLM\src\block_manager\prefix_cache_block_allocator.cpp) |
| `cowTracker_` | `CopyOnWriteTracker` | append 时若不可写就 CoW（多序列共享场景） | [prefix_cache_block_allocator.cpp:312-330](d:\design\MindIE-LLM\src\block_manager\prefix_cache_block_allocator.cpp) |
| `blockComputedAttr_` | `BlockComputedAttr` | 块是否标记为已计算（仅已计算块算"命中"） | [prefix_cache_block_allocator.cpp:481-497](d:\design\MindIE-LLM\src\block_manager\prefix_cache_block_allocator.cpp) |
| `hitRateCalculator_` | `std::shared_ptr<HitRateCalculator>` | `AllocateImmutableBlock` 时 `Record(hit/miss)` | [prefix_cache_block_allocator.cpp:95, 102, 437-442](d:\design\MindIE-LLM\src\block_manager\prefix_cache_block_allocator.cpp) |

> synthesis: **不存在 trie / radix tree**——`PrefixCacheBlockAllocator` 用的是 **hash table 单点查询**，前缀关系靠"块 → prevBlock 链表 + 链式 hash 组合"维持（[prefix_cache_block.cpp:101-118](d:\design\MindIE-LLM\src\block_manager\prefix_cache_block.cpp)）。这与 SGLang 的 `RadixCache`（[radix_cache.py:285](d:\design\sglang\python\sglang\srt\mem_cache\radix_cache.py)）和 vLLM 的 `BlockHashToBlockMap`（hash table）形成对比：MindIE ≈ vLLM 风格，**SGLang 是唯一用 trie 的**。

### Hash 算法

C++ 端 `PrefixCachingBlockObj::PrefixHash()`（[prefix_cache_block.cpp:90-121](d:\design\MindIE-LLM\src\block_manager\prefix_cache_block.cpp)）：

1. **触发时机**：`IsReadyToCalcPrefixHash()` 要求块**已写满**（`IsFull`）且**最后一个 token 不是 placeholder**（避免 spec decoding 中间状态污染缓存）（[prefix_cache_block.cpp:75-88](d:\design\MindIE-LLM\src\block_manager\prefix_cache_block.cpp)）。
2. **链式组合**：`seed = HashCombine(seed, prevBlock->PrefixHash())`，再对本块所有 token id `HashCombine`，最后 `HashCombine(seed, extraHash_)`。
3. **缓存复用**：算完后写 `cachedPrefixHash_`，下次直接返回（避免 O(块数) 重算）。

Python 端 `PrefixCachePlugin.hash_block` / `cpp_style_hash` / `hash_combine`（[prefix_cache_plugin.py:30-48, 228-235](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\prefix_cache\prefix_cache_plugin.py)）**显式模拟 C++ 的 boost-style `hash_combine`**：

```python
seed ^= (cpp_style_hash(token_id) + 0x9e3779b97f4a7c15
         + (seed << HASH_SHIFT_LEFT) + (seed >> HASH_SHIFT_RIGHT))
seed = 1 if seed == INVALID_HASH_VALUE else seed
seed = seed % 2**64
```

> [!warning] CONTRADICTION（潜在）：C++ 的 `HashCombine` 与 Python 的 `hash_combine` 必须**逐位等价**，否则 KV pool 中由 P 节点写入的 key 在 D 节点上查不到。Python 侧已加注释 `Simulate the default hash algorithm in C++`（[prefix_cache_plugin.py:30-42](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\prefix_cache\prefix_cache_plugin.py)），但 **C++ `HashCombine` 实现在 [math_utils.h](d:\design\MindIE-LLM\src\block_manager\math_utils.h) 内**，建议在改任何一侧时同步审。

### Allocate / Promote / Free 三段流程

| 操作 | 行为 | 锚点 |
|---|---|---|
| `AllocateMutableBlock` | 从 `freeBlockIndices_` 或 LRU 取一块；新块还没计算 hash 不进 `cachedBlocks_` | [prefix_cache_block_allocator.cpp:46-59, 148-161](d:\design\MindIE-LLM\src\block_manager\prefix_cache_block_allocator.cpp) |
| `AllocateImmutableBlock` | 算 hash → 在 `cachedBlocks_` 命中：复用块 ID + 引用计数 +1；未命中：降级为 mutable block；都过 `hitRateCalculator_->Record(true/false)` | [prefix_cache_block_allocator.cpp:75-107](d:\design\MindIE-LLM\src\block_manager\prefix_cache_block_allocator.cpp) |
| `PromoteToImmutableBlock` | append 写满后调用：cache miss 则插入 `cachedBlocks_[hash]=blockId` + `touchedBlocks_`；hit 则**释放新块、改用 cached 块 ID** | [prefix_cache_block_allocator.cpp:263-287](d:\design\MindIE-LLM\src\block_manager\prefix_cache_block_allocator.cpp) |
| `FreeBlockId` | 已 cached 的块走 `DecrRefCountCacheBlock` →ref==0 时**进 evictor**（不立即归 free pool）；未 cached 直接归 free pool | [prefix_cache_block_allocator.cpp:163-178, 234-261](d:\design\MindIE-LLM\src\block_manager\prefix_cache_block_allocator.cpp) |
| `MayBeAllocateEvictedBlockId` | free pool 空时 `evictor_->Evict()` 取一块，从 `cachedBlocks_` 抹掉对应 hash | [prefix_cache_block_allocator.cpp:122-146](d:\design\MindIE-LLM\src\block_manager\prefix_cache_block_allocator.cpp) |
| `MarkBlocksAsComputed` | 把 `touchedBlocks_` 中的块标记为 computed（每轮 schedule 末尾全局调用一次） | [prefix_cache_block_allocator.cpp:304-310](d:\design\MindIE-LLM\src\block_manager\prefix_cache_block_allocator.cpp)，触发 [scheduler.cpp:342](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp) |
| `GetCommonComputedBlockIds` | 多序列前缀求最长公共块 ID 序列（朴素逐位比对） | [prefix_cache_block_allocator.cpp:332-361](d:\design\MindIE-LLM\src\block_manager\prefix_cache_block_allocator.cpp) |
| `FindCachedBlocksPrefix` | 逐 hash 查 `cachedBlocks_`，遇 miss 立即 break（最长前缀语义） | [prefix_cache_block_allocator.cpp:424-435](d:\design\MindIE-LLM\src\block_manager\prefix_cache_block_allocator.cpp) |
| `ResetPrefixCache` | RLHF / benchmark 用：**仅在所有块空闲时**重建 `freeBlockIndices_` / `evictor_` / `refCounter_` | [prefix_cache_block_allocator.cpp:447-466](d:\design\MindIE-LLM\src\block_manager\prefix_cache_block_allocator.cpp) |

### 工厂入口

```
SchedulerConfig.enablePrefixCache  →  BlockManagerConfig.enableCaching
                                    ↓
SelfAttnBlockManager 构造函数 [self_attn_block_manager.cpp:45]
   allocatorConfig.allocatorType = enableCaching_ ? PREFIXCACHING : HASHLESS
                                    ↓
CpuNpuBlockAllocator [cpu_npu_block_allocator.cpp:39-59]
   blockObjPool 工厂注册 PrefixCachingBlockObj
   npuAllocators_ / cpuAllocators_ 装 PrefixCacheBlockAllocator
```

锚点：[self_attn_block_manager.cpp:24-63](d:\design\MindIE-LLM\src\block_manager\self_attn_block_manager.cpp)，[cpu_npu_block_allocator.cpp:39-59](d:\design\MindIE-LLM\src\block_manager\cpu_npu_block_allocator.cpp)。

> synthesis: 设计上 prefix cache 是 `BlockAllocator` 的一种实现，与 `HashLessBlockAllocator` 互斥。**关闭 prefix cache 的代价仅是 hash 计算开销 + map 查询**，不会改变 KV 块分配的整体框架，这点比 vLLM "重写 manager" 设计更轻量。

## C++ 调度层：每轮 schedule 怎么用

### 命中块计算（`Scheduler::CollectComputedBlocksInfo`）

[scheduler.cpp:1242-1274](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)：

```cpp
std::vector<BlockId> computedBlocks =
    blockManager_->GetCommonComputedBlockIds(runningSeqSPtrs);   // 本地块复用

std::vector<BlockId> remoteComputedBlocks;
if (schedulerConfig_->enableKvPool) {
    remoteComputedBlocks = blockManager_->GetRemoteComputedBlockIds(
        runningSeqSPtrs,
        computedBlocks.size(),                  // 跳过本地已命中部分
        schedulerConfig_->tpSize,
        schedulerConfig_->modelName);
} else {
    remoteComputedBlocks = computedBlocks;     // 没池化时两者相同
}
metaList[metaIndex].computedLens_.push_back(computedBlocks.size());
metaList[metaIndex].remoteComputedLens_.push_back(remoteComputedBlocks.size());
```

`SP * CP > 1` 时走 `AggregateComputedBlocksInfo`，使用 `GetAllrankComputedBlockNum` / `GetAllRankRemoteComputedBlockIds`（[scheduler.cpp:1257-1274](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)）。

### `GetRemoteComputedBlockIds`：Python MemPool 的 C++ 反向调用

`SelfAttnBlockManager::GetRemoteComputedBlockIds`（[self_attn_block_manager.cpp:373-405](d:\design\MindIE-LLM\src\block_manager\self_attn_block_manager.cpp)）：

1. 取本序列**未本地命中**的所有 hash（`GetSeqHashValues` 之后从 `computedLens` 起）。
2. 按 `tpSize` 展开成 `<hash>_<tpRank>_<tpSize>_<modelName>` 形式的 key 列表（**与 Python 的 `PrefixCachePlugin.get_prefix_keys` 完全同格式**，[prefix_cache_plugin.py:237-244](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\prefix_cache\prefix_cache_plugin.py)）。
3. `memPoolInstance_->LookUp(allKeys)` 调 Python `MemPool.lookup`（C++ pybind 持有 `py::object`，[self_attn_block_manager.cpp:55-61](d:\design\MindIE-LLM\src\block_manager\self_attn_block_manager.cpp)）。
4. 命中前缀 `numCachedBlocks += numElem / tpSize`，作为 `remoteComputedBlocks` 返回。

### Mark computed（**每轮 schedule 末尾必跑**）

[scheduler.cpp:341-342](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)：

```cpp
// 本轮调度结束，mark供下一轮调度使用
blockManager_->MarkBlocksAsComputed();
```

被 `PrefixCacheBlockAllocator::MarkBlocksAsComputed()`（[prefix_cache_block_allocator.cpp:304-310](d:\design\MindIE-LLM\src\block_manager\prefix_cache_block_allocator.cpp)）转化为 `blockComputedAttr_.SetComputed(blockId, true)`，使**这些块在下一轮才算"已计算可命中"**。

### `PolicyHelper`：剔除已 cached 部分参与 budget

[policy_helper.cpp:60-87](d:\design\MindIE-LLM\src\scheduler\policy\policy_helper.cpp)：

```cpp
if (!schedulerConfig_->enablePrefixCache) {
    numUncachedNewTokens += allNumNewTokensSeq;
    continue;
}
const size_t numCachedTokensSeq = blockManager_->GetSeqNumCachedTokens(seq);
...
const size_t numCachedNewTokensSeq =
    numCachedTokensSeq >= numComputedTokenSeq ? numCachedTokensSeq - numComputedTokenSeq : 0;
const size_t numUncachedNewTokensSeq = allNumNewTokensSeq - numCachedNewTokensSeq;
numUncachedNewTokens += numUncachedNewTokensSeq;
numCachedNewTokens   += numCachedNewTokensSeq;
```

> [!todo] VERIFY: `enablePrefixCache=false` 但 `enableCaching=false` 不一定意味着不算 hash——`SchedulerConfig.enablePrefixCache` 直接喂给 `BlockManagerConfig.enableCaching`（[scheduler.cpp:52, 68, 94, 109](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)），所以两个开关在源码里**强一致**。

## C++→Python 协议：computed_block_lens / remote_computed_block_lens

| 字段 | 序列化 | 反序列化 | 锚点 |
|---|---|---|---|
| `computed_block_lens` | `protoMeta.set_computed_block_lens(metaData.computedLens_.data(), ...)` | `convert_bytes_to_list(seq_group_metadata.computed_block_lens)` → `np.int64` | [construct_execute_request.cpp:90, 103](d:\design\MindIE-LLM\src\engine\construct_execute_request.cpp)，[input_metadata_builder.py:823, 854](d:\design\MindIE-LLM\mindie_llm\connector\common\input_metadata_builder.py) |
| `remote_computed_block_lens` | `protoMeta.set_remote_computed_block_lens(...)` | 同上 | [construct_execute_request.cpp:91, 104](d:\design\MindIE-LLM\src\engine\construct_execute_request.cpp)，[input_metadata_builder.py:824](d:\design\MindIE-LLM\mindie_llm\connector\common\input_metadata_builder.py) |
| protobuf schema | `bytes computed_block_lens = 9;` / `bytes remote_computed_block_lens = 27;` | — | [model_execute_data.proto:145, 163](d:\design\MindIE-LLM\proto\model_execute_data.proto) |

Python 端最终落地为 `InputMetadata.computed_blocks` / `remote_computed_blocks`（[input_metadata.py:106-107](d:\design\MindIE-LLM\mindie_llm\text_generator\utils\input_metadata.py)）。

`scp_size > 1` 时 reshape 成 `(-1, scp_size)`：每个 SP/CP rank 独立持有命中长度（[input_metadata_builder.py:843-851](d:\design\MindIE-LLM\mindie_llm\connector\common\input_metadata_builder.py)）。

## Python 插件层：`PrefixCachePlugin`

### 类与初始化

`PrefixCachePlugin(Plugin)`（[prefix_cache_plugin.py:51-121](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\prefix_cache\prefix_cache_plugin.py)）：

| 字段 | 来源 / 作用 |
|---|---|
| `sp_size`, `cp_size`, `scp_size`, `scp_rank` | 来自 `infer_context.spcp_parallel_info`（SP/CP 并行信息） |
| `tp_size`, `tp_rank` | `attn_tp.group_size > 1` 时取 `model_wrapper.mapping.attn_tp`，否则 `(1, 0)` |
| `prefix_cache_preprocess` | `PrefixCachePreprocess(infer_context, cp_size, scp_size, scp_rank)`（输入裁剪器） |
| `mempool_type` | `DISABLED` / `SYNC_WRITE` / `ASYNC_WRITE`，仅当 `kv_pool_backend` 与 `kv_pool_config_path` 都非空才走非 disabled 分支 |
| `m_store` | `MemPool.create_pool(...)` 的实例，按 `kv_pool_async_write` 决定 SYNC/ASYNC |
| 异步：`put_input_queue`/`put_task_queue`/`put_prefix_kvcache_thread`/`save_event` | 仅 `ASYNC_WRITE` 时分配；后台 daemon 线程跑 `_put_prefix_kvcache_thread`（[L391-409](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\prefix_cache\prefix_cache_plugin.py)） |
| `total_token_num`, `total_local_matched_token_num`, `total_remote_matched_token_num` | rank==0 上累计的命中率统计字段 |

### 关键方法

| 方法 | 行为 | 锚点 |
|---|---|---|
| `enable_local_prefixcache(metadata)` | `is_prefill and computed_blocks is not None` | [L123-125](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\prefix_cache\prefix_cache_plugin.py) |
| `enable_remmote_prefixcache(metadata)` | `is_prefill and remote_computed_blocks is not None`（拼写为 `remmote`，**保留原源拼写**） | [L127-129](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\prefix_cache\prefix_cache_plugin.py) |
| `model_inputs_update(...)` | prefill 阶段命中时：`get_prefix_kvcache_from_mempool` → `prefix_cache_preprocess.update_infer_input` → 重算 q_len/spec_mask；rank0 输出 `Prefix Cache Reporter` 命中率日志 | [L132-181](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\prefix_cache\prefix_cache_plugin.py) |
| `get_extra_infer_input(...)` | 把 prefill 转成 "decode 并行解码模式" 的 mask（300I 与 A2/A3 走不同分支，A2/A3 用 splitfuse mask） | [L183-211](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\prefix_cache\prefix_cache_plugin.py) |
| `hash_block(prefix_hash, tokens)` | 对应 C++ `PrefixCachingBlockObj::PrefixHash`：`HashCombine(prev) → 逐 token HashCombine → 加 EXTRA_HASH` | [L228-235](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\prefix_cache\prefix_cache_plugin.py) |
| `get_prefix_keys(hash)` | 拼成 `f"{hash}_{rank}_{size}_{model_name}"`（**与 C++ `GetRemoteComputedBlockIds` 拼 key 完全一致**） | [L237-244](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\prefix_cache\prefix_cache_plugin.py) |
| `get_prefix_kvcache_from_mempool(metadata)` | 命中部分 + remote 命中部分 → 拼 keys + 拼 `(layer, k_cache, v_cache)` 张量列表 → `m_store.get(keys, tensors)` 拉 KV 进 NPU | [L246-306](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\prefix_cache\prefix_cache_plugin.py) |
| `async_put_prefix_kvcache_to_mempool(metadata, cache_ids)` | 把 `(metadata, cache_ids)` 投到 `put_input_queue` 让后台线程处理 | [L308-311](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\prefix_cache\prefix_cache_plugin.py) |
| `put_prefix_kvcache_to_mempool(metadata, cache_ids)` | 同步：对本 rank 已算但远端未命中的块，拼 keys + tensor 后 `m_store.put(...)` | [L328-389](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\prefix_cache\prefix_cache_plugin.py) |
| `_put_prefix_kvcache_thread()` | 后台线程：`set_device → set_stream` → 取 `(metadata, cache_ids)` → 按层调度 `put_task_queue` → `wait_event(pipe_key)` → 末层调 `put_prefix_kvcache_to_mempool` | [L391-409](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\prefix_cache\prefix_cache_plugin.py) |
| `sample_preprocess` / `plugin_verify[*]` / `plugin_cache_update` / `plugin_cache_clear` | **空实现**（prefix cache 不修改 sample/verify 路径） | [L213-226](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\prefix_cache\prefix_cache_plugin.py) |

### `PrefixCachePreprocess.update_infer_input`：input 裁剪

[prefix_cache_preprocess.py:24-149](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\prefix_cache\prefix_cache_preprocess.py)：

1. 命中部分**直接**从 `input_ids` / `position_ids` / `slots` 砍掉：`seq_len -= cached_size`（`cached_size = computed_blocks[i] * block_size`）。
2. 砍后若 `seq_len <= 0`，**少用一个块的命中**确保有 input 进 forward（[L85-87](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\prefix_cache\prefix_cache_preprocess.py)）。
3. SCP 场景额外计算 `sp_computed_slots_padding_idx` / `sp_computed_slots_order` / `all_rank_prefix_lens` / `per_rank_prefix_lens`（[L151-242](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\prefix_cache\prefix_cache_preprocess.py)），做 SCP all_gather 后的块顺序映射。
4. **与 splitfuse 共存**：`split_start_position is not None` 时 `slot` 起点不为 0；被切的第 2、3、… 块**不再生效 computed_blocks**（[L88-90](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\prefix_cache\prefix_cache_preprocess.py)）。

> synthesis: **prefill 命中后转 PA（PagedAttention）decode 风格**——这点用户文档没写，但代码里 `get_extra_infer_input` 注释说"decode 并行解码模式代替 prefill"（[prefix_cache_plugin.py:142, 186-211](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\prefix_cache\prefix_cache_plugin.py)）。这是 prefix cache 与 splitfuse / context parallel / SP 共存时的关键解释路径。

## 与 PluginManager 的协作

### Plugin 注册与互斥

| 锚点 | 内容 |
|---|---|
| [plugin_utils.py:9](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_utils.py) | `PLUGIN_WHITE_LIST = ['la', 'memory_decoding', 'mtp', 'prefix_cache']` |
| [plugin_utils.py:81-84](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_utils.py) | `'prefix_cache': {required_fields: set(), validation_func: lambda data: True}`（**无必填字段**） |
| [plugin_utils.py:127-134](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_utils.py) | 错误文案明示支持的 5 类 plugin_type |
| [plugin_utils.py:11](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_utils.py) | `ASYNC_INFERENCE_UNSUPPORTED_OPTIONS = ["memory_decoding", "la"]`——**prefix_cache 不在其列**，可与 async scheduling 共存 |
| [plugin_utils.py:31-32](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_utils.py) | `InferenceMode`：`prefix_cache in enable_plugin_list → enable_prefill_pa = True`（开启 prefix cache **强制**用 paged attention 做 prefill） |

> synthesis: 用户文档明示"该特性不能和 Multi-LoRA 同时使用 / 不能与 prefix cache + context parallel + sequence parallel + function call(multiturn) 叠加"（[docs/zh/user_guide/feature/prefix_cache.md:21, 26](d:\design\MindIE-LLM\docs\zh\user_guide\feature\prefix_cache.md)），但**校验器中 `prefix_cache` 的 `validation_func` 是 `lambda data: True`**，**没有运行时强校验** Multi-LoRA / function call 叠加——这层互斥靠产品文档约束 + 上层 server 校验，不在 PluginParameterValidator 里阻断。

### PluginManager 调用点

[plugin_manager.py:201-202](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py)：

```python
if "prefix_cache" in self.plugin_list:
    self.mempool_type = self.prefix_cache.mempool_type
```

| 编排点 | 同步 (`generate_token`) | 异步 (`generate_token_async` / `forward_loop`) | 锚点 |
|---|---|---|---|
| `model_inputs_update` 走 plugin 链（包含 `PrefixCachePlugin.model_inputs_update`） | preprocess 后 | 同上，加 `hit_mask` 参数 | [plugin_manager.py:236-244, 359-369](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py) |
| **prefill 时投递异步写** `async_put_prefix_kvcache_to_mempool` | postprocess 前 | 主线程在 prefill 条件下 | [plugin_manager.py:245-252, 495-502](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py) |
| **同步写** `put_prefix_kvcache_to_mempool` | postprocess 前；`SYNC_WRITE` 分支 | `forward_loop` 内 verify 后 `SYNC_WRITE` 分支 | [plugin_manager.py:317-321, 803-807, 888-902](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py) |
| **等待异步落盘** `wait_put_finish` | `ASYNC_WRITE` 时 postprocess 前等 `save_event` | `ASYNC_WRITE` + `warmup_is_end` 时同步等 | [plugin_manager.py:208-212, 317-321, 888-902](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py) |

详见 [PluginManager.md §MemPool 与 prefix cache](../entities/PluginManager.md#mempool-与-prefix-cache)。

### Layerwise 边云：`PluginManagerLwd`

[plugin_manager_lwd.py:95-104](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager_lwd.py)：当 `cp_size > 1` 时**直接 import** 并实例化 `PrefixCachePlugin` 作为 `self.prefix_cache_plugin`，独立于 `plugin_list` 的动态加载路径。`model_inputs_update_manager_longseq_chunk_cp` 在所有插件链跑完后**额外**调一次 `prefix_cache_plugin.model_inputs_update`（[plugin_manager_lwd.py:221-228](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager_lwd.py)），用于 longseq chunk CP 场景下的 prefix 复用。`prepare_metadata_for_longseq_chunk_cp` 还会**人为构造** `remote_computed_blocks`（[plugin_manager_lwd.py:188-199](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager_lwd.py)）。

## 配置参数链

### 服务端入口（C++）

| 参数 | 类型 | 默认 | 锚点 |
|---|---|---|---|
| `plugin_params` 含 `"plugin_type":"prefix_cache"` | 字符串（JSON） | — | [docs/zh/user_guide/feature/prefix_cache.md:32-36](d:\design\MindIE-LLM\docs\zh\user_guide\feature\prefix_cache.md) |
| `EngineConfig.enablePrefixCache` | bool | false | [config_info.h:219](d:\design\MindIE-LLM\src\include\config\config_info.h)；由 `GetPluginEnable("prefix_cache", modelDeployParam)` 解析（[llm_manager_impl.cpp:687](d:\design\MindIE-LLM\src\llm_manager_v2\impl\llm_manager_impl.cpp)） |
| `ScheduleConfig.enablePrefixCache` | bool | false | [config_info.h:383, 502](d:\design\MindIE-LLM\src\include\config\config_info.h)；`schedulerConfig.enablePrefixCache = engineConfig.enablePrefixCache`（[llm_manager_impl.cpp:1076](d:\design\MindIE-LLM\src\llm_manager_v2\impl\llm_manager_impl.cpp)） |
| `BlockManagerConfig.enableCaching` | bool | false | [block_manager_interface.h:66](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h)；由 `cfg.enablePrefixCache` 喂入（[scheduler.cpp:52, 68, 94, 109](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)） |
| `enablePrefixCache` JSON 兼容 | — | — | [schedule_config.cpp:38-42, 188-198](d:\design\MindIE-LLM\src\config_manager\schedule_config.cpp)：保留字段但已无需配置，**老版本配置无影响**，预计 2026 Q1 下线 |

### 互斥规则（产品文档）

来自 [docs/zh/user_guide/feature/prefix_cache.md:15-26](d:\design\MindIE-LLM\docs\zh\user_guide\feature\prefix_cache.md)：

- **支持设备**：Atlas 800I A2、Atlas 300I Duo、Atlas 800I A3。
- **支持模型**：Qwen2/2.5/3 系列，DeepSeek-R1，DeepSeek-V3/V3.1。
- **量化支持**：W4A8 / W8A8 / PDMIX / 稀疏量化；其余暂不支持。
- **互斥**：不能与 **Multi-LoRA** 同时使用；不支持 `prefix cache + context parallel + sequence parallel + function call(multiturn)` 叠加。
- **可叠加**：PD 分离、并行解码、MTP、kvcache 池化、异步调度、SplitFuse、context parallel + sequence parallel、C8 量化。
- **PD 分离场景**：**仅 P 节点需要开启**该特性。
- **复用粒度**：跨 session 公共前缀 token 数 ≥ block size 才命中。

### 与 KV cache 池化叠加

[docs/zh/user_guide/feature/kv_cache_pool.md:5-14](d:\design\MindIE-LLM\docs\zh\user_guide\feature\kv_cache_pool.md)：**KV pool 必须叠加 prefix cache**——pool 是 prefix cache 的"二级存储"（DRAM/SSD），由 `MemPool` 后端实现。代码上即 `kv_pool_backend` / `kv_pool_config_path` 非空时把 `mempool_type` 切到 `SYNC_WRITE` / `ASYNC_WRITE`（[prefix_cache_plugin.py:91-116](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\prefix_cache\prefix_cache_plugin.py)）。

## PD 分离场景下的 prefix cache

### 数据通路（synthesis based on §scheduler 与 §plugin 锚点）

```
P 节点（仅 P 开 prefix_cache 即可）
  ┌── 完成 prefill ──────────────────────────────────────────┐
  │ MarkBlocksAsComputed ([scheduler.cpp:342])              │
  │  → PrefixCacheBlockAllocator.cachedBlocks_ += 本次块    │
  │ PrefixCachePlugin.async_put_prefix_kvcache_to_mempool   │
  │  → 后台线程将 KV 写入 MemPool（DRAM/SSD pool）           │
  └─────────────────────────────────────────────────────────┘

D 节点
  ┌── KV 拉取后开始 decode ─────────────────────────────────┐
  │ Scheduler.CollectComputedBlocksInfo                     │
  │  → GetCommonComputedBlockIds                            │
  │     （仅本地，D 多数为 0 因为 D 重启就空）              │
  │  → 若 enableKvPool: GetRemoteComputedBlockIds           │
  │     → memPoolInstance_->LookUp(<hash>_<tp>_<size>_<m>)  │
  │     → 命中数喂回 remoteComputedLens_                    │
  │ → 序列化进 protobuf remote_computed_block_lens          │
  └─────────────────────────────────────────────────────────┘
```

### 与 `BlockSpaceManager.GetRemoteComputedBlockIds` 的关系

详见 [BlockSpaceManager.md §"接口定义"](../entities/BlockSpaceManager.md) 的 [block_manager_interface.h:155-156](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) 行；**该方法是 prefix cache 在 PD 分离 + KV 池化场景下的核心入口**。`SelfAttnBlockManager::GetRemoteComputedBlockIds`（[self_attn_block_manager.cpp:373-405](d:\design\MindIE-LLM\src\block_manager\self_attn_block_manager.cpp)）通过 `memPoolInstance_->LookUp` 反向调 Python `MemPool.lookup`，把"key 在远端是否存在"映射为"该序列还能多复用几个块"。

> [!warning] CONTRADICTION（待人工 verify）：产品文档说"PD 分离场景下，**仅 P 节点需要开启**该特性"（[docs/zh/user_guide/feature/prefix_cache.md:24](d:\design\MindIE-LLM\docs\zh\user_guide\feature\prefix_cache.md)），但 D 节点要走 `GetRemoteComputedBlockIds` 走 KV pool 命中查询，**也需要 `enableKvPool=true` + 与 P 用同一 hash 算法**（key 拼接形式 `<hash>_<tp>_<size>_<model>` 必须等价）。所以"仅 P 开启"的精确语义是"**只在 P 端启用本地 prefix cache 的 hash table 主索引**，但 KV pool 二级存储两端都要开启"。

### 与 layerwise PD（边云）

`LwdSelfAttnBlockManager` 的 `lwd_self_attn_block_manager.h:26-34` 注释明示该实现**暂不支持 prefix caching / copy-on-write**（详见 [BlockSpaceManager.md §Layerwise SelfAttn 独有路径](../entities/BlockSpaceManager.md#layerwise-selfattn-独有路径)）。但 `PluginManagerLwd` 仍然实例化 `PrefixCachePlugin`（[plugin_manager_lwd.py:95-104](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager_lwd.py)）——这条路径下 prefix 复用走的是**层级长序列分块的 remote_computed_blocks 人为构造**（`prepare_metadata_for_longseq_chunk_cp`），而非 `GetCommonComputedBlockIds` 路径。

## Warmup

`Generator._update_request_for_prefix_cache`（[generator.py:1416-1438](d:\design\MindIE-LLM\mindie_llm\text_generator\generator.py)）：当 `enable_prefix_cache and backend_type != "torch"`，自动 warmup 时**人为给请求填 `computed_blocks` 与 `remote_computed_blocks`**，触发 prefix cache 的 `model_inputs_update` 走全路径，避免首请求时 NPU 算子 / mempool 调用未编译/未连。SCP 场景下按 `scp_size` 轮询填块数。

## §跨子系统引用（§5 step 3 hidden cross-reference grep）

按 [AGENTS.md §5 step 3](../../AGENTS.md#5-ingest-工作流) 5 类 grep 强制执行。

### 1. 跨语言绑定（C++↔Python）

- **C++ → Python（pybind 反向调用 MemPool）**：`PyGILState_Ensure` + `py::module_::import("mindie_llm.text_generator.mempool").attr("MemPool")` 在 [self_attn_block_manager.cpp:55-61](d:\design\MindIE-LLM\src\block_manager\self_attn_block_manager.cpp) — 这是 `BlockSpaceManager` 唯一的 Python 跨语言入口，专为 prefix cache 的 `LookUp` 远端命中查询服务。
- **Python class `PrefixCachePlugin` 是否被 C++ 直接调用**：**在 d:\design\MindIE-LLM\src\ 全 C++ 树 grep `PrefixCachePlugin` 0 命中**（**真**）；C++ 仅通过 `MemPool.lookup` / `MemPool.put` / `MemPool.get` 间接接入 Python prefix cache，不直接持有 plugin 实例。
- **C++ class `PrefixCacheBlockAllocator` / `PrefixCachingBlockObj` 是否被 Python 直接 import**：**在 d:\design\MindIE-LLM\mindie_llm\ 全 Python 树 grep 0 命中**（**真**）；Python 仅通过 protobuf 字段 `computed_block_lens` / `remote_computed_block_lens` 接收 C++ 端的命中长度，不持有任何 C++ 块对象。

### 2. 协作伙伴跨子系统引用

| 协作类 | grep 范围 | 命中数与位置 |
|---|---|---|
| `PrefixCachePlugin` 全仓库 | `d:\design\MindIE-LLM\` | **6 命中**（仅 mindie_llm/ Python 树），分布于 `prefix_cache_plugin.py`（自身）、`plugin_manager.py`、`plugin_manager_lwd.py`、`plugin_utils.py`、`generator.py`、自身 `__init__.py`；C++ 树 0 命中 |
| `PrefixCacheBlockAllocator` 全仓库 | `d:\design\MindIE-LLM\` | **3 文件**：`prefix_cache_block_allocator.{h,cpp}`、`cpu_npu_block_allocator.cpp`（构造点 [L48, L59](d:\design\MindIE-LLM\src\block_manager\cpu_npu_block_allocator.cpp)）；mindie_llm/ Python 树 0 命中 |
| `PrefixCachingBlockObj` 全仓库 | `d:\design\MindIE-LLM\` | **2 文件**：`prefix_cache_block.{h,cpp}` + `cpu_npu_block_allocator.cpp:39`（工厂注册）；mindie_llm/ Python 树 0 命中 |
| `BlockSpaceManager` × prefix cache | `d:\design\MindIE-LLM\src\` | 共 14 接口包含 `GetPrefixCacheHitRate`/`ResetPrefixCache`/`GetRankedHashValues`/`GetSeqHashValues`/`GetCommonComputedBlockIds`/`GetRemoteComputedBlockIds`/`GetAllRankRemoteComputedBlockIds`/`MarkBlocksAsComputed`/`GetNumCachedTokens`/`GetSeqNumCachedTokens` 等，详见 [BlockSpaceManager.md §接口定义](../entities/BlockSpaceManager.md) |
| `PluginManager` × prefix_cache | `d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py` | **5 个调用点**（[L201, L208, L247, L497, L804](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py)），详见 [PluginManager.md §MemPool 与 prefix cache](../entities/PluginManager.md#mempool-与-prefix-cache) |
| `MemPool` × prefix_cache | `d:\design\MindIE-LLM\` | C++ 端通过 [self_attn_block_manager.cpp:55-61, 393-403](d:\design\MindIE-LLM\src\block_manager\self_attn_block_manager.cpp) 调 `MemPool.LookUp`；Python 端 [prefix_cache_plugin.py:96-103, 306, 389](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\prefix_cache\prefix_cache_plugin.py) 调 `MemPool.create_pool`/`get`/`put` |
| `SeparateDeploymentEngine` × prefix_cache | `d:\design\MindIE-LLM\mindie_llm\` | **在 separate_deployment_engine.py 全 Python 树 grep `prefix_cache` 0 命中**（**真**）—— PD 路径里 prefix cache 不与 SeparateDeploymentEngine 直接耦合，靠 protobuf `remote_computed_block_lens` 字段隐式传递 |

### 3. 配置 / IPC 共享数据结构

- **protobuf `computed_block_lens` (field 9) / `remote_computed_block_lens` (field 27)**：[model_execute_data.proto:145, 163](d:\design\MindIE-LLM\proto\model_execute_data.proto)
  - **C++ 写**：[construct_execute_request.cpp:90, 91, 103, 104](d:\design\MindIE-LLM\src\engine\construct_execute_request.cpp)
  - **Python 读**：[input_metadata_builder.py:823-824](d:\design\MindIE-LLM\mindie_llm\connector\common\input_metadata_builder.py)
  - **测试覆盖**：[tests/pythontest/cpu/connector/common/test_input_metadata_builder.py:395-396](d:\design\MindIE-LLM\tests\pythontest\cpu\connector\common\test_input_metadata_builder.py) 用 `struct.pack('<2q', 0, 0)` 模拟字节流
- **`enablePrefixCache` 配置字段**：在 d:\design\MindIE-LLM\ 全树共 **15 命中**（Server config / EngineConfig / ScheduleConfig 三层 + 4 处 scheduler.cpp 喂入 BlockManagerConfig + 兼容字段已弃用日志）
- **`enableCaching` 配置字段**：仅 BlockManager 层使用（[block_manager_interface.h:66](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h)），上层不直接读

### 4. 测试覆盖反查

| 测试文件 | 覆盖点 |
|---|---|
| [tests/pythontest/npu/text_generator/test_plugins/test_prefix_cache_plugin.py](d:\design\MindIE-LLM\tests\pythontest\npu\text_generator\test_plugins\test_prefix_cache_plugin.py) | `PrefixCachePlugin` 单元测试，含 `FakeMemPool` mock |
| [tests/pythontest/npu/text_generator/test_plugins/test_prefix_cache_preprocess.py](d:\design\MindIE-LLM\tests\pythontest\npu\text_generator\test_plugins\test_prefix_cache_preprocess.py) | `PrefixCachePreprocess.update_infer_input` 单元测试 |
| [tests/dlt/ut/block_manager/prefix_cache_block_allocator_test.cpp](d:\design\MindIE-LLM\tests\dlt\ut\block_manager\prefix_cache_block_allocator_test.cpp) | `PrefixCacheBlockAllocator` 单元测试（**不少于 22 个测试用例**，详见 [test_prefix_cache_block_alloctor.md](d:\design\MindIE-LLM\tests\dlt\ut\block_manager\test_prefix_cache_block_alloctor.md)：`AllocateMutableBlock` / `AllocateImmutableBlock` / `MarkBlocksAsComputed` / Swap / Fork / Evictor / FindCachedBlocksPrefix / GetCommonComputedBlockIds 等） |
| [tests/dlt/ut/block_manager/prefix_cache_block_test.cpp](d:\design\MindIE-LLM\tests\dlt\ut\block_manager\prefix_cache_block_test.cpp) | `PrefixCachingBlockObj` 单元测试 |
| [tests/dlt/ut/block_manager/prefix_block_manager_test.cpp](d:\design\MindIE-LLM\tests\dlt\ut\block_manager\prefix_block_manager_test.cpp) | `SelfAttnBlockManager` + prefix cache 集成测试 |
| [tests/dlt/it/test_self_attn_block_manager_prefix.cpp](d:\design\MindIE-LLM\tests\dlt\it\test_self_attn_block_manager_prefix.cpp) | 集成测试 |
| [tests/dlt/ut/block_manager/cpu_npu_block_allocator_test.cpp](d:\design\MindIE-LLM\tests\dlt\ut\block_manager\cpu_npu_block_allocator_test.cpp) | CPU/NPU 双层 prefix cache 块分配测试 |
| [tests/pythontest/npu/text_generator/test_plugins/test_plugin_manager.py](d:\design\MindIE-LLM\tests\pythontest\npu\text_generator\test_plugins\test_plugin_manager.py) | PluginManager 内 prefix_cache 编排路径测试 |
| [tests/dlt/ut/scheduler/test_dynamicBatchSize.cpp](d:\design\MindIE-LLM\tests\dlt\ut\scheduler\test_dynamicBatchSize.cpp) / `test_fcfs_edge_cloud_policy.cpp` / `test_policy_factory.cpp` | 调度策略层 `enablePrefixCache` 通路测试 |
| [tests/fuzztest_llmmanager/llm_manager/test_llm_manager_test.cpp](d:\design\MindIE-LLM\tests\fuzztest_llmmanager\llm_manager\test_llm_manager_test.cpp) | LLM Manager 层 fuzz 测试 |

### 5. doc / config / yaml 反查

| 路径 | 命中内容 |
|---|---|
| [docs/zh/user_guide/feature/prefix_cache.md](d:\design\MindIE-LLM\docs\zh\user_guide\feature\prefix_cache.md) | 主用户文档（特性介绍、限制约束、参数表、curl 示例） |
| [docs/zh/user_guide/feature/kv_cache_pool.md](d:\design\MindIE-LLM\docs\zh\user_guide\feature\kv_cache_pool.md) | KV pool 文档：明示"必须叠加 prefix cache" |
| [docs/zh/user_guide/feature/README.md](d:\design\MindIE-LLM\docs\zh\user_guide\feature\README.md) | 特性目录入口（grep 命中） |
| [docs/zh/user_guide/user_manual/menu_user_manual.md](d:\design\MindIE-LLM\docs\zh\user_guide\user_manual\menu_user_manual.md) | 菜单引用（grep 命中） |
| [docs/zh/developer_guide/architecture_design/architecture_overview.md:39](d:\design\MindIE-LLM\docs\zh\developer_guide\architecture_design\architecture_overview.md) | 架构图列出 `prefix_cache` 目录位置 |
| 部署 yaml/json | **在 d:\design\MindIE-LLM\mindie_llm\examples\ 全树 grep `prefix_cache` 0 命中**（**真**），仅在 tests/dlt/ut/.../config*.json 模板中出现，**examples/ 没有可直接 copy 的 prefix cache 部署样例**（用户需手动按 docs/zh/user_guide/feature/prefix_cache.md 填配置） |

## 跨项目对照（synthesis）

> 非 MindIE 源码推导，仅结构对照，详细对比见 [comparison/topics/kv-cache.md](../../comparison/topics/kv-cache.md)。

| 维度 | MindIE | vLLM | SGLang |
|---|---|---|---|
| **数据结构** | `unordered_map<HashValue, BlockId>`（hash table）+ LRU evictor | `BlockHashToBlockMap`（hash table）在 `KVCacheManager` 内 | **Trie / Radix tree**（`RadixCache` [radix_cache.py:285](d:\design\sglang\python\sglang\srt\mem_cache\radix_cache.py)，及 C++ 加速版 `RadixCacheCpp`） |
| **抽象边界** | C++ Allocator + Python Plugin **两段** | 全 Python 在 `KVCacheManager` 内一段 | Python `BasePrefixCache` 多实现（`RadixCache` / `HiRadixCache` / `RadixCacheCpp` 等） |
| **Hash 算法** | boost-style `hash_combine` + 64-bit；Python 端**显式镜像** C++ | `hash_block_tokens(caching_hash_fn, prev, tokens, extra_keys)`（[kv_cache_utils.py:535](d:\design\vllm\vllm\v1\core\kv_cache_utils.py)） | trie 节点路径分裂，无单点 hash |
| **PD/远端命中** | `GetRemoteComputedBlockIds` → `MemPool.LookUp`（C++→Python pybind） | KV transfer connector 体系（[kv_transfer/kv_connector/v1](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1)） | HiRadixCache + HiCache Storage 7 后端 |
| **触发开关** | `plugin_params={plugin_type:prefix_cache}` + `enableCaching` | `--enable-prefix-caching`（默认 enabled） | `--disable-radix-cache` 反向开关 |
| **测试覆盖** | C++ unit ≥ 22 + Python unit + 集成 + scheduler 策略层 | v1/core/sched 系列 unit | mem_cache/ unit + integration |

> synthesis: MindIE 与 vLLM 都用 **hash table**（差别仅在抽象层数：MindIE C++/Python 双段，vLLM 全 Python 单段）；**SGLang 是唯一用 trie 的**——3 家路线两派分立。MindIE 把 KV pool 复用从核心 prefix cache 中**剥离**（pool 是可选二级存储 + Python 插件主导写回），与 vLLM 的 connector 抽象不同。

## Notes / Caveats

> [!warning] CONTRADICTION: 用户文档"PD 分离场景下，仅 P 节点需要开启该特性"（[docs/zh/user_guide/feature/prefix_cache.md:24](d:\design\MindIE-LLM\docs\zh\user_guide\feature\prefix_cache.md)）与 D 节点要走 `GetRemoteComputedBlockIds` + `MemPool.LookUp` 才能享受 KV pool 命中——精确含义是"**P 端开启本地 hash table 主索引**，KV pool 二级存储两端都要 enable"。

> [!warning] CONTRADICTION: `PrefixCachePlugin.enable_remmote_prefixcache` **拼写错误**（多了一个 m），[prefix_cache_plugin.py:127](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\prefix_cache\prefix_cache_plugin.py)；仅在插件类内自调，外部未引用，但未来重构需谨慎。

> [!todo] VERIFY: C++ `HashCombine`（[math_utils.h](d:\design\MindIE-LLM\src\block_manager\math_utils.h)） 与 Python `hash_combine`（[prefix_cache_plugin.py:45-48](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\prefix_cache\prefix_cache_plugin.py)）的 64-bit 截断 / 移位顺序是否完全等价；如果不等价，KV pool 跨 P/D 节点 key 不匹配，命中率为 0 且静默失败。

> [!todo] VERIFY: 用户文档"该特性不能和 Multi-LoRA 同时使用 / 不支持 prefix cache + CP + SP + function call(multiturn)"（[docs/zh/user_guide/feature/prefix_cache.md:21, 26](d:\design\MindIE-LLM\docs\zh\user_guide\feature\prefix_cache.md)）的**运行时强校验**位置——`PluginParameterValidator` 中 `prefix_cache` 的 `validation_func` 是 `lambda data: True`（[plugin_utils.py:81-84](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_utils.py)），**没有阻断**，依赖上层 server / model 配置层校验。

> [!todo] VERIFY: `PrefixCacheBlockAllocator::MarkBlocksAsAccessed` 内对 evictor 中块 update 时**直接抛异常**（[prefix_cache_block_allocator.cpp:296-300](d:\design\MindIE-LLM\src\block_manager\prefix_cache_block_allocator.cpp)）+ 注释 "理论上不应该走到这里，未明确为啥开源软件中这个实现"——上游 vLLM 的等价代码是否真不会触达此分支？

> [!todo] VERIFY: `Composite` 块管理器（[block_manager_interface.h:29-31](d:\design\MindIE-LLM\src\include\block_manager\block_manager_interface.h) 枚举槽位）若未来落地，prefix cache 在 `subManagers` 之间是按 manager 各自独立维持还是统一前缀视图？源码枚举有但工厂未实现（详 [BlockSpaceManager.md "5 个实现"对照表](../entities/BlockSpaceManager.md#5-个实现对照表)）。

## See also

- [BlockSpaceManager.md](../entities/BlockSpaceManager.md) — C++ 块管理器抽象，含 `GetRemoteComputedBlockIds` 详解
- [PluginManager.md](../entities/PluginManager.md) §MemPool 与 prefix cache — Python 插件编排同步/异步路径
- [Generator.md](../entities/Generator.md) — `enable_prefix_cache` warmup 路径
- [topics/kv-cache.md](kv-cache.md) — 三层 KV cache 体系（C++ BlockSpaceManager + Python KVCachePool + MemPool）
- [topics/connector.md](connector.md) — PD 拉 KV 链路 / Mooncake 等 mempool 后端
- [comparison/topics/prefix-cache.md](../../comparison/topics/prefix-cache.md) — 三家 prefix cache 深度对比（11 子维度，本页是 MindIE 端深化）
- [comparison/topics/kv-cache.md](../../comparison/topics/kv-cache.md) — 父级 KV cache 对比页
- [comparison/dimensions.md §dim-prefix-cache](../../comparison/dimensions.md) — 跨项目维度
- 上游产品文档：[prefix_cache.md](d:\design\MindIE-LLM\docs\zh\user_guide\feature\prefix_cache.md)、[kv_cache_pool.md](d:\design\MindIE-LLM\docs\zh\user_guide\feature\kv_cache_pool.md)
