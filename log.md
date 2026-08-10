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

> **如需检索归档内容**：直接 `Grep` / `Read` 对应的 `log-archive/log-NNN.md` 文件。Schema-update entries（AGENTS.md 演化轨迹）现都在 `log-002.md`，对未来 agent 仍可索引。

---

## [2026-04-19] archive | wiki | log rotation 005（P4 batch entry append 前预防触发，第 3 次同日轮转）

**触发**：用户 `ingest sglang modules all`。Main log 当前 109 KB / 962 行 / 10 entries（log-004 轮转后状态）。本轮 P4 batch entry 预估 ~25 KB，append 前主 agent 检查 size 必触发 110 KB 字节阈值；按 [AGENTS.md §11](AGENTS.md) 推荐"先轮转，后追加"模式主动触发 log-005 轮转。

**轮转结果**

| 文件 | 期间 | entries | 大小 |
|---|---|---|---|
| **新建** [log-archive/log-005.md](log-archive/log-005.md) | 2026-04-18（中段补移） | 4 | ~40 KB |
| **重写** [log.md](log.md) | 当前 | 6 + 本 archive + P4 ingest = **8** | ~95-100 KB |

**转移内容**（按 §11 保留策略 — 移除最旧 4 entries 一次到位）

| 转移条目 | 原 line | 操作日期 | 类型 |
|---|---|---|---|
| batch verify vLLM P0 + ingest sglang entities P0 + compare multiproc-ipc | 585 | 2026-04-18 | batch |
| archive log rotation 003 | 702 | 2026-04-18 | archive |
| compare executor-worker across all 3 projects | 775 | 2026-04-18 | compare |
| compare engine-architecture across all 3 projects | 882 | 2026-04-18 | compare |

**保留在主 log 的 6 条最新 entries**（newest → oldest，append 本 archive entry 后）

| # | 操作 | scope | 摘要 |
|---|---|---|---|
| 1 | archive | wiki | log rotation 004 |
| 2 | ingest | sglang modules P3 | function_call + constrained + parser + sampling + weight_sync + checkpoint_engine（5 subagent → 6 页） |
| 3 | ingest | sglang modules P1+P2 | speculative + eplb + elastic_ep + compilation + connector |
| 4 | ingest | sglang modules P0 | disaggregation + distributed + model_executor |
| 5 | lint | wiki | post compare-prefix-cache health check |
| 6 | compare | cross | prefix-cache across all 3 projects |

**链接修复**：[log-archive/log-005.md](log-archive/log-005.md) 内 wiki 路径已 rewrite 为相对从 `log-archive/` 出发（`../comparison/...` / `../mindie/...` / `../vllm/...` / `../sglang/...` 等）；`log-archive/log-NNN.md` 路径改为同目录相对 `log-NNN.md`；`d:\design\<...>` 绝对路径不变。

**净变化**

| | before | after |
|---|---|---|
| log.md 行数 | 962 | 584（节省 ~39%）→ append archive+P4 后约 ~870 行 |
| log.md 字节 | ~109 KB | **~71 KB**（合规：< 110 KB 阈值）→ append 后预估 ~95-100 KB |
| log.md entries | 10 | **6 + 本 archive + P4 = 8** |
| 归档文件数 | 4（log-001/002/003/004） | **5**（+ log-005） |
| 归档总字节 | ~275 KB | **~315 KB**（与 log.md 历史完全一致，无信息丢失） |

**Lessons learned**

1. **同日 3 次轮转的节奏**（log-003 上午 / log-004 下午 / log-005 晚 P4 前）：今天是 ingest 密集日，5 个 sglang module batch（P0 / P1+P2 / P3 / P4）每个 entry 都 ~25-30 KB，平均 1-2 个 ingest entry 就触发轮转。**这种节奏是健康的**——只要 archive 模板固化、链接 rewrite 自动化（本轮已用 PowerShell 脚本批量替换路径），轮转成本极低。
2. **轮转脚本化**：本轮首次用 PowerShell 脚本（`.tmp_archive_log005.ps1`）做 truncate + path-rewrite，比手写 archive entry + 逐处替换链接快 10x。脚本可固化为 wiki 工具：输入 `(start_line, end_line, archive_id)`，输出 `log-NNN.md` + 截断主 log + 提示更新 Archive Index + 新 archive entry 模板。**未来 ingest 密集期建议把这个脚本沉淀到 `wiki/tools/`**。
3. **预估 entry 大小指导轮转量**：P3 批 lessons learned 已沉淀"P3-级大 batch entry append 前应预估 entry 大小并多移 1-2 entries"；本轮一次到位移 4 entries 而非 6（因为 P4 entry 预估 25 KB 与 P3 30 KB 略小，且 archive entry 也只 ~5 KB），实测 archive+P4 后主 log 约 95-100 KB，留 10-15 KB 余量。

**下次轮转预期**：按当前 sglang module ingest 节奏，P5（剩余 14 模块）可能再分 2-3 batch；每个 batch 后预计触发 1 次轮转 → log-006 / 007 / ... 在本月内可能达 log-010+。

---

## [2026-04-19] ingest | sglang modules "all" P4 batch | entrypoints/openai + hardware_backend + multimodal + lora + model_loader（5 subagent 并行）

**触发**：用户 `ingest sglang modules all`。"all" 字面要求覆盖剩余 19 个 TODO 模块（详见 [sglang/index.md](sglang/index.md)），但按已沉淀 lessons learned「单 turn 5 subagent 上限 + `models`/`layers` 必须独立 turn」，本 turn 主 agent 选 **5 个最高 ROI 中-大模块** 作为 P4 batch 处理；剩余模块通过本 entry 末尾「下一步建议」清晰交付，由用户后续 turn 触发。

| 模块 | 文件量 | Tier | 选 P4 理由 |
|---|---|---|---|
| `entrypoints/openai` | 21 .py / 320 KB（实测 20 .py） | P4 | 最大 TODO；OpenAI 兼容层；与已 ingest 的 [function_call](sglang/modules/function_call.md) / [parser](sglang/modules/parser.md) / [constrained](sglang/modules/constrained.md) 强耦合，闭合"输出协议"链路 |
| `hardware_backend` | 22 .py / 312 KB | P4 | NPU/MUSA/MLX 设备子系统；已被 [`model_executor.md`](sglang/modules/model_executor.md) / [`compilation.md`](sglang/modules/compilation.md) 标记为关键依赖（`NPUGraphRunner` / `NPUPiecewiseBackend`）|
| `multimodal` | 46 .py / 360 KB | P4 | EPD encoder 上游依赖（[`disaggregation.md`](sglang/modules/disaggregation.md) `encode_*.py` 复用 `preprocess_video`）+ 36 模型族处理器矩阵 |
| `lora` | 33 .py / 380 KB | P4 | LoRA adapter pipeline；与 [`weight_sync.md`](sglang/modules/weight_sync.md) `flattened_bucket` 路径直连；S-LoRA / Punica 谱系 |
| `model_loader` | 6 .py / 285 KB | P4 | **闭合 4 条权重通道**：connector + weight_sync + checkpoint_engine + **model_loader**；vLLM 直接谱系 |

**执行方式**：5 个 explore subagent **完全并行**（read-only 模式返回 source-anchored 草稿）+ 主 agent 写入 + normalize anchor 格式 + 协同更新。**所有 5 subagent 输出长度均 > 23 KB**（function_call ~42K / hardware ~34K / multimodal ~40K / lora ~34K / model_loader ~44K），无 P3 批次中 constrained 的 token-budget 截断问题。

**新建页（5 个）**

| 文件 | 大小 | 关键涵盖 |
|---|---|---|
| [sglang/modules/entrypoints_openai.md](sglang/modules/entrypoints_openai.md) | ~25 KB | **20 .py**（无包根 `__init__.py`；纠正既有 wiki "21 .py" 推断）+ `OpenAIServingBase` 抽象 + 10 `OpenAIServing*` 子类（Chat/Completion/Responses/Embedding/Score/Rerank/Classify/Tokenize+Detokenize/Transcription，**`OpenAIServingResponses` 子类化 `OpenAIServingChat`**）+ `protocol.py` **77 BaseModel** + `OpenAIServingRequest` 8-element Union（`ResponsesRequest` 不在 Union；路由层手动构造）+ 11 `/v1/*` 路由全表 + `MCPToolServer` / `DemoToolServer` SSE 装配 + Reasoning + function_call + constrained 集成点 + `Adapted from vLLM OpenAIServingResponses` 谱系标注 |
| [sglang/modules/hardware_backend.md](sglang/modules/hardware_backend.md) | ~22 KB | **22 .py / 3 设备子包** npu/musa/mlx（无 `BaseHardwareBackend` 注册表抽象；模式为基类在 `model_executor`/`multimodal`/`speculative`，本目录提供覆盖实现）+ `NPUGraphRunner` 子类化 `CudaGraphRunner` + 4 个 `*GraphRunner` 子类（NPU / ViTNpu / EAGLEDraftNpu / EAGLEDraftExtendNpu）+ `NPUPiecewiseBackend` **位于 `compilation/` 而非本目录** + `init_npu_backend` import `sgl_kernel_npu` + `set_default_server_args` 按显存档位预设 + MUSA `MusaFlashAttentionBackend` + MLX `MlxTpModelWorker` 替换 `TpModelWorker`（绕过 PyTorch 权重加载）+ `_handle_npu_backends` 强制 `piecewise_cuda_graph_compiler='eager'` |
| [sglang/modules/multimodal.md](sglang/modules/multimodal.md) | ~24 KB | **46 .py / 37 processors/** + `BaseMultimodalProcessor` + `MultimodalSpecialTokens` + `import_processors` 启动注册到 `PROCESSOR_MAPPING`（按 HF `architectures` 分发）+ 36 模型族处理器矩阵（Qwen-VL / LLaVA / InternVL / Gemma3/4 / GLM-4V / DeepSeek-VL2/OCR / Kimi-VL/K2.5 / NVILA / Pixtral / Phi-4-MM / MiniCPM / Janus-Pro / Step3-VL / Mllama / Llama4 / Whisper / Voxtral / Qwen-Audio/ASR / GLM-ASR）+ `TransformersAutoMultimodalProcessor` 兜底 + 3 模态 IMAGE/VIDEO/AUDIO + Token 展开（grid_thw 驱动） + EPD 单向依赖 [`encode_server.py:40`](d:\design\sglang\python\sglang\srt\disaggregation\encode_server.py) `from sglang.srt.multimodal.processors.qwen_vl import preprocess_video` + `vit_cuda_graph_runner` ViT 图 + `evs/` 视频 token 剪枝 + 8 个 `--mm-*` CLI 字段 |
| [sglang/modules/lora.md](sglang/modules/lora.md) | ~26 KB | **33 .py** + `LoRAManager` 加载/卸载/批准备 + `LoRAMemoryPool` GPU 槽位 A/B 缓冲 + `LoRABatchInfo` `seg_indptr`/`weight_indices`/`lora_ranks` + 4 可运行后端（`triton` / `csgmv` / `ascend` / `torch_native`；`flashinfer` 弃用占位 raise）+ **13 个 `@triton.jit` 内核**（`_sgemm_lora_a/b` / `_qkv_lora_b` / `_gate_up_lora_b` / `_embedding_lora_a` / `_chunked_lora_shrink/expand` / `_chunked_embedding_lora_a` / `_fused_moe_lora` / `_fused_virtual_topk_ids` / `_fused_sanitize_expert_ids` / `_moe_lora_shrink_splitk` / `_resolve_token_positions`）+ **sgl-kernel CUDA adapter-LoRA = 0** 关键 N/A 强论断（命中均为 MLA `q_lora_rank` / `kv_lora_rank` 维度名，与 adapter LoRA 无关）+ S-LoRA / Punica 文档谱系 + `flattened_bucket` 张量直灌路径（[`tp_worker.py:192-207`](d:\design\sglang\python\sglang\srt\managers\tp_worker.py)）+ HTTP `/load_lora_adapter` / `/unload_lora_adapter` / `/load_lora_adapter_from_tensors` 三 endpoint + `LoRAOverlapLoader` 异步 H2D 重叠 |
| [sglang/modules/model_loader.md](sglang/modules/model_loader.md) | ~24 KB | **6 .py + `LoadFormat` 19 枚举**（CLI `LOAD_FORMAT_CHOICES` 16 项不一致：缺 `jax` / `rdma` / `local_cached`）+ `BaseModelLoader` **11 具体子类**（Default / Layered / QuantizedRL / ModelOpt / Dummy / ShardedState / BitsAndBytes / GGUF / RemoteInstance / Remote / RunaiModelStreamer；`Private` 动态 import）+ `get_model_loader` 工厂分支 [`loader.py:3147-3229`](d:\design\sglang\python\sglang\srt\model_loader\loader.py)（ModelOpt 量化标志优先 → 显式类 → 各 `LoadFormat` enum → 默认回落）+ `weight_utils.py` 9 类权重迭代器 + `default_weight_loader` 族 + 量化两阶段 `process_weights_after_loading` + `RemoteInstanceWeightLoaderBackend` 3 协议（nccl/transfer_engine/modelexpress）+ **代码血缘**：`__init__.py` / `loader.py` / `weight_utils.py` / `utils.py` / `load_config.py` 全部 `Adapted from vllm v0.6.3-0.6.4` + SGLang 上叠 `REMOTE` / `REMOTE_INSTANCE` / `FLASH_RL` / `RUNAI_STREAMER` / `PRIVATE` / ModelOpt / QuantizedRL 分支 + checkpoint_engine 集成点（`SGLangCheckpointEngineWorkerExtensionImpl.get_post_hook` 复刻 `DefaultModelLoader` 量化后处理） |

**关键 synthesis（5 模块综合）**

1. **「4 条权重通道」闭环**（最大 anti-误读价值；本批新建 [`model_loader.md`](sglang/modules/model_loader.md) 收束）：**加载期** → [`model_loader/`](sglang/modules/model_loader.md)（`LoadFormat` 19 枚举 / 11 loader 子类 / vLLM 直接谱系）+ [`connector/`](sglang/modules/connector.md)（远程权重 Redis/S3/instance:// → 经 `RemoteModelLoader` 消费）；**运行期** → [`weight_sync/`](sglang/modules/weight_sync.md)（训练 SPMD `flattened_bucket` + `update_weights_from_tensor`）+ [`checkpoint_engine/`](sglang/modules/checkpoint_engine.md)（Moonshot ParameterServer + ZMQ IPC `/update_weights_from_ipc`）。**4 条通道的 SGLang 实现已全部 ingest，可作为后续 cross compare topic 种子**。
2. **OpenAIServing 类层次**：`OpenAIServingBase` (ABC) → 10 子类 + `OpenAIServingResponses` 进一步子类化 `OpenAIServingChat`。**Anthropic 桥接层** [`AnthropicServing`](d:\design\sglang\python\sglang\srt\entrypoints\anthropic\serving.py) 转 ChatCompletionRequest 复用 `OpenAIServingChat`；**Ollama 完全独立**（不 import openai 包，自带 protocol）。这是三家 OpenAI 兼容路径中**最深的类层次**，主路径外还有 MCP tool server 注入到 Responses API。
3. **`hardware_backend` 不是统一注册表**：与"vLLM `vllm/platforms/` 统一平台接口"形成对比 —— SGLang 设备探测在 [`utils/common.py`](d:\design\sglang\python\sglang\srt\utils\common.py)（`is_npu` / `is_cuda` 等），设备专用代码收敛到 `srt/hardware_backend/` + `compilation/*npu*` + 各 `layers/quantization/*_npu.py`。**4 个 `*GraphRunner` 子类 + 1 个 `NPUPiecewiseBackend`** 是本目录核心可观测产出；**`NPUPiecewiseBackend` 位于 `compilation/` 而非 hardware_backend** 是命名陷阱。
4. **multimodal ? disaggregation 单向依赖**：[`encode_server.py:40`](d:\design\sglang\python\sglang\srt\disaggregation\encode_server.py) **import** `multimodal/processors/qwen_vl.preprocess_video`；`multimodal/` 不反向 import `disaggregation/`。这是 **EPD 编码器架构上 multimodal 是基础层、disaggregation/encode 是消费方** 的精确锚定 —— [`comparison/topics/pd-disaggregation.md`](comparison/topics/pd-disaggregation.md) 14 子维度未含 EPD，可作为后续 cross compare 种子。
5. **`SGLang lora` adapter-LoRA = 0 个 sgl-kernel C++/CUDA 算子**（**关键 N/A 强论断**）：在 `d:\design\sglang\sgl-kernel\csrc\` 全树 grep `lora` / `LoRA` / `Lora` / `BatchedLoRA` / `sgmv` / `bgmv` / `punica` —— 命中均为 **DeepSeek MLA 的 `q_lora_rank` / `kv_lora_rank` 维度名**（如 [`csrc/cpu/qkv_proj.cpp`](d:\design\sglang\sgl-kernel\csrc\cpu\qkv_proj.cpp)），**与 adapter serving 无关**。SGLang LoRA 全靠 **13 个 `@triton.jit` 内核** + Python 编排，与 vLLM 突出 Punica/BGMV CUDA 路径形成强对比。
6. **`model_loader` 是 SGLang 中 vLLM 谱系最深的模块**：5 个文件全部 `Adapted from vllm v0.6.3-0.6.4`（[`__init__.py:1`](d:\design\sglang\python\sglang\srt\model_loader\__init__.py) / [`loader.py:1`](d:\design\sglang\python\sglang\srt\model_loader\loader.py) / [`weight_utils.py:1`](d:\design\sglang\python\sglang\srt\model_loader\weight_utils.py) / [`utils.py:1`](d:\design\sglang\python\sglang\srt\model_loader\utils.py) / [`load_config.py:1`](d:\design\sglang\python\sglang\srt\configs\load_config.py)），**SGLang 仅在之上叠 `REMOTE` / `REMOTE_INSTANCE` / `FLASH_RL` / `RUNAI_STREAMER` / `PRIVATE` / ModelOpt / QuantizedRL** 分支。这种"基础层 vLLM 复用 + 业务层 SGLang 扩展"是健康的 fork 策略；与 [`compilation/`](sglang/modules/compilation.md) "Adapted from vllm v0.10.0" 同派。
7. **`LoadFormat` 19 枚举 vs `LOAD_FORMAT_CHOICES` 16 项 CLI 不一致**：`JAX` / `RDMA` / `LOCAL_CACHED` 是 enum 占位但 CLI 不可选 + `get_model_loader` 无对应分支 —— **死代码 / 未来扩展占位** 是数字精度类 CONTRADICTION 的高价值发现。
8. **数字精度核对**：5 模块的文件计数与类层次均经 Glob + grep 直接匹配；其中 `entrypoints/openai` 实测 **20 .py** 而既有 wiki 推断为 21 .py（已加 VERIFY），`hardware_backend` 22 .py 与既有数据一致，`multimodal` 46 .py 精确，`lora` 33 .py 精确，`model_loader` 6 .py 精确。

**协同更新（5 文件）**

| 文件 | 修改 |
|---|---|
| [sglang/index.md](sglang/index.md) | 5 行 TODO → DONE：entrypoints/openai / hardware_backend / multimodal / lora / model_loader（含 P4 标 + 一句话摘要 + 4 条权重通道闭环引用） |
| [sglang/modules/connector.md](sglang/modules/connector.md) | frontmatter `related:` 加 model_loader.md（4 通道闭环） |
| [sglang/modules/weight_sync.md](sglang/modules/weight_sync.md) | frontmatter `related:` 加 model_loader.md |
| [sglang/modules/checkpoint_engine.md](sglang/modules/checkpoint_engine.md) | frontmatter `related:` 加 model_loader.md |
| [sglang/modules/model_executor.md](sglang/modules/model_executor.md) | frontmatter `related:` 加 hardware_backend / model_loader / multimodal（3 新模块均为 ModelRunner 直接消费方） |
| [sglang/modules/compilation.md](sglang/modules/compilation.md) | frontmatter `related:` 加 hardware_backend.md（NPUPiecewiseBackend 互引） |
| [sglang/modules/disaggregation.md](sglang/modules/disaggregation.md) | frontmatter `related:` 加 multimodal.md（EPD encode_server 单向依赖） |
| [sglang/modules/entrypoints.md](sglang/modules/entrypoints.md) | frontmatter `related:` 加 entrypoints_openai.md |

**Markers added**

| 类型 | 数量 | 详情 |
|---|---|---|
| `[!todo] VERIFY` | **5** | entrypoints/openai 1（21 vs 20 .py 数字差异）+ hardware_backend 2（`sgl_kernel_npu` 包布局 / MUSA/MLX 无 `*GraphRunner` 子类）+ multimodal 1（`MMEmbeddingCache` 历史命名当前不存在，缓存实为 `MultiModalStaticCache` + `EmbeddingCacheController`）+ lora 1（PD 分离 / overlap scheduler 与 LoRA 交互细节） + model_loader 1（`LoadFormat.JAX/RDMA/LOCAL_CACHED` 占位是否为死代码） |
| `[!warning] CONTRADICTION` | **6** | entrypoints/openai 1（`EmbeddingResponse.model` 用 `model_path` 而非 `served_model_name`）+ hardware_backend 1（`NPUPiecewiseBackend` 字段名仍用 `cudagraph` 但实为 NPUGraph）+ multimodal 1（多模态全局 cache 跨 `mem_cache/` + `disaggregation/encode_server.py` + `multimodal/` 三处而非一个目录）+ lora 2（`flashinfer` 后端弃用占位仍 register 但 raise + SGMV/BGMV 命名混用）+ model_loader 1（CLI `LOAD_FORMAT_CHOICES` 16 项缺 enum 中 3 项 + `flattened_bucket` 文档与 `LoadFormat` 不同概念混用） |
| §跨子系统 N/A 强论断 | **15+** | 5 模块 × 各 3 处 |
| **sgl-kernel 算子绑定累计** | **0 新增 + 1 关键 N/A** | LoRA adapter-serving = 0（lora 命中均为 MLA 维度名）；本批 5 模块**均无新发现 sgl-kernel C++ 绑定**（hardware_backend 是 `sgl_kernel_npu` 消费方而非 `sgl-kernel/csrc/` 注册方）—— 维持 P3 批后的 **7 个累计**：`shm_allreduce` / `verify_tree_greedy` / `tree_speculative_sampling_target_only` / `weak_ref_tensor` / `top_k_renorm_probs` / `top_p_renorm_probs` / `apply_token_bitmask_inplace_cuda` |

**Anchor 抽查命中率**：5/5（entrypoints/openai `OpenAIServingBase.handle_request` L73-109 / hardware_backend `NPUGraphRunner` L73 子类化 + `init_npu_backend` L99-107 / multimodal `MultimodalSpecialTokens` L47-172 + `import_processors` 启动注册 / lora `ToolCallParserEnum`-style `LORA_BACKEND_CHOICES` L210 + 13 个 `@triton.jit` 内核全文件锚 / model_loader `LoadFormat` L15-34 19 枚举 + `get_model_loader` L3147-3229 分支表）

**净变化**

| | before | after |
|---|---|---|
| sglang/modules DONE 数 | 17/30 | **22/30**（+5：entrypoints/openai + hardware_backend + multimodal + lora + model_loader） |
| sglang 总页数 | 24 | **29** |
| 三家 entities 总数 | 24 | 24（无变化） |
| comparison topics DONE 数 | 14/24 | 14/24（本批深化 SGLang 模块层而非 cross compare） |
| 总 .md 文件数（excluding raw/ + log-archive/） | 88 | **93**（+5） |
| 4 条权重通道完整 ingest 状态 | 3/4（缺 model_loader） | **4/4 闭环**（model_loader 收束；可作为新 cross compare topic 种子） |
| sglang 模块剩余 TODO | 19 | **14**（model_loader/openai/hardware/multimodal/lora 完成） |

**Lessons learned**

1. **「all 不能 1 turn 完成」的明示交付**：用户 `ingest sglang modules all` 包含 19 个 TODO 模块，按 lessons learned「单 turn 5 subagent 上限」无法 1 turn 完成。**主 agent 选 5 个最高 ROI 模块 + entry 末尾「下一步建议」精确列出剩余 14 模块的分批方案 + 大模块独立 turn 警告**，而非默默放弃 / 错误声称完成。这是健康的「expectation management」模式，**未来类似「all / everything」字面要求都应这样响应**。
2. **subagent 输出大小验证**：本批 5 subagent 输出长度均 23-44 KB（远超 P3 批 constrained subagent 23 KB 截断阈值）。回顾 P3 批 lessons：**< 30 KB 的 subagent 产出需主动监测**；本批所有 subagent 都是 ≥ 23 KB（接近临界），但 multimodal 处理器矩阵完整、entrypoints/openai 全 endpoint 表完整 —— **23 KB 是临界值但仍可用**，30 KB 以下时主 agent 应抽查关键 section 完整性。
3. **「N 条权重通道闭环」是高价值 synthesis 模板**：本批 [`model_loader.md`](sglang/modules/model_loader.md) 收束 4 条通道，是 P3 批「3 条通道」synthesis 的自然升级。**这种"系统性枚举正交子系统"是 cross compare 的强模板**：未来对 vLLM / MindIE 做相同维度的 4 通道分类，可形成新 cross compare topic。
4. **预防式轮转脚本化**：本轮首次用 PowerShell 脚本（[`.tmp_archive_log005.ps1`](d:\design\wiki) 临时文件）做 truncate + path-rewrite，比手写 archive entry + 逐处链接替换快 10x。**未来应固化到 `wiki/tools/`**（与 lessons learned in archive 005 entry 同步）。
5. **既有 wiki 数字推断 vs 实测差异**：entrypoints/openai 既有 wiki 推断 21 .py 实测 20 .py。这种小数字差异是**前批 ingest entry 中"21 .py / 320 KB"被 P4 批用作 prompt 输入但 subagent Glob 实测后发现差异**的良性纠错路径。**未来 prompt 给 subagent 的"已知数字"应明确标注"待 verify"避免锚定效应**。
6. **同名异义模块陷阱再现**：本批 `hardware_backend` 与 vLLM `vllm/platforms/` 是「同语义异组织」（前者非统一注册表，后者统一接口）；`multimodal` 与 vLLM `vllm/multimodal/` 是「同语义异规模」（vLLM 112 .py 量级 vs SGLang 46 .py）。**Cross compare 时不能简单按目录名套等价**，需先做"组织模式"对照（这与 P0 批 `disaggregation` 命名陷阱 / P2 批 `connector` 命名陷阱同模式）。

**当前 wiki 状态**

| 项目 | overview | index | modules | entities | topics | 总数 |
|---|---|---|---|---|---|---|
| MindIE-LLM | ? | ? | 3/14 | 9/9 | 8/13 | 20 |
| vLLM | ? | ? | 2/26 | 10/17 | 5/14 | 17 |
| SGLang | ? | ? | **22/30**（+5） | 5/11 | 2/14 | **29** |
| comparison | — | ? | — | — | 14/24 | 14 + dimensions |
| 顶层 meta | 5 | — | — | — | — | 5 |

**总 .md 文件数**：**93**（excluding `wiki/raw/` + `wiki/log-archive/`）；**98**（including 5 archive files）。delta +5。

**§11 log rotation 阈值复核**（append 本 entry + 上面的 archive entry 后）：bytes ~95-100 KB / lines ~870 / entries 8 → 仍在 110 KB / 1500 行阈值内。下次轮转预计在 1-2 个大型 ingest entry 后触发。

**下一步建议**

按 ROI 排序，**剩余 14 个 sglang TODO 模块**分批方案：

1. **`ingest sglang modules small-rollup`**（~5 模块的小模块 rollup）—— `multiplex` (2.py) + `tokenizer` (1.py) + `batch_invariant_ops` (2.py) + `batch_overlap` (4.py) + `dllm` (7.py) 或 `ray` (5.py)。这些都是小模块（< 10 .py / < 65 KB），1 个 subagent 可同时产出 2-3 页（参考 P3 批 weight_sync + checkpoint_engine 配对模式）。
2. **`ingest sglang modules entrypoints-rest`** —— `entrypoints/anthropic` (3.py) + `entrypoints/ollama` (4.py) + `grpc` (1.py 占位) 三个小模块；与本批 `entrypoints/openai` 配套。
3. **`ingest sglang modules utility-3`** —— `observability` (10.py) + `configs` (44.py 多为 dataclass) + `debug_utils` (89.py 多小文件)。其中 configs / debug_utils 文件多但内容轻，可压缩到 1-2 页。
4. **`ingest sglang modules models`** —— **独立 turn**（186 .py / 3.9 MB；按模型族分组而非逐文件，参考 multimodal processors matrix 模式）
5. **`ingest sglang modules layers`** —— **独立 turn**（252 .py / 4.1 MB；按 attention / MoE / linear / quant / norm / rotary 等子目录分组）
6. **后续 cross compare 候选**：
   - `compare model-loading across all`（4 条权重通道在三家的对偶矩阵；vLLM `vllm/model_executor/model_loader/` + MindIE 待 grep）
   - `compare openai-compat across all`（vLLM `vllm/entrypoints/openai/` 是 SGLang 直接谱系；MindIE 兼容路径）
   - `compare multimodal across all`（vLLM 112 .py vs SGLang 46 .py 规模差异）
   - `compare lora-serving across all`（Punica/BGMV vs S-LoRA/SGMV 哲学差异）
7. **被动等待**：三仓 HEAD 都未动，无需触发 §12 增量

---

## [2026-04-19] verify | sglang/modules/* | 36 页 batch verify pass（6 subagent 并行；92 markers → 68 RESOLVED + 24 retained）

**触发**：用户 `verify sglang/modules/*`，承接 Turn 3+4+5 mega-batch entry 末尾「下一步建议」第 9 项「verify sglang/modules/* — 累计 40+ markers 待消化」。

**范围确认**：30 主入口 + 6 子页 = **36 页 sglang/modules** 全覆盖（与 [sglang/index.md](sglang/index.md) 当前 30 行 modules 表 + 5 entrypoints 子页 + grpc 一致）。Pre-pass 统计：

| 指标 | before |
|---|---|
| `[!todo] VERIFY` 总数 | 52 |
| `[!warning] CONTRADICTION` 总数 | 34 |
| **Markers 总和** | **86 + 6 (managers/disaggregation/multimodal/connector 重复计) = 92** |
| `verified_against: 2026-04-17` 页数 | 3（managers / entrypoints / mem_cache） |
| `verified_against: 2026-04-18` 页数 | 33 |
| `RESOLVED *` 标记数 | 0（首次大规模 verify pass） |

**执行方式**：6 subagent **完全并行**（沿用 Turn 3+4+5 验证的「6 subagent 是新可控上限」节奏）。每个 subagent 按 [§6 verify workflow](AGENTS.md) 执行：(1) 读全页；(2) 对每个 marker 用 Grep+Read 复查源码并尝试 RESOLVED；(3) 抽 3-5 anchor 实地核对；(4) RESOLVED 时 strikethrough 原 marker body + 追加 `> **RESOLVED 2026-04-19**: <结论> ([anchor](...))` 块；(5) 任意修改触发 frontmatter `verified_against` 升至 `2026-04-19`。

| Batch | 页面 | markers_before | resolved | retained |
|---|---|---|---|---|
| **A** entrypoints 家 + observability | entrypoints / entrypoints_openai / entrypoints_anthropic / entrypoints_ollama / grpc / observability | 7 | 6 | 1（grpc 命名陷阱永久） |
| **B** 调度/PD/parser | managers / disaggregation / multiplex / function_call / constrained / parser | 21 | 12 | 9（design caveats + 1 上游 Gemma4 对照） |
| **C** model exec + 大模块 | model_executor / model_loader / models / layers / lora / sampling | 14 | 9 | 5（observational + 1 PD/LoRA 跨页 unresolved） |
| **D** memory + spec + dllm | mem_cache / checkpoint_engine / weight_sync / speculative / dllm / batch_invariant_ops | 16 | **16** | **0** ? |
| **E** distributed + HW + connector | distributed / eplb / elastic_ep / ray / hardware_backend / compilation / connector | 24 | 18 | 6（4 design caveats + `eplb_simulator` 命名 + `sgl_kernel_npu` wheel 外部） |
| **F** utility 小 | tokenizer / multimodal / batch_overlap / debug_utils / configs | 10 | 7 | 3（1 git diff pending + 2 命名 caveats） |
| **总计** | **36 页** | **92** | **68** | **24** |

**Marker 处理总览**

| 处置 | 数量 | 性质 |
|---|---|---|
| ? **RESOLVED** | **68** | strikethrough + `> **RESOLVED 2026-04-19**: <结论> + [anchor](...)` 块；附直接源码证据 |
| ?? **Design caveat 永久保留** | 18 | 命名陷阱 / 永久反误读警示（grpc / multiplex 与 disaggregation / connector 4 系统 / SGMV vs BGMV / overlap 三层 / 4 同名 tokenizer 等）—— 不可消解的事实性提醒 |
| ?? **本地无法核证** | 4 | 上游 Gemma4 输出对照（无外部模型）/ `configs.md` git diff（工作区无 git）/ `sgl_kernel_npu` wheel 包布局（外部 wheel）/ `eplb_simulator` 命名/路线决策 |
| ?? **跨页 unresolved** | 2 | `lora.md` PD-disaggregation × LoRA 交互（需读 PD 主题页）/ `multimodal.md` 多模态分布在 multimodal+mem_cache+disaggregation 三处的提醒 |

**关键 RESOLVED synthesis**（高价值发现 8 条）

1. **`update_config.py` ≠ CLI JSON merge**（[configs.md](sglang/modules/configs.md) RESOLVED）：实测函数名 `adjust_config_with_unaligned_cpu_tp` 是 **TP head/intermediate 形状对齐补丁**（[update_config.py:112](d:\design\sglang\python\sglang\srt\configs\update_config.py)）。CLI JSON merge 实际在 [`hf_transformers/config.py`](d:\design\sglang\python\sglang\srt\utils\hf_transformers\config.py)。命名陷阱已永久写入 wiki。
2. **`LoadFormat` 19 vs `LOAD_FORMAT_CHOICES` 16 是 deliberate dead-end placeholder**（[model_loader.md](sglang/modules/model_loader.md) RESOLVED）：`JAX` / `RDMA` / `LOCAL_CACHED` 在 `get_model_loader` 无分支（[loader.py:3142-3229](d:\design\sglang\python\sglang\srt\model_loader\loader.py)），用户显式设置必触发 `ValueError: Unknown load_format`。**保留接口为上游同步预留**而非 bug。
3. **`MindSporeRunner` 实际不是类**（[model_executor.md](sglang/modules/model_executor.md) RESOLVED）：[`mindspore_runner.py`](d:\design\sglang\python\sglang\srt\model_executor\mindspore_runner.py) 仅模块级函数（`init_ms_distributed` / `set_ms_parallel_env` 等），是 NPU MindSpore 分布式 init helper，**非平行 Runner**。原 wiki "MindSporeRunner（与 ModelRunner 平行）" 措辞修正。
4. **`compile_sizes` 和 `FixFunctionalizationPass` 都是死代码 / 占位**（[compilation.md](sglang/modules/compilation.md) RESOLVED）：`self.compile_sizes = set([])` 写死空集（[cuda_piecewise_backend.py:77](d:\design\sglang\python\sglang\srt\compilation\cuda_piecewise_backend.py)），按形状二次 Inductor 编译分支永不可达；`FixFunctionalizationPass.__call__` 仅 `count += 1`（[fix_functionalization.py:26-48](d:\design\sglang\python\sglang\srt\compilation\fix_functionalization.py)），从未调用 `defunctionalize`。**vLLM 上游残留接口**。
5. **`ensure_model_parallel_initialized` 全树无运行时调用**（[distributed.md](sglang/modules/distributed.md) RESOLVED）：定义在 [parallel_state.py:2033-2063](d:\design\sglang\python\sglang\srt\distributed\parallel_state.py)，但全仓 grep 仅命中 docstring，**生产路径全走 `initialize_model_parallel`**。
6. **NPU 平台不支持 elastic_ep**（[elastic_ep.md](sglang/modules/elastic_ep.md) RESOLVED）：`_select_device` 显式 `raise NotImplementedError("Only CUDA and CPU support elastic ep now.")`（[elastic_ep.py:46-53](d:\design\sglang\python\sglang\srt\elastic_ep\elastic_ep.py)），但 Ascend 文档列 `--elastic-ep-backend` flag —— **文档与代码矛盾**，建议向上游报告。
7. **`SGLang srt/speculative/` 27 .py + `vllm v1/spec_decode/` 11 .py**（[speculative.md](sglang/modules/speculative.md) RESOLVED）：实测精确数字与 [comparison/topics/speculative-decoding.md](comparison/topics/speculative-decoding.md) L80/L439 旧 24/12 不符；**触发 cross-page lint 跟进项**（log 末「下一步建议」单列）。
8. **`@triton.jit` in `srt/layers/` = 60 文件**（[layers.md](sglang/modules/layers.md) RESOLVED）：旧 wiki 估值 71 偏高；实测 60 文件 / ~176 hits（聚集在 `moe/ep_moe/kernels.py` / `attention/utils.py` / `quantization/fp8_kernel.py`）。同时确认 `ATTENTION_BACKENDS` 17 个注册名顺序与既有 wiki 完全一致。

**Anchor spot-check 总结**：6 batch 累计抽查 ≥ 40 处 `[file:line]` anchor，全部 ±0 偏移命中（mem_cache.md 30+ anchors 实地复核 0 漂移最为亮点 — 该页 `verified_against: 2026-04-17` 最旧但 RadixCache 家族稳定）。**0 anchor fix 实施 / 0 新增 CONTRADICTION**。

**Frontmatter 净变化**

| | before | after |
|---|---|---|
| `verified_against: 2026-04-17` 页 | 3 | **0**（managers/entrypoints/mem_cache 全部升至 04-19）|
| `verified_against: 2026-04-18` 页 | 33 | **0** |
| `verified_against: 2026-04-19` 页 | 0 | **36 / 36 = 100%** |
| `status: verified` 页 | 36（全部已为 verified） | 36（status 不降级；剩余 marker 均非 falsifiable bug） |
| `RESOLVED 2026-04-19` 标记总数 | 0 | **66**（部分 batch 合并入表格 cell，与 68 resolved 总数差 = 2 处合并写入） |

**协同更新**：本 verify pass **不触发**子页协同（subagent 仅在 wiki/sglang/modules/* 内编辑 + frontmatter）。

**净变化**

| | before | after |
|---|---|---|
| sglang/modules `verified` 状态率 | 100%（status）+ 0%（04-19 pin） | **100% + 100% / 04-19 pin** |
| 累计 marker 解决率 | 0 / 92 = 0% | **68 / 92 = 73.9%** |
| 待办 marker（design caveats / unresolvable） | 92 | **24**（其中 18 永久 caveat / 4 本地不可证 / 2 跨页待跟进） |
| sglang 总页数 | 37 | 37（无新页） |
| 总 .md 文件数（excluding raw/ + log-archive/） | 107 | 107 |

**Lessons learned**

1. **「6 subagent + 6 batch 全 36 页」node 已稳定**：6 subagent 完全并行处理 36 页 = ~6 页/subagent；输出报告平均 ~3-5 KB；总执行时间约 1 个 turn。**对比 Turn 3+4+5 ingest（6 subagent → 8 页 + 大量 synthesis）**：verify 任务比 ingest **更适合并行**（无需主 agent cross-page synthesis，subagent 可直接 write）。
2. **Design caveat 与 falsifiable claim 区分**：18 / 24 retained markers 是「永久反误读警示」（grpc 命名 / multiplex vs disaggregation / 4 同名 tokenizer / SGMV vs BGMV 等），**不应也不能 RESOLVED** —— 它们的存在本身就是教学价值。**建议 [AGENTS.md §6](AGENTS.md) 加分类标签**：`> [!warning] CONTRADICTION (design-caveat)` vs `> [!warning] CONTRADICTION (falsifiable-bug)` 区分两种生命周期。
3. **跨页一致性 lint 项**：本 verify 暴露 [comparison/topics/speculative-decoding.md](comparison/topics/speculative-decoding.md) L80/L439 关于 SGLang/vLLM spec decode 文件数 (24/12) 与最新实测 (27/11) 不符 + [comparison/topics/distributed.md:116](comparison/topics/distributed.md) 已先一步同步。**未来 cross-page 数字应 grep wiki 全树同步更新**而非局部修改。
4. **死代码 / 占位发现率高**：8 个 high-value RESOLVED 中有 **3 个属"死代码 / 占位 / typo"**（`ensure_model_parallel_initialized` / `compile_sizes` / `FixFunctionalizationPass.__call__`）+ **2 个属"类型注解错误"**（`ExpertLocationDispatchInfo.ep_dispatch_algorithm` Literal / `RedisConnector.weight_iterator` 返回类型）。**verify pass 间接成为代码 audit 工具**，建议加入 [AGENTS.md §6 verify](AGENTS.md) 「附产物」章节。
5. **`mem_cache.md` 最旧 verified_against 复核 0 漂移**：`2026-04-17` pin 经过 2 天，`init_cache_with_memory_pool` 8 分支 + `RadixCache` 家族 ≥ 30 anchors 全部命中 → **稳定子系统的 stale lint 阈值可放宽** （当前 §11 lint 仅按"上游修改时间"判 stale，未利用 verify 反向证据）。
6. **`>> Stashed changes` upstream merge artifact 存在 ≥ 2 天**（[batch_invariant_ops.py:388](d:\design\sglang\python\sglang\srt\batch_invariant_ops\batch_invariant_ops.py)）：本次 verify 第二次确认；**建议向上游开 PR 清理**。

**§11 log rotation 阈值复核**（append 本 entry 后）

| | before append | after append（预估） |
|---|---|---|
| log.md 字节 | 65.6 KB | **~80 KB**（合规：< 110 KB 阈值；余 ~30 KB） |
| log.md 行数 | 511 | **~640**（合规：< 1500 行）|
| log.md entries | 5 | **6** |

**当前 wiki 状态**

| 项目 | overview | index | modules | entities | topics | 总数 | verify 状态 |
|---|---|---|---|---|---|---|---|
| MindIE-LLM | ? | ? | 3/14 | 9/9 | 8/13 | 20 | 部分 verified |
| vLLM | ? | ? | 2/26 | 10/17 | 5/14 | 17 | 部分 verified |
| SGLang | ? | ? | **30/30**（**36 页全 04-19 verified ?**） | 5/11 | 2/14 | **37** | **modules 100% verify-04-19** |
| comparison | — | ? | — | — | 14/24 | 14 + dimensions | — |
| 顶层 meta | 5 | — | — | — | — | 5 | — |

**下一步建议**

按 ROI 排序：

1. **同步 cross-page 数字漂移**（lint follow-up）：[comparison/topics/speculative-decoding.md](comparison/topics/speculative-decoding.md) L80/L439 SGLang spec_decode 文件数 24 → 27 + vLLM 12 → 11；可用 1 次小 lint pass。
2. **`compare unique-design-points across all`** —— 汇总三家独有特性矩阵（SGLang 现已有 4 个 unique 设计点 + dead-code/placeholder 累计审计）
3. **`compare model-loading across all`** —— 4 条权重通道（[connector.md](sglang/modules/connector.md) + [weight_sync.md](sglang/modules/weight_sync.md) + [checkpoint_engine.md](sglang/modules/checkpoint_engine.md) + [model_loader.md](sglang/modules/model_loader.md)）在三家的对偶矩阵
4. **`compare openai-compat across all`** —— [entrypoints_openai.md](sglang/modules/entrypoints_openai.md) 是 vLLM `vllm/entrypoints/openai/` 直接谱系
5. **`compare attention-backend across all`** —— [layers.md](sglang/modules/layers.md) 17 SGLang attention backend vs vLLM `vllm/v1/attention/`
6. **`ingest sglang topics P0`** —— 配合本轮 30 模块 + 36 页 verify 沉淀做 topic 深化（pd-disaggregation / kv-cache / speculative / scheduler-mixins / moe topic 页）
7. **`verify sglang/entities/*`** —— entities 5/11 已 ingest（Engine / TokenizerManager / Scheduler / TpModelWorker / DataParallelController），可做小批 verify pass
8. **`verify comparison/topics/*`** —— 14 个 comparison topic 页累积 markers 待做 batch verify
9. **AGENTS.md §6 schema 增量**：本 verify pass 沉淀的「design-caveat vs falsifiable-bug」分类，可作为 §6 marker schema 增强
10. **被动等待**：三仓 HEAD 都未动，无需触发 §12 增量

---

## [2026-04-19] verify | sglang/entities/* | 5 页 batch verify pass（2 subagent 并行；13 markers → 12 RESOLVED + 1 retained + 6 anchor silent fix）

**触发**：用户 `verify sglang/entities/*`，承接同日 sglang/modules verify pass 末尾「下一步建议」第 7 项。

**范围确认**：sglang/entities **5 页全覆盖**（Engine / TpModelWorker / DataParallelController / Scheduler / TokenizerManager；index.md 标 5/11，剩 6 个 P1-P3 仍 TODO 不在本批 verify 范围）。Pre-pass 统计：

| 指标 | before |
|---|---|
| `[!todo] VERIFY` 总数 | 9 |
| `[!warning] CONTRADICTION` 总数 | 4 |
| **Markers 总和** | **13** |
| `verified_against: 2026-04-17` 页数 | **2**（Scheduler + TokenizerManager — 3 天 lag） |
| `verified_against: 2026-04-18` 页数 | 3 |
| `RESOLVED *` 标记数 | 0 |

**执行方式**：2 subagent 并行（规模小于 modules 的 36 页，6 subagent 是 overkill；2 subagent 是合理 ROI）：

| Batch | 页面 | markers_before | resolved | retained | anchor_fixes |
|---|---|---|---|---|---|
| **A** Engine 家 | Engine / TpModelWorker / DataParallelController | 6 (5 VERIFY + 1 CONTRADICTION) | **6** ? | 0 | 0 |
| **B** Scheduler 家（旧 04-17 pin） | Scheduler / TokenizerManager | 7 (6 VERIFY + 1 CONTRADICTION) | 6 | 1（pickle vs msgspec 跨项目 synthesis） | **6**（旧 pin 预期漂移）|
| **总计** | **5 页** | **13** | **12 (92.3%)** | **1** | **6** |

**Marker 处理总览**

| 处置 | 数量 | 性质 |
|---|---|---|
| ? **RESOLVED** | **12** | strikethrough + `> **RESOLVED 2026-04-19**: <结论> + [anchor]` 块；附直接源码证据 |
| ?? **跨项目 synthesis 保留** | 1 | TokenizerManager.md `pickle vs msgspec` —— 与 vLLM ZMQ 序列化对比，超出本 verify 范围 |
| ?? **Anchor silent fix** | 6 | Scheduler.md（TpModelWorker 217-538→217-558 / forward_batch_generation 443-534→443-533 / forward_batch_split_prefill 535-...→535-558）+ TokenizerManager.md（abort_request 1459-1523→1459-1468 函数已瘦身 / auto_create_handle_loop 1586-1619→1586-1609 / dump_requests 2154-2311→2154-2272）|

**关键 RESOLVED synthesis**（高价值发现 6 条）

1. **`LoadBalanceMethod` enum 实际 4 项 = `ROUND_ROBIN` / `FOLLOW_BOOTSTRAP_ROOM` / `TOTAL_REQUESTS` / `TOTAL_TOKENS`**（[DataParallelController.md](sglang/entities/DataParallelController.md) RESOLVED）：subagent 任务 prompt 中给出的猜测名 `SHORTEST_QUEUE` / `MIN_TPS` 不存在；wiki 既有内容已正确，无需修改 ([data_parallel_controller.py:70-84](d:\design\sglang\python\sglang\srt\managers\data_parallel_controller.py))。
2. **`TOTAL_TOKENS` 闭环条件 = `dp_size > 1` 而非「仅 DP attention」**（[DataParallelController.md](sglang/entities/DataParallelController.md) RESOLVED）：Scheduler 每次 stream output 都发 `WatchLoadUpdateReq`（[scheduler_output_processor_mixin.py:967, 1212](d:\design\sglang\python\sglang\srt\managers\scheduler_output_processor_mixin.py)），TokenizerManager 在 `dp_size > 1` 时无条件 PUSH 回 DPC（[tokenizer_manager.py:1838-1844](d:\design\sglang\python\sglang\srt\managers\tokenizer_manager.py)）。**「仅 DP attention」是产品推荐而非硬约束**。
3. **`RayEngine._launch_scheduler_processes` 返回 `(SchedulerInitResult, None)`** 让 `scheduler_procs=None`（[Engine.md](sglang/entities/Engine.md) RESOLVED）：[`ray/engine.py:88-99`](d:\design\sglang\python\sglang\srt\ray\engine.py) docstring 显式说明；主路径 [`engine.py:746-747`](d:\design\sglang\python\sglang\srt\entrypoints\engine.py) 用 `processes = list(scheduler_procs or [])` 兜底 None —— 与 modules/ray.md 子类化设计闭环。
4. **`node_rank >= 1` 多节点 worker 路径根本不创建 `SubprocessWatchdog`**（[Engine.md](sglang/entities/Engine.md) RESOLVED）：早退分支返回 `(None, None, port_args, scheduler_init_result, None)`（[engine.py:689-715](d:\design\sglang\python\sglang\srt\entrypoints\engine.py)）；watchdog 仅在 rank 0 master 进程上启动，依赖子进程 `kill_itself_when_parent_died` 兜底。
5. **`Scheduler` 11 mixin 拆解 = 5 跨子系统外置 + 6 srt/managers 内部**（[Scheduler.md](sglang/entities/Scheduler.md) RESOLVED）：与 modules/managers.md 之前 RESOLVED 的「5 mixin file paths」**不冲突**，5 是外置（observability + disaggregation/decode + disaggregation/prefill + multiplex + dllm），其余 6 个（output_processor / update_weights / profiler / metrics 本地 / runtime_checker / pp / dp_attn）在 `srt/managers/*_mixin.py` 内部。
6. **`init_disaggregation` `TransferBackend` 实际 5 项不是 7**（[Scheduler.md](sglang/entities/Scheduler.md) RESOLVED）：[`disaggregation/utils.py:304-309`](d:\design\sglang\python\sglang\srt\disaggregation\utils.py) `MOONCAKE / MORI / NIXL / ASCEND / FAKE` —— wiki 原文 "7 backend" 高估（`common`/`base` 是抽象基类目录）。**与 modules/disaggregation.md verify 同 RESOLVED 一致**。

**Anchor 漂移分析**（旧 04-17 pin 实测）

| 页面 | 抽查 | ±0 命中 | ±1-5 silent fix | > ±5 contradiction |
|---|---|---|---|---|
| Scheduler.md | 6 | 3 | 3 | 0 |
| TokenizerManager.md | 6 | 3 | 3 | 0 |
| Engine.md | 6 | 6 | 0 | 0 |
| TpModelWorker.md | 5 | 5 | 0 | 0 |
| DataParallelController.md | 4 | 4 | 0 | 0 |
| **合计** | **27** | **21** | **6** | **0** |

**结论**：3 天 lag 的 Scheduler.md / TokenizerManager.md 出现 6 次小幅函数体 reshape（`abort_request` 由 65 行瘦身到 10 行最显著），**100% 在 ≤ ±5 行容差内**；0 次符号丢失。**04-17 pin 在重活跃模块（Scheduler / TM）的 stale 临界值约 ≤ 3 天**。

**新发现 — 反向 modules 跟进项**

1. **mem_cache.md `init_cache_with_memory_pool` 8 → 10 branches**：本批 Scheduler.md verify 顺手 grep 实测 [scheduler.py:754-915](d:\design\sglang\python\sglang\srt\managers\scheduler.py) 实际 10 个 `tree_cache` 分支：`ChunkCache / SWAChunkCache / RadixCacheCpp / HiMambaRadixCache / HiRadixCache / UnifiedRadixCache / SWARadixCache / MambaRadixCache / LMCRadixCache / RadixCache`。**modules/mem_cache.md 同日 verify pass 中 RESOLVED 为 8 已滞后** —— 可能是 ingest 时 SWA / Mamba / LMCRadixCache 3 分支未列。**触发 mem_cache.md follow-up verify**（已加入下一步）。
2. **`tokenizer_manager.__init__` 实际 10 个 init 子方法**（wiki 行文 "8 步初始化"）：列表细节本身已正确；叙述措辞需小修（subagent 按 user rule "不发明 marker 之外改动" 未改）。
3. **`forward_batch_generation` 实际 5 条分支**（DLLM / is_verify / overlap+grammar / prefill_only / normal sample）+ 1 条非末段 PP 分支；wiki 既有 mermaid 已正确捕获，subagent 任务 prompt 中「DLLM_EXTEND added → 4 branches」hint 不准（DLLM_EXTEND 是 ForwardMode 枚举值，非分支）。

**Frontmatter 净变化**

| | before | after |
|---|---|---|
| `verified_against: 2026-04-17` 页 | 2 (Scheduler + TokenizerManager) | **0** |
| `verified_against: 2026-04-18` 页 | 3 | **0** |
| `verified_against: 2026-04-19` 页 | 0 | **5 / 5 = 100%** |
| `status: verified` 页 | 5 | 5（不降级；剩余 1 marker 是跨项目 synthesis 不在 falsifiable 范畴）|

**协同更新**：[`Engine.md`](sglang/entities/Engine.md) `sources:` 追加 `d:\design\sglang\python\sglang\srt\ray\engine.py`（RayEngine 子类源）。其余 4 页无 sources/related 变化。

**净变化**

| | before | after |
|---|---|---|
| sglang/entities `verified_against: 2026-04-19` 率 | 0% | **100% (5/5)** |
| 累计 marker 解决率（entities） | 0/13 = 0% | **12/13 = 92.3%** |
| 待办 marker | 13 | **1**（pickle vs msgspec 跨项目 synthesis） |
| sglang 总页数 | 37 | 37（无新页） |
| 总 .md 文件数（excluding raw/ + log-archive/） | 107 | 107 |
| **同日累计 RESOLVED**（modules 68 + entities 12） | 0 | **80** |

**Lessons learned**

1. **2 subagent + 5 页 = 适中粒度**：与 modules 6 subagent + 36 页 节奏对比，**entities 5 页单 subagent 可吃 2-3 页**（B subagent 处理 Scheduler + TokenizerManager 两个最大的 entity 页 + 6 anchor fix，仍稳定输出）；**未来「小集合 verify」推荐 2-3 subagent 并行**而非过度拆分。
2. **subagent prompt 中给出的「猜测 enum 名」可能误导**：本批 prompt 写「`LoadBalanceMethod` 4 strategies (ROUND_ROBIN / SHORTEST_QUEUE / MIN_TPS / etc — verify exact enum)」其中 `SHORTEST_QUEUE` / `MIN_TPS` 不存在；幸而 subagent 实测确认 wiki 已正确未误改。**未来 verify prompt 中 hint 应明确标「假设需验证」**减弱锚定。
3. **跨页 verify 的「反向跟进」机制**：本批 entities verify 实测出 modules/mem_cache.md 8 → 10 branches 漂移。**这是 entity verify 反推 module verify 的高价值副产物** —— 建议未来 entity batch verify 都附「反向 module 检视」section。
4. **04-17 pin 在重活跃模块 ≤ 3 天稳定**：Scheduler.md + TokenizerManager.md 共 12 anchor 抽查，6 次 ±1-5 silent fix（50%）+ 6 次 ±0 命中（50%）+ 0 次符号丢失。**说明 §11 lint stale 阈值（"上游修改时间"）对调度核心模块是 ≤ 3 天合理**。
5. **`init_disaggregation` "7 backend" 修正与 modules/disaggregation.md 形成同源 RESOLVED 对**：两个 verify pass 独立确认 `TransferBackend` 5 项 enum，**双重锁死真相**；未来跨页同条 fact 出现时优先查既有 RESOLVED 而非重做 grep。
6. **anchor silent fix 是健康 verify 副产物**：本批 6 次 silent fix 占 27 抽查的 22%，全部因函数体 refactor / 瘦身。**保留原 symbol 名 + 重新定位行号** 比 marker 化更轻量。

**§11 log rotation 阈值复核**（append 本 entry 后）

| | before append | after append（预估） |
|---|---|---|
| log.md 字节 | 78.9 KB | **~85 KB**（合规：< 110 KB；余 ~25 KB） |
| log.md 行数 | 627 | **~770**（合规：< 1500） |
| log.md entries | 6 | **7** |

**当前 wiki 状态**

| 项目 | overview | index | modules | entities | topics | 总数 | verify-04-19 状态 |
|---|---|---|---|---|---|---|---|
| MindIE-LLM | ? | ? | 3/14 | 9/9 | 8/13 | 20 | 部分 |
| vLLM | ? | ? | 2/26 | 10/17 | 5/14 | 17 | 部分 |
| SGLang | ? | ? | **30/30 ?** | **5/11 ? verify-04-19** | 2/14 | **37** | **modules + entities = 41 页 100% 04-19** |
| comparison | — | ? | — | — | 14/24 | 14 + dimensions | — |
| 顶层 meta | 5 | — | — | — | — | 5 | — |

**下一步建议**

按 ROI 排序：

1. **mem_cache.md follow-up verify** —— 本批反向发现 `init_cache_with_memory_pool` 8 → 10 branches 漂移；可用单 subagent 1 页快速修订
2. **同步 cross-page 数字漂移** —— [comparison/topics/speculative-decoding.md](comparison/topics/speculative-decoding.md) 24 → 27 + 12 → 11（modules verify 已沉淀 lint 项）
3. **`compare unique-design-points across all`** —— SGLang 4 unique 设计点 + dead-code/placeholder audit 维度
4. **`compare model-loading across all`** —— 4 条权重通道在三家
5. **`ingest sglang topics P0`** —— pd-disaggregation / kv-cache / speculative / scheduler-mixins / moe topic 页（**11 mixin** 拆分清单已沉淀，可作 scheduler-mixins topic 种子）
6. **`ingest sglang entities P1-P3`** —— 6 个 TODO entity（`SchedulePolicy` / `ScheduleBatch` 110KB 最大单文件 / `CacheController` / `DisaggService` / `HiSparseCoordinator` 等）
7. **`verify comparison/topics/*`** —— 14 个 comparison topic 页累积 markers
8. **AGENTS.md §6 schema 增量** —— 「design-caveat vs falsifiable-bug」+ 「subagent prompt 中 enum hint 减弱锚定」+ 「反向跟进机制」
9. **被动等待**：三仓 HEAD 都未动

---

## [2026-04-19] ingest | sglang topics P0 | 5 个 deep-dive topic 页（5 subagent 完全并行 + 主 agent 合并 3 subagent 独立 entry → 1 mega-entry）

**触发**：用户 `ingest sglang topics P0`，承接 entities verify pass 末尾「下一步建议」第 5 项。P0 = 配合本日 30 模块 + 5 entities verify 沉淀做 topic 深化的 5 个高 ROI topic（[index.md §Topics](sglang/index.md) 中 13 TODO 中的 5 个核心）。

**执行方式**：5 subagent 完全并行（在「6 subagent 可控上限」内）。每个 subagent 探索源码 + 利用既有已 verified 的 36 modules + 5 entities anchor，按 [§4 标准模板](AGENTS.md) 写入新 topic 页。

**新建页（5 个）**

| 文件 | KB / 行 | 关键涵盖 |
|---|---|---|
| [sglang/topics/scheduler-mixins.md](sglang/topics/scheduler-mixins.md) | **23.2 KB / 247** | **11 mixin** 完整拆解表（每行带 [file:line] 锚 + 触发条件）+ External 5 (`observability` / `disaggregation/decode`+`/prefill` / `multiplex` / `dllm`) vs Internal 6 (`output_processor` / `update_weights` / `profiler` / `runtime_checker` / `pp` / `dp_attn`) 切分原则 + `process_batch_result` 6-way 分派 + `dispatch_event_loop` 8 模式 + `TYPE_CHECKING + self: Scheduler` 注解 trick + 6 跨 mixin 协作链 |
| [sglang/topics/pd-disaggregation.md](sglang/topics/pd-disaggregation.md) | **31.0 KB / 294** | 5 backend **3 层继承树**（`Base*→Common*→具体`，`ascend` 唯一子类化 `MooncakeKVManager`，`fake` 直接落 Base）+ 2 Scheduler mixin 不对称（prefill 9 方法 / decode 6 方法）+ `init_disaggregation` L1051-1167 + `get_kv_class` L342-428 dispatch + EPD encode 三段（11 模型族白名单）+ PD-Disagg vs PD-Mux 互斥本质（调度循环结构不兼容）+ 4 维度对偶矩阵 |
| [sglang/topics/kv-cache.md](sglang/topics/kv-cache.md) | **32.0 KB / 226** | **10 RadixCache 变体矩阵**（ChunkCache / SWAChunkCache / RadixCacheCpp / HiMambaRadixCache / HiRadixCache / UnifiedRadixCache / SWARadixCache / MambaRadixCache / LMCRadixCache / RadixCache）**纠正 mem_cache.md 8 → 10 stale** + `BasePrefixCache` 子类继承树 + `UnifiedRadixCache` 4-component 进行中重构 + HiCache `init_load_back/ready_to_load_host_cache` 仅 `Hi*RadixCache` 实现（其余 8 留 `NotImplementedError`）+ 多模态 embedding cache 3 子图正交 + 8 触发器 → 10 分支 不一一对应 |
| [sglang/topics/speculative.md](sglang/topics/speculative.md) | **27.5 KB / 280** | **27 .py**（实测；comparison 页 stale 24）+ 6 algorithm enum + **5 算法族 × V1/V2 双轨**（EAGLE/EAGLE3 V1 子类 + EAGLE V2 组合 `BaseSpecWorker` 持 `EagleDraftWorker` 内嵌 `TpModelWorker(is_draft_worker=True)` + MultiLayer EAGLE V1/V2 + Standalone V1/V2 + DFlash duck-typed 共享 target.model_runner + NGRAM CPU trie）+ subclass vs duck-typed 二元根因（scheduler `tp_worker.get_worker_info()` L705 从 target 读）+ sgl-kernel 2 CUDA 算子 `verify_tree_greedy` + `tree_speculative_sampling_target_only` |
| [sglang/topics/moe.md](sglang/topics/moe.md) | **30.5 KB / ~250** | **42 .py** 中 `layers/moe/` + **8 enum vs 7 CLI Literal** `MoeA2ABackend`（缺 `customized` —— 新发现 contradiction）+ token_dispatcher 7 类（StandardDispatcher / DeepEPDispatcher / MooncakeEPDispatcher / NixlEPDispatcher / MoriEPDispatcher / FlashinferDispatcher / NpuFuseEPDispatcher）+ `MoeRunnerBackend` 11 取值 + sgl-kernel **14 MoE 算子**（含 DeepSeek 专属 `dsv3_router_gemm` / `dsv3_fused_a_gemm`，Kimi K2 `kimi_k2_moe_fused_gate`）+ `is_deepep_class_backend()` 漏写 nixl bug + EPLB 3 算法 + Elastic EP NPU 不支持 + TBO/SBO 正交 |

**5 subagent 并行规模**：所有页 ≥ 23 KB ≤ 32 KB（**总 144 KB**），均为 `confidence: high` / `verified_against: 2026-04-19` / `status: draft`。

**关键 cross-topic synthesis（10 条 — 跨 5 topic 页才能解构的发现）**

1. **「mixin 之间不通过基类、可以直接互调」**（scheduler-mixins ↔ pd-disaggregation）：`SchedulerDisaggregationPrefillMixin.get_next_disagg_prefill_batch_to_run` 在 PD 模式下仍调 `SchedulerDPAttnMixin.maybe_prepare_mlp_sync_batch`（[disaggregation/prefill.py:381](d:\design\sglang\python\sglang\srt\disaggregation\prefill.py)）。`process_batch_result` 6-way 分派点是 4 mixin（Output/Dllm/DisaggPrefill）共享的关键证据。
2. **「PD-Disagg vs PD-Mux 互斥本质 = 调度循环结构不兼容」**（pd-disaggregation ↔ scheduler-mixins）：两者都进 11 mixin MRO 但运行时 `dispatch_event_loop` 8 模式分支互斥；与 modules/multiplex.md `server_args` 4 互斥前提（PD-disagg + chunked + overlap + PP）形成完整防护链。
3. **「mem_cache.md 8 → 10 RadixCache 反向 lint」**（kv-cache ↔ entities verify）：本批 kv-cache topic 页确认 [scheduler.py:754-915](d:\design\sglang\python\sglang\srt\managers\scheduler.py) 实有 10 个 `tree_cache` 分支（SWA/Mamba/LMC 3 个子类被 modules/mem_cache.md verify RESOLVED 为 8 时遗漏）—— 已加 `[!warning] CONTRADICTION` 块；**触发 `verify mem_cache.md` follow-up**（与 entities verify pass 反向发现一致）。
4. **「3 子图正交是 SGLang KV cache 哲学」**（kv-cache）：prefix cache（10 子类）/ HiCache 后端（7 后端）/ 多模态 embedding cache（process 全局单例）三层独立；**多模态不进 `BasePrefixCache` 是因 embedding 与 token 异质**；与 vLLM `KVConnector` 钩子模式哲学不同（vLLM 是「单一抽象 + 后端插件」，SGLang 是「多继承爆炸 + UnifiedRadixCache 进行中重构」）。
5. **「subclass vs duck-typed worker 根因 = scheduler 资源信息从 target 读」**（speculative ↔ entities）：`Scheduler.init_model_worker` 中 `model_worker = draft_worker`（[scheduler.py:686-689](d:\design\sglang\python\sglang\srt\managers\scheduler.py)）但 step4 `tp_worker.get_worker_info()`（L705）**仍从 target 读** —— 所以 DFlash/NGRAM 没必要做完整 TpModelWorker 子类，只需暴露 forward 协议即可。**两条 worker 协议路径 = 真子类（EAGLE 系，独立 KV pool）vs duck-typed（DFlash 共享 target / NGRAM CPU trie）**。
6. **「V1/V2 双轨匹配 overlap scheduler」**（speculative ↔ batch_overlap）：EAGLE 系沿 `enable_overlap` 拆 V1（继承 `TpModelWorker`）/ V2（组合 `BaseSpecWorker` 持 `EagleDraftWorker` 内嵌 `TpModelWorker`）—— V2 用「**组合而非继承**」匹配 overlap scheduler；DFlash/NGRAM 因强制 disable overlap 只有 V1 实现。**5 算法族 × V1/V2 × subclass/duck-typed 三维落到 27 .py，无冗余**。
7. **「MoE 三层正交夹心」**（moe ↔ layers）：MoE = 逻辑模块（FusedMoE/DeepEPMoE）× 通信 dispatcher（7 backend token_dispatcher）× 算子 runner（11 backend MoeRunnerBackend），由 `FusedOpPool` 与 `PermuteMethodPool` 把 `(a2a, runner)` 二元组映射到融合/非融合路径。**同一 `FusedMoE` 实例在不同 backend 组合下 kernel 路径完全不同**。
8. **「sgl-kernel C++ 算子专属化集中在 DeepSeek 模型族」**（moe ↔ models）：`dsv3_router_gemm` / `dsv3_fused_a_gemm` 仅 DeepSeek 独享；Kimi K2 仅获 fused gate 变体；其它 MoE 模型族共享通用 `moe_fused_gate` + `topk_softmax` + `cutlass_w4a8_moe_mm`。**SGLang sgl-kernel C++ 加速 = DeepSeek-V3 专属投资 + 通用 MoE 算子**两层。
9. **「TBO vs SBO 正交开关 vs PD/PP 互斥」**（moe ↔ batch_overlap ↔ pd-disaggregation）：TBO 切 micro-batch（**跨** MoE 层粒度）vs SBO 切 combine/down-gemm（**MoE 层内** kernel 粒度）—— `enable_two_batch_overlap + moe_a2a_backend == "none"` 互斥并强校验报错（[server_args.py:6629-6633](d:\design\sglang\python\sglang\srt\server_args.py)），SBO 无此约束；与 PD-Disagg/PD-Mux 互斥构成 SGLang 5 层调度模式互斥矩阵。
10. **「mixin / fork / 占位 / 命名陷阱 / 死代码」5 大反误读模式跨 5 topic 全覆盖**：scheduler-mixins.md（mixin 不依赖 cooperative super().__init__）+ pd-disaggregation.md（命名陷阱 disaggregation ≠ vLLM kv_connector）+ kv-cache.md（HiCache 仅 2/10 实现 init_load_back）+ speculative.md（24→27 数字 stale + comparison 措辞精化）+ moe.md（enum 8 vs CLI 7 / `is_deepep_class_backend` 漏 nixl）—— **每个 topic 都贡献 1+ 个反误读发现**，与 modules verify 沉淀的 18 个 design caveat 模式一致。

**Markers added 总览**

| 页面 | `[!warning] CONTRADICTION` | `[!todo] VERIFY` | `synthesis:` 前缀块 |
|---|---|---|---|
| scheduler-mixins.md | 0 | 3 | 5 |
| pd-disaggregation.md | 2 | 3 | 7 |
| kv-cache.md | 1（mem_cache.md 8→10 stale）| 4 | 4 |
| speculative.md | 2（comparison 24→27 / 12→11 + 措辞）| 2 | 5 |
| moe.md | 3 | 2 | 4 |
| **合计** | **8** | **14** | **25** |

**新发现 falsifiable bugs / typos**（高价值副产物）

1. **`MoeA2ABackend` enum 8 成员 vs CLI Literal 7 取值不一致**（[utils.py:23-32](d:\design\sglang\python\sglang\srt\layers\moe\utils.py) vs [server_args.py:528-530](d:\design\sglang\python\sglang\srt\server_args.py)）：`CUSTOMIZED` 在 enum 中存在但不在 CLI Literal —— 可能是死代码占位
2. **`is_deepep_class_backend()` 漏写 nixl**（[utils.py:260-263](d:\design\sglang\python\sglang\srt\layers\moe\utils.py)）：函数把 deepep/mooncake/mori 视同族但**未含 nixl**；而 `create_moe_dispatcher` 把 nixl 与前三者一同包成 `MaybeTboDeepEPDispatcher` —— **internal 分类不一致**
3. **`UnifiedRadixCache` 4-component 重构进行中**（[unified_cache_components/](d:\design\sglang\python\sglang\srt\mem_cache\unified_cache_components)）：env 默认 false，与旧 4 子类（RadixCache / SWARadixCache / MambaRadixCache / Hi*RadixCache）并存 —— 目标用「组合而非继承」取代多继承爆炸
4. **8 个 BasePrefixCache 子类把 `init_load_back / ready_to_load_host_cache` 留 `NotImplementedError`**：仅 `Hi*RadixCache` 真正闭环 host_pool + storage_backend —— **HiCache 是一等公民**

**协同更新（10 文件）**

| 文件 | 修改 |
|---|---|
| [sglang/index.md §Topics](sglang/index.md) | 5 行 TODO → DONE：scheduler-mixins / pd-disaggregation / kv-cache / speculative / moe（每行附一句话摘要 + 关键发现注脚） |
| [sglang/entities/Scheduler.md](sglang/entities/Scheduler.md) | `related:` + `## See also` 加 scheduler-mixins.md（11 mixin 详细拆解） |
| [sglang/modules/managers.md](sglang/modules/managers.md) | `related:` 加 scheduler-mixins.md |
| [sglang/modules/disaggregation.md](sglang/modules/disaggregation.md) | `related:` 加 scheduler-mixins.md |
| [sglang/modules/multiplex.md](sglang/modules/multiplex.md) | `related:` 加 scheduler-mixins.md |
| [sglang/modules/dllm.md](sglang/modules/dllm.md) | `related:` 加 scheduler-mixins.md |
| [sglang/modules/observability.md](sglang/modules/observability.md) | `related:` 加 scheduler-mixins.md |
| [sglang/topics/manager-pipeline.md](sglang/topics/manager-pipeline.md) | `related:` 加 scheduler-mixins.md |

**未应用的 cross-page 扩散（pd-disaggregation + kv-cache + moe + speculative subagent 推荐，留待 lint pass）**

| 待修页 | 推荐 |
|---|---|
| [sglang/modules/disaggregation.md](sglang/modules/disaggregation.md) `## See also` | 增加回链 topics/pd-disaggregation.md |
| [sglang/modules/multimodal.md](sglang/modules/multimodal.md) | EPD 章节回链 topics/pd-disaggregation.md §EPD 三段架构 |
| [sglang/modules/mem_cache.md](sglang/modules/mem_cache.md) `related:` | 加 topics/kv-cache.md（**且需 verify follow-up 修复 8 → 10 stale**）|
| [comparison/topics/kv-cache.md](comparison/topics/kv-cache.md) `related:` | 加 sglang/topics/kv-cache.md |
| [comparison/topics/prefix-cache.md](comparison/topics/prefix-cache.md) `related:` | 加 sglang/topics/kv-cache.md |
| [sglang/modules/layers.md](sglang/modules/layers.md) `## See also` | 加 topics/moe.md |
| [sglang/modules/eplb.md](sglang/modules/eplb.md) / [elastic_ep.md](sglang/modules/elastic_ep.md) / [batch_overlap.md](sglang/modules/batch_overlap.md) `related:` | 加 sglang/topics/moe.md |
| [sglang/modules/models.md](sglang/modules/models.md) | DeepSeek 段链 topics/moe.md sgl-kernel 章节 |
| [comparison/topics/speculative-decoding.md L80/L439](comparison/topics/speculative-decoding.md) | **数字 fix**：SGLang 24→27 + vLLM 12→11；措辞精化「独立 TpModelWorker 仅适用 EAGLE 系」 |

**Cross-project topic 候选（高价值 seed）**

- **`comparison/topics/moe.md`** 当前**不存在**；本 topic 已构建 7 a2a backend × 11 runner × EPLB/Elastic/Overlap 三方对照框架，是高价值 seed
- **`comparison/topics/scheduler-architecture.md`** 候选：vLLM EngineCore 单类 vs SGLang 11 mixin vs MindIE TextGenerator+Manager 切分

**净变化**

| | before | after |
|---|---|---|
| sglang topics DONE | 2/15 (13.3%) | **7/15 = 46.7%**（+5）|
| sglang 总页数 | 37 | **42**（+5） |
| 总 .md 文件数（excluding raw/ + log-archive/） | 107 | **112**（+5） |
| comparison topic 候选种子（新增）| — | 2（moe + scheduler-architecture） |
| **同日累计 RESOLVED + 新页**（modules verify 68 RESOLVED + entities verify 12 + topics 5 新页）| 0 | **80 RESOLVED + 5 新页** |

**Lessons learned**

1. **subagent 不应 append log entry**（**重要管理修正 + 编码事故**）：本批 5 subagent 中有 3 个（scheduler-mixins / moe / speculative）独立 append 了 ingest entry 到 log.md，导致：(a) 同 turn 出现 3 个独立 entry 违反「单 turn 单 mega-entry」惯例；(b) log.md 一度膨胀到 112.5 KB > 110 KB 阈值；(c) 主 agent 用 PowerShell `Set-Content -NoNewline` 截断时**默认用系统 GBK 编码导致整个文件中文 mangled**，必须额外做 `[System.Text.Encoding]::GetEncoding(936)` 读 + `UTF8Encoding($false)` 重写 还原。**未来 subagent prompt 必须明确「DO NOT touch log.md」**；**主 agent 操作含中文文件务必显式指定 UTF-8 编码**。
2. **5 subagent 并行 + 主 agent 合并 = topic ingest 最优 ROI**：5 页 23-32 KB 范围（**总 144 KB**）远超 modules ingest 的 ~30 KB / 页；topic 页含密集 cross-cut synthesis，subagent 内独立合成空间充裕。**6 subagent 上限对 topic ingest 仍可用**，但本批 5 已是 P0 边界。
3. **「topic 页 vs module/entity 页职责切分」沉淀**：module 页负责 inventory（X 文件 / Y enum / Z CLI 字段全表）；topic 页负责 deep-dive synthesis（subclass vs duck-typed / V1/V2 双轨 / 三层正交 / 4 维度对偶）。**两类页都引用同一 anchor 但讲不同故事** —— module 讲「有什么」，topic 讲「为什么」。本日同日 entities + topics 双 ingest 验证了这种切分的健康。
4. **「反向 lint 跟进项」累计**：本批新发现 `mem_cache.md 8→10 stale`（continuing entities verify 反向发现）+ `comparison/topics/speculative-decoding.md 24→27 / 12→11 stale` + `MoeA2ABackend 8 vs 7 enum/CLI 不一致` —— **3 个新 lint 项加入 lint TODO 队列**。
5. **「死代码 + typo + 命名陷阱」audit 模式延伸到 topic 层**：modules verify 已沉淀 5 个死代码 / typo（`ensure_model_parallel_initialized` / `compile_sizes` / `FixFunctionalizationPass.__call__` / `ExpertLocationDispatchInfo` Literal / `RedisConnector.weight_iterator` 注解）；本批 topic ingest 又发现 2 个新（`MoeA2ABackend.CUSTOMIZED` 死代码 / `is_deepep_class_backend` 漏 nixl）。**verify + ingest 双 pass 已成为 wiki 间接的代码 audit 工具**。
6. **size 控制策略沉淀**：5 个 topic 页平均 ~29 KB，超 25 KB 软上限主因 = anchor 密度 + cross-module synthesis 表。**trim 优先级**：(1) CLI 全表合并；(2) Sources 表 → bullet；(3) 重复 URL 单链；(4) 多段长解释 → 表格 + 4-bullet 总结。本批 speculative.md 三轮 trim 37→27.5 KB 是好范本。

**§11 log rotation 阈值复核**（截断 3 subagent entry + append 本 mega-entry 后）

| | original 3-entry-bloat | after 截断 + 本 mega-entry |
|---|---|---|
| log.md 字节 | 112.5 KB（超阈值） | **~103 KB**（合规：< 110 KB；余 ~7 KB）|
| log.md 行数 | 689 | **~920**（合规：< 1500） |
| log.md entries | 11（含 3 散乱 subagent ingest） | **9**（合并节省）|

> **截断 3 subagent entry 节省 ~30 KB**；本 mega-entry 添加 ~16 KB；**净节省 ~14 KB 相比让 3 entry 共存**（112 → 103，避免触发 log-008 轮转一次）。

**当前 wiki 状态**

| 项目 | overview | index | modules | entities | topics | 总数 | verify-04-19 状态 |
|---|---|---|---|---|---|---|---|
| MindIE-LLM | ✅ | ✅ | 3/14 | 9/9 | 8/13 | 20 | 部分 |
| vLLM | ✅ | ✅ | 2/26 | 10/17 | 5/14 | 17 | 部分 |
| SGLang | ✅ | ✅ | **30/30 ✅** | 5/11 (verify-04-19) | **7/15** (+5) | **42** (+5) | **modules + entities = 41 页 100% 04-19；topics 7/15** |
| comparison | — | ✅ | — | — | 14/24 | 14 + dimensions | — |
| 顶层 meta | 5 | — | — | — | — | 5 | — |

**总 .md 文件数**：**112**（excluding `wiki/raw/` + `wiki/log-archive/`）；**119**（including 7 archive files）。delta +5。

**下一步建议**

按 ROI 排序：

1. **lint pass — 3 项 cross-page stale**：(a) mem_cache.md 8→10 RadixCache 修复；(b) comparison/topics/speculative-decoding.md L80/L439 数字 fix（24→27 / 12→11）；(c) 4 个 module/topic 页 `related:` 扩散更新（详见上方表）
2. **`ingest sglang topics P1`** —— 剩余 8 TODO topic（`continuous-batching` / `dp-attention` / `constrained` / `function-call` / `multi-api` / `weight-sync` / `hardware-backends` / `debug-utils`）；建议 4-5 subagent 一批，**subagent prompt 明确禁止动 log.md**
3. **`ingest sglang entities P1-P3`** —— 6 个 TODO entity（`SchedulePolicy` / `ScheduleBatch` 110KB 最大单文件 / `CacheController` / `DisaggService` / `HiSparseCoordinator` 等）
4. **`compare moe across all`** —— 用本 topic 页 7 a2a × 11 runner 矩阵作 SGLang-row seed
5. **`compare unique-design-points across all`** —— SGLang 4 unique 设计点 + 7 死代码/typo/命名陷阱 audit 维度
6. **`verify comparison/topics/*`** —— 14 个 cross-compare 页累积 markers（含 speculative-decoding stale）
7. **AGENTS.md §6 + §5 schema 增量** —— 「subagent 禁止动 log.md」+ 「主 agent 中文文件操作必须显式 UTF-8」+ 「design-caveat vs falsifiable-bug」+ 「反向跟进机制」
8. **被动等待**：三仓 HEAD 都未动

---

## [2026-04-19] archive | wiki | log rotation 008（lint pass 前预防触发，第 6 次同日轮转）

**触发**：用户 `LINT`。Main log 当前 109.5 KB / 671 行 / 9 entries（topics-P0 ingest 后 0.5 KB 余阈值边缘）。lint pass 必 append entry 预估 ~12-15 KB，append 后必超 110 KB；按预防式轮转模式主动触发 log-008。

**轮转结果**

| 文件 | 期间 | entries | 大小 |
|---|---|---|---|
| **新建** [log-archive/log-008.md](log-archive/log-008.md) | 2026-04-19（晚段） | 4 | ~38 KB |
| **重写** [log.md](log.md) | 当前 | 5 + 本 archive + lint = **7** | ~85-90 KB |

**转移内容**（按 §11 保留策略 — 移除最旧 4 entries 一次到位）

| 转移条目 | 原 line | 操作日期 | 类型 |
|---|---|---|---|
| archive log rotation 007 | 28 | 2026-04-19 | archive |
| ingest sglang modules Turn 3+4+5 mega-batch | 75 | 2026-04-19 | ingest |
| archive log rotation 006 | 188 | 2026-04-19 | archive |
| ingest sglang modules Turn 2 small-rollup | 237 | 2026-04-19 | ingest |

**保留在主 log 的 5 条最新 entries**

| # | 操作 | scope | 摘要 |
|---|---|---|---|
| 1 | archive | wiki | log rotation 005 |
| 2 | ingest | sglang modules P4 | entrypoints/openai + hardware_backend + multimodal + lora + model_loader |
| 3 | verify | sglang/modules/* | 36 页 / 92→24 markers / 68 RESOLVED |
| 4 | verify | sglang/entities/* | 5 页 / 13→1 markers / 12 RESOLVED |
| 5 | ingest | sglang topics P0 | 5 页 / scheduler-mixins+pd-disagg+kv-cache+spec+moe |

**编码事故规避**：本轮主 agent 用 PowerShell + `[System.Text.UTF8Encoding]::new($false)` 显式 UTF-8 编码读写（不再用默认 `Set-Content`），避免上一轮 topics-P0 ingest 中遇到的 GBK 中文 mangled 事故。

**净变化**

| | before | after |
|---|---|---|
| log.md 行数 | 671 | 433 → append archive + lint 后 ~620 行 |
| log.md 字节 | 109.5 KB | **~71 KB** → append 后 ~85-90 KB（合规：< 110 KB；余 ~20-25 KB）|
| log.md entries | 9 | **5 + 本 archive + lint = 7** |
| 归档文件数 | 7 | **8**（+ log-008） |
| 归档总字节 | ~382 KB | ~420 KB |

---

## [2026-04-19] lint | wiki | post topics-P0 健康检查 + 3 项 cross-page stale fix + 9 项 related: 扩散

**触发**：用户 `LINT`。承接 topics-P0 ingest 末尾「下一步建议」第 1 项（**lint pass — 3 项 cross-page stale**），同时执行 §7 全套健康检查。

**执行方式**：3 subagent 并行：
- **Subagent A（fix-applier）**：应用 3 项已知 lint follow-up（mem_cache 8→10 / spec-decoding 24→27 + 12→11 / 9 项 related: 扩散）
- **Subagent B（sglang/* health check, 50 页）**：frontmatter / dead link / orphan / marker count 统计；ROI 高的 trivial `related:` 同步
- **Subagent C（comparison + mindie + vllm health check, 57 页）**：cross-project 一致性 + stale verified_against 列表 + 项目状态报告

主 agent 额外应用 1 项 trivial fix（comparison/dimensions.md `related:` 缺 6 个新 topic）。

**应用的 fixes（20 文件）**

### Fix #1 — mem_cache.md 8 → 10 RadixCache stale

[`sglang/modules/mem_cache.md`](sglang/modules/mem_cache.md)：
- RESOLVED 块 "8 priority branches" 改写为 "**10 分支**"，新增完整 10 类清单（ChunkCache / SWAChunkCache / RadixCacheCpp / HiMambaRadixCache / HiRadixCache / UnifiedRadixCache / SWARadixCache / MambaRadixCache / LMCRadixCache / RadixCache）
- §See also 末段 `8+ 实现工厂分支` → `10 实现工厂分支`
- frontmatter `related:` += `sglang/topics/kv-cache.md`
- 新增 `> **NOTE 2026-04-19 lint fix**: 8 → 10 branches stale 修复（含 SWA/Mamba/LMC 3 子类显式列出）` 行
- **数字精度厘清**：8 = 顶层 if/elif 分支数（① ~ ⑧）；10 = 可被实例化的 tree_cache 类数（其中 ① 含 ChunkCache+SWAChunkCache 2 类，③ 含 HiMambaRadixCache+HiRadixCache 2 类）。两者都对，仅维度不同——已在 NOTE 行解释。

### Fix #2 — comparison/topics/speculative-decoding.md 数字 stale + TpModelWorker 措辞精化

[`comparison/topics/speculative-decoding.md`](comparison/topics/speculative-decoding.md)：
- L80 / L142 / L439 三处 SGLang `24 .py` → **`27 .py（递归含 cpp_ngram/ 子目录）`**；vLLM `12 .py` → **`11 .py（不含 __init__.py）`**
- L82 "独立 TpModelWorker" 加 `[^tpw-precision]` 脚注（L85），明确：**EAGLE / EAGLE3 / STANDALONE** 走独立 `TpModelWorker(is_draft_worker=True)`；**DFLASH / NGRAM** 是 duck-typed wrapper（无第二个 `TpModelWorker`）
- frontmatter `verified_against` 升级为 `2026-04-19 (verify pass: 2026-04-18; lint fix: 2026-04-19)`
- frontmatter `related:` += `sglang/topics/speculative.md`
- `## See also` 加 `[sglang/topics/speculative.md](../../sglang/topics/speculative.md)` 全景链接
- `## Notes / Caveats` 段首部 `> **NOTE 2026-04-19 lint fix**: 24→27 SGLang / 12→11 vLLM stale + EAGLE-only TpModelWorker precision` 一行
- **数字精度厘清**：vLLM `v1/spec_decode/` 实际 12 个 .py 文件，但其中 1 个是 `__init__.py`；topics/speculative.md 报「11」是「业务文件数」。SGLang 27 是递归含 `cpp_ngram/` + `triton_ops/` 子目录的全树计数。两个数字都正确，本 fix 显式标注计数口径。

### Fix #3 — 11 项 cross-page `related:` 扩散

| page | 追加 `related:` |
|---|---|
| [sglang/modules/disaggregation.md](sglang/modules/disaggregation.md) | `topics/pd-disaggregation.md` |
| [sglang/modules/multimodal.md](sglang/modules/multimodal.md) | `topics/pd-disaggregation.md` |
| [sglang/modules/mem_cache.md](sglang/modules/mem_cache.md) | `topics/kv-cache.md`（合并入 Fix #1）|
| [comparison/topics/kv-cache.md](comparison/topics/kv-cache.md) | `sglang/topics/kv-cache.md` |
| [comparison/topics/prefix-cache.md](comparison/topics/prefix-cache.md) | `sglang/topics/kv-cache.md` |
| [sglang/modules/layers.md](sglang/modules/layers.md) | `topics/moe.md` |
| [sglang/modules/eplb.md](sglang/modules/eplb.md) | `topics/moe.md` |
| [sglang/modules/elastic_ep.md](sglang/modules/elastic_ep.md) | `topics/moe.md` |
| [sglang/modules/batch_overlap.md](sglang/modules/batch_overlap.md) | `topics/moe.md` |
| [sglang/modules/models.md](sglang/modules/models.md) | `topics/moe.md` |
| [comparison/topics/speculative-decoding.md](comparison/topics/speculative-decoding.md) | `sglang/topics/speculative.md`（合并入 Fix #2）|

### Fix #4 — Subagent B 8 项 sglang/* `related:` 同步（已存在 See also 但缺 frontmatter）

| page | 追加 `related:` | 根因 |
|---|---|---|
| [sglang/topics/speculative.md](sglang/topics/speculative.md) | `sglang/modules/distributed.md` | See also 已有，frontmatter 漏 |
| [sglang/topics/manager-pipeline.md](sglang/topics/manager-pipeline.md) | `sglang/entities/TokenizerManager.md` + `sglang/entities/Scheduler.md` | 2026-04-17 老页未跟进 |
| [sglang/entities/Scheduler.md](sglang/entities/Scheduler.md) | `sglang/topics/request-lifecycle.md` | 多处反向引用 |
| [sglang/entities/TokenizerManager.md](sglang/entities/TokenizerManager.md) | `sglang/topics/request-lifecycle.md` | 同上 |
| [sglang/modules/managers.md](sglang/modules/managers.md) | `sglang/topics/request-lifecycle.md` | 同上 |
| [sglang/modules/mem_cache.md](sglang/modules/mem_cache.md) | `sglang/topics/request-lifecycle.md` | 同上 |
| [sglang/modules/entrypoints.md](sglang/modules/entrypoints.md) | `sglang/entities/Scheduler.md` + `sglang/topics/request-lifecycle.md` | 同上 |
| [sglang/modules/speculative.md](sglang/modules/speculative.md) | `sglang/modules/distributed.md` | 同上 |

> **复发模式**：`topics/request-lifecycle.md`（verified 2026-04-17，先于大部分 module 页）被多个后写的页加入 See also 但未传播到 frontmatter `related:`。**未来 ingest 应在写 See also 时同步 `related:`**。

### Fix #5 — comparison/dimensions.md `related:` 补 6 个新 topic（主 agent 直接应用）

[`comparison/dimensions.md`](comparison/dimensions.md) frontmatter `related:` 追加：`speculative-decoding` / `prefix-cache` / `multiproc-ipc` / `engine-architecture` / `executor-worker` / `async-schedule`（body 已引用但 frontmatter 漏）

**§7 全套健康检查结果（107 页）**

| 检查项 | 检查范围 | 结果 |
|---|---|---|
| **Frontmatter 完整性** | 107 页（50 sglang + 57 cross-project + mindie + vllm）| **0 issues** ✅ — 所有 6 必填字段（type/project/status/confidence/verified_against/sources）均合规 |
| **Dead links（采样 65+）** | 内部 markdown 链接 + cross-project 链接 | **0 dead** ✅ — 涵盖 sglang/index → modules/topics/entities，topics → comparison/topics，comparison/topics → mindie/vllm/sglang，dimensions → topics |
| **Orphan pages** | 全 107 页 | **0 真正 orphan** ✅ — 弱-inbound 2 页（`topics/moe.md` / `topics/speculative.md` 仅有 `index.md` 链接，**等待 module 页 body 反向链接** — 见非 trivial 待跟进 #1）|
| **Marker count 校验** | 全 sglang/* | `RESOLVED 2026-04-19` 块 = **80**（与 modules verify 68 + entities verify 12 = 80 一致 ✅）/ `[!todo] VERIFY` 总 88 / 划线 61 / 保留 27（log 文档 ~37 偏高）/ `[!warning] CONTRADICTION` 总 49 / 划线 19 / **保留 30**（log 文档 ~10 偏低 —— 大部分是永久 NAMING-TRAP 类设计 caveat，建议加新 marker 类型）|
| **Stale verified_against** | 全 107 页 vs source-versions.md 当前 pin (2026-04-18) | **5 页** `2026-04-17`（comparison/topics/scheduler / kv-cache / cp-sp / flashcomm / chunked-prefill）+ **20 页** mindie/vllm 也是 04-17 —— 但 §12 规定「上游 HEAD 未动则不算 stale」，本轮**不降级 status** |

**5 个非 trivial 待跟进发现**（不在本 lint 直接修复，留为下次 schema-update / lint pass 候选）

1. **`topics/moe.md` / `topics/speculative.md` body 反向链接 asymmetric**：两个 topic 页 link out 到 module 页（modules/eplb.md / modules/elastic_ep.md / modules/layers.md / modules/batch_overlap.md / modules/speculative.md），但 module 页 body 未反向 link 到 topic 页。**建议**：在 module 页的 `## Architecture` 或 `## Notes` 段加 `> 跨子系统视角见 [topics/moe.md](...)` 一行。Fix #3 已修 frontmatter `related:`，但 body backlink 待补。
2. **`[!warning] CONTRADICTION` 50% 是永久 NAMING-TRAP / 设计 caveat**（30 retained 中 ~20 个）：包括 `disaggregation/ ≠ vLLM kv_connector/` / `model_executor ≠ vLLM Executor` / `update_config.py ≠ CLI merge` / 4 同名 tokenizer / SGMV vs BGMV 等。**建议 schema-update**：引入 `> [!info] NAMING-TRAP` 或 `[!info] DESIGN-CAVEAT` marker 类型，与 actionable `CONTRADICTION` 区分；这是 modules verify pass 已沉淀过的 lessons learned 第 2 项的延续。
3. **`verified_against` 字段格式漂移**（7 页含 `(verify pass: <date>)` 注解）：[speculative-decoding.md](comparison/topics/speculative-decoding.md) / [vllm/entities/AsyncLLM.md](vllm/entities/AsyncLLM.md) / LLMEngine / GPUWorker / GPUModelRunner / EngineCoreClient / `mindie/topics/moe.md`（已删）。§3 接受但解析不友好。**建议 schema-update**：移到独立 `verify_pass:` 字段。
4. **vLLM `index.md` 6 处 "DEMO" status 非 §3 enum**：`engine.md` / `executor.md` / `request-lifecycle.md` / `multiproc-ipc.md` / `EngineCore.md` / `MultiprocExecutor.md` 用 `DEMO` 标记。§3 `status: draft|verified|stale` 未注册 DEMO。**建议**：要么注册 enum，要么升 verified（这些页都已经过 verify pass）。
5. **comparison/index.md merge markers 不一致**：L55 "Continuous batching" / L58 "KV 传输" 用自然语言合并标记（"已合并到 chunked-prefill" / "覆盖在 PD 分离对比 §3"），未来如机器解析表格会 fragile。**建议**：加专用 state 列。

**项目状态报告**（Subagent C 摘录）

| 项目 | overview | index | modules | entities | topics | 总数 |
|---|---|---|---|---|---|---|
| MindIE-LLM | ✅ | ✅ | 3/14 | 9/13 | 8/12 | 22 |
| vLLM | ✅ | ✅ | 2/26 | 10/17 | 5/13 | 19 |
| SGLang | ✅ | ✅ | **30/30 ✅** | 5/11 | **7/15**（topics-P0 后） | **42** |
| comparison | — | ✅ | — | — | 14/14 | 14 + dimensions |
| 顶层 meta | 5 | — | — | — | — | 5 |

**总 .md 文件数**：**112**（excluding `wiki/raw/` + `wiki/log-archive/`）；**120**（including 8 archive files）。

**净变化**

| | before | after |
|---|---|---|
| 应用的 trivial fixes | 0 | **20**（Fix #1-5 总计 20 文件被 modify）|
| sglang/* 内 dead link | 0（验证）| 0（已检）|
| sglang/* 内 frontmatter 问题 | 0 | 0 |
| `topics/request-lifecycle.md` frontmatter 反向引用 | 6 缺失 | 6 同步 ✅ |
| `comparison/dimensions.md` `related:` 完整性 | 8/14（缺 6）| 14/14 ✅ |
| `verified_against` 不一致页 | 7（注解漂移）| 7（待 schema-update）|
| 待跟进非 trivial 发现 | — | **5**（schema-update 4 + body backlink 1）|

**Lessons learned**

1. **「subagent 禁止动 log.md」prompt 显式约束首次成功**：本轮 3 subagent 在 prompt 中明确 `IMPORTANT: DO NOT touch d:\design\wiki\log.md` + `IMPORTANT: When using PowerShell to write files with Chinese characters, ALWAYS specify UTF-8 explicitly`。**全部 3 subagent 均未触碰 log.md**，主 agent 也用 `[System.Text.UTF8Encoding]::new($false)` 显式 UTF-8 编码写文件，**0 编码事故**。两条约束已验证有效，应固化到 [`AGENTS.md §5 / §7`](AGENTS.md)。
2. **「fix-applier subagent + parallel health-check subagents」分工模式有效**：Subagent A 拿明确 task list 高效应用 11 文件 fix（顺手发现 2 个数字精度口径冲突并加注解）；Subagent B/C 做覆盖广但低风险的 health check + trivial fix。**总 19 文件 modified + 65+ 链接抽查 + 5 非 trivial 发现 1 turn 完成**，比串行节省 ~50% 时间。
3. **`request-lifecycle.md` 反向引用同步**：6 个 `related:` 缺失（Subagent B Fix #4）暴露「**老页 + 新页混合时，See also 与 related: 容易脱节**」。建议 [AGENTS.md §5 ingest workflow](AGENTS.md) step 5 加「**写 See also 时同步反向更新 related:**」明确步骤。
4. **「数字精度口径冲突」需明示标注**：mem_cache 8 vs 10（顶层 if/elif vs 实例化类数）+ vLLM 11 vs 12（业务文件 vs 含 __init__.py）+ SGLang 24 vs 27（顶层 vs 递归含子目录）= **3 处同 turn 数字口径冲突**。**统一原则**：wiki 数字论断必须显式标注「计数口径」（vendor 文件数 / 业务文件数 / 含子目录 / enum 成员数 / CLI choices 数），避免后续 verify 反复反复。
5. **CONTRADICTION marker 类型分化已成必需**：本轮 Subagent B 实测 30 retained CONTRADICTION 中 ~20 是永久 NAMING-TRAP/DESIGN-CAVEAT，**只有 ~10 是 actionable**（与 modules verify pass log 中的 ~10 数字吻合）。**建议下次 schema-update 引入 `[!info] NAMING-TRAP`**，这是连续 3 次 verify/lint pass 沉淀的同一发现。
6. **预防式 log rotation 已稳定为「lint pass 前默认动作」**：本轮 log 在 109.5 KB / 110 KB 阈值边缘（仅 0.5 KB 余地），lint entry 必触发轮转。**主动在 lint 开始时鸣枪轮转**比 lint append 后回滚更高效。同日 6 次轮转节奏（log-003 ~008）已成 ingest 密集日的稳态。

**§11 log rotation 阈值复核**（append 本 archive + lint entry 后）

| | before append | after append |
|---|---|---|
| log.md 字节 | 71.3 KB | **~85-90 KB**（合规：< 110 KB；余 ~20-25 KB）|
| log.md 行数 | 433 | **~620**（合规：< 1500） |
| log.md entries | 5 | **5 + archive + lint = 7** |
| 归档文件数 | 7 | **8**（+ log-008） |

**当前 wiki 状态**

| 项目 | overview | index | modules | entities | topics | 总数 | 04-19 verify 状态 |
|---|---|---|---|---|---|---|---|
| MindIE-LLM | ✅ | ✅ | 3/14 | 9/13 | 8/12 | 22 | 部分 |
| vLLM | ✅ | ✅ | 2/26 | 10/17 | 5/13 | 19 | 部分 |
| SGLang | ✅ | ✅ | **30/30 ✅** | **5/11 ✅** | **7/15** | **42** | **modules + entities = 41 页 100% 04-19；topics 7/15** |
| comparison | — | ✅ | — | — | 14/14 ✅ | 14 + dimensions | — |
| 顶层 meta | 5 | — | — | — | — | 5 | — |

**总 .md 文件数**：**112**（excluding raw + log-archive）；**120**（including 8 archives）。

**下一步建议**

按 ROI 排序：

1. **schema-update — `[!info] NAMING-TRAP` 新 marker 类型 + 「subagent 禁止动 log.md + 必须 UTF-8」约束写入 AGENTS.md §5/§7** —— 沉淀 3 轮 verify/lint 累计发现
2. **`ingest sglang topics P1`** —— 剩余 8 TODO topic（continuous-batching / dp-attention / constrained / function-call / multi-api / weight-sync / hardware-backends / debug-utils）；4-5 subagent 并行
3. **`ingest sglang entities P1-P3`** —— 6 个 TODO entity（最大 ScheduleBatch 110KB）
4. **body 反向链接补充** —— modules/speculative.md → topics/speculative.md / modules/eplb.md+elastic_ep.md+layers.md → topics/moe.md（5 个 module 页 body 加 `> 跨子系统视角见 ...` 行）
5. **`compare moe across all`** —— 用 sglang/topics/moe.md 7×11 矩阵作 SGLang-row seed，新建 comparison/topics/moe.md
6. **`compare scheduler-architecture across all`** —— vLLM EngineCore 单类 vs SGLang 11 mixin vs MindIE TextGenerator+Manager 切分（用 sglang/topics/scheduler-mixins.md 作 seed）
7. **`verify comparison/topics/*`** —— 14 个 cross-compare 页累积 markers
8. **被动等待**：三仓 HEAD 都未动；如有 increment 触发，按 §12 流程

---

## [2026-04-19] compare | cross moe + scheduler-architecture | 2 个新跨项目对比页（2 subagent 并行；首次「sglang topic 作 SGLang-row seed」模式）

**触发**：用户 `compare moe / scheduler-architecture across all`，承接 lint pass 末尾「下一步建议」第 5 项。**首次实践 sglang topic 页（topics-P0 ingest 沉淀）作为 cross-compare 的 SGLang-row seed**——验证「sglang topic 深 → cross compare 浅」分工模式。

**执行方式**：2 subagent 完全并行。subagent prompt 明确禁止动 log.md + 必须 UTF-8 + 必跑 §8 rule 6 anchor-driven cross-check。

**新建页（2 个）**

| 文件 | KB / 行 | 子维度数 | 关键涵盖 |
|---|---|---|---|
| [comparison/topics/moe.md](comparison/topics/moe.md) | **39 KB / 211** | **13** | **三家 MoE 体系深度对比**：SGLang 三层正交（FusedMoE/DeepEPMoE × 7 dispatcher × 11 runner）vs vLLM 双轴正交 + oracle 子包（**68 .py fused_moe / 9 .py eplb / 3 .py elastic_ep / 13 .cu csrc/moe / 10 All2AllBackend Literal / 9 MoEBackend / 8 BaseRouter 派生**）vs MindIE 单枚举二合一（6 .py + 6 .cpp）+ 13 N/A cells（MindIE EPLB / Elastic EP / TBO+SBO 全 N/A reaffirmed）|
| [comparison/topics/scheduler-architecture.md](comparison/topics/scheduler-architecture.md) | **35.3 KB / 279** | 9 子节 / 13+ dim cells | **三家 Scheduler 架构分解对比**（区别于既有 [topics/scheduler.md](comparison/topics/scheduler.md) 的策略视角）：MindIE 双语言双层（C++ LlmEngine + C++ BatchScheduler + **15 Policy .h** + Python Generator+PluginManager 跨 pybind 紧耦合）vs vLLM 单类双层（EngineCore 0 mixin + Scheduler 0 mixin + AsyncScheduler 子类 + 6 EngineCoreClient 分支；**但 Worker 层 GPUModelRunner 有 3 mixin** —— 项目内立场不一致）vs SGLang 1 类 + 11 mixin（**5 external + 6 internal** + dispatch_event_loop **8 mode** + process_batch_result **6-way 共享分派**） |

**5 subagent 并行规模**：2 subagent 输出 35-39 KB / 211-279 行。两页都为 `confidence: high` / `verified_against: 2026-04-19` / `status: draft`。

**关键 cross-project synthesis（10 条 — 来自两个 topic 的 high-value 发现）**

### MoE compare（5 条）

1. **三家用完全不同 product space 切 MoE**：MindIE 单枚举二合一（dispatch+FFN+combine 单算子）/ vLLM 双轴正交（a2a × runner）+ oracle 子包 / SGLang 三层正交 + FusedOpPool。**架构哲学差异比实现差异更大**。
2. **dispatcher 抽象的同构性 ≫ 命名差异**：三家都把通信形态独立成 ABC（vLLM `FusedMoEPrepareAndFinalize` / SGLang `*Dispatcher` / MindIE `MOE_COMM_STRATEGIES`），但命名让初看无法识别。
3. **三家都有「枚举先于实现」dead-branch 问题**：vLLM 显式 deprecation 处理最优（`__post_init__` auto-fallback `naive`/`pplx` → `allgather_reducescatter`，[parallel.py:417-423](d:\design\vllm\vllm\config\parallel.py)）→ SGLang enum 8 vs CLI Literal 7 不同步（`CUSTOMIZED` 死代码）→ MindIE 三方源互相矛盾。**vLLM 的 deprecation 自动 fallback 是好实践**。
4. **EPLB / Elastic EP / Overlap 三件套覆盖度排序**：SGLang 全有（含 EPLB 3 算法 + ElasticEPStateManager + TBO+SBO）> vLLM 全有但单档（EplbConfig 1 算法 + 3 通信 backend + use_async；ElasticEPScalingExecutor 4 状态机；DBO with 双 token threshold）>> MindIE 全 N/A（runtime/ 0 命中 / 全仓 0 命中）；**MindIE 独有 dispatch+FFN+combine 单算子（FUSED_MC2）**——一来一回 trade-off。
5. **DeepSeek-V3 是三家共同灯塔模型** + **`routed_experts_capturer.py` 文件头 attribution 显式引用 SGLang URL**（vLLM csrc/moe → sgl_kernel/csrc/gemm 同款命名 `dsv3_router_gemm`）—— **稀有跨项目代码继承显式案例**，sglang→vllm 反向影响首次记录。

### Scheduler-architecture compare（5 条）

1. **三家「调度器复杂度」放置维度不同**：MindIE 用语言边界 + Policy 类层次（15 .h Policy）/ vLLM 用单类巨函数 + 字段开关 / SGLang 用 mixin 多继承 + 多 event-loop 分派。**复杂度总和相近，散布维度不同**。
2. **`process_batch_result` 6-way 分派点是 SGLang mixin 模式最关键的协同点**：3 个 mixin（OutputProcessor / Dllm / DisaggPrefill）通过共享同一 dispatch 表「**互不感知地共存**」—— mixin 拆分能可持续的硬证据。
3. **vLLM Scheduler 层 0 mixin vs Worker 层 3 mixin** 是不同设计选择**项目内并存**：项目内对「是否用 mixin」在不同层级有相反立场。**不能一概而论「vLLM 不用 mixin」** —— scheduler.md / engine-architecture.md 此前措辞需精化。
4. **MindIE `SwitchRole` 一等公民设计** 在 vLLM/SGLang 缺失：vLLM/SGLang 把 `kv_role` / `disaggregation_mode` 钉在启动配置，**弹性 PD 改造需先在 scheduler 接口加这一抽象** —— MindIE 架构独有可借鉴点。
5. **SGLang 8 event_loop 模式带来 PP+PDMux 组合爆炸隐患**：当前 PP 与 PDMux 互斥但源码无注释明示，对 PD+PP 联合优化的项目要先评估三家在该组合上的成熟度（MindIE 草稿 / vLLM 部分 / SGLang 互斥）。

**§8 rule 6 — Anchor-driven cross-check 结果（4 个 anchor）**

| Anchor | 来自 | 另两家结果 |
|---|---|---|
| **SGLang `MoeA2ABackend.CUSTOMIZED` 预留不可达** | SGLang 已知 | MindIE: 命中同型 `FusedMC2Strategy` 类存在但未入 `MOE_COMM_STRATEGIES` 列表（同 dead-branch 模式）；vLLM: **唯一显式处理** —— `__post_init__` auto-fallback deprecated → recommended（[parallel.py:417-423](d:\design\vllm\vllm\config\parallel.py)），全树 grep `register_dispatcher` / `customized` / `register_runner` **0 命中** |
| **SGLang `is_deepep_class_backend()` 漏 nixl** | SGLang 已知 | vLLM: 用「每 backend 独立 bool flag + `use_batched_dp_moe` 显式同族集合」避开（[parallel.py:625-631](d:\design\vllm\vllm\config\parallel.py) 把 nixl 与 deepep_ll 显式归同族）；MindIE: 单枚举无族概念无该问题 |
| **SGLang `dispatch_event_loop` 8-mode** | SGLang 已知 | vLLM 等价 = 2 step_fn × 1 run_busy_loop（PP 与非 PP 二分，无 dispatch_event_loop，**verified grep 0 命中**）；MindIE 等价 = 单一 `LlmEngine::SchedulerThreadEntry` 内部 if/else（**grep `dispatch` 作 mode router 0 命中**）。**结论：SGLang 唯一把主循环按 mode 拆成多独立函数** |
| **vLLM `EngineCore.step()` 单点** | vLLM | MindIE 等价 = `SchedulerThreadEntry` 每轮 `Schedule(needSync) + AsyncExecuteModel`，但有 `GetAsyncBatchNum() >= asyncScheduleRound` 门控；SGLang 等价 = scheduler 子进程内每个 `event_loop_*` while 循环体（无独立 "engine.step"，driver 责任由子进程 event_loop 自驱）。**三家都有「每帧 schedule + execute」语义但触发位置不同** |

**Markers added 总览**

| 页面 | `[!warning] CONTRADICTION` | `[!todo] VERIFY` | N/A 单元 |
|---|---|---|---|
| comparison/topics/moe.md | 3（MoE backend 命名 / FUSED 命名 / Mooncake 覆盖域） | 5（EPLB ROI / vLLM-SGLang capturer drift / MC2 vs FuseEP benchmark / DBO 默认关原因 / capturer SGLang commit drift） | **13** |
| comparison/topics/scheduler-architecture.md | 2（MindIE Engine↔Scheduler 跨语言双层 vs engine-architecture.md §4 描述差异 / SGLang 5+6=11 mixin vs 物理文件名只有 7 个 *_mixin.py） | 3（SGLang PP+PDMux 互斥假设 / vLLM Scheduler 0 mixin 全验证 / MindIE 测试粒度未读测试内容验证） | 多处（§2 物理位置 / §9 cheat sheet）|
| **合计** | **5** | **8** | 13+ |

**协同更新（5 文件）**

| 文件 | 修改 |
|---|---|
| [comparison/index.md](comparison/index.md) | 2 行 TODO → DONE：moe（line 68 提升 + bold）+ scheduler-architecture（**取代** line 72 旧 `scheduler-modularity` TODO，名称合并）；frontmatter `related:` += topics/moe.md + topics/scheduler-architecture.md；`已建对比页` 表新增 2 行（高密度摘要）|
| [sglang/topics/moe.md](sglang/topics/moe.md) | 末尾 See also 「暂无 [comparison/topics/moe.md]」placeholder 改为正向链接 + DONE 注脚；frontmatter `related:` += `comparison/topics/moe.md` |
| [sglang/topics/scheduler-mixins.md](sglang/topics/scheduler-mixins.md) | frontmatter `related:` += `comparison/topics/scheduler-architecture.md` |

**未应用的 cross-page 扩散（subagent 推荐，留待下次 lint pass）**

| 待修页 | 推荐 `related:` 追加 |
|---|---|
| [vllm/entities/EngineCore.md](vllm/entities/EngineCore.md) | comparison/topics/scheduler-architecture.md |
| [vllm/entities/Scheduler.md](vllm/entities/Scheduler.md) | comparison/topics/scheduler-architecture.md |
| `mindie/entities/BatchScheduler.md`（已删） | comparison/topics/scheduler-architecture.md |
| `mindie/entities/LlmEngine.md`（已删） | comparison/topics/scheduler-architecture.md |
| [comparison/topics/scheduler.md](comparison/topics/scheduler.md) | comparison/topics/scheduler-architecture.md（兄弟页互链） |
| [comparison/topics/engine-architecture.md](comparison/topics/engine-architecture.md) | comparison/topics/scheduler-architecture.md |
| `mindie/topics/moe.md`（已删） | comparison/topics/moe.md |

**净变化**

| | before | after |
|---|---|---|
| comparison topics DONE | 14/24 (58.3%) | **16/24 = 66.7%**（+2）|
| comparison/topics/ 总数 | 14 | **16**（+2）|
| 总 .md 文件数（excluding raw + log-archive）| 112 | **114**（+2）|
| 跨项目 anchor-driven cross-check 累计 N/A 强论断 | — | +13 (moe) + ≥6 (sched) = **19+ 新增** |
| **同日累计**（modules verify 68 + entities verify 12 + topics-P0 5 新页 + lint 20 文件 + compare 2 新页）| 0 | **80 RESOLVED + 7 新页 + 20 lint fixes** |

**Lessons learned**

1. **「sglang topic 作 SGLang-row seed」模式效率验证**：SGLang topic 页（[scheduler-mixins](sglang/topics/scheduler-mixins.md) 23.2 KB + [moe](sglang/topics/moe.md) 30.5 KB）已沉淀「内部架构 deep-dive」，**cross-compare subagent 不需重新 grep SGLang 源码**，仅做 vLLM/MindIE fresh grep + 三方对照表 + anchor cross-check。**比无 seed 的 cross-compare 节省 ~40-50% subagent 时间**。建议未来「先 ingest project topic → 后 compare across」成为标准 ROI 路径。
2. **vLLM fused_moe/ 68 .py 首次 fresh grep 落地**：本批 MoE compare 是 vLLM MoE 子树**首份 wiki 落地**（无 vllm/topics/moe.md seed）。subagent 实测 vLLM `model_executor/layers/fused_moe/` 5 子包结构（prepare_finalize / runner / router / experts / oracle）+ 9 .py eplb / 3 .py elastic_ep / 13 .cu csrc/moe / All2AllBackend 10 字面值（含 2 deprecated 自动 fallback）—— **未来可独立 ingest `vllm/topics/moe.md`**（68 .py 子树值得独立页）。
3. **跨项目代码继承显式 attribution 模式首次发现**：vLLM `routed_experts_capturer.py` 文件头显式引用 SGLang URL（vLLM 借鉴 SGLang 的 expert distribution capture 设计）—— **不是 vLLM → SGLang 单向 fork（如 SGLang `model_loader/` Adapted from vLLM v0.6.4），而是 SGLang → vLLM 反向**。建议 `compare unique-design-points across all` 时把这种「**双向代码继承事件**」作为单独维度。
4. **scheduler-modularity → scheduler-architecture 命名收敛**：subagent 自然把「mixin / 模块化」topic 名收敛到「架构分解」，避免与 `topics/scheduler.md`（策略视角）混淆。**两个独立 topic 页（架构 vs 策略）+ 显式 prelude 区分** 比一个混合页更清晰。建议 [AGENTS.md §8 rule 7](AGENTS.md) 加「**cross compare topic 划分应按视角而非主题**」标准。
5. **「subagent 禁止动 log.md + 必须 UTF-8」prompt 约束二次验证**：本批 2 subagent 全部遵守，0 编码事故 / 0 log entry 散乱 append。已成标准 prompt template 必备项。**建议在 [AGENTS.md §5 ingest workflow](AGENTS.md) 与 [§8 cross-project rules](AGENTS.md) 中显式加入这两条 prompt 约束**。

**§11 log rotation 阈值复核**（append 本 entry 后预估）

| | before append | after append |
|---|---|---|
| log.md 字节 | 88.9 KB | **~100 KB**（合规：< 110 KB；余 ~10 KB）|
| log.md 行数 | 584 | **~770**（合规：< 1500） |
| log.md entries | 7 | **8** |

**当前 wiki 状态**

| 项目 | overview | index | modules | entities | topics | 总数 | 04-19 verify 状态 |
|---|---|---|---|---|---|---|---|
| MindIE-LLM | ✅ | ✅ | 3/14 | 9/13 | 8/12 | 22 | 部分 |
| vLLM | ✅ | ✅ | 2/26 | 10/17 | 5/13 | 19 | 部分 |
| SGLang | ✅ | ✅ | **30/30 ✅** | **5/11 ✅** | **7/15** | **42** | **modules + entities = 41 页 100% 04-19；topics 7/15** |
| comparison | — | ✅ | — | — | **16/24**（+2）| **16 + dimensions** | — |
| 顶层 meta | 5 | — | — | — | — | 5 | — |

**总 .md 文件数**：**114**（excluding raw + log-archive）；**122**（including 8 archives）。delta +2。

**下一步建议**

按 ROI 排序：

1. **`compare unique-design-points across all`** —— 用本批新发现的「**双向代码继承**」（SGLang `routed_experts_capturer` → vLLM）+ 之前累计的 SGLang 4 unique 设计点 + 7 死代码/typo/命名陷阱 audit 维度作为 dim
2. **`ingest sglang topics P1`** —— 剩余 8 TODO topic（continuous-batching / dp-attention / constrained / function-call / multi-api / weight-sync / hardware-backends / debug-utils）；继续 sglang topic→cross compare seed 模式
3. **`ingest sglang entities P1-P3`** —— 6 个 TODO entity（最大 ScheduleBatch 110KB）
4. **`ingest vllm/topics/moe.md`** —— 本批 MoE compare 已生成 vLLM `fused_moe/` 68 .py 详细 grep 结果，可直接独立 ingest 一页 vllm/topics/moe.md
5. **lint follow-up** —— 7 项 cross-page `related:` 扩散（vllm/entities/EngineCore + Scheduler / mindie/entities/BatchScheduler + LlmEngine / comparison/topics/scheduler + engine-architecture / mindie/topics/moe）
6. **schema-update** —— 「`[!info] NAMING-TRAP` 新 marker 类型」+ 「subagent 禁止动 log.md + 必须 UTF-8」+ 「**cross compare topic 按视角划分**」+ 「**双向代码继承事件**作为 unique-design dim」 写入 AGENTS.md §5/§7/§8
7. **被动等待**：三仓 HEAD 都未动

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

