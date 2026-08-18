---
type: module
project: sglang
status: stale
confidence: high
verified_against: 2026-04-19
sources:
  - d:\design\sglang\python\sglang\srt\batch_invariant_ops\batch_invariant_ops.py
  - d:\design\sglang\python\sglang\srt\batch_invariant_ops\__init__.py
  - d:\design\sglang\python\sglang\srt\model_executor\model_runner.py
  - d:\design\sglang\python\sglang\srt\server_args.py
  - d:\design\sglang\python\sglang\srt\layers\layernorm.py
  - d:\design\sglang\python\sglang\srt\layers\moe\fused_moe_triton\fused_moe_triton_kernels.py
related:
  - sglang/modules/batch_overlap.md
  - sglang/modules/sampling.md
  - sglang/modules/model_executor.md
---

> [!todo] VERIFY: **lint 2026-08-18** — 本页正文存在 **5** 处源码死锚（多为 sglang 上游 test 树重组 / docs 站点 mdx 化 / 文件迁移所致，锚点写于 2026-04 快照），已按 §7 标 `status: stale`，待重校对。死锚清单见 log.md lint entry。

# `srt/batch_invariant_ops` — 确定性 / batch-invariant 算子（Triton GEMM / log-softmax / mean / BMM / RMSNorm）

## Summary

本模块提供 **Triton 实现的持久化 GEMM / log-softmax / mean / BMM / RMSNorm** 等算子，并通过 [`torch.library.Library("aten", "IMPL")`](d:\design\sglang\python\sglang\srt\batch_invariant_ops\batch_invariant_ops.py) 在 CUDA 上 **注册 `aten::mm`、`aten::addmm`、`aten::_log_softmax`、`aten::mean.dim`**，可选 **`aten::bmm` + 对 `torch.bmm` 的 monkeypatch**（见 [`enable_batch_invariant_mode`](d:\design\sglang\python\sglang\srt\batch_invariant_ops\batch_invariant_ops.py)）。当 [`ServerArgs.enable_deterministic_inference`](d:\design\sglang\python\sglang\srt\server_args.py) 为真时，[`ModelRunner`](d:\design\sglang\python\sglang\srt\model_executor\model_runner.py) 在初始化阶段调用 [`enable_batch_invariant_mode()`](d:\design\sglang\python\sglang\srt\batch_invariant_ops\batch_invariant_ops.py)。

> synthesis: 文件头标明 **改编自** [thinking-machines-lab/batch_invariant_ops](https://github.com/thinking-machines-lab/batch_invariant_ops)；RMS 包装注释另指向 **vLLM** `batch_invariant` 实现（见 [`rms_norm_batch_invariant`](d:\design\sglang\python\sglang\srt\batch_invariant_ops\batch_invariant_ops.py)）。

## Sources

- [d:\design\sglang\python\sglang\srt\batch_invariant_ops\batch_invariant_ops.py](d:\design\sglang\python\sglang\srt\batch_invariant_ops\batch_invariant_ops.py)（主实现，约 L1–L995）
- [d:\design\sglang\python\sglang\srt\batch_invariant_ops\__init__.py](d:\design\sglang\python\sglang\srt\batch_invariant_ops\__init__.py)
- 集成入口：[d:\design\sglang\python\sglang\srt\model_executor\model_runner.py:676-680](d:\design\sglang\python\sglang\srt\model_executor\model_runner.py)
- CLI / 环境：[d:\design\sglang\python\sglang\srt\server_args.py](d:\design\sglang\python\sglang\srt\server_args.py)（`enable_deterministic_inference`、`SGLANG_ENABLE_DETERMINISTIC_INFERENCE`）
- 调用方示例：[d:\design\sglang\python\sglang\srt\layers\layernorm.py:199-208](d:\design\sglang\python\sglang\srt\layers\layernorm.py)、[d:\design\sglang\python\sglang\srt\layers\moe\fused_moe_triton\fused_moe_triton_kernels.py:11](d:\design\sglang\python\sglang\srt\layers\moe\fused_moe_triton\fused_moe_triton_kernels.py)

## Architecture / Data flow

1. **模式开关**：[`_batch_invariant_MODE`](d:\design\sglang\python\sglang\srt\batch_invariant_ops\batch_invariant_ops.py) 与 [`is_batch_invariant_mode_enabled()`](d:\design\sglang\python\sglang\srt\batch_invariant_ops\batch_invariant_ops.py) 暴露当前是否启用。
2. **注册 vs 直接调用**：[`enable_batch_invariant_mode`](d:\design\sglang\python\sglang\srt\batch_invariant_ops\batch_invariant_ops.py) 注册 ATen CUDA 实现；[`RMSNorm`](d:\design\sglang\python\sglang\srt\layers\layernorm.py) 在 `is_batch_invariant_mode_enabled()` 为真且无 residual 等条件时 **直接调用** [`rms_norm_batch_invariant`](d:\design\sglang\python\sglang\srt\batch_invariant_ops\batch_invariant_ops.py)（不经过 `aten` 包装路径）。
3. **GEMM 路径**：[`matmul_persistent`](d:\design\sglang\python\sglang\srt\batch_invariant_ops\batch_invariant_ops.py) 在 BF16 + JIT DeepGEMM 等条件满足时走 [`_matmul_persistent_deepgemm`](d:\design\sglang\python\sglang\srt\batch_invariant_ops\batch_invariant_ops.py)，否则 Triton [`_matmul_persistent_triton`](d:\design\sglang\python\sglang\srt\batch_invariant_ops\batch_invariant_ops.py)；环境变量 `SGLANG_BATCH_INVARIANT_OPS_ENABLE_MM_DEEPGEMM` 等见文件顶部。
4. **Attention 块大小**：[`get_batch_invariant_attention_block_size`](d:\design\sglang\python\sglang\srt\batch_invariant_ops\batch_invariant_ops.py) 返回固定 `AttentionBlockSize(block_m=16, block_n=16)`，供确定性 attention 配置使用。

## File inventory

| 文件 | 角色 |
|------|------|
| [`__init__.py`](d:\design\sglang\python\sglang\srt\batch_invariant_ops\__init__.py) | 重导出公开 API |
| [`batch_invariant_ops.py`](d:\design\sglang\python\sglang\srt\batch_invariant_ops\batch_invariant_ops.py) | Triton 核、ATen 注册、上下文管理器 [`set_batch_invariant_mode`](d:\design\sglang\python\sglang\srt\batch_invariant_ops\batch_invariant_ops.py) |

## Triton kernel inventory（`@triton.jit`）

| 符号 | 作用 | 锚点 |
|------|------|------|
| `matmul_kernel_persistent` | 2D GEMM，persistent grid，`tl.range` 遍历 tile | [batch_invariant_ops.py:69-160](d:\design\sglang\python\sglang\srt\batch_invariant_ops\batch_invariant_ops.py) |
| `_log_softmax_kernel` | 末维 log-softmax，每行一 block | [batch_invariant_ops.py:308-378](d:\design\sglang\python\sglang\srt\batch_invariant_ops\batch_invariant_ops.py) |
| `mean_kernel` / `mean_dim` | 单维 mean | [batch_invariant_ops.py:424-564](d:\design\sglang\python\sglang\srt\batch_invariant_ops\batch_invariant_ops.py) |
| `bmm_kernel_persistent` | Batched GEMM，注释强调 **batch-major 确定性顺序** | [batch_invariant_ops.py:596-712](d:\design\sglang\python\sglang\srt\batch_invariant_ops\batch_invariant_ops.py) |
| `_rms_norm_kernel` / `rms_norm` | 末维 RMSNorm | [batch_invariant_ops.py:812-907](d:\design\sglang\python\sglang\srt\batch_invariant_ops\batch_invariant_ops.py) |

## Deterministic inference integration

- **模型 runner**：[`if server_args.enable_deterministic_inference:`](d:\design\sglang\python\sglang\srt\model_executor\model_runner.py) → [`enable_batch_invariant_mode()`](d:\design\sglang\python\sglang\srt\batch_invariant_ops\batch_invariant_ops.py)。
- **Server 参数**：[`enable_deterministic_inference: bool = False`](d:\design\sglang\python\sglang\srt\server_args.py)；[`_handle_deterministic_inference`](d:\design\sglang\python\sglang\srt\server_args.py) 将 sampling 设为 `pytorch` 等；[`_handle_environment_variables`](d:\design\sglang\python\sglang\srt\server_args.py) 设置 `envs.SGLANG_ENABLE_DETERMINISTIC_INFERENCE`。
- **图编译**：[`if self.enable_deterministic_inference: self.disable_piecewise_cuda_graph = True`](d:\design\sglang\python\sglang\srt\server_args.py)（与 batch-invariant 路径交互时需知）。

## CLI / 环境变量（摘录）

| 机制 | 锚点 |
|------|------|
| `--enable-deterministic-inference` | [server_args.py:677](d:\design\sglang\python\sglang\srt\server_args.py) |
| `SGLANG_ENABLE_DETERMINISTIC_INFERENCE` | [server_args.py:3696-3698](d:\design\sglang\python\sglang\srt\server_args.py) |
| `SGLANG_BATCH_INVARIANT_OPS_ENABLE_MM_DEEPGEMM` 等 | [batch_invariant_ops.py:18-27](d:\design\sglang\python\sglang\srt\batch_invariant_ops\batch_invariant_ops.py) |

## §跨子系统 5 grep（batch_invariant / deterministic）

1. **sgl-kernel C++**：`d:\design\sglang\sgl-kernel\` 全树 grep `batch_invariant` → **0 命中**。
2. **Collaborator imports（`srt/`）**：`model_runner`（`enable_batch_invariant_mode`）、`layers/layernorm.py`、`layers/moe/fused_moe_triton/fused_moe_triton_kernels.py`（`is_batch_invariant_mode_enabled`）；另有包内 `__init__.py` 导出。
3. **CLI / configs**：`server_args.py` 中 `enable_deterministic_inference` 多处（定义、piecewise cuda graph、env 同步、`_handle_deterministic_inference`）。
4. **Tests**：[d:\design\sglang\test\registered\unit\batch_invariant_ops\test_batch_invariant_ops.py](d:\design\sglang\test\registered\unit\batch_invariant_ops\test_batch_invariant_ops.py)；其它测试文件仅 **mock** 或参数里出现 `enable_deterministic_inference`。
5. **Docs**：[d:\design\sglang\docs\advanced_features\deterministic_inference.md](d:\design\sglang\docs\advanced_features\deterministic_inference.md)、[server_arguments.md](d:\design\sglang\docs\advanced_features\server_arguments.md)、[sglang_for_rl.md](d:\design\sglang\docs\advanced_features\sglang_for_rl.md) 等引用 deterministic / batch-invariant。

## Notes

> [!todo] VERIFY: ~~[`log_softmax` 的 docstring](d:\design\sglang\python\sglang\srt\batch_invariant_ops\batch_invariant_ops.py) 中含异常行 `>> Stashed changes`（约 L388），疑似合并残留，应以上游为准是否删除。~~
> **RESOLVED 2026-04-19**: **仍存在**。在 [`batch_invariant_ops.py L388`](d:\design\sglang\python\sglang\srt\batch_invariant_ops\batch_invariant_ops.py) 的 `log_softmax(...)` docstring（L382-L391）正文中含 `>> Stashed changes` 单独一行，明显是 git stash pop / merge 工具未清理的残留。**未影响运行时行为**（位于 docstring 内）。建议向上游 sglang 提 patch；本页继续标注。

> [!warning] CONTRADICTION: ~~本页不声称与 vLLM 当前 tree 的 **逐行** 等价；源码仅 **注释链接** 到 vLLM `batch_invariant` 与 TML repo。~~
> **RESOLVED 2026-04-19**: 立场仍正确——本模块文件头注释指向 [thinking-machines-lab/batch_invariant_ops](https://github.com/thinking-machines-lab/batch_invariant_ops)（[batch_invariant_ops.py 文件头](d:\design\sglang\python\sglang\srt\batch_invariant_ops\batch_invariant_ops.py)），`rms_norm_batch_invariant`（[L910](d:\design\sglang\python\sglang\srt\batch_invariant_ops\batch_invariant_ops.py)）注释另指向 vLLM 实现。本页不主张逐行对比，仅作引用关系说明，立场保留。

## See also

- [batch_overlap.md](batch_overlap.md)（调度/前向 overlap，另一概念层）
- [sampling.md](sampling.md)（`--enable-deterministic-inference` 与 `flashinfer` sampling backend 的交互）
- [model_executor.md](model_executor.md)（启用入口）
- [deterministic_inference.md（上游 doc）](d:\design\sglang\docs\advanced_features\deterministic_inference.md)
