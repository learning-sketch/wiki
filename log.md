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

## [2026-08-18] compare + ingest | cross/sglang | comparison 4 页 SGLang 列刷新 + speculative/parser 重 ingest（4 subagent 并行）

- **comparison 刷新**（SGLang 列 → f7101b0a；vLLM `5f7fab88` / MindIE `f032cd3f` pin 未动，MindIE 未检出 cross-check 跳过已页内注明）：
  - [speculative-decoding.md](comparison/topics/speculative-decoding.md)：组织形式（48 .py / V2 单轨）、enum 8 成员、verify 机制等 8+ cell 重写；vLLM spec 文件数 12→11 死锚顺带修复。cross-check：`DSPARK`/`FROZEN_KV_MTP` 在 vLLM pin 0 命中（N/A）。
  - [scheduler.md](comparison/topics/scheduler.md)：6 mixin + 19 组件、~5085 行；**+3 新维度行**（配置读取途径 `_ConfigBag` vs `get_current_vllm_config()`、prefill batch size 自适应、spec draft 跨步透传）。cross-check：`RecentPrefillBatchSizeTracker` 在 vLLM pin 0 命中。
  - [kv-cache.md](comparison/topics/kv-cache.md)：+构造工厂入口行、cache salt 三方表（**vLLM 真等价物命中** kv_cache_utils.py:385）、HiCache retraction 在 vLLM 0 命中（N/A）。
  - [pd-disaggregation.md](comparison/topics/pd-disaggregation.md)：§10「D 侧始终 ChunkCache」叙事作废（decode 侧现有 3 条 prefix 复用路径）；§13 send_kv_chunk 7 步表整表重写；prefill mixin 9→18。
- **重 ingest**：[sglang/topics/speculative.md](sglang/topics/speculative.md)（V2 单轨 + BaseSpecWorker 继承树 + draft 承载三形态 + NGRAM 也走 `eagle_sample`）、[sglang/modules/parser.md](sglang/modules/parser.md)（22 detector / 26 键全表 + template_detection/template_manager/inkling 新小节 + **发现 Rust 语义镜像** `rust/sglang-server/.../reasoning.rs`）。两页升回 verified。
- **重大更正（写错传播链已全部修复）**：~~sgl-kernel 树已移出仓库~~ → 实为 `c32c4ef79c` #32648 移入 **`python/sglang/kernels/aot/`**（独立 `sglang-kernel` wheel、import 名不变；仅 `sgl_kernel_npu` 是外部包）。已同步修正：topics/speculative（源头自愈）、modules/speculative、index.md、comparison/speculative-decoding、本 log 上一 entry 划线。
- **Lessons learned**：主 agent 简报里的"已确认事实"也可能错——重 ingest 的 hidden cross-ref grep（§5 step 3）是发现该错误的机制，证明该步骤不可跳过。

## [2026-08-18] lint | wiki | 全量 lint：735 → 245 issues（死链清零至 stale 页内）

- **范围**：97 页（跳过 log-archive 锁定区）。检查：frontmatter 必填字段（**0 缺失**）、孤立页（**0**）、wiki 内部死链、源码死锚（`d:\design\sglang\`→/tmp/sglang@f7101b0a、`d:\design\vllm\`→/tmp/vllm-pin@5f7fab88 实测；MindIE 未检出跳过）。
- **批量修复（~470 处）**：① 行号误写进链接 URL（`(...py:123-456)`→`(...py)`，35 文件）；② sgl-kernel 路径迁移（`sgl-kernel\`→`python\sglang\kernels\aot\`）；③ 改名定点修（`communicator_nsa_cp`→`communicator_dsa_cp` #25821、`scheduler_dp_attn_mixin`→`scheduler_components/dp_attn`、`scheduler_update_weights_mixin`→`scheduler_components/weight_updater`、dimensions.md 一处 wiki 链接层级）。
- **定点修复（9 页）**：docs 站点 mdx 化（object_storage / server_arguments / ollama_api / post_training_integration / openai_api_completions / ascend support_features）+ test 树重组（anthropic test_serving、specv2 unit/ 前缀）+ comparison/moe 两处文件迁移（routed_experts→state_capturer、fuseep→hardware_backend/npu）+ grpc.md 已删文件 de-link。
- **标 stale（13 页 + index 同步）**：compilation(23 死锚)/observability(15)/checkpoint_engine(8)/model_loader(8)/weight_sync(8)/constrained(7)/dllm(7)/batch_invariant_ops(5)/batch_overlap(5)/debug_utils(5)/eplb(4)/multiplex(4)/sampling(4)——4 月正文快照 vs f7101b0a 的真实漂移，按 §7 语义降级，各页加 `[!todo] VERIFY` lint 标记。
- **终态**：剩余 242 处死锚**全部**位于 `status: stale` 页面内 + vllm/overview.md（draft，待 vLLM 增量处理）；2 处 `](set)`/`](self)` 为代码片段误匹配（非链接，false positive 不处理）。

## [2026-08-18] increment | vllm | pin 5f7fab88 → d29dc3ab：19 页全处理（4 subagent 并行）

**Diff 摸底**：4 个月增量 **4273 commits**，wiki 相关子树 ~200 文件 / ±6 万行。旧 pin worktree（/tmp/vllm-pin）用于对照与 comparison cross-check。

**结构性大变化（已落页）**：
- **V2 model runner 转正**：`worker/gpu/` 包（30b44a1598 引入，旧 pin 已有——任务简报"本期新增"被子代理 git 考古纠正）从 env opt-in 变为 `VllmConfig.use_v2_model_runner`（[config/vllm.py:614-660]）四级判定下**白名单架构默认**；PCP/DSpark/DFlash/diffusion 强制 V2；V1（gpu_model_runner.py 8018 行）仍是全功能 fallback，两套长期共存（[GPUModelRunner.md](vllm/entities/GPUModelRunner.md)）。
- **spec 架构重写**：eagle.py 被 cde8d24710 抽空（`SpecDecodeBaseProposer` → llm_base_proposer.py 1892 行）；`worker/gpu/spec_decode/` 扩成 `BaseSpeculator` 家族（AutoRegressive/Eagle/MTP/Gemma4/DFlash→DSpark/MultiModuleMTP）+ `AdaptiveVerificationManager`；RejectionSampler 三模式换血 strict/probabilistic/synthetic → **standard/synthetic/block**（[spec-decode-eagle.md](vllm/topics/spec-decode-eagle.md) 标 stale 待 verify pass）。
- **v0 物理删除**：顶层 `executor/`/`worker/`/`attention/` 目录已删（[overview.md](vllm/overview.md) 划线 RESOLVED）；KV connector 14→**16** backend（P2P 删、NIXL pull/push 拆、+MooncakeStore）。
- **engine 层新框架**：`EngineShutdownState` 优雅停机、`EngineCoreSentinel`/`WorkerSentinel` 容错、EEP 两阶段（prepare/commit + `ElasticScalingCache`）、内部 LB 重写（inflight 计数 + KV 压力惩罚）、`"ray"` 默认翻转 `RayExecutorV2`。
- **KV 管理**：`get_computed_blocks` 2→3 元组（`shared_prefix_boundary` Marconi 式保留）、hybrid partial prefix hit、KV watermark、spec 家族 +4 类；`can_fit_full_sequence`/`TQFullAttentionSpec` 删除（RESOLVED）。

**页面清单**：19/19 处理。**verified 13**（EngineCore/AsyncLLM/LLMEngine/EngineCoreClient/OutputProcessor/Scheduler/KVCacheManager/GPUWorker/MultiprocExecutor + engine/executor 模块页 + multiproc-ipc + index）；**stale 4（诚实降级 + VERIFY 注明未校范围）**（GPUModelRunner 细粒度区间 / prefix-cache grep 统计 / request-lifecycle 跨子系统次级细节 / spec-decode-eagle 旧锚点）+ kv-connector（backend 内部行号）+ overview 已 verified。~400+ 锚点修正。

**Lessons learned**：① "新增大目录"要先 `git log --diff-filter=A` 查引入 commit 再定性——`worker/gpu/` 实为旧 pin 已有，本期变化是**默认值翻转**，两者叙事完全不同；② 4k+ commits 的增量下 entity 页仍可保住 verified（类层次稳定），topic 页（跨子系统统计）最易积累不可校债务，stale 降级 + VERIFY 范围声明是正确姿势。
