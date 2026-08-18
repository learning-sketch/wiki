---
type: module
project: sglang
status: draft
confidence: high
verified_against: 2026-08-18
sources:
  - d:\design\sglang\python\sglang\srt\rust_extensions\__init__.py
  - d:\design\sglang\python\sglang\srt\rust_extensions\loader.py
  - d:\design\sglang\python\sglang\srt\environ.py
  - d:\design\sglang\python\sglang\srt\entrypoints\http_server.py
  - d:\design\sglang\python\sglang\srt\managers\rust_server.py
related:
  - sglang/modules/grpc.md
  - sglang/modules/multimodal.md
  - sglang/modules/entrypoints.md
---

# `srt/rust_extensions/` — Rust PyO3 扩展的按需构建加载器

## Summary

新顶层模块（commit `67e12131df` "Build Rust extensions on demand in source checkouts" #34994 引入，2026-08-16），共 2 个文件 417 行：[`__init__.py`](d:\design\sglang\python\sglang\srt\rust_extensions\__init__.py)（5 行，re-export `load_rust_extension` / `RustBuildMode`）+ [`loader.py`](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py)（412 行）。核心 API [`load_rust_extension(python_module)`](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py) 导入一个 PyO3 扩展模块：wheel 安装场景直接 import 打包好的 `.so`；**源码 checkout 场景**在缺失时用 Cargo 从 `rust/` workspace 现场编译并缓存到 `~/.cache/sglang/rust_extensions/`，使源码开发者无需手动预构建 Rust 扩展。

## Sources

- [d:\design\sglang\python\sglang\srt\rust_extensions\__init__.py](d:\design\sglang\python\sglang\srt\rust_extensions\__init__.py)（entire file，5 行）
- [d:\design\sglang\python\sglang\srt\rust_extensions\loader.py](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py)（entire file，412 行）
- [d:\design\sglang\python\sglang\srt\environ.py:L1504-1506](d:\design\sglang\python\sglang\srt\environ.py)（`SGLANG_RUST_BUILD_MODE`）、[L991](d:\design\sglang\python\sglang\srt\environ.py)（`SGLANG_CACHE_DIR`）

## 机制说明

### 三种构建模式（`RustBuildMode = "auto" | "never" | "force"`）

模式声明见 [`loader.py:L33`](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py)，默认取环境变量 `SGLANG_RUST_BUILD_MODE`（默认 `"auto"`，[`environ.py:L1504-1506`](d:\design\sglang\python\sglang\srt\environ.py)；[`loader.py:L85-86`](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py)）：

| 模式 | 语义（docstring 原文 [`loader.py:L80-83`](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py)） |
|---|---|
| `auto` | 优先 wheel 内打包模块（[`_import_bundled_extension`, L134-140](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py)）→ 其次本地构建缓存（[L113-114](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py)）→ 最后调 Cargo 现场编译（[L123-131](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py)） |
| `never` | 允许前两者，绝不调 Cargo；缺失时抛 `ModuleNotFoundError`（[L116-121](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py)） |
| `force` | 从源码重建并替换缓存条目；若模块已被 import 则拒绝并要求新进程（[L96-100](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py)） |

### Crate 发现：零注册

[`_discover_crate`](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py)（[L143-203](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py)）遍历 `rust/` workspace（路径 = `Path(__file__).parents[4] / "rust"`，[L45](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py)）下所有 `Cargo.toml`，匹配声明 `[package.metadata.sglang] python-module = "<目标模块>"` 的 crate——与 [`python/setup.py`](d:\design\sglang\python\setup.py) wheel 构建读的是**同一份 metadata**，因此"新 crate 无需在 loader 里注册"（docstring [L75-79](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py)）。要求 `Cargo.lock` 存在以支撑 `cargo build --locked` 可复现构建（[L151-154](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py)）。

### 缓存与指纹

- **缓存根**：`SGLANG_CACHE_DIR`（默认 `~/.cache/sglang`，[`environ.py:L991`](d:\design\sglang\python\sglang\srt\environ.py)）下的 `rust_extensions/` 子目录（[`_cache_root`, L296-300](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py)）；产物路径 `artifacts/<package>/<fingerprint>/<module_leaf><EXT_SUFFIX>`（[`_cached_extension_path`, L303-316](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py)）。
- **指纹**（[`_build_context`, L206-245](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py)）= SHA-256 over：`rust/` 全源码树 digest（跳过 `.git`/`target` 等，[L36-38](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py)、[L248-260](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py)）+ cargo/rustc 工具链版本 + Python ABI（cache_tag / EXT_SUFFIX / platform / SOABI）+ 构建环境变量（`CARGO_BUILD_TARGET` / `RUSTFLAGS` 等，[L39-43](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py)）——任一变化即触发重建。
- **并发与原子性**：`fcntl.flock` 文件锁串行化同一 target 的并发构建（[`_filesystem_lock`, L319-327](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py)）；产物经 tempfile + `os.replace` + 双重 `fsync` 原子落盘（[`_stage_atomically`, L371-393](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py)）；构建期间源码变化则拒绝缓存该结果（[L125-129](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py)）。
- **构建命令**：`cargo build --release --locked --package <crate>`，`PYO3_PYTHON` 指向当前解释器（[`_cargo_build`, L330-360](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py)）。

### 使用方（全仓 grep `load_rust_extension` / `rust_extensions`）

| 扩展模块 | 使用方 | 锚点 |
|---|---|---|
| `sglang.srt.rust_extensions._grpc` | HTTP server 内嵌的 Rust 原生 gRPC server | [`http_server.py:L2728-2754`](d:\design\sglang\python\sglang\srt\entrypoints\http_server.py)（`_start_native_grpc_server_for_runtime`，配合 [`grpc_bridge.RuntimeHandle`](d:\design\sglang\python\sglang\srt\entrypoints\grpc_bridge.py)） |
| `sglang.srt.rust_extensions._server` | Rust server 管理器 | [`managers/rust_server.py:L370-372`](d:\design\sglang\python\sglang\srt\managers\rust_server.py)（TYPE_CHECKING import 见 [L40](d:\design\sglang\python\sglang\srt\managers\rust_server.py)） |
| `sglang.srt.rust_extensions._multimodal` | Inkling 图像预处理 Rust 实现 | [`multimodal/inkling/image_processing_rust.py:L14-16`](d:\design\sglang\python\sglang\srt\multimodal\inkling\image_processing_rust.py)；缺失提示见 [`multimodal/processors/inkling.py:L137`](d:\design\sglang\python\sglang\srt\multimodal\processors\inkling.py) |

- 测试覆盖：[`test/registered/rust/test_rust_extension.py`](d:\design\sglang\test\registered\rust\test_rust_extension.py)（loader 单测，覆盖三模式与 `_server`/`_grpc`/`_multimodal` 三模块名）、[`test/registered/unit/multimodal/rust/_mm_rust_utils.py`](d:\design\sglang\test\registered\unit\multimodal\rust\_mm_rust_utils.py)；[`python/sglang/test/test_utils.py:L222-225`](d:\design\sglang\python\sglang\test\test_utils.py) 以 `find_spec("...._server")` 判定 Rust 扩展可用性。
- synthesis: 三个扩展分别对应 `rust/sglang-grpc`、`rust/sglang-server`、`rust/sglang-mm` 三个 crate（同一 commit stat 中三者的 `pyproject.toml`/`Cargo.toml` 均改为声明 `python-module` metadata）。

## Key APIs / Entities

| 名称 | 位置 | 作用 |
|---|---|---|
| `load_rust_extension` | [`loader.py:L66-131`](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py) | 唯一公开入口：bundled → cache → cargo 三级回退导入 PyO3 扩展 |
| `RustBuildMode` | [`loader.py:L33`](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py) | `Literal["auto", "never", "force"]` |
| `_CrateSpec` / `_discover_crate` | [`loader.py:L48-56`](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py)、[L143-203](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py) | 由 Cargo metadata 反查 crate，零注册 |
| `_build_context` | [`loader.py:L206-245`](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py) | 源码 + 工具链 + ABI + 环境的组合指纹 |
| `_cargo_build` / `_stage_atomically` | [`loader.py:L330-360`](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py)、[L371-393](d:\design\sglang\python\sglang\srt\rust_extensions\loader.py) | locked release 构建；原子缓存落盘 |

## Notes / Caveats

> [!todo] VERIFY: 三个 Rust crate（`sglang-grpc` / `sglang-server` / `sglang-mm`）各自的功能边界未在本页深入（属 `rust/` 树，非 `srt/` Python 模块范围）；`_server` 与 `managers/rust_server.py` 的完整交互待 managers 页 verify 时补。

## See also

- [grpc.md](grpc.md)（`_grpc` 扩展的消费方；`srt/grpc/` 占位包在同一 commit 被删）
- [multimodal.md](multimodal.md)（`_multimodal` 扩展的消费方，Inkling 预处理）
- [entrypoints.md](entrypoints.md)（`http_server.py` 内嵌原生 gRPC 启动路径）
- 源码根：[`d:\design\sglang\python\sglang\srt\rust_extensions\`](d:\design\sglang\python\sglang\srt\rust_extensions)
