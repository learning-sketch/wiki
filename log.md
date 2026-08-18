# Wiki Activity Log

> **Append-only**。每条以 `## [YYYY-MM-DD] <op> | <scope> | <summary>` 开头，方便 `Grep` 检索。
> 操作类型：`bootstrap` / `ingest` / `query` / `lint` / `verify` / `compare` / `refactor` / `lint-fix` / `schema-update` / `archive`。

---

## Archive Index

> 当 `log.md` 行数 > **1500** 或字节 > **110 KB** 时触发轮转（详见 [AGENTS.md §11 Log rotation](AGENTS.md)）。
> 历史 entries 转储到 `wiki/log-archive/log-NNN.md`（append-only，从不重写）。
> 主 `log.md` 保留：schema header + 本表 + 最近 ~10 条 entries（涵盖 ingest/verify/compare/lint）。

| Archive 文件 | 期间 | entries | 大小 | 主要内容 |
|---|---|---|---|---|
| [log-archive/log-001.md](log-archive/log-001.md) | 2026-04-17 | 21 | ~70 KB | bootstrap 全套 + L1 ingest 三家 + scheduler 4 文件批 + kv-cache 4 文件批 + comparison topic 3 批（chunked-prefill / flashcomm / cp-sp）+ full health-check + verify flashcomm |
| [log-archive/log-002.md](log-archive/log-002.md) | 2026-04-18（早段） | 17 | ~110 KB | compare pd-disaggregation + verify + sync-schedule + distributed + lint-fix p0-p2/p3-p4 + async-schedule + verify + 2 schema-update（anchor-driven cross-check / hidden cross-reference）+ ingest mindie entities P0/P1 + ingest mindie topics P0 + verify moe.md + ingest mindie structured-output / prefix-cache + ingest comparison speculative-decoding |
| [log-archive/log-003.md](log-archive/log-003.md) | 2026-04-18（中段） | 4 | ~50 KB | archive log rotation 001/002 + pin source-versions（initial pin + §12 增量工作流 schema 沉淀）+ lint post pin/rotation health check（含 §11/§12 首次集成验证）+ verify comparison/topics/speculative-decoding（5 markers 全部 RESOLVED）|
| [log-archive/log-004.md](log-archive/log-004.md) | 2026-04-18（晚段） | 8 | ~45 KB | ingest mindie modules/connector + lint post-batch acceptance（mindie P1 batch）+ ingest vllm topics P1（spec-decode-eagle / prefix-cache / kv-connector）+ lint vllm topics P1 batch acceptance + verify vllm entities（EngineCoreClient + GPUModelRunner）+ ingest vllm entities P0+P1 |
| [log-archive/log-005.md](log-archive/log-005.md) | 2026-04-18（中段补移） | 4 | ~40 KB | batch verify vLLM P0 + ingest sglang entities P0 + compare multiproc-ipc + archive log rotation 003 + compare executor-worker + compare engine-architecture |
| [log-archive/log-006.md](log-archive/log-006.md) | 2026-04-19（早段） | 3 | ~30 KB | ingest sglang modules P0 + lint post compare-prefix-cache + compare prefix-cache across all 3 projects |
| [log-archive/log-007.md](log-archive/log-007.md) | 2026-04-19（中段） | 3 | ~37 KB | archive log rotation 004 + ingest sglang modules P3（function_call+constrained+parser+sampling+weight_sync+checkpoint_engine）+ ingest sglang modules P1+P2（speculative+eplb+elastic_ep+compilation+connector） |
| [log-archive/log-008.md](log-archive/log-008.md) | 2026-04-19（晚段） | 4 | ~38 KB | archive log rotation 007 + ingest sglang modules Turn 3+4+5 mega-batch（8 模块完成 sglang/modules 30/30 全覆盖）+ archive log rotation 006 + ingest sglang modules Turn 2 small-rollup（multiplex+tokenizer+batch_invariant_ops+batch_overlap+dllm+ray） |
| [log-archive/log-009.md](log-archive/log-009.md) | 2026-04-19（全部滞留主 log 的 8 条） | 8 | ~98 KB | archive rotation 005/008 记录 + ingest sglang modules P4 batch + verify sglang modules 36 页 / entities 5 页 batch + ingest sglang topics P0 + lint post topics-P0 + compare moe & scheduler-architecture |

> **如需检索归档内容**：直接 `Grep` / `Read` 对应的 `log-archive/log-NNN.md` 文件。Schema-update entries（AGENTS.md 演化轨迹）现都在 `log-002.md`，对未来 agent 仍可索引。

---

## [2026-08-10] lint-fix | cross | remove mindie wiki + neutralize dead links

**触发**：用户删除 `mindie/` 目录后要求清理死链。

**变更**
1. 删除整个 `mindie/` 子树（22 页）——见同分支先前 commit。
2. 活跃 wiki 页（排除 `log-archive/`，archive 不可重写）中所有指向 `mindie/**` 的 markdown 链接与 `related:` 条目已清除或标为 N/A。
3. 入口页同步：
   - [README.md](README.md)：去掉 mindie 入口与目录树
   - [index.md](index.md)：MindIE 项目行 / 关键词表改为 `N/A（wiki removed）`
   - [comparison/index.md](comparison/index.md)：项目跳转去掉 mindie
   - [AGENTS.md](AGENTS.md)：§8/§10/§11/§12 示例与映射注明 `wiki/mindie/` 已删
   - [source-versions.md](source-versions.md)：working-tree caveat 去掉失效 wiki 链接
4. comparison / vllm / sglang 页的 See also / related / inline wiki 回链已清理；**对比页 MindIE 列的源码锚点（`d:\design\MindIE-LLM\...`）保留**。

**未改**
- `log-archive/log-00N.md`（§11 不可重写；历史死链保留）
- 对比表三列结构（MindIE | vLLM | SGLang）——仅去掉失效 wiki 页链接


## [2026-08-10] increment | sglang | pin 34fef07a → 06f32bab | 7 new modules + core entity re-ingest

**触发**：用户要求把 `sglang/` wiki 更新到最新，并按 [AGENTS.md §12](AGENTS.md) 增量工作流执行。

**Diff 摸底**（源：GitHub `sgl-project/sglang`）
- pin `34fef07a` → HEAD `06f32bab`（2026-08-09）：**4669** commits
- `python/sglang/srt`：**1929** files changed（+347k / -131k）
- 顶层包：34 → **41**（新增 `arg_groups` / `kv_canary` / `platforms` / `plugins` / `session` / `state_capturer` / `weight_cache`）

**本轮产出**
1. **新建模块页 7**：`weight_cache` / `kv_canary` / `arg_groups` / `platforms` / `plugins` / `session` / `state_capturer`
2. **re-ingest**：[entities/Scheduler.md](sglang/entities/Scheduler.md)、[entities/TokenizerManager.md](sglang/entities/TokenizerManager.md)、[topics/scheduler-mixins.md](sglang/topics/scheduler-mixins.md)
3. **re-anchor verify**：Engine / TpModelWorker / DataParallelController / modules/managers
4. **overview + index** 同步到 41 包与新模块 DONE 行；高 churn 未深读页标 `status: stale` + VERIFY
5. **[source-versions.md](source-versions.md)** SGLang pin → `06f32bab`；History append

**显式未做（留 follow-up）**
- 深 verify：`mem_cache` / `speculative` / `model_executor` / `layers` / `models` / `multimodal` 等（已标 stale）
- comparison 页 SGLang 单元格行号未批量重锚
- MindIE / vLLM pin 未动（本环境未检出源码）

**Lessons**
- 跨 ~4 个月 / 4k+ commits 时，§12 应先做**顶层包 inventory + 核心 entity re-ingest**，其余标 stale，避免假 verified。


## [2026-08-10] ingest | sglang topics kv-cache | re-ingest against HEAD 06f32bab

**触发**：用户 `ingest sglang topics kv-cache`。

**范围（源）**
- 主链：`managers/scheduler.py:516+` → `mem_cache/kv_cache_builder.py` → `mem_cache/registry.py`
- 抽象 / 实现：`base_prefix_cache`、`allocator/`、`radix_*`、`hiradix`、`unified_*`、`chunk_cache`、`pure_swa_*`、`storage/{lmcache,flexkv}`、`hicache_storage`、`backend_factory`、`session/streaming_session`、`multimodal_cache`、`evict_policy`、`environ`/`server_args`

**重大漂移（相对旧页）**
1. `Scheduler.init_cache_with_memory_pool` **已删除** → `kv_cache_builder.build_kv_cache` + `registry.create_tree_cache`
2. `HiMambaRadixCache` / `session_aware_cache.py` / `unified_cache_components/` **已删**
3. hybrid SWA/SSM (+ hierarchical/DSA) 默认汇入 **`UnifiedRadixCache`**；`SWARadixCache`/`MambaRadixCache` 类仍在但工厂 0 构造
4. `StreamingSession` 迁 `session/`；HiCache storage 注册名 **9**；新增 FlexKV；`allocator.py` → `allocator/` 包
5. `kv_canary.attach_radix_cache(tree_cache)` 新协作点

**页面**
- 重写 [sglang/topics/kv-cache.md](sglang/topics/kv-cache.md)（`status:verified`，`verified_against:2026-08-10`）
- [sglang/index.md](sglang/index.md) topics 行更新
- [sglang/modules/mem_cache.md](sglang/modules/mem_cache.md) 加 CONTRADICTION + SUPERSEDED 旧 10-branch RESOLVED
- [sglang/entities/Scheduler.md](sglang/entities/Scheduler.md) See also 回链

**Hidden cross-ref**
- C++：`cpp_radix_tree/` + `RadixCacheCpp`；`HiMamba` 全树 0
- 协作：Scheduler / TpWorker / kv_canary / session / platforms / Mooncake…
- 测试：storage 内 `test_simm` / `test_mooncake_store` / aibrix unit_test
- docs：`hicache_design.mdx` / `session_radix_cache.mdx` 等

**未做**
- 未整页 re-ingest `modules/mem_cache.md`（仍 stale）
- 未批量更新 `comparison/topics/{kv-cache,prefix-cache}.md` 单元格


## [2026-08-18] increment | sglang | module 页批量增量 (06f32bab → f7101b0a) + 新页 rust_extensions

13 页轻量增量 + 1 新页（重点结构 3 / 轻量注记 7 / 高 churn 标 stale 3）：

- **[grpc.md](sglang/modules/grpc.md)**：`srt/grpc/` 占位包已被上游删除（`67e12131df` #34994）；实现仍在 `entrypoints/grpc_server.py`（行号漂移 L66→L156）+ `grpc_bridge.py`（Rust 原生桥，新记）。
- **[rust_extensions.md](sglang/modules/rust_extensions.md)**（新页）：`load_rust_extension` bundled→cache→cargo 三级回退按需构建 PyO3 扩展；缓存 `~/.cache/sglang/rust_extensions`；消费方 `_grpc`/`_server`/`_multimodal`。
- **[multimodal.md](sglang/modules/multimodal.md)**：+3 子包 `cache/`(#34398) `transport/`(#33949) `media_artifacts/`(#34404) + `encoder_preprocessing.py` + `processors/muse_glimmer.py`；70→**81 .py**。
- **[observability.md](sglang/modules/observability.md)**：+`trace_async.py`（ZMQ→独立 exporter 进程，#30023）；13→**14 .py**。
- **[model_loader.md](sglang/modules/model_loader.md)**：GGUF 原生加载（`gguf_name_maps.py`+`gguf_native.py`，随 #34262）+ startup weight load overlap #32017；7→**8 .py**。
- **[model_executor.md](sglang/modules/model_executor.md)**：+`step_span_utils.py`、`startup_weight_load.py`；`war_event.py`→`shared_read_event.py` (#34916)；54→**56 .py**；保持 stale。
- **[function_call.md](sglang/modules/function_call.md)**：+Muse Glimmer detector（键 `"muse"`）；37→**39 .py**、注册表 33→**34 键**。
- **[configs.md](sglang/modules/configs.md)**：+muse_glimmer 配置 ×2（61→**63 .py**）；config bag 机制（#35022-#35028）确认落在 **`srt/runtime_context.py` `_ConfigBag`**（非 configs/、非 server_args）。
- **[entrypoints_openai.md](sglang/modules/entrypoints_openai.md)**：+`audio_chunking.py` 等音频转写扩展 #33604；路由/serving 类数不变；29→**30 .py**。
- **[distributed.md](sglang/modules/distributed.md)**：`vmm_utils.py` 迁出为顶层 `srt/cuda_vmm_utils.py`（#34199/#34358）；29→**28 .py**。
- **stale 3 页**：[models.md](sglang/modules/models.md)（244→245，+muse_glimmer.py 1027 行）、[layers.md](sglang/modules/layers.md)（312→312 净不变，104 文件 churn）、[hardware_backend.md](sglang/modules/hardware_backend.md)（73→79，DSV4 DSpark #33676 等；旧「22」系 2026-04-19 口径）。
- [sglang/index.md](sglang/index.md) 13 行条目同步更新。

## [2026-08-18] increment | sglang | pin 06f32bab → f7101b0a 主体：5 entity + 8 topic/module 深 verify + 元文件（4 subagent 并行）

**Diff 摸底**：`06f32bab..f7101b0a` 共 **433 commits** / srt 子树 **407 文件** / +30k -9.4k 行。结构性变化：`srt/grpc/` 占位包删除、+顶层 `rust_extensions/`、`multimodal/` +3 子包、`mem_cache/pool_host/` 拆出、+Muse Glimmer 模型族、横切 **config bags** 重构（`runtime_context.py` `_ConfigBag` L593，"config: ... read the bags" #35022-#35028）。**总体结论：无一处架构级论断被推翻**（6-mixin MRO / 三进程 ZMQ 拓扑 / PD 5-backend 继承树 / mem_cache 工厂均成立）；变化 = bags 迁移行号漂移 + 文件级重组 + pin 前漂移补漏。

- **5 entity 页**（[Scheduler](sglang/entities/Scheduler.md) ~70 锚点 / [TokenizerManager](sglang/entities/TokenizerManager.md) ~60 / [Engine](sglang/entities/Engine.md) ~55 / [TpModelWorker](sglang/entities/TpModelWorker.md) ~35 / [DataParallelController](sglang/entities/DataParallelController.md) ~30）：全部 re-anchor + 增量注记（bags getter 化、#34284 RecentPrefillBatchSizeTracker、#35198 ngram→FutureMap、#32017 startup weight load、#34916 WAR→shared-read）；两处任务线索经 git log 证伪未写入（#30023 未触及 tokenizer_manager、#32017 未触及 engine.py）。
- **[topics/kv-cache.md](sglang/topics/kv-cache.md)**（~60 锚点）：工厂矩阵 +分支 0.5（disable_radix ∧ host_pool retraction → Unified+HiCache #34801）；MambaPoolHost 迁出 #31180、嵌入缓存脱钩 Mooncake #30392、+l2_transfer #34793、cache salt #30827。[mem_cache.md](sglang/modules/mem_cache.md) 116→**119 .py** 保持 stale；[managers.md](sglang/modules/managers.md) 仍 49 .py 骨架不变；[scheduler-mixins.md](sglang/topics/scheduler-mixins.md) 19 组件不变、`init_metrics_reporter` 前移 L1198。
- **[topics/pd-disaggregation.md](sglang/topics/pd-disaggregation.md)**（~60 锚点）：**计数修正 prefill mixin 9→18 方法**（pin 时已 17，#35070 +1）；5 backend 树成立（枚举移 utils.py:L592）。[disaggregation.md](sglang/modules/disaggregation.md) 28→**29 .py**。
- **[topics/speculative.md](sglang/topics/speculative.md)**（~55 锚点，**失效最重**，标 stale）：算法族 5→**7 内建 + 插件**（+DSPARK/FROZEN_KV_MTP）；V1/V2 双轨 → **V1 已删单轨**（统一 `BaseSpecWorker`）；~~sgl-kernel 树已移出仓库~~（**更正见后续 re-ingest entry**：实为移入 `python/sglang/kernels/aot/`，`c32c4ef79c` #32648）。[speculative.md 模块页](sglang/modules/speculative.md) 实测 **48 .py** / enum **8 成员**。新文件 `dflash_disaggregation.py` HEAD 内 0 调用方（VERIFY 已标）。
- **主 agent 页**：[source-versions.md](source-versions.md) pin 表 + History；[overview.md](sglang/overview.md)（41 顶层包：-grpc +rust_extensions）；[parser.md](sglang/modules/parser.md) **重大补漏**（4 月快照：5→9 .py / 17→26 键 / 10→22 detector，标 stale）；[arg_groups.md](sglang/modules/arg_groups.md)（锚点全有效，DSpark NPU/MegaMoE）；[eplb.md](sglang/modules/eplb.md) / [ray.md](sglang/modules/ray.md) 轻量注记；[sglang/index.md](sglang/index.md) entity/topic/managers/disaggregation/speculative/mem_cache/parser 行刷新。
- **跨页遗留**：[comparison/topics/speculative-decoding.md](comparison/topics/speculative-decoding.md) 计数已再标过期（本轮范围不含 comparison 页，留待下次 verify）。

**Lessons learned**：① 页面若连续两轮增量未被列入受影响清单，不代表没漂移——parser/observability/configs/hardware_backend 的文件数在 pin 时就已偏差，增量影响面分析应对 `verified_against` 过老（>1 个 pin 周期）的页面做抽样核对；② 子代理提供的 commit 线索必须逐条 git log 证伪后再写入页面。

## [2026-08-18] archive | wiki | log rotation 009（increment entry append 后触发，字节阈值）

- **触发**：append 2026-08-18 两条 increment entries 后 `log.md` = 997 行 / **114269 字节** > 110 KB 阈值（行数 997 < 1500 未触发）。
- **归档**：2026-04-19 的全部 **8 条** entries（863 行 / ~100 KB）→ [log-archive/log-009.md](log-archive/log-009.md)（相对链接已修复 `../` 前缀；绝对 `d:\design\` 路径不变）。
- **轮转后**：主 `log.md` = header + Archive Index（9 行）+ 3 条 2026-08-10 + 2 条 2026-08-18 increment + 本条 = **6 entries**，~20 KB。
