---
type: topic
project: vllm
status: stale
confidence: high
verified_against: 2026-08-18 (vLLM d29dc3ab, increment from 5f7fab88)
sources:
  - d:\design\vllm\vllm\v1\core\kv_cache_manager.py
  - d:\design\vllm\vllm\v1\core\block_pool.py
  - d:\design\vllm\vllm\v1\core\kv_cache_utils.py
  - d:\design\vllm\vllm\v1\core\kv_cache_coordinator.py
  - d:\design\vllm\vllm\v1\core\single_type_kv_cache_manager.py
  - d:\design\vllm\vllm\v1\kv_cache_interface.py
  - d:\design\vllm\vllm\v1\core\sched\scheduler.py
  - d:\design\vllm\vllm\v1\engine\core.py
  - d:\design\vllm\vllm\v1\request.py
  - d:\design\vllm\vllm\config\cache.py
  - d:\design\vllm\vllm\utils\hashing.py
  - d:\design\vllm\vllm\distributed\kv_events.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\simple_cpu_offload_connector.py
  - d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\offloading\scheduler.py
  - d:\design\vllm\vllm\v1\simple_kv_offload\manager.py
  - d:\design\vllm\tests\v1\core\test_prefix_caching.py
  - d:\design\vllm\tests\v1\core\test_kv_cache_utils.py
  - d:\design\vllm\tests\v1\core\test_reset_prefix_cache_e2e.py
  - d:\design\vllm\tests\v1\core\test_single_type_kv_cache_manager.py
  - d:\design\vllm\docs\features\automatic_prefix_caching.md
  - d:\design\vllm\docs\design\prefix_caching.md
  - d:\design\vllm\examples\offline_inference\automatic_prefix_caching.py
  - d:\design\vllm\examples\offline_inference\prefix_caching.py
related:
  - vllm/entities/KVCacheManager.md
  - vllm/entities/Scheduler.md
  - comparison/topics/prefix-cache.md
  - comparison/topics/kv-cache.md
  - comparison/dimensions.md
---

# Prefix Cache（vLLM v1）

## Summary

vLLM v1 的 **prefix cache 不是独立子系统**，而是 [`KVCacheManager`](../entities/KVCacheManager.md) 的内置能力——由 `cache_config.enable_prefix_caching` 单开关（**默认 True**，[cache.py:107](d:\design\vllm\vllm\config\cache.py)）翻转 `BlockPool` 内的 **`BlockHashToBlockMap` hash table** 是否参与查询/插入。三段关键设计：

- **Hash 算法**：`hash_block_tokens(hash_function, parent_block_hash, curr_block_token_ids, extra_keys)`（[kv_cache_utils.py:577-604](d:\design\vllm\vllm\v1\core\kv_cache_utils.py)）链式哈希 `(parent_hash, tokens_tuple, extra_keys)`，`hash_function` 由 `prefix_caching_hash_algo` ∈ `{sha256, sha256_cbor, xxhash, xxhash_cbor}` 选（[hashing.py:82-100](d:\design\vllm\vllm\utils\hashing.py)，default `sha256`）。
- **数据结构**：**`BlockHashToBlockMap` = `dict[BlockHashWithGroupId, KVCacheBlock | dict[block_id, KVCacheBlock]]`**（[block_pool.py:33-141](d:\design\vllm\vllm\v1\core\block_pool.py)），是 **hash table（不是 trie/radix tree）**——同 hash 多 block 不去重以保持 block table append-only。
- **3 种 coordinator 拓扑**：`KVCacheCoordinatorNoPrefixCache` / `UnitaryKVCacheCoordinator` / `HybridKVCacheCoordinator`（[kv_cache_coordinator.py:420-922](d:\design\vllm\vllm\v1\core\kv_cache_coordinator.py)），由 `get_kv_cache_coordinator(...)` 工厂按 `enable_caching` + 是否多 KV 组路由。

> synthesis：与 `MindIE 的 prefix cache`（已删）（C++ Allocator + Python Plugin 双段）不同，vLLM 是**全 Python 一段抽象**——`BlockPool` / `Coordinator` / `SingleTypeKVCacheManager` 都在 [v1/core/](d:\design\vllm\vllm\v1\core/) 下；csrc/ 树**对 prefix cache 0 命中**（见 §跨子系统引用 §1）。与 SGLang `RadixCache`（trie）也不同，vLLM 与 MindIE 同属 hash table 派。

## Sources

| 资源 | 锚点 |
|---|---|
| Manager 主入口 | [kv_cache_manager.py](d:\design\vllm\vllm\v1\core\kv_cache_manager.py)（887 行） |
| 块池 + hash 索引 | [block_pool.py](d:\design\vllm\vllm\v1\core\block_pool.py)（831 行） |
| Hash / Block 类型 / FreeQueue / extra_keys | [kv_cache_utils.py](d:\design\vllm\vllm\v1\core\kv_cache_utils.py)（2343 行） |
| 3 种 Coordinator | [kv_cache_coordinator.py](d:\design\vllm\vllm\v1\core\kv_cache_coordinator.py)（979 行） |
| 单类型 Manager 群（`find_longest_cache_hit` 多态） | [single_type_kv_cache_manager.py](d:\design\vllm\vllm\v1\core\single_type_kv_cache_manager.py)（1953 行） |
| KV cache spec | [kv_cache_interface.py](d:\design\vllm\vllm\v1\kv_cache_interface.py) |
| 配置 | [config/cache.py:107-109, 221-260](d:\design\vllm\vllm\config\cache.py) |
| Hash 函数实现 | [utils/hashing.py](d:\design\vllm\vllm\utils\hashing.py) |
| Scheduler 集成 | [v1/core/sched/scheduler.py:129-159, 826-880, 2511-2582](d:\design\vllm\vllm\v1\core\sched\scheduler.py) |
| EngineCore 启动 hash 函数 | [v1/engine/core.py:223-232](d:\design\vllm\vllm\v1\engine\core.py) |
| Request 端 block_hashes | [v1/request.py:140, 214-276](d:\design\vllm\vllm\v1\request.py) |
| KV connector 接口 | [kv_connector/v1/base.py:475-](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py) |
| KV cache 事件 | [distributed/kv_events.py](d:\design\vllm\vllm\distributed\kv_events.py)、[block_pool.py:344-444, 592-606, 796](d:\design\vllm\vllm\v1\core\block_pool.py) |
| 用户 / 设计文档 | [docs/features/automatic_prefix_caching.md](d:\design\vllm\docs\features\automatic_prefix_caching.md)、[docs/design/prefix_caching.md](d:\design\vllm\docs\design\prefix_caching.md) |

## Architecture / Data flow

```
                     CacheConfig.enable_prefix_caching = True (默认)
                                  │
                                  ▼
              EngineCore.__init__ ([engine/core.py:223-232])
                                  │
              ├── caching_hash_fn = get_hash_fn_by_name(prefix_caching_hash_algo)
              ├── init_none_hash(caching_hash_fn)
              └── request_block_hasher = get_request_block_hasher(block_size, caching_hash_fn)
                                  │
                                  ▼
              Request.update_block_hashes ([request.py:214-219, 273-276])
              ├── 每次新 token 追加都增量算 BlockHash 并 append 到 self.block_hashes
              └── hash_block_tokens(hash_function, prev, tokens, extra_keys)
                                  │
                                  ▼
              Scheduler.__init__ ([sched/scheduler.py:277-287])
              └── KVCacheManager(enable_caching=cache_config.enable_prefix_caching, ...)
                                  │
                                  ▼
              get_kv_cache_coordinator(...)  →  3 选 1
              ├── enable_caching=False               → KVCacheCoordinatorNoPrefixCache
              ├── len(kv_cache_groups)==1            → UnitaryKVCacheCoordinator
              └── len(kv_cache_groups)>1 (hybrid)    → HybridKVCacheCoordinator
                                  │
                                  ▼
              ├── BlockPool: blocks[KVCacheBlock] + FreeKVCacheBlockQueue + BlockHashToBlockMap
              └── single_type_managers: tuple[SingleTypeKVCacheManager, ...]  (一个 group 一个，经 KVCacheSpecRegistry 路由)
                       ├── FullAttentionManager   (FullAttn / MLA / HiddenStateCache)
                       ├── RSWAManager            (新增: Reference SWA)
                       ├── SlidingWindowManager   (含 SlidingWindowMLASpec)
                       ├── ChunkedLocalAttentionManager
                       ├── MambaManager
                       ├── CrossAttentionManager  (no prefix cache)
                       └── SinkFullAttentionManager

每轮 schedule (Scheduler.schedule, [sched/scheduler.py:748-1162] WAITING 阶段) :

   新请求 (request.num_computed_tokens == 0):
     ├── KVCacheManager.get_computed_blocks(request)             # 本地命中
     │   ├── coordinator.find_longest_cache_hit(block_hashes, max_cache_hit_length)
     │   │   ├── Unitary: 调一次 single_type.find_longest_cache_hit
     │   │   └── Hybrid: 多 group 迭代 fixed-point，full-attn 优先 + LCM 对齐
     │   └── PrefixCacheStats.record(num_tokens, num_hits)
     │
     ├── connector.get_num_new_matched_tokens(request, num_local_computed_tokens)  # 远端命中
     │   └── 返回 num_external_computed_tokens (P/D 拉 KV)
     │
     ├── KVCacheManager.allocate_slots(request, num_new_tokens, num_new_computed_tokens, new_computed_blocks, ...)
     │   ├── coordinator.remove_skipped_blocks   # sliding window: 砍掉窗口外块
     │   ├── coordinator.get_num_blocks_to_allocate
     │   ├── coordinator.allocate_new_computed_blocks  # touch + 拼 prefix hit
     │   ├── coordinator.allocate_new_blocks           # BlockPool.get_new_blocks
     │   └── coordinator.cache_blocks(request, num_tokens_to_cache)  # 块写满后 → BlockPool.cache_full_blocks → BlockHashToBlockMap.insert
     │
     └── 返回 KVCacheBlocks 给 Scheduler

   请求结束: KVCacheManager.free(request) → 反向归还 free queue
```

锚点：
- 工厂选择：[kv_cache_coordinator.py:923-979](d:\design\vllm\vllm\v1\core\kv_cache_coordinator.py)
- get_computed_blocks 主入口：[kv_cache_manager.py:232-299](d:\design\vllm\vllm\v1\core\kv_cache_manager.py)（现返 3 元组，含 `shared_prefix_boundary`）
- allocate_slots 主入口：[kv_cache_manager.py:347-569](d:\design\vllm\vllm\v1\core\kv_cache_manager.py)
- Scheduler 调用点：[sched/scheduler.py:866-880 (新请求 hit)](d:\design\vllm\vllm\v1\core\sched\scheduler.py), [1033 (allocate_slots)](d:\design\vllm\vllm\v1\core\sched\scheduler.py)

## Hash 计算细节

### `BlockHash` 与 `BlockHashWithGroupId`

`BlockHash` 是 `NewType("BlockHash", bytes)`（[kv_cache_utils.py:45](d:\design\vllm\vllm\v1\core\kv_cache_utils.py)），不是 dataclass——直接用 `bytes` 储存以省 GC 与 dict 哈希成本。`BlockHashWithGroupId` = `BlockHash` + `group_id.to_bytes(4, "big")`（[kv_cache_utils.py:50, 59-77](d:\design\vllm\vllm\v1\core\kv_cache_utils.py)，pack/unpack 现拆为 `make_block_hash_with_group_id` / `get_block_hash` / `get_group_id` 三个 helper），用作 `BlockHashToBlockMap` 的 key：**不同 KV cache group 的同 hash 块物理上是不同 entries**。

```python
# kv_cache_utils.py:577-604
def hash_block_tokens(
    hash_function: Callable[[Any], bytes],
    parent_block_hash: BlockHash | None,
    curr_block_token_ids: Sequence[int],
    extra_keys: tuple[Any, ...] | None = None,
) -> BlockHash:
    if not parent_block_hash:
        parent_block_hash = NONE_HASH        # 全局 seed; init_none_hash() 设
    curr_block_token_ids_tuple = tuple(curr_block_token_ids)
    return BlockHash(
        hash_function((parent_block_hash, curr_block_token_ids_tuple, extra_keys))
    )
```

| 字段 | 来源 | 锚点 |
|---|---|---|
| `parent_block_hash` | 上一块的 BlockHash；首块用 `NONE_HASH` | [kv_cache_utils.py:598-599](d:\design\vllm\vllm\v1\core\kv_cache_utils.py) |
| `curr_block_token_ids` | `request.all_token_ids[start:end]`（仅 full block） | [kv_cache_utils.py:685-728](d:\design\vllm\vllm\v1\core\kv_cache_utils.py) |
| `extra_keys` | LoRA name + MM `(identifier, offset)` + cache_salt（仅首块）+ prompt_embeds hash | [kv_cache_utils.py:539-576](d:\design\vllm\vllm\v1\core\kv_cache_utils.py) |
| `hash_function` | `caching_hash_fn`，default `sha256`；其余可选 `sha256_cbor` / `xxhash` / `xxhash_cbor`（后两者要 `xxhash` 包） | [config/cache.py:109](d:\design\vllm\vllm\config\cache.py)、[hashing.py:82-100](d:\design\vllm\vllm\utils\hashing.py) |
| `NONE_HASH` | `os.urandom(32)`（无 PYTHONHASHSEED）或 `hash_fn(PYTHONHASHSEED)` | [kv_cache_utils.py:96-117](d:\design\vllm\vllm\v1\core\kv_cache_utils.py) |

`get_request_block_hasher(hash_block_size, caching_hash_fn)` 是工厂，返回闭包 `request_block_hasher(request) -> list[BlockHash]`，由 `Request.update_block_hashes` 在 `__init__` 与每次 token 追加后调用，把新 hash 拼到 `request.block_hashes`（[request.py:214-219, 273-276](d:\design\vllm\vllm\v1\request.py)、[kv_cache_utils.py:671-728](d:\design\vllm\vllm\v1\core\kv_cache_utils.py)）。hash 现固定按 `hash_block_size` 粒度链式计算，供更粗的 group block size 直接复用（`BlockHashListWithBlockSize`，[kv_cache_utils.py:2310](d:\design\vllm\vllm\v1\core\kv_cache_utils.py) 附近）。

> synthesis：vLLM hash 链是**纯 token + extra_keys**——与 MindIE C++ `HashCombine`（boost-style XOR + magic constant，[mindie prefix_cache_block.cpp:101-118](d:\design\MindIE-LLM\src\block_manager\prefix_cache_block.cpp)）不同。MindIE Python 端必须**显式镜像** C++ 算法以保 KV pool key 跨进程等价；vLLM 不存在这个问题，因为整个 hash 链路在单一 Python 进程内（`Request` 端算 + `BlockPool` 端比对都用同一 `caching_hash_fn`）。

### `extra_keys` 隔离场景

`generate_block_hash_extra_keys(request, start_token_idx, end_token_idx, start_mm_idx)`（[kv_cache_utils.py:539-576](d:\design\vllm\vllm\v1\core\kv_cache_utils.py)）依赖：

| 来源 | 入参 | 作用 |
|---|---|---|
| LoRA | `request.lora_request.lora_name` | LoRA 适配器隔离（[kv_cache_utils.py:498-512](d:\design\vllm\vllm\v1\core\kv_cache_utils.py)） |
| 多模态 | `request.mm_features[i].(identifier, offset)` | 同 placeholder 块若 MM 不同则 hash 不等（[kv_cache_utils.py:431-497](d:\design\vllm\vllm\v1\core\kv_cache_utils.py)） |
| `cache_salt` | `request.cache_salt`，仅首块 | 多租户隔离（[kv_cache_utils.py:560-561](d:\design\vllm\vllm\v1\core\kv_cache_utils.py)）；用户文档 [docs/design/prefix_caching.md](d:\design\vllm\docs\design\prefix_caching.md) |
| Prompt embeds | `sha256(tensor_data(block_prompt_embeds))` 缓存在 `request._prompt_embeds_per_block_hashes` | embeds 输入隔离（[kv_cache_utils.py:513-538](d:\design\vllm\vllm\v1\core\kv_cache_utils.py)） |

~~`need_extra_keys(request)` 快速判定：mm_features / lora_request / cache_salt 任一非空即需要。~~ **RESOLVED 2026-08-18**：`need_extra_keys` 已删除（在 [d:\design\vllm\vllm\](d:\design\vllm\vllm) 全树 grep 0 命中）；`generate_block_hash_extra_keys` 直接组装 4 类 keys 后判空返回 None（[kv_cache_utils.py:566-576](d:\design\vllm\vllm\v1\core\kv_cache_utils.py)）。

## 数据结构

### `BlockHashToBlockMap`（[block_pool.py:33-141](d:\design\vllm\vllm\v1\core\block_pool.py)）

```python
self._cache: dict[BlockHashWithGroupId, KVCacheBlock | dict[int, KVCacheBlock]]
```

- 单 block：`KVCacheBlock` 直接存
- 同 hash 多 block（碰撞或 cache_salt 不同的同 prefix tokens）：升级为 `dict[block_id, KVCacheBlock]`

| 方法 | 行号 | 用途 |
|---|---|---|
| `get_one_block(key)` | [61-73](d:\design\vllm\vllm\v1\core\block_pool.py) | 取任一匹配 block（O(1) hash + O(1) dict.next iter） |
| `contain(key, block_id)` | [74-87](d:\design\vllm\vllm\v1\core\block_pool.py) | **新增**：判断特定 block 是否在 cache 中 |
| `insert(key, block)` | [88-105](d:\design\vllm\vllm\v1\core\block_pool.py) | 单值 → 升级 dict；dict → append；不去重 |
| `pop(key, block_id)` | [106-135](d:\design\vllm\vllm\v1\core\block_pool.py) | evict 时按 block_id 摘除 |

> [!warning] CONTRADICTION（已知 trade-off）：注释 [block_pool.py:48-52](d:\design\vllm\vllm\v1\core\block_pool.py) 明示"We currently don't de-duplicate the blocks in the cache"——同 hash 块不去重以保 **block IDs 不变 + block tables append-only**。这是性能 vs 内存的明确取舍，与 [KVCacheManager.md `[!warning] CONTRADICTION`](../entities/KVCacheManager.md#blockpool-blockpoolpy130-509) 同源。

### `KVCacheBlock`（[kv_cache_utils.py:119-179](d:\design\vllm\vllm\v1\core\kv_cache_utils.py)）

`@dataclass(slots=True)`，字段现为 7 个：`block_id`, `ref_cnt`, `_block_hash`, `_block_hash_num_tokens`（**新增**，partial block 缓存的 token 数）, `prev_free_block`, `next_free_block`, `is_null`。~~`block_hash` setter 带 assert：一旦设过就不能再改~~ **RESOLVED 2026-08-18**：property setter 已改为显式 `set_block_hash(block_hash, num_tokens=None)` 方法（partial-tail 缓存需要携带 token 数，[kv_cache_utils.py:149-160](d:\design\vllm\vllm\v1\core\kv_cache_utils.py)）——`reset_hash()` 显式清空（evict 时调）语义不变。另新增 `KVCacheBlockCopy` NamedTuple（[kv_cache_utils.py:180-184](d:\design\vllm\vllm\v1\core\kv_cache_utils.py)，块拷贝任务描述）。

### `FreeKVCacheBlockQueue`（[kv_cache_utils.py:185-430](d:\design\vllm\vllm\v1\core\kv_cache_utils.py)）

**双向链表**，核心要点：
- 节点指针**直接挂在 `KVCacheBlock` 自身**——避免 `deque` wrapper GC 开销（注释 [kv_cache_utils.py:186-192](d:\design\vllm\vllm\v1\core\kv_cache_utils.py)）。
- `popleft(237) / popleft_n(274) / append(327) / append_n(371) / remove(307, O(1))` 全套操作；`remove` 取 O(1) 是这个数据结构存在的核心理由（注释 [kv_cache_utils.py:185-192](d:\design\vllm\vllm\v1\core\kv_cache_utils.py)）。
- LRU 顺序：**老块在头**，新归还的块加到尾。`free_blocks(ordered_blocks)` 调用方需 reverse 序传入（[kv_cache_manager.py:570-582](d:\design\vllm\vllm\v1\core\kv_cache_manager.py)）。
- `fake_free_list_head/tail` 哨兵简化分支判断（[kv_cache_utils.py:217-235](d:\design\vllm\vllm\v1\core\kv_cache_utils.py)）。

### `BlockPool` 主流程方法

| 方法 | 行号 | 关键逻辑 |
|---|---|---|
| `get_cached_block(block_hash, kv_cache_group_ids)` | [198-224](d:\design\vllm\vllm\v1\core\block_pool.py) | 按 group_id 列表逐个查 hash table，**任一 group miss 即全 miss** |
| `cache_full_blocks(request, blocks, num_cached_blocks, num_full_blocks, block_size, kv_cache_group_id)` | [225-343](d:\design\vllm\vllm\v1\core\block_pool.py) | 把已写满的块 → 设 `block_hash` → `insert` 到 hash table；可选 emit `BlockStored` 事件 |
| `emit_cached_block_events(...)` | [373-444](d:\design\vllm\vllm\v1\core\block_pool.py) | **新增**：full report 模式下为 prefix-cache 复用的块补发 `BlockStored`（PR #45261） |
| `cache_partial_block(...)` | [445-570](d:\design\vllm\vllm\v1\core\block_pool.py) | **新增**：sub-block prompt 尾部 partial 块缓存（partial-tail KV offload，PR #49502） |
| `get_new_blocks(num_blocks)` | [647-678](d:\design\vllm\vllm\v1\core\block_pool.py) | `popleft_n` + 若启 caching 调 `_maybe_evict_cached_block` |
| `_maybe_evict_cached_block(block)` | [679-701](d:\design\vllm\vllm\v1\core\block_pool.py) | 已 cached 块从 hash table `pop` + reset_hash + emit `BlockRemoved` |
| `touch(blocks)` | [702-718](d:\design\vllm\vllm\v1\core\block_pool.py) | prefix hit 时 ref_cnt+1，若 ref==0 在 free queue 中需 `remove` |
| `free_blocks(ordered_blocks)` | [719-744](d:\design\vllm\vllm\v1\core\block_pool.py) | ref_cnt-1，仅 ref==0 且非 null 加回 free queue 尾 |
| `evict_blocks(block_ids)` | [745-763](d:\design\vllm\vllm\v1\core\block_pool.py) | KV connector 用：强制把指定 block 从 hash table 摘除（不释放） |
| `reset_prefix_cache()` | [764-799](d:\design\vllm\vllm\v1\core\block_pool.py) | 仅当所有非 null 块都 free 才能成功；否则 `Failed to reset prefix cache because some blocks (N) are not freed yet` |
| `take_events()` | [821-831](d:\design\vllm\vllm\v1\core\block_pool.py) | 取并清空 `kv_event_queue` |

## 3 种 Coordinator 拓扑

由 `get_kv_cache_coordinator(...)` ([kv_cache_coordinator.py:923-979](d:\design\vllm\vllm\v1\core\kv_cache_coordinator.py)) 工厂决定：

```python
if not enable_caching:
    return KVCacheCoordinatorNoPrefixCache(...)
if len(kv_cache_config.kv_cache_groups) == 1:
    return UnitaryKVCacheCoordinator(...)
return HybridKVCacheCoordinator(...)
```

| Coordinator | 适用 | `find_longest_cache_hit` 行为 | 锚点 |
|---|---|---|---|
| `KVCacheCoordinatorNoPrefixCache` | `enable_caching=False`（含 0 group 的 attention-free 模型） | 直接返 `(空 blocks, 0)` | [kv_cache_coordinator.py:420-471](d:\design\vllm\vllm\v1\core\kv_cache_coordinator.py) |
| `UnitaryKVCacheCoordinator` | 单 KV group：所有层同类型（FullAttn 或 SW 或 MLA 或 ChunkedLocal） | 单调一次 single_type_managers[0].find_longest_cache_hit | [472-544](d:\design\vllm\vllm\v1\core\kv_cache_coordinator.py) |
| `HybridKVCacheCoordinator` | 多 KV group（hybrid 模型如 Gemma3 5:1 sw:full、LLaMA4 3:1 local:full） | **iterative fixed-point**：full attn 先扫 → 其它类型 reduce hit_length → 反复直到收敛；对齐粒度按 `_cache_hit_alignment_tokens`；另支持 **per-group 分歧命中**（`find_longest_cache_hit_per_group` [891-922](d:\design\vllm\vllm\v1\core\kv_cache_coordinator.py)，交给有能力的 KV connector，PR #46384 / #48425 / #50344） | [560-922](d:\design\vllm\vllm\v1\core\kv_cache_coordinator.py) |

`HybridKVCacheCoordinator.verify_and_split_kv_cache_groups()` ([kv_cache_coordinator.py:664-714](d:\design\vllm\vllm\v1\core\kv_cache_coordinator.py)) 把 manager class 相同的 group 合并扫（辅助结构 `SpecGroup` NamedTuple，[545-558](d:\design\vllm\vllm\v1\core\kv_cache_coordinator.py)），**FullAttention 排第一**——其左到右扫提供更紧的 hit 上界，减少其它类型的工作量。`_cache_hit_alignment_tokens` / `_align_cacheable` 保证多类型间的 hit 长度对齐（[kv_cache_coordinator.py:655-663, 715-726](d:\design\vllm\vllm\v1\core\kv_cache_coordinator.py)）。

> [!todo] VERIFY：`HybridKVCacheCoordinator` 的 fixed-point 算法注释仍引 issue [#32802](https://github.com/vllm-project/vllm/issues/32802)：复杂多 attn 类型混合时按 candidate length 迭代（[kv_cache_coordinator.py:790-800](d:\design\vllm\vllm\v1\core\kv_cache_coordinator.py)）；simple hybrid（1 full + 1 other）之外的收敛保证需重新确认（新 HEAD 下算法已被 PR #46384 partial-hit 重写）。

## `SingleTypeKVCacheManager.find_longest_cache_hit` 多态

~~`spec_manager_map` dict 路由~~ **RESOLVED 2026-08-18**：路由机制已重构为 **`KVCacheSpecRegistry`** 注册表 + `get_manager_for_kv_cache_spec(...)` 工厂（[single_type_kv_cache_manager.py:1852-1897](d:\design\vllm\vllm\v1\core\single_type_kv_cache_manager.py)，注册块在文件尾 [~1898-1953](d:\design\vllm\vllm\v1\core\single_type_kv_cache_manager.py)），按 `KVCacheSpec` 类型路由到 7 个具体 manager：

| Spec | Manager | `find_longest_cache_hit` 算法 | 锚点 |
|---|---|---|---|
| `FullAttentionSpec` / `MLAAttentionSpec` / `HiddenStateCacheSpec` | `FullAttentionManager` | **从左到右扫**，第一次 miss 即停；`drop_eagle_block=True` 多丢最后一块 | [680-833](d:\design\vllm\vllm\v1\core\single_type_kv_cache_manager.py)（flch [684](d:\design\vllm\vllm\v1\core\single_type_kv_cache_manager.py)） |
| `RSWASpec` | `RSWAManager`（**新增**，FullAttention 子类） | prefill 全局可见 + 仅保留最近 `rswa_window` 生成 token | [834-879](d:\design\vllm\vllm\v1\core\single_type_kv_cache_manager.py) |
| `SlidingWindowSpec` / `SlidingWindowMLASpec` | `SlidingWindowManager` | **从右到左扫**，找连续命中窗口；左侧用 `null_block` 填充；`use_eagle` 时窗口 +1 块（[891-894](d:\design\vllm\vllm\v1\core\single_type_kv_cache_manager.py)） | [880-1109](d:\design\vllm\vllm\v1\core\single_type_kv_cache_manager.py)（flch [903](d:\design\vllm\vllm\v1\core\single_type_kv_cache_manager.py)） |
| `ChunkedLocalAttentionSpec` | `ChunkedLocalAttentionManager` | 把 chunk 边界外的块标 null，仅扫窗口内 | [1110-1267](d:\design\vllm\vllm\v1\core\single_type_kv_cache_manager.py)（flch [1116](d:\design\vllm\vllm\v1\core\single_type_kv_cache_manager.py)） |
| `MambaSpec` | `MambaManager` | 从右到左找最后一个 hit（mamba 只需最后状态）；前面填 null；新 HEAD 下大幅扩展（mamba_cache_mode / align，1268-1762 共 ~500 行） | [1268-1762](d:\design\vllm\vllm\v1\core\single_type_kv_cache_manager.py)（flch [1295](d:\design\vllm\vllm\v1\core\single_type_kv_cache_manager.py)） |
| `CrossAttentionSpec` | `CrossAttentionManager` | **`raise NotImplementedError`**——cross attn 不支持 prefix cache | [1763-1825](d:\design\vllm\vllm\v1\core\single_type_kv_cache_manager.py)（raise [1823](d:\design\vllm\vllm\v1\core\single_type_kv_cache_manager.py)） |
| `SinkFullAttentionSpec` | `SinkFullAttentionManager` (FullAttention 子类) | 同 FullAttention，但额外预占 `sink_blocks` | [1826-1851](d:\design\vllm\vllm\v1\core\single_type_kv_cache_manager.py) |

eagle 多丢一块的语义保留，但传参方式变了：`find_longest_cache_hit` 签名现在接 `drop_eagle_block: bool`（[single_type_kv_cache_manager.py:684-695](d:\design\vllm\vllm\v1\core\single_type_kv_cache_manager.py)），SlidingWindow 端在窗口块数上 +1（[891-894, 1028](d:\design\vllm\vllm\v1\core\single_type_kv_cache_manager.py)）。`Scheduler.use_eagle` 流到 `KVCacheManager.use_eagle` 的链路锚点更新为 [sched/scheduler.py:247-266, 282](d:\design\vllm\vllm\v1\core\sched\scheduler.py)。

`KVCacheBlocks` 上的两个关键 helper：

- `__add__(other)`（[kv_cache_manager.py:56-64](d:\design\vllm\vllm\v1\core\kv_cache_manager.py)）—— 合并 prefix hit 与新分配的两段 block list；与 MindIE 的 "append blocks" 角色等价。
- `Coordinator.find_longest_cache_hit` + `KVCacheManager.get_computed_blocks`（[kv_cache_manager.py:232-299](d:\design\vllm\vllm\v1\core\kv_cache_manager.py)）—— "find longest common prefix" 角色；返 `(KVCacheBlocks, num_computed_tokens, shared_prefix_boundary)` 给 Scheduler。

> 注意：vLLM 没有 `KVCacheBlocks.append_blocks` / `find_longest_common_prefix` 同名方法，对应实现是 `__add__` / `find_longest_cache_hit` / `get_num_common_prefix_blocks`。

## Scheduler / EngineCore 集成

### EngineCore 启动序

[engine/core.py:223-232](d:\design\vllm\vllm\v1\engine\core.py)：

```python
self.request_block_hasher: Callable[[Request], list[BlockHash]] | None = None
if vllm_config.cache_config.enable_prefix_caching or kv_connector is not None:
    caching_hash_fn = get_hash_fn_by_name(
        vllm_config.cache_config.prefix_caching_hash_algo
    )
    init_none_hash(caching_hash_fn)
    self.request_block_hasher = get_request_block_hasher(
        hash_block_size, caching_hash_fn
    )
```

> synthesis：**只要有 KV connector 就强制启用 hash 计算**（即便 `enable_prefix_caching=False`）——因为 connector 的 `get_num_new_matched_tokens` 也按 hash 命中查询。这是 prefix cache 与 P/D 解耦的边界。

### Scheduler 内的查询/分配链

| 阶段 | 锚点 | 行为 |
|---|---|---|
| init | [sched/scheduler.py:277-287](d:\design\vllm\vllm\v1\core\sched\scheduler.py) | 创建 KVCacheManager，把 `enable_caching=cache_config.enable_prefix_caching` 直接喂入（现另穿 `max_in_flight_tokens` / `num_prefill_lookahead` / `watermark`） |
| WAITING 新请求查命中 | [sched/scheduler.py:866-880](d:\design\vllm\vllm\v1\core\sched\scheduler.py) | `kv_cache_manager.get_computed_blocks(request)`（返 3 元组，含 `shared_prefix_boundary`） |
| Connector 远端命中 | [sched/scheduler.py:826-865](d:\design\vllm\vllm\v1\core\sched\scheduler.py) | `connector.get_num_new_matched_tokens(request, block_aligned_local)` 加到 `num_external_computed_tokens`；新 HEAD 下要先扣 partial-tail（sub-block 尾部本地命中，PR #49502） |
| 分配 slots | [sched/scheduler.py:1033-1041](d:\design\vllm\vllm\v1\core\sched\scheduler.py) | `allocate_slots(request, num_new_tokens, num_new_computed_tokens=local, new_computed_blocks=local_blocks, ...)` |
| reset_prefix_cache | [sched/scheduler.py:2511-2582](d:\design\vllm\vllm\v1\core\sched\scheduler.py) | RLHF / benchmark 用；可选 `reset_running_requests` 强制抢占 |
| stats | [sched/scheduler.py:130, 146-147, 1067-1078, 2592-2613](d:\design\vllm\vllm\v1\core\sched\scheduler.py) | `prefix_cache_stats` (本地，现经 `kv_cache_manager.make_prefix_cache_stats()`) + `connector_prefix_cache_stats` (远端) 双轨 |

`KVCacheManager.allocate_slots` 内的关键 trade-offs（[kv_cache_manager.py:347-569](d:\design\vllm\vllm\v1\core\kv_cache_manager.py)）：

- `delay_cache_blocks`：P/D 异步拉 KV 时**跳过 cache_blocks**（[kv_cache_manager.py:355, 375, 554](d:\design\vllm\vllm\v1\core\kv_cache_manager.py)）——避免把"还在传输的块"加进 hash table。
- `num_tokens_to_cache = min(...)`（[kv_cache_manager.py:562-566](d:\design\vllm\vllm\v1\core\kv_cache_manager.py)）——**仅 cache 已最终化（finalized）的 token**，spec decode 草稿 token 不进 cache。
- `max_cache_hit_length = request.num_tokens - 1`（[kv_cache_manager.py:257-262](d:\design\vllm\vllm\v1\core\kv_cache_manager.py)）——所有 token 全命中时仍要保留最后 1 token 重算以拿 logits。
- pooling / prompt logprobs 请求经 `prefix_cache_lookup_enabled`（`request.skip_reading_prefix_cache`）跳过命中查询（[kv_cache_manager.py:217-219](d:\design\vllm\vllm\v1\core\kv_cache_manager.py)）。

## 兼容性

### 默认值与可选 hash 算法

| 配置 | 默认 | 锚点 |
|---|---|---|
| `enable_prefix_caching` | **`True`**（默认开） | [config/cache.py:107](d:\design\vllm\vllm\config\cache.py) |
| `prefix_caching_hash_algo` | `"sha256"` | [config/cache.py:109](d:\design\vllm\vllm\config\cache.py) |
| `block_size` | 16（`DEFAULT_BLOCK_SIZE`；现为 pydantic Field，用户是否显式指定另有 `user_specified_block_size` 跟踪） | [config/cache.py:59-66](d:\design\vllm\vllm\config\cache.py) |
| `enable_kv_cache_events` | False（按 `kv_events_config`） | [sched/scheduler.py:117-120](d:\design\vllm\vllm\v1\core\sched\scheduler.py) |

`compute_hash` 把 `enable_prefix_caching` / `prefix_caching_hash_algo` 列进 `ignored_factors`（[config/cache.py:221-260](d:\design\vllm\vllm\config\cache.py)）——开关 prefix cache **不会**改变编译图哈希，可热切换不重编译。

### 兼容矩阵

| 特性 | 状态 | 锚点 / 备注 |
|---|---|---|
| **spec decode (Eagle)** | ✅ `use_eagle` 流到 find_longest_cache_hit（现经 `drop_eagle_block` 参数），**多丢一块** | [single_type_kv_cache_manager.py:684-695, 891-894](d:\design\vllm\vllm\v1\core\single_type_kv_cache_manager.py)；test [test_prefix_caching.py](d:\design\vllm\tests\v1\core\test_prefix_caching.py)（行号未按新 HEAD 复核） |
| **chunked prefill** | ✅ 透明：每个 chunk 内 `cache_blocks` 都做一次 | scheduler 不区分 chunk 与全 prompt；test [test_hybrid_chunked_prefill.py](d:\design\vllm\tests\v1\e2e\test_hybrid_chunked_prefill.py) |
| **hybrid memory（hybrid attn）** | ✅ `HybridKVCacheCoordinator` 专门处理，现支持 **partial prefix cache hit**（PR #46384） | [kv_cache_coordinator.py:560-922](d:\design\vllm\vllm\v1\core\kv_cache_coordinator.py)；test [test_prefix_caching.py](d:\design\vllm\tests\v1\core\test_prefix_caching.py)（行号未按新 HEAD 复核） |
| **MLA** | ✅ `MLAAttentionSpec → FullAttentionManager`（经 `KVCacheSpecRegistry` 注册） | [single_type_kv_cache_manager.py:~1898-1953](d:\design\vllm\vllm\v1\core\single_type_kv_cache_manager.py) |
| **sliding window** | ✅ `SlidingWindowManager` 专属算法（含 `SlidingWindowMLASpec`） | [single_type_kv_cache_manager.py:880-1109](d:\design\vllm\vllm\v1\core\single_type_kv_cache_manager.py) |
| **mamba** | ✅ 仅末尾状态，前面填 null；新 HEAD 增加 `mamba_block_size` / cache mode（`all` 等，[config/cache.py:145-174](d:\design\vllm\vllm\config\cache.py)） | [single_type_kv_cache_manager.py:1268-1762](d:\design\vllm\vllm\v1\core\single_type_kv_cache_manager.py)；test [e2e/general/test_mamba_prefix_cache.py](d:\design\vllm\tests\v1\e2e\general\test_mamba_prefix_cache.py) |
| **encoder-only** | N/A — encoder 不出 KV cache | `EncoderOnlyAttentionSpec`（[kv_cache_interface.py:720-721](d:\design\vllm\vllm\v1\kv_cache_interface.py)） |
| **cross attention** | ❌ `CrossAttentionManager` raise NotImplementedError（"does not support caching"） | [single_type_kv_cache_manager.py:1763-1825](d:\design\vllm\vllm\v1\core\single_type_kv_cache_manager.py) |
| **LoRA** | ✅ extra_keys 加入 `lora_name` 隔离 | [kv_cache_utils.py:498-512](d:\design\vllm\vllm\v1\core\kv_cache_utils.py) |
| **MultiModal** | ✅ extra_keys 加入 `(mm_identifier, offset)` | [kv_cache_utils.py:431-497](d:\design\vllm\vllm\v1\core\kv_cache_utils.py) |
| **prompt embeds** | ✅ extra_keys 加入 sha256(embeds) | [kv_cache_utils.py:513-538](d:\design\vllm\vllm\v1\core\kv_cache_utils.py) |
| **cache salt** | ✅ extra_keys 首块加入 salt | [kv_cache_utils.py:560-561](d:\design\vllm\vllm\v1\core\kv_cache_utils.py) |
| **pooling models with all pooling / prompt logprobs** | 跳过命中查询（`prefix_cache_lookup_enabled` 检查 `request.skip_reading_prefix_cache`） | [kv_cache_manager.py:217-219](d:\design\vllm\vllm\v1\core\kv_cache_manager.py) |
| **DCP / PCP（context parallel）** | ✅ `block_size *= dcp_world_size`（Unitary，[kv_cache_coordinator.py:510-513](d:\design\vllm\vllm\v1\core\kv_cache_coordinator.py)）；**Hybrid 现接受 DCP**、仅 `assert pcp_world_size == 1`（[kv_cache_coordinator.py:611](d:\design\vllm\vllm\v1\core\kv_cache_coordinator.py)）；SW / chunked-local / mamba manager 仍 assert dcp==pcp==1（[single_type_kv_cache_manager.py:918-919, 1170-1171, 1310-1311](d:\design\vllm\vllm\v1\core\single_type_kv_cache_manager.py)） |

## KV connector 协作

`KVConnectorBase_V1.get_num_new_matched_tokens(request, num_computed_tokens)` ([kv_connector/v1/base.py:475-509](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py)) 是远端命中协议 entry：

```python
ext_tokens, load_kv_async = connector.get_num_new_matched_tokens(
    request, block_aligned_local   # 已扣本地命中数（且扣掉 partial-tail 后按块对齐）
)
num_external_computed_tokens = ext_tokens
```

返 `None` 表示"我还没准备好"，scheduler 把请求挂回 skipped 队列（[sched/scheduler.py:835-841](d:\design\vllm\vllm\v1\core\sched\scheduler.py)）。新 HEAD 下若本地命中含 partial-tail（sub-block 尾部），远端命中需严格超过 partial-tail 才截断本地 tail 换远端加载（[sched/scheduler.py:843-864](d:\design\vllm\vllm\v1\core\sched\scheduler.py)，PR #49502）。

`SimpleCpuOffloadConnector.__init__` 读取并要求 `enable_prefix_caching=True`（[simple_cpu_offload_connector.py:71, 127](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\simple_cpu_offload_connector.py)）——CPU offload 必须叠加 prefix cache（与 `MindIE KV pool 必须叠加 prefix cache`（已删） 同结构）。（旧 pin 中该文件的 `reset_prefix_cache` 调用已不存在：在新 HEAD 该文件 grep `reset_prefix_cache` 0 命中。）

`SimpleKVOffloadManager` 的 `cpu_coordinator.find_longest_cache_hit` ([simple_kv_offload/manager.py:270](d:\design\vllm\vllm\v1\simple_kv_offload\manager.py)) 和 `cpu_pool.cached_block_hash_to_block.get_one_block(bhash)` ([simple_kv_offload/manager.py:509, 619](d:\design\vllm\vllm\v1\simple_kv_offload\manager.py)) 直接复用 `BlockHashToBlockMap` 在 CPU 端做 mirror —— **prefix cache hash 表是 KV offload 的共享数据结构**。

`OffloadingScheduler` 在 `enable_prefix_caching=True` 时初始化 `set()` 跟踪 cached blocks（[offloading/scheduler.py:545](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\offloading\scheduler.py)）。

> synthesis：vLLM 把 `BlockHashToBlockMap` **暴露给 connector** 作为复用基础——这是为什么 KV offload / KV transfer 都不需要重新算 hash。MindIE 的 KV pool 走 `MemPool.LookUp` Python pybind 反向调用，则要求 C++ `HashCombine` 与 Python `hash_combine` 逐位等价才能跨节点查到（原 `mindie/topics/prefix-cache.md` wiki 页已删除）。vLLM 全 Python 单语言**没有这个约束**。

## KV cache events

`enable_kv_cache_events=True` 时 `BlockPool.kv_event_queue` 收集三类事件，由 `Scheduler.make_stats` → `EventPublisher` 推送：

| 事件 | 何时 emit | 锚点 |
|---|---|---|
| `BlockStored` | `cache_full_blocks` 写入 hash table 时（builder `_build_block_stored_event` [344-372](d:\design\vllm\vllm\v1\core\block_pool.py)）；full report 模式下 prefix-cache 复用块也补发（`emit_cached_block_events`，PR #45261）；partial block 缓存时另发（[527-528](d:\design\vllm\vllm\v1\core\block_pool.py)） | [block_pool.py:331, 344-444, 527-528](d:\design\vllm\vllm\v1\core\block_pool.py) |
| `BlockRemoved` | `_emit_block_removed_events` 摘除时 | [block_pool.py:592-606](d:\design\vllm\vllm\v1\core\block_pool.py) |
| `AllBlocksCleared` | `reset_prefix_cache()` 成功时 | [block_pool.py:796](d:\design\vllm\vllm\v1\core\block_pool.py) |

`maybe_convert_block_hash` 控制 event 中 hash 是 `bytes` 还是 `int`（依 `VLLM_KV_EVENTS_USE_INT_BLOCK_HASHES` env，[kv_cache_utils.py:80-83](d:\design\vllm\vllm\v1\core\kv_cache_utils.py)）。事件现携带 per-group 的 spec kind + sliding window 元数据（`kv_cache_event_metadata`，PR #40984，[kv_cache_manager.py:177-183](d:\design\vllm\vllm\v1\core\kv_cache_manager.py)）。

## §跨子系统引用（§5 step 3 hidden cross-reference grep）

按 [AGENTS.md §5 step 3](../../AGENTS.md#5-ingest-工作流) 5 类全仓库 grep 强制执行。

### 1. 跨语言绑定（C++ ↔ Python）

vLLM v1 prefix cache **完全是 Python**——核心代码全在 [vllm/v1/core/](d:\design\vllm\vllm\v1\core/) 下，与 C++ kernel 无直接耦合。证据：

- **`d:\design\vllm\csrc\` 全 C++/CUDA 树 grep `prefix_cache|PrefixCache|cached_block_hash` 0 命中**（已用 Grep 验证）。
- **`d:\design\vllm\csrc\` 全 C++/CUDA 树 grep `BlockHash|hash_block_tokens` 0 命中**——所有 hash 计算与索引都在 Python 进程内。
- **kernels（[csrc/](d:\design\vllm\csrc/)）只暴露 attention / quant / MoE / norm 等算子**，不持有任何 prefix cache 状态。

> synthesis：与 `MindIE prefix cache 跨语言绑定`（已删）（C++ `PrefixCacheBlockAllocator` + Python `PrefixCachePlugin` 双段 + `MemPool` pybind 反向调用）形成强对比。vLLM 的设计选择是**全 Python 一段**，性能上靠 `dataclass(slots=True)` + 自维护 doubly linked list 减 GC。

### 2. 协作伙伴跨子系统引用

| 协作类 | grep 范围 | 命中数与位置 |
|---|---|---|
| `Scheduler` × `KVCacheManager.get_computed_blocks` | `d:\design\vllm\vllm\` Python 全树 | 调用点 [v1/core/sched/scheduler.py:448-451, 873](d:\design\vllm\vllm\v1\core\sched\scheduler.py)（新 HEAD 另有 connector 视角的 `get_computed_blocks_for_connector`）；`KVCacheManager` 仍是 Scheduler 唯一的 prefix cache 入口（详 [KVCacheManager.md](../entities/KVCacheManager.md)） |
| `Scheduler` × `KVCacheManager.allocate_slots` | `d:\design\vllm\vllm\` Python 全树 | RUNNING [v1/core/sched/scheduler.py:629-631](d:\design\vllm\vllm\v1\core\sched\scheduler.py) + WAITING [1033](d:\design\vllm\vllm\v1\core\sched\scheduler.py) |
| `KVConnectorBase_V1` × prefix cache | `d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\` 全树 | `enable_prefix_caching` **2 处**（[simple_cpu_offload_connector.py:56, 82-86](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\simple_cpu_offload_connector.py)、[offloading/scheduler.py:125](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\offloading\scheduler.py)）；`reset_prefix_cache` **1 处**（[simple_cpu_offload_connector.py:244](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\simple_cpu_offload_connector.py)） |
| `EngineCore` × prefix cache | `d:\design\vllm\vllm\v1\engine\core.py` | **1 处**（[L223-232](d:\design\vllm\vllm\v1\engine\core.py)）：启动时构造 `request_block_hasher`，**`enable_prefix_caching=True OR kv_connector 非空** 都触发 |
| `BlockHashToBlockMap` 全仓库 | `d:\design\vllm\` 全树 | **8 文件**：3 prod（[block_pool.py](d:\design\vllm\vllm\v1\core\block_pool.py) + [kv_cache_utils.py](d:\design\vllm\vllm\v1\core\kv_cache_utils.py) import + [simple_kv_offload/manager.py](d:\design\vllm\vllm\v1\simple_kv_offload\manager.py) 3 处）+ 5 tests（[test_prefix_caching.py](d:\design\vllm\tests\v1\core\test_prefix_caching.py) 23 处、[test_single_type_kv_cache_manager.py](d:\design\vllm\tests\v1\core\test_single_type_kv_cache_manager.py)、[test_scheduler.py](d:\design\vllm\tests\v1\core\test_scheduler.py)、[test_remote_prefill_lifecycle.py](d:\design\vllm\tests\v1\kv_connector\unit\test_remote_prefill_lifecycle.py)、[test_cache_pollution_prevention.py](d:\design\vllm\tests\v1\kv_connector\unit\test_cache_pollution_prevention.py)） |
| `BlockHash` / `hash_block_tokens` 全仓库 | `d:\design\vllm\vllm\` 全树 | **6 文件**：[v1/core/kv_cache_utils.py](d:\design\vllm\vllm\v1\core\kv_cache_utils.py)（定义）、[v1/core/block_pool.py](d:\design\vllm\vllm\v1\core\block_pool.py)、[v1/core/single_type_kv_cache_manager.py](d:\design\vllm\vllm\v1\core\single_type_kv_cache_manager.py)、[v1/core/kv_cache_coordinator.py](d:\design\vllm\vllm\v1\core\kv_cache_coordinator.py)、[v1/engine/core.py](d:\design\vllm\vllm\v1\engine\core.py)、[v1/request.py](d:\design\vllm\vllm\v1\request.py) + [distributed/kv_events.py](d:\design\vllm\vllm\distributed\kv_events.py) 用 `ExternalBlockHash`、[distributed/kv_transfer/kv_connector/v1/lmcache_mp_connector.py](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\lmcache_mp_connector.py) 引用 |

### 3. 配置 / IPC 共享数据结构

| 字段 / 协议 | 位置 | 所属层 |
|---|---|---|
| `CacheConfig.enable_prefix_caching: bool = True` | [config/cache.py:107](d:\design\vllm\vllm\config\cache.py) | 顶层配置 |
| `CacheConfig.prefix_caching_hash_algo: PrefixCachingHashAlgo = "sha256"` | [config/cache.py:109](d:\design\vllm\vllm\config\cache.py) | 顶层配置 |
| `BlockHash` 类型 | `NewType("BlockHash", bytes)` [kv_cache_utils.py:45](d:\design\vllm\vllm\v1\core\kv_cache_utils.py) — **不是 dataclass** | 数据类型 |
| `BlockHashWithGroupId` 类型 | `NewType("BlockHashWithGroupId", bytes)` [kv_cache_utils.py:50](d:\design\vllm\vllm\v1\core\kv_cache_utils.py)（pack/unpack helper [59-77](d:\design\vllm\vllm\v1\core\kv_cache_utils.py)） | 数据类型 |
| `ExternalBlockHash` | `bytes \| int` 联合体（KV events 传输用） [kv_cache_utils.py:55](d:\design\vllm\vllm\v1\core\kv_cache_utils.py) | KV event 协议 |
| `KVCacheEvent` 体系 | [distributed/kv_events.py](d:\design\vllm\vllm\distributed\kv_events.py)：`BlockStored` / `BlockRemoved` / `AllBlocksCleared` | 跨进程事件流 |
| 环境变量 `VLLM_KV_EVENTS_USE_INT_BLOCK_HASHES` | `maybe_convert_block_hash` [kv_cache_utils.py:80-83](d:\design\vllm\vllm\v1\core\kv_cache_utils.py) | 与外部 event 消费者对齐 hash 表示 |
| 环境变量 `PYTHONHASHSEED` | `init_none_hash` [kv_cache_utils.py:96-117](d:\design\vllm\vllm\v1\core\kv_cache_utils.py)：未设时用 `os.urandom(32)`；CBOR-based hash 函数会 warn | 跨进程 hash 一致性（仅 cbor 系列必须） |
| `enable_prefix_caching` 全仓库引用数 | 48+ 命中（**旧 pin 2026-04-18 统计**，新 HEAD 未重跑） | 通用配置开关 |

> synthesis：vLLM 把 `BlockHash` 直接定义为 `NewType[bytes]` 而非 dataclass —— 看似过度简化，但这正是性能关键：dict 查询直接用 bytes hash，**省去 dataclass 包装的 Python 对象层**。MindIE 用 `HashValue`（uint64）+ `unordered_map` 的 C++ 结构，性能数量级相当但跨语言成本更高。

### 4. 测试覆盖反查

| 测试文件 | 覆盖点 | 测试数 |
|---|---|---|
| [tests/v1/core/test_prefix_caching.py](d:\design\vllm\tests\v1\core\test_prefix_caching.py) | 主测：prefill / hybrid / eagle / mamba / MM / salt / LoRA / KV events / reset / evict / hash collision | **32 tests**（包含 `test_prefill_hybrid_model_eagle` / `test_prefill_hybrid_model_combinations[_eagle]` / `test_eagle_with_partial_blocks` / `test_eagle_with_sliding_window` / `test_kv_cache_events[*]` / `test_block_lookup_cache_*`） |
| [tests/v1/core/test_kv_cache_utils.py](d:\design\vllm\tests\v1\core\test_kv_cache_utils.py) | hash 算法、`generate_block_hash_extra_keys`（mm/lora/cache_salt/prompt_embeds）、`FreeKVCacheBlockQueue` | **41 tests** |
| [tests/v1/core/test_reset_prefix_cache_e2e.py](d:\design\vllm\tests\v1\core\test_reset_prefix_cache_e2e.py) | `Scheduler.reset_prefix_cache` e2e（含 `reset_running_requests`） | — |
| [tests/v1/core/test_single_type_kv_cache_manager.py](d:\design\vllm\tests\v1\core\test_single_type_kv_cache_manager.py) | 单类型 manager 的 `find_longest_cache_hit` 行为 | **7 tests** |
| [tests/v1/e2e/general/test_mamba_prefix_cache.py](d:\design\vllm\tests\v1\e2e\general\test_mamba_prefix_cache.py) | Mamba + prefix cache e2e | — |
| [tests/v1/e2e/test_hybrid_chunked_prefill.py](d:\design\vllm\tests\v1\e2e\test_hybrid_chunked_prefill.py) | hybrid + chunked prefill 共存 | — |
| [tests/v1/e2e/general/test_correctness_sliding_window.py](d:\design\vllm\tests\v1\e2e\general\test_correctness_sliding_window.py) | sliding window prefix cache 正确性 | — |
| [tests/v1/kv_connector/unit/test_remote_prefill_lifecycle.py](d:\design\vllm\tests\v1\kv_connector\unit\test_remote_prefill_lifecycle.py) | KV connector 命中与本地 prefix cache 协作 | — |
| [tests/v1/kv_connector/unit/test_cache_pollution_prevention.py](d:\design\vllm\tests\v1\kv_connector\unit\test_cache_pollution_prevention.py) | 防止 connector 提前 cache 未完成 KV transfer 的块（`delay_cache_blocks`） | — |
| [tests/v1/kv_connector/unit/test_invalid_blocks_correctness.py](d:\design\vllm\tests\v1\kv_connector\unit\test_invalid_blocks_correctness.py) | 失败 KV transfer 后的 prefix cache 状态 | — |
| [tests/v1/simple_kv_offload/test_scheduler.py](d:\design\vllm\tests\v1\simple_kv_offload\test_scheduler.py) | CPU offload 内 mirror `cached_block_hash_to_block` 行为 | — |
| [tests/v1/kv_offload/test_cpu_offloading.py](d:\design\vllm\tests\v1\kv_offload\test_cpu_offloading.py) | CPU offloading 集成 | — |
| [tests/v1/core/test_scheduler.py](d:\design\vllm\tests\v1\core\test_scheduler.py) | Scheduler 集成 prefix cache（27 处 `enable_prefix_caching`） | — |
| [tests/v1/metrics/test_stats.py](d:\design\vllm\tests\v1\metrics\test_stats.py) | `PrefixCacheStats` / `connector_prefix_cache_stats` 双轨 | — |
| [tests/models/language/pooling/test_auto_prefix_cache_support.py](d:\design\vllm\tests\models\language\pooling\test_auto_prefix_cache_support.py) | pooling 模型 `skip_reading_prefix_cache` 路径 | — |

### 5. doc / config / yaml 反查

| 路径 | 命中内容 |
|---|---|
| [docs/features/automatic_prefix_caching.md](d:\design\vllm\docs\features\automatic_prefix_caching.md) | 用户主文档（"APC" 概念 / 启用方式 / 工作负载场景 / 限制） |
| [docs/design/prefix_caching.md](d:\design\vllm\docs\design\prefix_caching.md) | 设计文档（hash 链结构图、`KVCacheBlock` 字段、Block Pool / Free Queue / Cache blocks / Request blocks 4 元件、APC operations 工作流、Cache Isolation `cache_salt`） |
| [docs/design/hybrid_kv_cache_manager.md](d:\design\vllm\docs\design\hybrid_kv_cache_manager.md) | hybrid 模型 KV cache 管理设计 |
| [docs/usage/v1_guide.md](d:\design\vllm\docs\usage\v1_guide.md) | v1 用户指南 |
| [docs/features/disagg_prefill.md](d:\design\vllm\docs\features\disagg_prefill.md) | PD 解耦下的 prefix cache 行为 |
| [docs/usage/security.md](d:\design\vllm\docs\usage\security.md) | `cache_salt` 多租户隔离用法 |
| [examples/offline_inference/automatic_prefix_caching.py](d:\design\vllm\examples\offline_inference\automatic_prefix_caching.py) | 官方示例：长文档 + 多 query 复用 |
| [examples/offline_inference/prefix_caching.py](d:\design\vllm\examples\offline_inference\prefix_caching.py) | 基础示例 |
| [examples/offline_inference/prefix_caching_flexkv.py](d:\design\vllm\examples\offline_inference\prefix_caching_flexkv.py) | FlexKV 后端 prefix caching |
| **csrc / kernel 反查** | **`d:\design\vllm\csrc\` 全 C++/CUDA 树 grep `prefix_cache\|PrefixCache\|cached_block_hash\|BlockHash\|hash_block_tokens` 0 命中**（**真**）—— 与 §1 跨语言绑定结论一致 |

## 跨项目对照（synthesis）

> 三方对比详见 [comparison/topics/kv-cache.md](../../comparison/topics/kv-cache.md) 与 [comparison/dimensions.md §dim-prefix-cache](../../comparison/dimensions.md)。

| 维度 | MindIE | vLLM | SGLang |
|---|---|---|---|
| **数据结构** | `unordered_map<HashValue, BlockId>`（hash table） + LRU evictor | `BlockHashToBlockMap`（hash table）= `dict[BlockHashWithGroupId, KVCacheBlock \| dict[block_id, KVCacheBlock]]` ([block_pool.py:33](d:\design\vllm\vllm\v1\core\block_pool.py)) | **Trie / Radix tree**（`RadixCache`） |
| **抽象边界** | C++ `PrefixCacheBlockAllocator` + Python `PrefixCachePlugin` **两段** | **全 Python 一段**：`KVCacheManager` → `KVCacheCoordinator` → `BlockPool` + `SingleTypeKVCacheManager` 多态 | Python `BasePrefixCache` 多实现（RadixCache / HiRadixCache / RadixCacheCpp 等） |
| **Hash 算法** | C++ boost-style `HashCombine` + Python 镜像 `hash_combine` | `hash_block_tokens(hash_function, parent, tokens, extra_keys)` ([kv_cache_utils.py:577](d:\design\vllm\vllm\v1\core\kv_cache_utils.py))；4 种可选 hash function（[hashing.py:82-100](d:\design\vllm\vllm\utils\hashing.py)），default `sha256` | trie 节点路径分裂，无单点 hash |
| **Eviction** | LRU evictor（C++） | LRU 双向链表 `FreeKVCacheBlockQueue` ([kv_cache_utils.py:185-430](d:\design\vllm\vllm\v1\core\kv_cache_utils.py)) | LRU + radix prefix 共享 |
| **PD/远端命中** | `GetRemoteComputedBlockIds` → `MemPool.LookUp`（C++→Python pybind） | `KVConnectorBase_V1.get_num_new_matched_tokens` ([base.py:475-509](d:\design\vllm\vllm\distributed\kv_transfer\kv_connector\v1\base.py))；`SimpleCpuOffloadConnector` 强制 enable_prefix_caching | HiRadixCache + HiCache Storage 后端 |
| **触发开关** | `plugin_params={plugin_type:prefix_cache}` + `enableCaching` | `--enable-prefix-caching`（**默认 True**）+ `--prefix-caching-hash-algo` | `--disable-radix-cache`（默认 enabled） |
| **Hybrid 模型** | 单一 `BlockSpaceManager` | `HybridKVCacheCoordinator` 显式 fixed-point 算法 + partial per-group hit ([kv_cache_coordinator.py:560-922](d:\design\vllm\vllm\v1\core\kv_cache_coordinator.py)) | radix + 多 KV 池 |
| **Spec decode (Eagle)** | MTP 不在 prefix_cache 路径上互斥（plugin_utils） | `use_eagle` 流到 `find_longest_cache_hit`（`drop_eagle_block` 参数），**主动多丢 1 块** ([single_type_kv_cache_manager.py:684-695](d:\design\vllm\vllm\v1\core\single_type_kv_cache_manager.py)) | 类似处理 |
| **多模态隔离** | extra_hash 字段 | `extra_keys` 拼 `(mm_identifier, offset)` ([kv_cache_utils.py:431-497](d:\design\vllm\vllm\v1\core\kv_cache_utils.py)) | trie 节点 metadata |
| **测试覆盖** | C++ unit ≥ 22 + Python unit + 集成 + scheduler 策略层 | **80+ tests**（test_prefix_caching.py 32 + test_kv_cache_utils.py 41 + e2e + KV connector + offload） | mem_cache/ unit + integration |

> synthesis：vLLM 与 MindIE 都用 **hash table**，差别仅在抽象层数（vLLM 全 Python 单段；MindIE C++/Python 双段）；**SGLang 是唯一用 trie 的**。vLLM 的设计选择是**单语言代价换简化**——`BlockHashToBlockMap` 直接被 KV connector / KV offload mirror 复用（[simple_kv_offload/manager.py](d:\design\vllm\vllm\v1\simple_kv_offload\manager.py)），无需任何跨语言 hash 等价性约束。

## Increment 2026-08-18 (vLLM 5f7fab88 → d29dc3ab)

prefix cache 相关 5 个核心文件在增量区间全部大改（kv_cache_utils.py 1693 → 2343、single_type_kv_cache_manager.py 1133 → 1953、kv_cache_coordinator.py 591 → 979、block_pool.py 509 → 831、kv_cache_manager.py 552 → 887 行）。本页主链路锚点（hash 计算 / 数据结构 / coordinator / manager / Scheduler / EngineCore 集成 / KV connector / KV events）已按新 HEAD 修正。要点：

- **manager 路由重构**：`spec_manager_map` dict → `KVCacheSpecRegistry` 注册表 + `get_manager_for_kv_cache_spec` 工厂（[d:\design\vllm\vllm\v1\core\single_type_kv_cache_manager.py:1852-1953](d:\design\vllm\vllm\v1\core\single_type_kv_cache_manager.py)）；新增 `RSWAManager`（[834-879](d:\design\vllm\vllm\v1\core\single_type_kv_cache_manager.py)）。
- **hybrid 模型 partial prefix cache hit**（PR #46384 / #48425 / #50344 / #51843）：`HybridKVCacheCoordinator.find_longest_cache_hit` 重写为按 candidate length 迭代 + `find_longest_cache_hit_per_group`（[d:\design\vllm\vllm\v1\core\kv_cache_coordinator.py:757-922](d:\design\vllm\vllm\v1\core\kv_cache_coordinator.py)）。
- **Marconi 式选择性 cache 保留**（PR #47782 / #52216）：`get_computed_blocks` 返回 3 元组（新增 `shared_prefix_boundary`）；`prefix_cache_retention_interval` 成为正式参数、默认 0。
- **partial-tail KV offload**（PR #49502）：`BlockPool.cache_partial_block`（[d:\design\vllm\vllm\v1\core\block_pool.py:445-570](d:\design\vllm\vllm\v1\core\block_pool.py)）+ `KVCacheBlock._block_hash_num_tokens` 字段 + scheduler 端 partial-tail 与远端命中的裁剪逻辑（[d:\design\vllm\vllm\v1\core\sched\scheduler.py:843-864](d:\design\vllm\vllm\v1\core\sched\scheduler.py)）。
- **eagle 传参方式变更**：`find_longest_cache_hit` 签名接 `drop_eagle_block: bool`（替代直接读 `use_eagle`）。
- **删除项**：`need_extra_keys`（extra keys 判定内联进 `generate_block_hash_extra_keys`）、`TQFullAttentionSpec`、`SimpleCpuOffloadConnector.reset_prefix_cache` 调用（三者在 [d:\design\vllm\vllm\](d:\design\vllm\vllm) 全树 grep 均 0 命中）。
- **hash 粒度语义**：hash 固定按 `hash_block_size` 链式计算，粗粒度 group 经 `BlockHashListWithBlockSize` 复用（[d:\design\vllm\vllm\v1\core\kv_cache_utils.py:671-683, 2310](d:\design\vllm\vllm\v1\core\kv_cache_utils.py)）。
- **KV events 增强**：full report 模式补发复用块 `BlockStored`（PR #45261）、per-group spec kind / sliding window 元数据（PR #40984）。

> [!todo] VERIFY（本次增量未覆盖范围）：§跨子系统引用 与 §测试覆盖反查 中的 **grep 命中数统计**（如 `BlockHashToBlockMap` 8 文件、`enable_prefix_caching` 48+ 命中、test_prefix_caching.py 32 tests 等）与 **测试文件行号**均为旧 pin（2026-04-18）数据，新 HEAD 未重跑；docs/ 锚点亦未逐一复核。故本页 `status: stale`。核心代码锚点（vllm/v1/core/ + engine/core.py + request.py + kv_connector base）已全部按 d29dc3ab 复核。

## Notes / Caveats

> [!warning] CONTRADICTION（已记录的取舍）：`BlockHashToBlockMap` 不去重同 hash 块（[block_pool.py:48-52](d:\design\vllm\vllm\v1\core\block_pool.py)，NOTE #1 注释新 HEAD 下行号未变），保留 block table append-only 不变性。已在 [KVCacheManager.md](../entities/KVCacheManager.md) 记录同源 marker。

> [!todo] VERIFY：`HybridKVCacheCoordinator.find_longest_cache_hit` 在**复杂多 attn 类型**（>2 group）+ `use_eagle` 下仍存在 EAGLE spiral block dropping 问题（[kv_cache_coordinator.py:790-800](d:\design\vllm\vllm\v1\core\kv_cache_coordinator.py) 注释引 issue [#32802](https://github.com/vllm-project/vllm/issues/32802)）。新 HEAD 下算法已被 partial-hit 重写，收敛保证范围需重新确认。生产环境若用 3+ attn type 模型 + spec decode 需关注。

> [!todo] VERIFY：`prefix_caching_hash_algo` 在 `sha256_cbor` / `xxhash_cbor` 模式下，未设 `PYTHONHASHSEED` 会 warn 但**不报错**（[kv_cache_utils.py:96-117](d:\design\vllm\vllm\v1\core\kv_cache_utils.py)）。跨进程 KV events 消费方若依赖可重现 hash，必须显式设 `PYTHONHASHSEED`——这是部署时的常见坑。

> [!todo] VERIFY：`KVCacheBlocks.__add__` 不验证 `len(self.blocks) == len(other.blocks)`（[kv_cache_manager.py:56-64](d:\design\vllm\vllm\v1\core\kv_cache_manager.py) 用 `zip` 截短）——hybrid 多 group 下若 group 数不一致会**静默丢块**。当前调用方（`Scheduler` 与 `KVCacheManager`）保证 group 数一致，但若未来开放外部直接构造需谨慎。

> [!warning] CONTRADICTION（命名陷阱）：用户文档 [docs/features/automatic_prefix_caching.md](d:\design\vllm\docs\features\automatic_prefix_caching.md) 说 "APC in general does not reduce the performance of vLLM"，但 `enable_prefix_caching=True` 时**仍要付 hash 计算开销**（每 token 追加都跑 `request_block_hasher`，[engine/core.py:229-231](d:\design\vllm\vllm\v1\engine\core.py)、[request.py:273-276](d:\design\vllm\vllm\v1\request.py)）。短 prompt + 全 unique workload 时这是净开销，文档没明示。

> [!todo] VERIFY：`reset_prefix_cache()` 失败条件是"非 null 块未全 free"（[block_pool.py:764-799](d:\design\vllm\vllm\v1\core\block_pool.py)），但 `Scheduler.reset_prefix_cache(reset_running_requests=True)` 通过抢占所有 running 请求来强制满足该条件（[sched/scheduler.py:2511-2582](d:\design\vllm\vllm\v1\core\sched\scheduler.py)）；这意味着 RLHF 场景下 reset 的成本与 running batch 大小线性相关——大 batch 下 reset 延迟需测试覆盖。

## See also

- [vllm/entities/KVCacheManager.md](../entities/KVCacheManager.md) — `KVCacheManager` / `BlockPool` / `KVCacheCoordinator` / `KVCacheSpec` 实体页（**本主题主源**）
- [vllm/entities/Scheduler.md](../entities/Scheduler.md) — Scheduler 集成 / `reset_prefix_cache` / connector 命中协议
- [comparison/topics/prefix-cache.md](../../comparison/topics/prefix-cache.md) — 三家 prefix cache 深度对比（11 子维度，本页是 vLLM 端深化）
- [comparison/topics/kv-cache.md](../../comparison/topics/kv-cache.md) — 父级 KV cache 对比页（含 prefix-cache cell）
- [comparison/dimensions.md §dim-prefix-cache](../../comparison/dimensions.md) — 跨项目维度
- 上游产品文档：[docs/features/automatic_prefix_caching.md](d:\design\vllm\docs\features\automatic_prefix_caching.md)、[docs/design/prefix_caching.md](d:\design\vllm\docs\design\prefix_caching.md)、[docs/design/hybrid_kv_cache_manager.md](d:\design\vllm\docs\design\hybrid_kv_cache_manager.md)
