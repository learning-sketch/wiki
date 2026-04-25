---
type: module
project: sglang
status: verified
confidence: high
verified_against: 2026-04-19
sources:
  - d:\design\sglang\python\sglang\srt\grpc\__init__.py
  - d:\design\sglang\python\sglang\srt\entrypoints\grpc_server.py
related:
  - sglang/modules/entrypoints_openai.md
  - sglang/modules/observability.md
---

# `srt/grpc/` 与 gRPC 相关实现（占位 + 真实在 `entrypoints/`）

## Summary

[`d:\design\sglang\python\sglang\srt\grpc\__init__.py`](d:\design\sglang\python\sglang\srt\grpc\__init__.py) 仅含一行注释 `# SGLang gRPC module`，**无**可执行代码、无 `grpcio` import，可视为 **空包占位**。**实际** gRPC 服务入口在 [`d:\design\sglang\python\sglang\srt\entrypoints\grpc_server.py`](d:\design\sglang\python\sglang\srt\entrypoints\grpc_server.py)：[`serve_grpc`](d:\design\sglang\python\sglang\srt\entrypoints\grpc_server.py:66) 委托外部包 `smg_grpc_servicer.sglang.server.serve_grpc`，缺失依赖时抛出明确 `ImportError`（[L69-76](d:\design\sglang\python\sglang\srt\entrypoints\grpc_server.py)）；同文件还包含可选 Prometheus metrics HTTP（[L13-63](d:\design\sglang\python\sglang\srt\entrypoints\grpc_server.py)）。

> [!warning] CONTRADICTION（命名 — 必读）
>
> **目录名** `srt/grpc/` 易让人以为实现位于此包内；**事实**是核心逻辑在外部 `smg-grpc-servicer`，且仓库内另有多处 **其它** gRPC 用途（如测试里 `grpc` + `smg_grpc_proto` 多模态 encoder），与 `srt/grpc/__init__.py` **无** import 关系。

## Sources

- [d:\design\sglang\python\sglang\srt\grpc\__init__.py](d:\design\sglang\python\sglang\srt\grpc\__init__.py)（**1 .py / 22 字节占位**）
- [d:\design\sglang\python\sglang\srt\entrypoints\grpc_server.py](d:\design\sglang\python\sglang\srt\entrypoints\grpc_server.py)（服务启动与 metrics）

## Architecture（占位 vs 实际）

```mermaid
flowchart TB
  subgraph stub["srt/grpc/__init__.py"]
    S["注释占位，无 API"]
  end
  subgraph real["entrypoints/grpc_server.py"]
    G["serve_grpc → smg_grpc_servicer..."]
    M["可选 /metrics HTTP (aiohttp)"]
  end
  stub -.->|无调用边| real
```

## §跨子系统检索

| 类别 | 结果 |
|------|------|
| **sgl-kernel** | `srt/grpc/__init__.py` **不适用**；内容无 kernel 引用 |
| **`from sglang.srt.grpc` / `srt.grpc`** | 全 `python/sglang` 下 **0** 处匹配（裸包未被引用） |
| **`grpc_server.py`** | 唯一命中：[entrypoints/grpc_server.py](d:\design\sglang\python\sglang\srt\entrypoints\grpc_server.py) |
| **`grpcio` / `grpc.`** | 分布于测试与分布式等（例：`test/registered/distributed/test_epd_disaggregation.py` 含 `import grpc`）；与 `srt/grpc/` **无**直接文件级耦合 |
| **`*.proto` / `protos/`** | 仓库内存在 **其它** gRPC 生成桩引用（如测试中 `smg_grpc_proto` / `sglang_encoder_pb2`），**不**在 `srt/grpc/` 内 |

## Notes / Caveats

- 若 wiki 页标题写「`srt/grpc` 模块」，应在正文首段声明：**实现空洞**，避免读者在 `srt/grpc/` 下查找服务逻辑。
- 与 HTTP 主入口文档的关系：`serve_grpc` 为 **并行** serving 模式入口（与 `launch_server` 流程相关）。
- gRPC 模式下 metrics HTTP 由 [`grpc_server.py`](d:\design\sglang\python\sglang\srt\entrypoints\grpc_server.py) 在独立端口暴露；详见 [observability.md](observability.md) `--metrics-http-port` 字段。

## See also

- [entrypoints/grpc_server.py](d:\design\sglang\python\sglang\srt\entrypoints\grpc_server.py)（实际实现）
- [observability.md](observability.md)（gRPC 模式的 metrics HTTP）
- [entrypoints_openai.md](entrypoints_openai.md)（HTTP 模式对照）
