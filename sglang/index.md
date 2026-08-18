---
type: index
project: sglang
status: verified
confidence: high
verified_against: 2026-08-18
sources:
  - d:\design\sglang\python\sglang\srt
related:
  - ../index.md
  - overview.md
  - ../comparison/index.md
---

# SGLang Index

> 项目内所有 wiki 页的目录。
> 入口架构鸟瞰：[overview.md](overview.md)。
> 主代码根：[d:\design\sglang\python\sglang\srt\](d:\design\sglang\python\sglang\srt)。

---

## Modules（按 [overview.md](overview.md) 的功能分组）

### 入口与服务化
| 模块 | wiki 页 | 状态 |
|---|---|---|
| `entrypoints` | [modules/entrypoints.md](modules/entrypoints.md) | **DONE** / **stale@06f32bab**（56 `.py`；待深 verify） |
| `entrypoints/openai` | [modules/entrypoints_openai.md](modules/entrypoints_openai.md) | **DONE** (P4)（`OpenAIServingBase` + 10 `OpenAIServing*` 子类 + `protocol.py` 77 BaseModel + 11 `/v1/*` 路由 + `MCPToolServer` / `DemoToolServer` + Reasoning + function_call + constrained 集成） ；**file count @f7101b0a = 30.py**（@06f32bab = 29；「20」为旧口径；increment 2026-08-18：+`audio_chunking.py`，音频转写扩展 #33604） |
| `entrypoints/anthropic` | [modules/entrypoints_anthropic.md](modules/entrypoints_anthropic.md) | **DONE** (P6)（**2 .py**：`AnthropicServing` 转 `ChatCompletionRequest` 复用 `OpenAIServingChat`；Anthropic SSE 事件 `message_start` / `content_block_*` / `message_delta` / `message_stop`；`/v1/messages` + `/v1/messages/count_tokens`） |
| `entrypoints/ollama` | [modules/entrypoints_ollama.md](modules/entrypoints_ollama.md) | **DONE** (P6)（4 .py + `OllamaServing` 直连 `TokenizerManager`（**不**依赖 OpenAI 包）+ NDJSON 流式 + `/api/chat` / `/api/generate` / `/api/tags` / `/api/show` 4 路由 + `SGLANG_OLLAMA_*` env 覆盖路径） |
| `grpc` | [modules/grpc.md](modules/grpc.md) | **DONE** (P6)（~~1 .py 占位~~ **@f7101b0a：`srt/grpc/` 目录已删**（commit `67e12131df` #34994）；实现仍在外部包 `smg-grpc-servicer` + `entrypoints/grpc_server.py` + `grpc_bridge.py`（Rust 原生桥）；increment 2026-08-18） |
| `rust_extensions` | [modules/rust_extensions.md](modules/rust_extensions.md) | **DONE** (increment 2026-08-18 新页)（**2 .py / 417 行**：`load_rust_extension` 按需 Cargo 构建 PyO3 扩展（bundled→cache→build 三级回退）+ `~/.cache/sglang/rust_extensions` 指纹缓存；消费方 `_grpc` / `_server` / `_multimodal`） |

### 调度与请求生命周期
| 模块 | wiki 页 | 状态 |
|---|---|---|
| `managers` | [modules/managers.md](modules/managers.md) | **DONE**（verify 2026-08-18：仍 49 `.py`；Scheduler 6 mixin + `scheduler_components/` 不变；本期 cache_controller L2Transfer 接入 #34793、prefill_delayer `RecentPrefillBatchSizeTracker` #34284、io_struct +cache_salt #30827） |
| **`disaggregation`** | [modules/disaggregation.md](modules/disaggregation.md) | **DONE** (P0；verify 2026-08-18：**29 .py**（原记 28，pin 前 +decode_hicache_mixin.py）；5 backend + EPD encode_*.py + 2 Scheduler mixin 骨架不变；本期 HiCache retraction 保 KV #34801、unified-memory PD #33362） |
| `multiplex` | [modules/multiplex.md](modules/multiplex.md) | **DONE** (P5)（2 .py + PD-Multiplexing：同一 GPU 上 Prefill/Decode SM 分区 + green context streams + `SchedulerMultiplexMixin.event_loop_pdmux` + `--enable-pdmux` / `--pdmux-config-path` / `--sm-group-num`；与 disaggregation **互斥**） |
| `session` | [modules/session.md](modules/session.md) | **DONE** (P2)（3 .py + `SessionController` open/close/reap + `StreamingSession` KV/Mamba slot 保活；Scheduler 持有 controller） |
| `function_call` | [modules/function_call.md](modules/function_call.md) | **DONE** (P3)（26 .py + `FunctionCallParser` 注册表 24 字符串键 / 21 detector + `BaseFormatDetector` 流式状态机 + `structural_tag` / `json_schema` 约束分支；产出**约束元组**交 `constrained/` 编译） ；**file count @f7101b0a = 39.py / 注册表 34 键**（increment 2026-08-18：+Muse Glimmer `"muse"` detector） |
| `constrained` | [modules/constrained.md](modules/constrained.md) | **DONE** (P3)（9 .py + 4 backend xgrammar/outlines/llguidance/none + Reasoner wrap + outlines jump-forward FSM + sgl-kernel `apply_token_bitmask_inplace_cuda` + Triton fallback） |

### 模型执行层
| 模块 | wiki 页 | 状态 |
|---|---|---|
| `model_executor` | [modules/model_executor.md](modules/model_executor.md) | **DONE** (P0)（13 .py + 命名陷阱：≠ vLLM Executor 抽象；ModelRunner / ForwardBatch / CudaGraphRunner / Attention 注册表） ；**file count @f7101b0a = 56.py**（increment 2026-08-18：+`step_span_utils.py`、`model_runner_components/startup_weight_load.py`；`war_event.py`→`shared_read_event.py`） |
| `model_loader` | [modules/model_loader.md](modules/model_loader.md) | **DONE** (P4)（`LoadFormat` 含 `IPC_CACHE` + `BaseModelLoader` 子类 + `get_model_loader` 工厂；权重通道：connector / weight_sync / checkpoint_engine / **weight_cache**） ；**file count @f7101b0a = 8.py**（@06f32bab = 7；「6」为旧口径；increment 2026-08-18：+`gguf_name_maps.py` GGUF 原生加载 + startup weight load overlap #32017） |
| `models` | [modules/models.md](modules/models.md) | **DONE** (Turn 4)（**185 .py**（实测；既有 wiki 推断 186）+ ~22 模型族矩阵 + `ModelRegistry.register("sglang.srt.models")` 自动发现 + `EntryClass` 注册 + 共 **54** 文件 `Adapted from vllm-project/vllm` URL + Llama 模板 / DeepSeek MLA+MoE / Qwen3-VL / LLaDA2 DLLM / Llama embedding 等代表实现） ；**file count @f7101b0a = 245.py**（increment 2026-08-18：+`muse_glimmer.py`；38 文件高 churn，页面 stale） |
| `layers` | [modules/layers.md](modules/layers.md) | **DONE** (Turn 5)（**248 .py**（实测；既有 wiki 推断 252）+ 5 子系统 attention(94)/quantization(66)/moe(42)/rotary_embedding(9)/utils(6) + 顶层 31 + `ATTENTION_BACKENDS` **17** 注册名 + `moe_a2a_backend` 7 取值 + Triton `@triton.jit` ≥71 文件 + `from sgl_kernel import` 33 文件） ；**file count @f7101b0a = 312.py**（净不变；increment 2026-08-18：104 文件高 churn——`dwdp/vmm.py`、`torchao_utils.py` 删，`kda_helion.py`、modelslim w4a8-mxfp4 增；页面 stale） |
| `lora` | [modules/lora.md](modules/lora.md) | **DONE** (P4)（33 .py + `LoRAManager` + 4 backend triton/csgmv/ascend/torch_native + `LoRAMemoryPool` 槽位池 + 13 Triton kernel + S-LoRA/Punica 谱系；CUDA adapter LoRA = 0） ；**file count @06f32bab = 45.py**（待深 verify） |
| `sampling` | [modules/sampling.md](modules/sampling.md) | **DONE** (P3)（9 .py + `SamplingParams` 25 字段 + `SamplingBatchInfo` 批张量 / merge / filter + `BatchedPenalizerOrchestrator` 4 子类 + custom logits dill 序列化 + sgl-kernel `top_k_renorm_probs` / `top_p_renorm_probs` / `apply_token_bitmask_inplace_cuda` 3 算子 + FlashInfer 采样核 delegate） |
| `speculative` | [modules/speculative.md](modules/speculative.md) | **DONE**（verify 2026-08-18：**48 .py** 实测（34 根 + 12 dspark_components + 2 cpp_ngram）；`SpeculativeAlgorithm` enum **8 成员** + `CustomSpecAlgo` 插件；V1 worker 已删、`create_worker` 一维分发；~~sgl-kernel 移出仓库~~（更正 2026-08-18：移入 `python/sglang/kernels/aot/`，独立 `sglang-kernel` wheel）+ Triton fallback；本期 dflash_utils +217 行、DSpark logprobs #34696） |
| `state_capturer` | [modules/state_capturer.md](modules/state_capturer.md) | **DONE** (P2)（4 .py + `RoutedExpertsCapturer` / `IndexerTopkCapturer` + host pin_memory + `meta_info` base64；门控 `enable_return_*`） |

### 内存与缓存
| 模块 | wiki 页 | 状态 |
|---|---|---|
| `mem_cache` | [modules/mem_cache.md](modules/mem_cache.md) | **DONE** / **stale@f7101b0a**（**119 `.py`**（@06f32bab 116、原 62）；9 子目录（+pool_host/cpp_utils/layout）；本期 MambaPoolHost 迁出 #31180、+embedding_store/l2_transfer、增量小节已补；待深 verify） |
| `kv_canary` | [modules/kv_canary.md](modules/kv_canary.md) | **DONE** (P0)（~50 .py + `install_canary` patch `model.forward` + pool_patcher MHA/SWA/DSV4 + runner/perturb/token_oracle；`--kv-canary {none,log,raise}`） |
| `checkpoint_engine` | [modules/checkpoint_engine.md](modules/checkpoint_engine.md) | **DONE** (P3)（3 .py + Moonshot **checkpoint-engine==0.1.2** ParameterServer + ZMQ IPC + `/update_weights_from_ipc` HTTP；与 `weight_sync` 并列正交通道） |
| `weight_sync` | [modules/weight_sync.md](modules/weight_sync.md) | **DONE** (P3)（2 .py 无 `__init__` + `FlattenedTensorBucket` `uint8` flatten + `update_weights` 训练 SPMD 桥接；NCCL broadcast 主体在 `model_runner`；与 `connector`/`checkpoint_engine`/`weight_cache` 形成**4 条正交权重通道**） |
| `weight_cache` | [modules/weight_cache.md](modules/weight_cache.md) | **DONE** (P0)（4 .py + `WeightCacheDaemon` Unix sock + `IpcModelLoader` CUDA IPC 零拷贝；`--weight-cache-mode {off,daemon,client}`；与 sync/connector/checkpoint_engine 正交的**第四条权重通道**） |

### 分布式与硬件
| 模块 | wiki 页 | 状态 |
|---|---|---|
| `distributed` | [modules/distributed.md](modules/distributed.md) | **DONE** (P0)（22 .py = 5 顶层 + 17 device_communicators / 13 逻辑后端；GroupCoordinator + 7 子组；Scheduler 6 rank 形参；唯一 sgl-kernel 算子 `shm_allreduce`） ；**file count @f7101b0a = 28.py**（increment 2026-08-18：`vmm_utils.py` 迁出为顶层 `srt/cuda_vmm_utils.py`，#34199/#34358） |
| `eplb` | [modules/eplb.md](modules/eplb.md) | **DONE** (P1)（12 .py + `EPLBManager` 周期重平衡 + 3 套算法 DeepSeek/DeepSeek-vec/Elasticity-aware + simulator 离线 reader） |
| `elastic_ep` | [modules/elastic_ep.md](modules/elastic_ep.md) | **DONE** (P1)（3 .py + `ElasticEPStateManager` rank 活性 + `ExpertBackup{Manager,Client}` 子进程 Mooncake TE RDMA；**vLLM 有但实现不同**） |
| `ray` | [modules/ray.md](modules/ray.md) | **DONE** (P5)（5 .py + `RayEngine(Engine)` 子类化 + `SchedulerActor` Ray actor + `RayDataParallelController` 子类化 + Placement Group + `--use-ray`；**比 vLLM `RayDistributedExecutor` 更轻**：仅替换 scheduler 进程） |
| `platforms` | [modules/platforms.md](modules/platforms.md) | **DONE** (P1)（7 .py + 惰性 `current_platform` + `SRTPlatform` 工厂/能力；CUDA/ROCm/CPU/XPU + OOT entry_points `sglang.srt.platforms`） |
| `hardware_backend` | [modules/hardware_backend.md](modules/hardware_backend.md) | **DONE** (P4)（22 .py + 3 设备子包 npu/musa/mlx + `NPUGraphRunner` 子类化 `CudaGraphRunner` + 4 个 `*GraphRunner` 子类 + `sgl_kernel_npu` import + `_handle_npu_backends` 默认参数注入） ；**file count @f7101b0a = 79.py**（@06f32bab = 73；「22」为 2026-04-19 旧口径；increment 2026-08-18：DSV4 DSpark / ascend_kda / muse_glimmer_mlx 等 +6，页面 stale） |
| `compilation` | [modules/compilation.md](modules/compilation.md) | **DONE** (P1)（13 .py + `SGLangBackend` torch.compile + Inductor 适配 + PCG 子图 CUDAGraph；唯一 sgl-kernel 算子 `weak_ref_tensor`） |
| `connector` | [modules/connector.md](modules/connector.md) | **DONE** (P2)（**命名陷阱**：≠ KV connector，是远程**权重**加载；9 .py + Redis/S3/RemoteInstance + serde） |
| `plugins` | [modules/plugins.md](modules/plugins.md) | **DONE** (P2)（2 .py + `load_plugins` / `HookRegistry` BEFORE/AFTER/AROUND/REPLACE；groups `sglang.srt.plugins` + platforms） |

### 其他
| 模块 | wiki 页 | 状态 |
|---|---|---|
| `tokenizer` | [modules/tokenizer.md](modules/tokenizer.md) | **DONE** (P5)（**1 .py**：`TiktokenTokenizer` 加载 xtok JSON + HF 兼容 API；`get_tokenizer` 在 `tokenizer_name.endswith(".json")` 时分发；**4 个同名"tokenizer"对象消歧**） |
| `multimodal` | [modules/multimodal.md](modules/multimodal.md) | **DONE** (P4)（46 .py + 36 模型族处理器 + `MultimodalSpecialTokens` + `import_processors` 启动注册 + EPD encode_server 单向依赖 + 3 模态 IMAGE/VIDEO/AUDIO + `vit_cuda_graph_runner` ViT 图） ；**file count @f7101b0a = 81.py**（increment 2026-08-18：+`cache/`、`transport/`、`media_artifacts/` 3 子包 + `encoder_preprocessing.py` + `processors/muse_glimmer.py`） |
| `parser` | [modules/parser.md](modules/parser.md) | **DONE**（re-ingest 2026-08-18 / HEAD `f7101b0a`：**9 .py**（template_manager/template_detection 已迁入 + inkling_renderer/inkling_tokenizer）；`DetectorMap` **26 键**（含 `auto`）→ 21 类 / **22** detector 类；holdback 流式状态机 #34458；`register_conv_template` 实为 27 次（口径修正）；新发现 Rust 语义镜像 `rust/sglang-server/.../reasoning.rs`（VERIFY 键集合同步）） |
| `dllm` | [modules/dllm.md](modules/dllm.md) | **DONE** (P5)（7 .py + Diffusion LLM 调度：`DllmConfig` + `SchedulerDllmMixin` + 2 算法 `LowConfidence` / `JointThreshold` 迭代 mask 填充 + 3 HF 架构（LLaDA2/SDAR/SDARMoe）；**SGLang-unique** 设计点） |
| `batch_invariant_ops` | [modules/batch_invariant_ops.md](modules/batch_invariant_ops.md) | **DONE** (P5)（2 .py + Triton GEMM/log-softmax/mean/BMM/RMSNorm 确定性算子 + `torch.library.Library` ATen 注册 + `--enable-deterministic-inference` 触发；改编自 thinking-machines-lab） |
| `batch_overlap` | [modules/batch_overlap.md](modules/batch_overlap.md) | **DONE** (P5)（4 .py + TBO 双 micro-batch 前向交错 + SBO 单 batch MoE combine/down-gemm CUDA stream/event + `--enable-two-batch-overlap`；与 `--disable-overlap-schedule` 是不同层概念） |
| `debug_utils` | [modules/debug_utils.md](modules/debug_utils.md) | **DONE** (Turn 3)（**82 .py**（实测；既有 wiki 推断 89）+ 3 子包 comparator/schedule_simulator/source_patcher + 5 子系统：tensor dump（DUMPER_*）/ comparator / text_comparator / schedule_simulator / 辅助 cuda_coredump+log_parser+model_truncator） |
| `observability` | [modules/observability.md](modules/observability.md) | **DONE** (Turn 3)（`SchedulerMetricsCollector` 70+ 指标 + `TokenizerMetricsCollector` + OTel `process_tracing_init` + KV events 发布 + multiprocess Prometheus + Grafana JSON dashboard） ；**file count @f7101b0a = 14.py**（@06f32bab = 13；「10」为旧口径；increment 2026-08-18：+`trace_async.py` tracing v2 异步导出 #30023） |
| `metrics` | — | **(N/A)** 路径不存在；指标系统在 `observability/` |
| `configs` | [modules/configs.md](modules/configs.md) | **DONE** (Turn 3)（`ModelConfig` 主类（25 字段）+ `LoadConfig` 正交 + `update_config.py` TP 对齐补丁 + ~95 个 srt 文件消费 + 多文件 Adapted from vLLM/HF transformers） ；**file count @f7101b0a = 63.py**（@06f32bab = 61；「44」为旧口径；increment 2026-08-18：+muse_glimmer 配置 ×2；config bag 机制在 `srt/runtime_context.py` #35022-#35028） |
| `arg_groups` | [modules/arg_groups.md](modules/arg_groups.md) | **DONE** (P1)（9 .py + `A`/`Arg`/`NS` CLI 派生 + `overrides.py` 声明式 arch override + speculative/PD/DeepSeekV4/HiSparse/KimiK3 hooks） |
| `utils` | — | 未 ingest（横切工具包；非本批） |

---

## Entities（关键 class）

| 实体 | 源文件 | wiki 页 | 状态 |
|---|---|---|---|
| `Engine` | [entrypoints/engine.py](d:\design\sglang\python\sglang\srt\entrypoints\engine.py) (~1877 行) | [entities/Engine.md](entities/Engine.md) | **DONE** (P0；verify 2026-08-18 / HEAD `f7101b0a`：`Engine` L207；`_launch_scheduler_processes` L856；`_launch_subprocesses` L1060（新增 `publish(role="tokenizer")` 步骤）；+`get_model_info` API、`cache_salt`/`mm_content_hashes` 参数) |
| `TokenizerManager` | [managers/tokenizer_manager.py](d:\design\sglang\python\sglang\srt\managers\tokenizer_manager.py) (~3665 行) | [entities/TokenizerManager.md](entities/TokenizerManager.md) | **DONE**（verify 2026-08-18：`ReqState` L215；`TokenizerManager` L386（`TokenizerControlMixin` + `TokenizerManagerScoreMixin`）；含 `DetokenizerManager` L92；本期 config bags 迁移 + VLM 预处理缓存/cache_salt 透传） |
| `Scheduler` | [managers/scheduler.py](d:\design\sglang\python\sglang\srt\managers\scheduler.py) (~5085 行) + [scheduler_components/](d:\design\sglang\python\sglang\srt\managers\scheduler_components) | [entities/Scheduler.md](entities/Scheduler.md) | **DONE**（verify 2026-08-18 / HEAD `f7101b0a`：`class Scheduler` L383，MRO **6 mixin 不变**；本期 config bags getter 化、`RecentPrefillBatchSizeTracker` #34284、ngram accept tokens 走 FutureMap #35198；拆解见 [topics/scheduler-mixins.md](topics/scheduler-mixins.md)） |
| `DetokenizerManager` | [managers/detokenizer_manager.py](d:\design\sglang\python\sglang\srt\managers\detokenizer_manager.py) | 见 [entities/TokenizerManager.md](entities/TokenizerManager.md) | DONE（合并；verify 2026-08-18：L92，+`publish(role="detokenizer")`） |
| `TpModelWorker` | [managers/tp_worker.py](d:\design\sglang\python\sglang\srt\managers\tp_worker.py) | [entities/TpModelWorker.md](entities/TpModelWorker.md) | **DONE** (P0；verify 2026-08-18：`BaseTpWorker` L74；`TpModelWorker` L299；`forward_batch_generation` L574；本期 +`start/finalize_startup_weight_load` #32017、delay-sample 条件扩展 #32637、WAR→shared-read 更名 #34916) |
| `DataParallelController` | [managers/data_parallel_controller.py](d:\design\sglang\python\sglang\srt\managers\data_parallel_controller.py) | [entities/DataParallelController.md](entities/DataParallelController.md) | **DONE** (P0；verify 2026-08-18：`DataParallelController` L132；`LoadBalanceMethod` L79；`DPBudget` L96；`run_...` L811；本期 DP/EP 拓扑读取改走 `get_parallel()` bag；构造签名与进程拓扑不变) |
| `SchedulePolicy` | [managers/schedule_policy.py](d:\design\sglang\python\sglang\srt\managers\schedule_policy.py) | `entities/SchedulePolicy.md` | TODO (P1) |
| `ScheduleBatch` | [managers/schedule_batch.py](d:\design\sglang\python\sglang\srt\managers\schedule_batch.py) | `entities/ScheduleBatch.md` | TODO (P1，110 KB 最大单文件) |
| `CacheController` | [managers/cache_controller.py](d:\design\sglang\python\sglang\srt\managers\cache_controller.py) | `entities/CacheController.md` | TODO (P2) |
| `DisaggService` | [managers/disagg_service.py](d:\design\sglang\python\sglang\srt\managers\disagg_service.py) | `entities/DisaggService.md` | TODO (P2，PD 优化相关) |
| `HiSparseCoordinator` | [managers/hisparse_coordinator.py](d:\design\sglang\python\sglang\srt\managers\hisparse_coordinator.py) | `entities/HiSparseCoordinator.md` | TODO (P3) |

---

## Topics（跨模块主题）

| 主题 | wiki 页 | 状态 |
|---|---|---|
| **Request lifecycle** | [topics/request-lifecycle.md](topics/request-lifecycle.md) | **DONE** |
| **Manager 模式 + ZMQ pipeline** | [topics/manager-pipeline.md](topics/manager-pipeline.md) | **DONE** |
| **Scheduler mixin / composition 拆解** | [topics/scheduler-mixins.md](topics/scheduler-mixins.md) | **DONE**（verify 2026-08-18 / HEAD `f7101b0a`：MRO **6 mixin** 不变 + `scheduler_components/` 仍 19 模块；`init_metrics_reporter` 前移 L1198；`dispatch_event_loop` PP 判定改 `configured_pp_size()`） |
| **PD 分离 多后端**（NIXL/Mooncake/MORI/Ascend/Fake） | [topics/pd-disaggregation.md](topics/pd-disaggregation.md) | **DONE**（verify 2026-08-18：5 backend 3 层继承树仍成立（枚举移至 utils.py:L592）；2 mixin 方法数修正为 **prefill 18 / decode 6**（旧记 9 系 4 月口径）；本期 HiCache retraction 保 KV #34801、unified-memory PD #33362、mooncake/conn +148 行） |
| **KV cache**（unified / paged / radix / sparsity / hierarchical / mamba / SWA / LMC） | [topics/kv-cache.md](topics/kv-cache.md) | **DONE**（verify 2026-08-18 / HEAD `f7101b0a`：工厂 `kv_cache_builder`+`registry` 叙事成立，HiCache storage 仍 **9** 注册名；工厂矩阵 +分支 0.5（disable_radix ∧ host_pool retraction → Unified+HiCache #34801）；本期 MambaPoolHost 迁出 pool_host/、+embedding_store/l2_transfer、cache salt #30827） |
| Continuous batching / chunked prefill | `topics/continuous-batching.md` | TODO |
| **MoE**（dispatch/combine, EPLB, Elastic EP, TBO/SBO, sgl-kernel 绑定） | [topics/moe.md](topics/moe.md) | **DONE** |
| DP attention | `topics/dp-attention.md` | TODO |
| Speculative decoding | [topics/speculative.md](topics/speculative.md) | **DONE**（re-ingest 2026-08-18 / HEAD `f7101b0a`：**7 内建族 + `CustomSpecAlgo` 插件**、enum 8 成员；V1 已删，全 worker 统一继承 `BaseSpecWorker`，draft 承载三形态（EagleDraftWorkerBase 包裹 / plain TpModelWorker / None）；NGRAM 也走统一 `eagle_sample` verify；sgl-kernel 源树在仓内 `kernels/aot/`（独立 `sglang-kernel` wheel，`sgl_kernel_npu` 才是外部包）+ Triton fallback；`dflash_disaggregation.py` 0 调用方 VERIFY 保留） |
| Constrained / structured output | `topics/constrained.md` | TODO |
| Function calling / tool use | `topics/function-call.md` | TODO |
| Multi-API 兼容（OpenAI / Anthropic / Ollama） | `topics/multi-api.md` | TODO |
| RLHF weight sync | `topics/weight-sync.md` | TODO |
| Hardware backends（CUDA / MUSA / Ascend） | `topics/hardware-backends.md` | TODO |
| Schedule simulator / debug 工具 | `topics/debug-utils.md` | TODO |

---

## Sources of truth

- 主代码根：[d:\design\sglang\python\sglang\srt\](d:\design\sglang\python\sglang\srt)
- 前端 lang：[d:\design\sglang\python\sglang\lang\](d:\design\sglang\python\sglang\lang)
- 测试：[d:\design\sglang\python\sglang\test\](d:\design\sglang\python\sglang\test)
- benchmark：[d:\design\sglang\benchmark\](d:\design\sglang\benchmark)
- C++ kernel：[d:\design\sglang\sgl-kernel\](d:\design\sglang\sgl-kernel)
- model gateway：[d:\design\sglang\sgl-model-gateway\](d:\design\sglang\sgl-model-gateway)
