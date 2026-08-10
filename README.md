# d:\design Wiki

> A persistent, source-anchored knowledge base for the LLM inference engines under `d:\design\`:
> **vLLM**, **SGLang**（对比页中仍保留 MindIE-LLM 源码锚点；`mindie/` 子 wiki 已于 2026-08-10 删除）。
>
> Pattern inspired by Karpathy's [llm-wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f).

---

## 三层模型

```
raw\          外部资料（不可变）
项目代码      d:\design\{MindIE-LLM,vllm,sglang}\  事实来源（不可变）
wiki\         LLM 派生物（可变，每条论断必须能溯源到上面两层）
```

详细 schema 见 [AGENTS.md](AGENTS.md)。

---

## 入口

| 入口 | 说明 |
|---|---|
| [index.md](index.md) | 全局总目录 |
| [AGENTS.md](AGENTS.md) | LLM agent 工作手册（**任何 ingest 前必读**） |
| [log.md](log.md) | 全部 ingest / query / lint 操作的时间序记录 |
| [vllm/index.md](vllm/index.md) | vLLM 项目内目录 |
| [sglang/index.md](sglang/index.md) | SGLang 项目内目录 |
| [comparison/index.md](comparison/index.md) | 跨项目对比 |

---

## 目录结构

```
d:\design\wiki\
├── README.md                 本文件
├── AGENTS.md                 schema 与工作流
├── index.md                  全局总目录
├── log.md                    全局 append-only 日志
├── raw\                      外部资料副本
├── vllm\                     vLLM 子 wiki
│   ├── index.md / overview.md
│   ├── modules\              模块概念页
│   ├── entities\             类 / 函数实体页
│   └── topics\               跨模块主题页
├── sglang\                   SGLang 子 wiki（同上结构）
└── comparison\               跨项目对比
    ├── index.md
    ├── dimensions.md         对比维度清单
    └── topics\               按主题的三方对照页
```

> `mindie/` 子 wiki 已删除（2026-08-10）。对比页 MindIE 列仍可指向 `d:\design\MindIE-LLM\` 源码锚点。

---

## 怎么用

### 作为读者
1. 想知道某概念在某项目里怎么实现的：进 `<project>/index.md` 查 → 跳到对应 `modules` / `entities` / `topics` 页。
2. 想跨项目比较：进 [comparison/index.md](comparison/index.md)。
3. 看 `## Sources` 节获取真实代码位置，永远以源码为准。

### 作为驱动者
1. **新增源码理解**：跟 LLM 说 `ingest <project> <topic>`（例：`ingest vllm pd-disaggregation`）。
2. **提问**：直接问，LLM 会先 grep wiki，不够再读源码，并将高价值答案回写为新页。
3. **健康检查**：周期性说 `lint`，LLM 跑 [AGENTS.md §7](AGENTS.md) 的检查表。

---

## 当前状态

- [x] 目录骨架与 schema (AGENTS.md)
- [x] vLLM / SGLang 的 `overview.md` + `index.md` + 若干 modules/entities/topics
- [x] 跨项目对比维度清单与已建 comparison topics
- [x] `mindie/` 子 wiki 已移除；相关死链已清理（见 [log.md](log.md)）

后续触发示例：

```
ingest sglang scheduler
compare scheduler across all
lint
```

详见 [log.md](log.md) 跟踪进度。
