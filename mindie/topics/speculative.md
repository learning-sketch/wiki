---
type: topic
project: mindie
status: verified
confidence: high
verified_against: 2026-04-18
sources:
  - d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\mtp\mtp_plugin.py
  - d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\la\la_plugin.py
  - d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\memory_decoding\memory_decoding_plugin.py
  - d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\spec_worker.py
  - d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\model_runner.py
  - d:\design\MindIE-LLM\mindie_llm\runtime\models\deepseek_v3\deepseek_v3_mtp.py
  - d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py
  - d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_utils.py
  - d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\mtp\decoding_policy.py
  - d:\design\MindIE-LLM\src\scheduler\scheduler.cpp
  - d:\design\MindIE-LLM\src\include\config\config_info.h
  - d:\design\MindIE-LLM\src\config_manager\config_interaction.cpp
  - d:\design\MindIE-LLM\docs\zh\user_guide\feature\speculative_decoding.md
  - d:\design\MindIE-LLM\docs\zh\user_guide\feature\mtp.md
  - d:\design\MindIE-LLM\docs\zh\developer_guide\architecture_design\mtp.md
related:
  - mindie/entities/PluginManager.md
  - mindie/entities/BatchScheduler.md
  - mindie/entities/LlmEngine.md
  - mindie/topics/moe.md
  - comparison/dimensions.md
  - comparison/topics/async-schedule.md
  - comparison/topics/speculative-decoding.md
---

# Speculative Decoding（MTP / LA / Memory Decoding 三 plugin + spec_worker）

## Summary

> synthesis: MindIE-LLM 的「投机解码」在 **Plugin 层** 拆成三个独立算法包：`mtp`（多 token 草稿 + 主模型校验，配合 **MtpWorker** 双 ModelRunner）、`la`（Lookahead / Jacobi 风格猜测 + `plugin_verify`）、`memory_decoding`（前缀树缓存历史 IO 做候选 + `plugin_verify`）。**C++ Scheduler** 用 `speculationGamma` 与 `tokenNumPerIter = 1 + speculationGamma` 做 **placeholder token** 预留，异步调度下 `AddNextTokenPlaceHolder` 追加占位符，`ReplacePlaceHolderWithToken` 用引擎返回的真实 token 回填。**Model 层** DeepSeek MTP（`DeepseekV3MTP`）是 **模型内嵌草稿头**，与通用 `Plugin` 解耦，由 `forward_context.mtp_metadata.last_hidden_states` 驱动。

## Sources

见 frontmatter `sources`；跨项目对照额外参考：[`d:\design\vllm\vllm\v1\spec_decode\eagle.py`](d:\design\vllm\vllm\v1\spec_decode\eagle.py)、[`d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py`](d:\design\sglang\python\sglang\srt\speculative\eagle_worker_v2.py) 等（**synthesis**，非 MindIE 源码内嵌）。

## 三 plugin 子目录结构

| 子目录 | 文件（Glob） | 主类 |
|--------|----------------|------|
| `mindie_llm/text_generator/plugins/mtp/` | `mtp_plugin.py`, `decoding_policy.py`, `__init__.py` | `MtpPlugin(Plugin)` [mtp_plugin.py:24-25](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\mtp\mtp_plugin.py) |
| `.../plugins/la/` | `la_plugin.py`, `decoding_policy.py`, `la_statistics.py`, `__init__.py` | `LaPlugin(Plugin)` [la_plugin.py:26-27](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\la\la_plugin.py) |
| `.../plugins/memory_decoding/` | `memory_decoding_plugin.py`, `decoding_policy.py`, `dynamic_decoding.py`, `tokens_knowledge_base_cache.py`, `__init__.py` | `MemoryDecodingPlugin(Plugin)` [memory_decoding_plugin.py:21-22](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\memory_decoding\memory_decoding_plugin.py) |

三者均 **继承 `Plugin`**（从 `..plugin` 导入），与 `PluginManager.initialize()` 约定一致：按 `plugin_list` 动态 `importlib` 加载 `mindie_llm.text_generator.plugins.{name}.{name}_plugin`，类名为 `{Name}Plugin` [plugin_manager.py:186-200](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_manager.py)。

## MTP（Multi-Token Prediction）

- **主类** `MtpPlugin`：负责 `model_inputs_update`（经 `DecodingPolicy`）、`sample_preprocess`（展开 `all_sequence_ids` / `all_token_ids`）、`plugin_verify`（`verify_greedy_one_batch` 贪婪校验草稿）、`fill_in_model_result` / `prepare_masks_for_filling`（异步命中路径）、`plugin_cache_update`（`CacheEngine` 写 hidden 前缀）等 [mtp_plugin.py](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\mtp\mtp_plugin.py)。
- **与 `deepseek_v3_mtp.py` 协作**：`DeepseekV3MtpModel.forward` 从 `get_forward_context().mtp_metadata.last_hidden_states` 取主模型末层 hidden，与 `embed_tokens`、`eh_proj`、单层 `DeepseekV3MtpLayer` 算草稿 hidden，再经 `ParallelLMHead` 出 logits [deepseek_v3_mtp.py:141-170, 217-229](d:\design\MindIE-LLM\mindie_llm\runtime\models\deepseek_v3\deepseek_v3_mtp.py)。这是 **DeepSeek 架构自带的 MTP 模块**，通过 `router` 选 `DeepseekV3MTP` 作为 **draft** `model_cls`（`is_draft_model=True`）与主模型成对加载。
- **`PluginDataParam`**：`decode_model_input_update` 写入 `mtp_model_inputs` 与 `hidden_states` [decoding_policy.py:129-134](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\mtp\decoding_policy.py)；`fill_in_model_result` / `fill_in_model_result_exp` 消费 `sub_model_inputs`、`hidden_states` 做 hit 回填 [mtp_plugin.py:107-158, 332-420](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\mtp\mtp_plugin.py)。
- **通用性**：**Plugin 框架通用**（`MtpPlugin` + `DecodingPolicy`）；**模型侧 MTP 前向**当前实现绑定 **DeepSeek V3 MTP 层**（固定 `layer_idx = 61` 等）[deepseek_v3_mtp.py:131-138](d:\design\MindIE-LLM\mindie_llm\runtime\models\deepseek_v3\deepseek_v3_mtp.py)。非 DeepSeek 需另有 draft 模型定义。

## LA（Lookahead）

- **主类** `LaPlugin`：`level` / `window` / `guess_set_size`（默认 4/5/5）记录于 `__init__` [la_plugin.py:41-44](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\la\la_plugin.py)。
- **decode**：`sample_preprocess` 按 `store_guess_tokens` 计算每 batch `logits_num_per_batch = 1 + guess_token_num`，从展平 logits 中截取"下一跳猜测"对应行到 `next_guess_logits`，并展开 `all_sequence_ids` / 采样参数 [la_plugin.py:67-117](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\la\la_plugin.py)。
- **verify**：`plugin_verify` → `la_token_verify_not_sample` → `decoding_policy.la_verify_greedy_one_batch`，必要时 `la_cache.set_need_cal_kv` [la_plugin.py:119-144, 184-189](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\la\la_plugin.py)。
- **与 PluginManager**：与其它 plugin 一样挂在 `plugin_list` 上；**不**走 `MtpWorker`（见下节）。

## Memory Decoding

- **主类** `MemoryDecodingPlugin`：`decoding_length` 默认 16，裁剪在 `[1,16]` [memory_decoding_plugin.py:39-49](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\memory_decoding\memory_decoding_plugin.py)。
- **算法要点**：`calc_decoding_info` / `decoding_policy.update_infer_input` 在非 prefill 时更新解码支路；`all_token_ids_padding` 用 `memory_decoding_decoding_ids`（候选链）填充 `-1` 与猜测 token [memory_decoding_plugin.py:76-143](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\memory_decoding\memory_decoding_plugin.py)；`plugin_verify` 调 `decoding_policy.verify` [memory_decoding_plugin.py:225-234](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\memory_decoding\memory_decoding_plugin.py)；`plugin_cache_update` 维护 `prefix_ids` 与 `tokens_knowledge_base_cache` [memory_decoding_plugin.py:236-265](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\memory_decoding\memory_decoding_plugin.py)（与文档"前缀树缓存历史 IO"一致 [speculative_decoding.md:19-22](d:\design\MindIE-LLM\docs\zh\user_guide\feature\speculative_decoding.md)）。

## SpecWorker（Worker 层）

文件 [spec_worker.py](d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\spec_worker.py) 全文件要点：

- **`BaseWorkerProxy`**：抽象 `forward` / `__getattr__` [spec_worker.py:25-33](d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\spec_worker.py)。
- **`MtpWorker`**：持有 `main_model_runner`、`draft_model_runner`（`is_draft_model=True`）[spec_worker.py:43-64](d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\spec_worker.py)；`forward_mtp_prefill` 主模型 prefill → 更新 MTP 参数 → draft forward 刷 cache [spec_worker.py:121-150](d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\spec_worker.py)；`forward_mtp_decode` 先 `forward_draft_decode` 再主模型 [spec_worker.py:280-310](d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\spec_worker.py)；`forward` 按 `is_prefill` 分支 [spec_worker.py:312-320](d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\spec_worker.py)。
- **`MtpWorkerExp`**：实验路径（`ENV.model_runner_exp`），签名含 `npu_cache`、`forward_context`、`build_forward_context` 合并 sub context [spec_worker.py:323-659](d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\spec_worker.py)。
- **`auto_speculative_method_router` + `speculative_worker_selector`**：`num_speculative_tokens > 0` 时选 `MtpWorkerExp`（exp）或 `MtpWorker`，否则返回 `None` 用原始类 [spec_worker.py:693-707](d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\spec_worker.py)。
- **与 `ModelRunner` 关系**：`ModelRunner` 类被装饰器包裹，仅在 MTP 场景替换为 `MtpWorker*` [model_runner.py:44-45](d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\model_runner.py)。
- **三 plugin 是否共用一个 Worker**：**仅 MTP** 使用 `MtpWorker`；**LA / memory_decoding** 不增加该 proxy，仍为单 `ModelRunner` + Plugin 逻辑。

## C++ Scheduler placeholder 机制

- **`speculationGamma`**：`ModelParam` / `ModelDeployConfig` 均含 `uint32_t speculationGamma` [config_info.h:118, 282](d:\design\MindIE-LLM\src\include\config\config_info.h)。
- **`tokenNumPerIter`**：`1 + speculationGamma`（注释：1 为主模型 token，`speculationGamma` 为 MTP/投机侧）[scheduler.cpp:765-766](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)。
- **`maxPlaceHolderNum`**：`maxScheduledBatch_ * tokenNumPerIter + tokenNumPerIter` [scheduler.cpp:769-771](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)；与 [BatchScheduler.md](../entities/BatchScheduler.md) 占位公式一致。
- **`AddNextTokenPlaceHolder`**：调度出队前向输出侧追加 `PLACEHOLDER_TOKEN` [scheduler.cpp:802-824](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)；`speculationGamma > 0` 时 KV pull 场景也会追加 [scheduler.cpp:431-434](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)。
- **`ReplacePlaceHolderWithToken`**：从 `predictedTokensBySeqId_` 取真实 token，替换尾部占位；校验 `placeholderCount` 与 `numGenTokens`、`tokenNumPerIter` 关系 [scheduler.cpp:854-915](d:\design\MindIE-LLM\src\scheduler\scheduler.cpp)；调用链见同文件约 1654/1677/1824 行。
- **与 plugin verify**：Python 侧 `plugin_verify` 决定接受长度；C++ 侧重在 **异步占位与 KV 槽位** 一致性，引擎回填 token 后 scheduler 替换占位。

## 三算法对比表（核心 deliverable）

| 维度 | MTP | Lookahead (LA) | Memory Decoding |
|------|-----|----------------|-------------------|
| 草稿来源 | 同权重复制的 **draft ModelRunner**（DeepSeek MTP 层） | Jacobi + prompt/历史生成的 **guess 集合** | **Trie / 知识库** 历史 IO 候选 |
| 每步 logits 形状 | 固定 `num_speculative_tokens+1` 每请求 | **变长** `1 + guess_token_num` | **变长**（`q_len` 由 policy 定） |
| Worker | **MtpWorker / MtpWorkerExp** | 无 | 无 |
| `speculationGamma` 约束（validator） | `max(num_st, 2*num_st-2) <= gamma`，且 `num_st<=5` [plugin_utils.py:50-55](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_utils.py) | `(N-1)*(W+G) <= gamma` [plugin_utils.py:46-47](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_utils.py) | `decoding_length <= gamma` [plugin_utils.py:73-75](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_utils.py) |
| 文档性能/场景 | 吞吐优先；`num_speculative_tokens` 1~2 等 [mtp 用户文档](d:\design\MindIE-LLM\docs\zh\user_guide\feature\mtp.md) | 文本/对话 [speculative_decoding.md](d:\design\MindIE-LLM\docs\zh\user_guide\feature\speculative_decoding.md) | 代码/检索 [speculative_decoding.md](d:\design\MindIE-LLM\docs\zh\user_guide\feature\speculative_decoding.md) |
| 互斥 | 与 **并行解码（LA/memory）** 文档声明不可与 **MTP** 同时 [speculative_decoding.md:29, 34](d:\design\MindIE-LLM\docs\zh\user_guide\feature\speculative_decoding.md) | 与 **memory_decoding** 不可同时 [speculative_decoding.md:34](d:\design\MindIE-LLM\docs\zh\user_guide\feature\speculative_decoding.md) | 同上 |
| splitfuse / prefix_cache | 文档：并行解码与 SplitFuse 等互斥；`plugin_utils` 白名单含 `mtp`+`prefix_cache` 组合（需查部署约束） | 并行解码限制同文档 | 同左 |

## 配置依赖

- **JSON**：`ModelDeployConfig.speculationGamma`、`ModelConfig.plugin_params`（见文档示例 [speculative_decoding.md:83-127](d:\design\MindIE-LLM\docs\zh\user_guide\feature\speculative_decoding.md)）。
- **`PluginParameterValidator`**：按 plugin 类型校验 `speculation_gamma` 与 JSON 字段 [plugin_utils.py:63-80](d:\design\MindIE-LLM\mindie_llm\text_generator\plugins\plugin_utils.py)。
- **Server**：`mtpEnabled` 与 **structured output** 互斥（`response_format`）[infer_param.cpp:217-221](d:\design\MindIE-LLM\src\server\endpoint\utils\infer_param.cpp)。
- **`config_interaction`**：`speculationGamma > 0` 或 `plugin_params` 含 `mtp` → `CheckMtpEnabled` [config_interaction.cpp:114-127](d:\design\MindIE-LLM\src\config_manager\config_interaction.cpp)。

## 跨子系统引用（[AGENTS.md §5 step 3](../../AGENTS.md) — 5 类 grep 结果）

### 1. 跨语言绑定（`d:\design\MindIE-LLM\src\`）

| 模式 | 结果 |
|------|------|
| `SpecWorker` | **0 命中**（N/A：Worker 名在 Python 为 `MtpWorker`，C++ 无 `SpecWorker` 符号） |
| `mtp` | 命中：`scheduler.cpp`、`config_interaction.cpp`、`infer_param`、`model_deploy_config.cpp`、脚本 `A2_single_machine.md` 等 |
| `lookahead` | **0 命中**（N/A：字面未出现；LA 以插件名 **`la`** 出现） |
| `memory_decoding` | 命中：`config_interaction.cpp` 中 `UNSUPPORTED_PLUGINS` 列表 [config_interaction.cpp:23](d:\design\MindIE-LLM\src\config_manager\config_interaction.cpp) |

### 2. 协作伙伴（全仓库精选）

| 模式 | 结果摘要 |
|------|-----------|
| `speculationGamma` | `scheduler.cpp`、`llm_manager_impl.cpp`、`model_deploy_config.cpp`、`config_interaction.cpp`、`test_scheduler.cpp` 等 |
| `SpecWorker` | 仅 Python `spec_worker.py` 文件名与注释语义；**无** C++ 绑定 |
| `mtp_model_inputs` | `plugin_manager.py`、`mtp_plugin.py`、`decoding_policy.py`、测试 `test_decoding_policy.py` |
| `hidden_states` | 广谱命中；**spec 相关**集中：`plugin_utils.PluginDataParam`、`mtp/decoding_policy`、`mtp_plugin.fill_in_model_result`、`spec_worker` `kwargs.get("hidden_states")`、`model_runner` `hidden_states_mtp` |
| `ReplacePlaceHolderWithToken` | `scheduler.h`、`scheduler.cpp` 定义与实现；`test_scheduler.cpp` 单测 |

### 3. 配置 / IPC

| 模式 | 结果摘要 |
|------|-----------|
| `speculation` / `speculative` | `scheduler`、引擎、`model_deploy_config`、`docs` 多处 |
| `num_spec_tokens` | **0 命中**（N/A：全仓库无此标识符；使用 `num_speculative_tokens` / `speculationGamma`） |
| `gamma` | 多为 `speculation_gamma` / `speculationGamma` 字段名，非裸 `gamma` |

### 4. 测试覆盖（`tests/`）

| 区域 | 文件示例 |
|------|-----------|
| MTP | `tests/pythontest/npu/text_generator/test_plugins/test_mtp_plugin.py`、`test_async_mtp.py` |
| LA | `test_la_plugin.py` |
| Memory | `test_md_plugin.py` |
| Policy | `test_decoding_policy.py` |
| C++ | `tests/dlt/ut/scheduler/test_scheduler.cpp`（`ReplacePlaceHolderWithToken`、`speculationGamma`） |

### 5. doc / yaml / json

| 模式 | 结果摘要 |
|------|-----------|
| `docs/` 下 `speculative` / `mtp` | `speculative_decoding.md`、`mtp.md`、`developer_guide/architecture_design/mtp.md`、`prefix_cache.md`（mtp+prefix 示例）等 |
| `*.yaml` | **0 命中**（N/A：`MindIE-LLM` 根下未检出含 `speculation`/`mtp` 的 yaml） |
| `*.json` | 配置示例多在文档内嵌；测试 `config.json` 多为模型结构非 spec 字段 |

## 跨项目对照（synthesis）

| MindIE | vLLM（`v1/spec_decode/`） | SGLang（`srt/speculative/`） |
|--------|---------------------------|------------------------------|
| **MTP**：同模型双 runner + DeepSeek MTP 层 + 贪婪 verify | **MTP / EAGLE**：`eagle.py` 等统一 `SpecDecodeBaseProposer`，Eagle3 多草稿、树注意力元数据 | **Eagle v2**：`eagle_worker_v2.py`、`multi_layer_eagle_worker_v2.py`，与 Triton/CUDA graph 深度集成 |
| **LA**：Jacobi 变长 guess，无独立 draft 网络 | **N-gram / Medusa / DFlash** 等并行 proposer | **ngram_worker**、corpus 辅助 |
| **Memory decoding**：Trie 历史 IO | 语义接近 **n-gram / external corpus** 路线 | `cpp_ngram`、`external_corpus_manager.py` |
| **Scheduler placeholder** | 由 vLLM 连续批处理与 spec metadata 管理，无同名 `ReplacePlaceHolderWithToken` | `spec_info` / `standalone_worker_v2` 等另一套编排 |

> synthesis: 较 [comparison/dimensions.md §dim-spec](../../comparison/dimensions.md) 的一行深入：**MindIE 把"算法"固定在 Python Plugin + 可选 `MtpWorker`，Scheduler 用 `speculationGamma` 统一预留 KV；vLLM/SGLang 则把多种 proposer（Eagle/N-gram/MTP）收进同一 spec 元数据与 kernel 路径，Scheduler 不负责 MindIE 式 `-1` 占位符链。**

## Notes / Caveats

> [!warning] CONTRADICTION: `spec_worker.py` 文件名易误解为"泛型 SpecWorker"；实际类名为 **`MtpWorker`**，且仅 **`num_speculative_tokens > 0`** 启用 [spec_worker.py:693-707](d:\design\MindIE-LLM\mindie_llm\runtime\model_runner\spec_worker.py)。
> [!todo] VERIFY: `ConfigInteraction::UNSUPPORTED_PLUGINS` 标记 `mtp, la, memory_decoding` 为 **服务端某类能力检测用"不支持"列表**（与 Python 插件白名单并存，需按调用语义理解）[config_interaction.cpp:21-23](d:\design\MindIE-LLM\src\config_manager\config_interaction.cpp)。
> [!todo] VERIFY: 并行解码文档列出的与 **PD 分离、SplitFuse、MTP、异步调度** 等互斥需以产品版本为准 [speculative_decoding.md:29](d:\design\MindIE-LLM\docs\zh\user_guide\feature\speculative_decoding.md)。

## See also

- [mindie/entities/PluginManager.md](../entities/PluginManager.md)（`initialize` 与 `mtp_model_inputs` 传递）
- [mindie/entities/BatchScheduler.md](../entities/BatchScheduler.md)（`speculationGamma` / `tokenNumPerIter`）
- [mindie/entities/LlmEngine.md](../entities/LlmEngine.md)（C++ engine 主循环对 `asyncBatchNum_` / placeholder 的协作）
- [mindie/topics/moe.md](moe.md)（DeepSeek-V3 MTP 与 MoE 模型实现的关系）
- [comparison/topics/async-schedule.md](../../comparison/topics/async-schedule.md)（占位与异步推理）
- 官方文档：[mtp.md](d:\design\MindIE-LLM\docs\zh\developer_guide\architecture_design\mtp.md)（MTP 数据流详解）
