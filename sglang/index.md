---
type: index
project: sglang
status: verified
confidence: high
verified_against: 2026-08-10
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
| `entrypoints` | [modules/entrypoints.md](modules/entrypoints.md) | **DONE** |
| `entrypoints/openai` | [modules/entrypoints_openai.md](modules/entrypoints_openai.md) | **DONE** (P4)（20 .py + `OpenAIServingBase` + 10 `OpenAIServing*` 子类 + `protocol.py` 77 BaseModel + 11 `/v1/*` 路由 + `MCPToolServer` / `DemoToolServer` + Reasoning + function_call + constrained 集成）|
| `entrypoints/anthropic` | [modules/entrypoints_anthropic.md](modules/entrypoints_anthropic.md) | **DONE** (P6)（**2 .py**：`AnthropicServing` 转 `ChatCompletionRequest` 复用 `OpenAIServingChat`；Anthropic SSE 事件 `message_start` / `content_block_*` / `message_delta` / `message_stop`；`/v1/messages` + `/v1/messages/count_tokens`） |
| `entrypoints/ollama` | [modules/entrypoints_ollama.md](modules/entrypoints_ollama.md) | **DONE** (P6)（4 .py + `OllamaServing` 直连 `TokenizerManager`（**不**依赖 OpenAI 包）+ NDJSON 流式 + `/api/chat` / `/api/generate` / `/api/tags` / `/api/show` 4 路由 + `SGLANG_OLLAMA_*` env 覆盖路径） |
| `grpc` | [modules/grpc.md](modules/grpc.md) | **DONE** (P6)（**1 .py / 22 字节占位** — 实际实现在外部包 `smg-grpc-servicer` + [`entrypoints/grpc_server.py`](modules/grpc.md)；命名陷阱） |

### 调度与请求生命周期
| 模块 | wiki 页 | 状态 |
|---|---|---|
| `managers` | [modules/managers.md](modules/managers.md) | **DONE** |
| **`disaggregation`** | [modules/disaggregation.md](modules/disaggregation.md) | **DONE** (P0)（5 backend + EPD encode_*.py + 2 Scheduler mixin） |
| `multiplex` | [modules/multiplex.md](modules/multiplex.md) | **DONE** (P5)（2 .py + PD-Multiplexing：同一 GPU 上 Prefill/Decode SM 分区 + green context streams + `SchedulerMultiplexMixin.event_loop_pdmux` + `--enable-pdmux` / `--pdmux-config-path` / `--sm-group-num`；与 disaggregation **互斥**） |
| `session` | [modules/session.md](modules/session.md) | **DONE** (P2)（3 .py + `SessionController` open/close/reap + `StreamingSession` KV/Mamba slot 保活；Scheduler 持有 controller） |
| `function_call` | [modules/function_call.md](modules/function_call.md) | **DONE** (P3)（26 .py + `FunctionCallParser` 注册表 24 字符串键 / 21 detector + `BaseFormatDetector` 流式状态机 + `structural_tag` / `json_schema` 约束分支；产出**约束元组**交 `constrained/` 编译） |
| `constrained` | [modules/constrained.md](modules/constrained.md) | **DONE** (P3)（9 .py + 4 backend xgrammar/outlines/llguidance/none + Reasoner wrap + outlines jump-forward FSM + sgl-kernel `apply_token_bitmask_inplace_cuda` + Triton fallback） |

### 模型执行层
| 模块 | wiki 页 | 状态 |
|---|---|---|
| `model_executor` | [modules/model_executor.md](modules/model_executor.md) | **DONE** (P0)（13 .py + 命名陷阱：≠ vLLM Executor 抽象；ModelRunner / ForwardBatch / CudaGraphRunner / Attention 注册表） |
| `model_loader` | [modules/model_loader.md](modules/model_loader.md) | **DONE** (P4)（6 .py + `LoadFormat` 19 枚举 + `BaseModelLoader` 11 子类 + `get_model_loader` 工厂 + Adapted from vLLM v0.6.4 + **4 条权重通道闭环**：connector/weight_sync/checkpoint_engine/model_loader） |
| `models` | [modules/models.md](modules/models.md) | **DONE** (Turn 4)（**185 .py**（实测；既有 wiki 推断 186）+ ~22 模型族矩阵 + `ModelRegistry.register("sglang.srt.models")` 自动发现 + `EntryClass` 注册 + 共 **54** 文件 `Adapted from vllm-project/vllm` URL + Llama 模板 / DeepSeek MLA+MoE / Qwen3-VL / LLaDA2 DLLM / Llama embedding 等代表实现） |
| `layers` | [modules/layers.md](modules/layers.md) | **DONE** (Turn 5)（**248 .py**（实测；既有 wiki 推断 252）+ 5 子系统 attention(94)/quantization(66)/moe(42)/rotary_embedding(9)/utils(6) + 顶层 31 + `ATTENTION_BACKENDS` **17** 注册名 + `moe_a2a_backend` 7 取值 + Triton `@triton.jit` ≥71 文件 + `from sgl_kernel import` 33 文件） |
| `lora` | [modules/lora.md](modules/lora.md) | **DONE** (P4)（33 .py + `LoRAManager` + 4 backend triton/csgmv/ascend/torch_native + `LoRAMemoryPool` 槽位池 + 13 Triton kernel + S-LoRA/Punica 谱系；CUDA adapter LoRA = 0） |
| `sampling` | [modules/sampling.md](modules/sampling.md) | **DONE** (P3)（9 .py + `SamplingParams` 25 字段 + `SamplingBatchInfo` 批张量 / merge / filter + `BatchedPenalizerOrchestrator` 4 子类 + custom logits dill 序列化 + sgl-kernel `top_k_renorm_probs` / `top_p_renorm_probs` / `apply_token_bitmask_inplace_cuda` 3 算子 + FlashInfer 采样核 delegate） |
| `speculative` | [modules/speculative.md](modules/speculative.md) | **DONE** (P1)（27 .py + 6 算法 enum + EAGLE/MultiLayer/Standalone/DFlash/NGRAM 多家族 + sgl-kernel `verify_tree_greedy` CUDA 绑定） |
| `state_capturer` | [modules/state_capturer.md](modules/state_capturer.md) | **DONE** (P2)（4 .py + `RoutedExpertsCapturer` / `IndexerTopkCapturer` + host pin_memory + `meta_info` base64；门控 `enable_return_*`） |

### 内存与缓存
| 模块 | wiki 页 | 状态 |
|---|---|---|
| `mem_cache` | [modules/mem_cache.md](modules/mem_cache.md) | **DONE** |
| `kv_canary` | [modules/kv_canary.md](modules/kv_canary.md) | **DONE** (P0)（~50 .py + `install_canary` patch `model.forward` + pool_patcher MHA/SWA/DSV4 + runner/perturb/token_oracle；`--kv-canary {none,log,raise}`） |
| `checkpoint_engine` | [modules/checkpoint_engine.md](modules/checkpoint_engine.md) | **DONE** (P3)（3 .py + Moonshot **checkpoint-engine==0.1.2** ParameterServer + ZMQ IPC + `/update_weights_from_ipc` HTTP；与 `weight_sync` 并列正交通道） |
| `weight_sync` | [modules/weight_sync.md](modules/weight_sync.md) | **DONE** (P3)（2 .py 无 `__init__` + `FlattenedTensorBucket` `uint8` flatten + `update_weights` 训练 SPMD 桥接；NCCL broadcast 主体在 `model_runner`；与 `connector`/`checkpoint_engine` 形成**3 条正交权重通道**） |
| `weight_cache` | [modules/weight_cache.md](modules/weight_cache.md) | **DONE** (P0)（4 .py + `WeightCacheDaemon` Unix sock + `IpcModelLoader` CUDA IPC 零拷贝；`--weight-cache-mode {off,daemon,client}`；与 sync/connector/checkpoint_engine 正交的**第四条权重通道**） |

### 分布式与硬件
| 模块 | wiki 页 | 状态 |
|---|---|---|
| `distributed` | [modules/distributed.md](modules/distributed.md) | **DONE** (P0)（22 .py = 5 顶层 + 17 device_communicators / 13 逻辑后端；GroupCoordinator + 7 子组；Scheduler 6 rank 形参；唯一 sgl-kernel 算子 `shm_allreduce`） |
| `eplb` | [modules/eplb.md](modules/eplb.md) | **DONE** (P1)（12 .py + `EPLBManager` 周期重平衡 + 3 套算法 DeepSeek/DeepSeek-vec/Elasticity-aware + simulator 离线 reader） |
| `elastic_ep` | [modules/elastic_ep.md](modules/elastic_ep.md) | **DONE** (P1)（3 .py + `ElasticEPStateManager` rank 活性 + `ExpertBackup{Manager,Client}` 子进程 Mooncake TE RDMA；**vLLM 有但实现不同**） |
| `ray` | [modules/ray.md](modules/ray.md) | **DONE** (P5)（5 .py + `RayEngine(Engine)` 子类化 + `SchedulerActor` Ray actor + `RayDataParallelController` 子类化 + Placement Group + `--use-ray`；**比 vLLM `RayDistributedExecutor` 更轻**：仅替换 scheduler 进程） |
| `platforms` | [modules/platforms.md](modules/platforms.md) | **DONE** (P1)（7 .py + 惰性 `current_platform` + `SRTPlatform` 工厂/能力；CUDA/ROCm/CPU/XPU + OOT entry_points `sglang.srt.platforms`） |
| `hardware_backend` | [modules/hardware_backend.md](modules/hardware_backend.md) | **DONE** (P4)（22 .py + 3 设备子包 npu/musa/mlx + `NPUGraphRunner` 子类化 `CudaGraphRunner` + 4 个 `*GraphRunner` 子类 + `sgl_kernel_npu` import + `_handle_npu_backends` 默认参数注入） |
| `compilation` | [modules/compilation.md](modules/compilation.md) | **DONE** (P1)（13 .py + `SGLangBackend` torch.compile + Inductor 适配 + PCG 子图 CUDAGraph；唯一 sgl-kernel 算子 `weak_ref_tensor`） |
| `connector` | [modules/connector.md](modules/connector.md) | **DONE** (P2)（**命名陷阱**：≠ KV connector，是远程**权重**加载；9 .py + Redis/S3/RemoteInstance + serde） |
| `plugins` | [modules/plugins.md](modules/plugins.md) | **DONE** (P2)（2 .py + `load_plugins` / `HookRegistry` BEFORE/AFTER/AROUND/REPLACE；groups `sglang.srt.plugins` + platforms） |

### 其他
| 模块 | wiki 页 | 状态 |
|---|---|---|
| `tokenizer` | [modules/tokenizer.md](modules/tokenizer.md) | **DONE** (P5)（**1 .py**：`TiktokenTokenizer` 加载 xtok JSON + HF 兼容 API；`get_tokenizer` 在 `tokenizer_name.endswith(".json")` 时分发；**4 个同名"tokenizer"对象消歧**） |
| `multimodal` | [modules/multimodal.md](modules/multimodal.md) | **DONE** (P4)（46 .py + 36 模型族处理器 + `MultimodalSpecialTokens` + `import_processors` 启动注册 + EPD encode_server 单向依赖 + 3 模态 IMAGE/VIDEO/AUDIO + `vit_cuda_graph_runner` ViT 图） |
| `parser` | [modules/parser.md](modules/parser.md) | **DONE** (P3)（5 .py + `ReasoningParser.DetectorMap` 17 键 / 10 detector 子类 + `HarmonyParser` GPT-OSS channel + `Conversation` 28 内置模板（FastChat 谱系）+ Jinja AST format 检测 + 3 套 FIM 模板） |
| `dllm` | [modules/dllm.md](modules/dllm.md) | **DONE** (P5)（7 .py + Diffusion LLM 调度：`DllmConfig` + `SchedulerDllmMixin` + 2 算法 `LowConfidence` / `JointThreshold` 迭代 mask 填充 + 3 HF 架构（LLaDA2/SDAR/SDARMoe）；**SGLang-unique** 设计点） |
| `batch_invariant_ops` | [modules/batch_invariant_ops.md](modules/batch_invariant_ops.md) | **DONE** (P5)（2 .py + Triton GEMM/log-softmax/mean/BMM/RMSNorm 确定性算子 + `torch.library.Library` ATen 注册 + `--enable-deterministic-inference` 触发；改编自 thinking-machines-lab） |
| `batch_overlap` | [modules/batch_overlap.md](modules/batch_overlap.md) | **DONE** (P5)（4 .py + TBO 双 micro-batch 前向交错 + SBO 单 batch MoE combine/down-gemm CUDA stream/event + `--enable-two-batch-overlap`；与 `--disable-overlap-schedule` 是不同层概念） |
| `debug_utils` | [modules/debug_utils.md](modules/debug_utils.md) | **DONE** (Turn 3)（**82 .py**（实测；既有 wiki 推断 89）+ 3 子包 comparator/schedule_simulator/source_patcher + 5 子系统：tensor dump（DUMPER_*）/ comparator / text_comparator / schedule_simulator / 辅助 cuda_coredump+log_parser+model_truncator） |
| `observability` | [modules/observability.md](modules/observability.md) | **DONE** (Turn 3)（10 .py 无 `__init__.py` + `SchedulerMetricsCollector` 70+ 指标 + `TokenizerMetricsCollector` + OTel `process_tracing_init` + KV events 发布 + multiprocess Prometheus + Grafana JSON dashboard） |
| `metrics` | — | **(N/A)** 路径不存在；指标系统在 `observability/` |
| `configs` | [modules/configs.md](modules/configs.md) | **DONE** (Turn 3)（44 .py + `ModelConfig` 主类（25 字段）+ `LoadConfig` 正交 + `update_config.py` TP 对齐补丁 + ~95 个 srt 文件消费 + 多文件 Adapted from vLLM/HF transformers） |
| `arg_groups` | [modules/arg_groups.md](modules/arg_groups.md) | **DONE** (P1)（9 .py + `A`/`Arg`/`NS` CLI 派生 + `overrides.py` 声明式 arch override + speculative/PD/DeepSeekV4/HiSparse/KimiK3 hooks） |
| `utils` | — | 未 ingest（横切工具包；非本批） |

---

## Entities（关键 class）

| 实体 | 源文件 | wiki 页 | 状态 |
|---|---|---|---|
| `Engine` | [entrypoints/engine.py](d:\design\sglang\python\sglang\srt\entrypoints\engine.py) | [entities/Engine.md](entities/Engine.md) | **DONE** (P0)（含 `HttpServerEngineAdapter`） |
| `TokenizerManager` | [managers/tokenizer_manager.py](d:\design\sglang\python\sglang\srt\managers\tokenizer_manager.py) | [entities/TokenizerManager.md](entities/TokenizerManager.md) | **DONE**（含 `DetokenizerManager`） |
| `Scheduler` | [managers/scheduler.py](d:\design\sglang\python\sglang\srt\managers\scheduler.py) (~5046 行) + [scheduler_components/](d:\design\sglang\python\sglang\srt\managers\scheduler_components) | [entities/Scheduler.md](entities/Scheduler.md) | **DONE**（re-ingest 2026-08-10 / HEAD `06f32bab`：MRO **6 mixin + MlxOverlap**；前 Output/Weights/Profiler/Metrics/RuntimeChecker/DPAttn mixin → composition；`IdleSleeper`/`SenderWrapper` 已迁出；TpModelWorker 见独立页）。**Caveat:** [topics/scheduler-mixins.md](topics/scheduler-mixins.md) 仍写 11 mixin → CONTRADICTION / stale |
| `DetokenizerManager` | [managers/detokenizer_manager.py](d:\design\sglang\python\sglang\srt\managers\detokenizer_manager.py) | 见 [entities/TokenizerManager.md](entities/TokenizerManager.md) | DONE（合并） |
| `TpModelWorker` | [managers/tp_worker.py](d:\design\sglang\python\sglang\srt\managers\tp_worker.py) | [entities/TpModelWorker.md](entities/TpModelWorker.md) | **DONE** (P0)（独立页，含 `BaseTpWorker`、`forward_batch_generation` 三分支、`get_memory_pool` spec 共享） |
| `DataParallelController` | [managers/data_parallel_controller.py](d:\design\sglang\python\sglang\srt\managers\data_parallel_controller.py) | [entities/DataParallelController.md](entities/DataParallelController.md) | **DONE** (P0)（含 `LoadBalanceMethod` 4 策略 + `routed_dp_rank` 钉死） |
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
| **Scheduler mixin / composition 拆解** | [topics/scheduler-mixins.md](topics/scheduler-mixins.md) | **STALE** (P1) — 仍按 11 mixin 撰写；HEAD 已改为 6 mixin + `scheduler_components/` composition（见 [entities/Scheduler.md](entities/Scheduler.md) CONTRADICTION）。需 re-ingest |
| **PD 分离 多后端**（NIXL/Mooncake/MORI/Ascend/Fake） | [topics/pd-disaggregation.md](topics/pd-disaggregation.md) | **DONE** (P1)（5 backend 3 层继承树 `Base*→Common*→具体` + 2 mixin（prefill 9 方法 / decode 6 方法 不对称）+ EPD encode 三段 + PD-Disagg vs PD-Mux 互斥本质 + 4 维度对偶矩阵） |
| **KV cache**（unified / paged / radix / sparsity / hierarchical / mamba / SWA / LMC） | [topics/kv-cache.md](topics/kv-cache.md) | **DONE** (P1)（**10** RadixCache 变体（**纠正 mem_cache.md 8 → 10**）+ `BasePrefixCache` 子类树 + `UnifiedRadixCache` 4-component 重构进行中 + HiCache `init_load_back/ready_to_load_host_cache` 仅 `Hi*RadixCache` 实现 + multimodal embedding cache 3 子图正交 + 8 触发器→10 分支 不一一对应） |
| Continuous batching / chunked prefill | `topics/continuous-batching.md` | TODO |
| **MoE**（dispatch/combine, EPLB, Elastic EP, TBO/SBO, sgl-kernel 绑定） | [topics/moe.md](topics/moe.md) | **DONE** |
| DP attention | `topics/dp-attention.md` | TODO |
| Speculative decoding | [topics/speculative.md](topics/speculative.md) | **DONE** (P1)（5 算法族 EAGLE/MultiLayer/Standalone/DFlash/NGRAM × V1/V2 + 2 worker 接入协议 subclass vs duck-typed + sgl-kernel 2 CUDA 算子 `verify_tree_greedy` / `tree_speculative_sampling_target_only` + `model_runner_list` MTP 多 ModelRunner + 完整 CLI 表 + cross-page lint 待修：comparison 24→27 / 12→11） |
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
