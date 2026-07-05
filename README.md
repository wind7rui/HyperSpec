# HyperSpec

> v1.1.0 · [更新日志](CHANGELOG.md)

规格驱动 + 工程纪律的完整开发工作流 Skill，协调 [OpenSpec](https://github.com/fission-ai/openspec)（规格管理）和 [Superpowers](https://github.com/obra/superpowers)（TDD + 子代理审查），从需求到实现到归档一条流程走完。

OpenSpec 管「做什么和为什么」，Superpowers 管「怎么做和做得对不对」。HyperSpec 是**轻量编排框架**：以编排 OpenSpec/Superpowers 为主（项目感知、状态检测、阶段路由、commit 纪律），不重写原生 skill 的功能；另含自有的规格一致性验证、计划质量审查等显式增强（不伪装为纯透传）。

## 核心价值

- **项目感知**：自动探测语言/框架/构建工具，自适应生成规格和执行策略
- **需求先行**：强制先产出规格文档再写代码，避免 AI 闷头实现方向跑偏
- **轻量编排框架**：以编排 OpenSpec/Superpowers 为主，不重写其功能；另含自有的规格一致性验证等显式增强
- **断点恢复**：结构化状态文件 + 实际文件双重验证，任何中断点可精确恢复
- **智能执行**：根据任务数量、依赖关系、跨模块性等多因子选择最优执行模式
- **多语言支持**：自动适配 Java/Node/Go/Rust/Python 等不同技术栈的编译和测试命令
- **知识图谱感知（可选）**：叠加 CodeGraph（代码结构层）+ Graphify（文档知识层），提供调用链/影响面/历史规格的结构化检索；不可用时自动回退 grep/Read，核心流程不依赖知识图谱

## 前置依赖

| 依赖 | 用途 | 检查方式 | 安装方式 |
|------|------|----------|----------|
| **Superpowers** skill | TDD、计划编写、子代理开发、代码审查 | 检查 brainstorming 等 skill 是否可用 | `/plugin install superpowers@claude-plugins-official` |
| **OpenSpec** CLI | 规格文档管理（变更提案、设计文档、任务拆分、归档） | 检查项目根目录是否有 `openspec/` | `npx @fission-ai/openspec init` |

**可选依赖（知识图谱感知，不可用时自动降级为 grep/Read）：**

| 依赖 | 用途 | 可用性检测（两态） |
|------|------|----------|
| **CodeGraph** MCP | 代码结构层：AST 解析 → 符号/调用链/依赖图/影响面，零 LLM 依赖 | ① `indexed`：`.codegraph/` 在不在（项目属性）② `tool_reachable`：试调 `codegraph_explore` 探活（会话属性，会失效） |
| **Graphify** Skill/CLI | 文档知识层：对 `openspec/` 构建语义图谱，检索历史规格和归档经验 | ① `indexed`：`graphify-out/` 在不在（项目属性）② `tool_reachable`：试调 `graphify` CLI 探活（会话属性，会失效） |

> 知识图谱工具是可选增强。可用性按**两态**判断：`indexed`（索引在不在，项目属性）与 `tool_reachable`（工具在当前会话调不调得通，会话属性），`available = 两者皆真`——用"目录在"推断"工具通"是范畴错误（装了 CLI 但没配 MCP 时目录在、工具却不可达）。未安装或检测不可用时标记 `available: false` 自动回退 grep/Read，不影响主流程；运行时若任一 KG 工具调用返回「Unknown tool / 工具不存在 / 连接错误」则立即判本次不可达、回退 grep/Read 并把 `tool_reachable` 置 `false`，堵死"目录在但 MCP 未注入 → 静默失败"。
>
> **CodeGraph 安装**（colbymchenry/codegraph，按官方手册）：
> ```bash
> npm i -g @colbymchenry/codegraph     # 或 curl -fsSL https://raw.githubusercontent.com/colbymchenry/codegraph/main/install.sh | sh
> codegraph install                    # 在一个新的终端中，运行安装程序以将 CodeGraph 连接到你使用AI agent
> cd <your-project> && codegraph init       # 构建该项目的 .codegraph/ 索引（一次性；⚠️ MCP stdio 下不自动同步，apply 改动后须手动 `codegraph sync`）
> ```
> 其 MCP 服务**默认只暴露 `codegraph_explore` 一个工具**（单次调用已返回源码 + 调用链 + 影响面）。若需 `codegraph_search`/`callers`/`callees`/`impact` 等，给 MCP 服务设环境变量 `CODEGRAPH_MCP_TOOLS=explore,node,search,callers,callees,impact`，或直接用 CLI 等价命令（`codegraph query`/`callers`/`callees`/`impact`）。详见 SKILL.md「CodeGraph 工具面说明」。
>
> **Graphify 安装**（`graphifyy` 包，提供 `/graphify` Skill + CLI）：
> ```bash
> uv tool install graphifyy        # 或 pip install graphifyy
> graphify install                 # 在你的AI中注册该技能
> cd <your-project>                # 进入项目目录
> /graphify openspec               # 对openspec目录构建文档知识图谱 → graphify-out/graph.json
> graphify query "<问题>"          # 语义检索历史规格；path/explain/update/merge-graphs 见 --help
> ```
> **无需任何外部 API key**：`/graphify` 默认用 Claude Code 子代理做语义抽取，仅当设了 `GEMINI_API_KEY`/`GOOGLE_API_KEY` 才改走 Gemini API。增量合并/清理用 Python API `graphify.build.build_merge(new_chunks, graph_path, prune_sources=...)`。详见 SKILL.md「Graphify 工具面说明」。

## 安装

将 HyperSpec skill clone或下载到本地：

```bash
# 克隆仓库
git clone https://github.com/wind7rui/HyperSpec hyperspec
```

**Claude Code**安装：

```bash
cp -r hyperspec ~/.claude/skills/hyperspec
```

**Cursor**安装：

```bash
cp -r hyperspec .cursor/skills/hyperspec
```

**Codex CLI**安装：

```bash
cp -r hyperspec ~/.codex/skills/hyperspec
```

## 使用方式

在 Claude Code / Cursor / Codex 对话中输入：

```
用hyperspec开发一个用户认证功能
```

或更自然地表达：

```
规格驱动开发：给订单模块加上导出Excel功能
完整流程开发一个定时任务，每天凌晨同步数据
```

skill 会自动检测项目状态，判断应进入哪个阶段。你也可以显式指定阶段：

- `先做规格` → 强制进入 propose 阶段
- `直接开始实现` → 跳到 apply 阶段（需已有实现计划）
- `归档收尾` → 进入 archive 阶段

## 编排协议

HyperSpec 是**轻量编排框架**，以编排原生 skill 为主，核心职责如下（前四件为核心编排，第五件为可选叠加）：

1. **项目感知** — 自动探测语言/框架/构建工具，生成 `project_profile` 驱动后续阶段的自适应行为
2. **状态检测** — 通过结构化状态文件（`.hyperspec-state.yaml`）+ 实际文件验证确定当前阶段和断点位置
3. **阶段路由** — 加载对应 prompt 文件，按其中的流程调用原生 skill
4. **Commit 纪律** — 每个 task/fix 完成后自动 commit，编译前置，不做 push
5. **知识感知（可选）** — 叠加 CodeGraph（代码结构）/Graphify（文档知识）做结构化知识检索，不可用时回退 grep/Read

HyperSpec **不做**：

- 不手动创建 openspec artifacts（由 `openspec-propose` 负责）
- 不手动转 tasks → plan（由 `superpowers:writing-plans` 负责）
- 不手动执行归档操作（由 `openspec-archive-change` 负责）
- 不重申 TDD 规则（由 `superpowers:subagent-driven-development` 负责）
- 不重申审查规则（由 `superpowers:requesting-code-review` 负责）

## 工作流概览

HyperSpec 将一次完整的开发周期分为三个阶段，每个阶段委托给原生 skill 执行：

```
+==================+     +================+     +================+
| propose（规格）   | --> | apply（实现）    | --> | archive（归档） |
| 项目分析          |     | TDD 实现        |     | 一致性验证       |
| 需求确认          |     | verification   |     | archive-change  |
| openspec-propose |     | code-review    |     | specs 合并      |
| writing-plans    |     | 禁止改规格       |     |                |
| 禁止写代码         |     |                |     |                |
+==================+     +================+     +================+
```

**各阶段委托的原生 Skill：**

| 阶段 | 委托 Skill | 职责 |
|------|-----------|------|
| propose | `openspec-propose` | 通过 CLI 创建变更目录 + 生成所有 artifacts |
| propose | `superpowers:writing-plans` | 读取 openspec artifacts，生成实现计划 |
| apply | `superpowers:subagent-driven-development` 或 inline | 按计划执行实现 |
| apply | `superpowers:verification-before-completion` | 全量验证 |
| apply | `superpowers:requesting-code-review` | 全局代码审查 |
| archive | `openspec-archive-change` | 通过 CLI 归档变更 |

**各阶段产出：**

| 阶段 | 产出 |
|------|------|
| propose | `proposal.md` / `design.md` / `specs/` / `tasks.md` + `superpowers/plans/` 下的实现计划 |
| apply | 可执行代码 / 编译通过 / 审查通过 |
| archive | 归档记录 / specs 合并到主规格库 |

## 三阶段详解

### propose 阶段 — 把模糊想法变成可执行任务

将用户需求从模糊描述转化为完整的规格文档和实现计划。**本阶段禁止写任何代码或创建分支。**

**步骤：**

1. **项目分析** — 自动探测语言、框架、构建工具、测试框架，生成 project_profile
2. **需求确认** — 与用户交互确认需求（HyperSpec 自身逻辑，不委托）
3. **调用 openspec-propose** — 通过 CLI 创建变更目录，按依赖顺序生成 proposal → design → specs → tasks
4. **调用 writing-plans** — 读取 openspec artifacts + project_profile，生成适配技术栈的实现计划
5. **用户确认** — 展示产出摘要，请用户确认进入 apply

**产出文件：**

| 文件 | 生成者 | 内容 |
|------|--------|------|
| `proposal.md` | openspec-propose | 变更提案 — 背景、目标、影响范围 |
| `design.md` | openspec-propose | 技术方案 — 架构、选型、决策 |
| `specs/` | openspec-propose | 规格增量 — ADDED/MODIFIED/REMOVED |
| `tasks.md` | openspec-propose | 任务清单 — 按依赖排序 |
| `superpowers/plans/*.md` | writing-plans | 实现计划 — 带 checkbox 的微步骤 |

### apply 阶段 — 用工程纪律实现规格

按 propose 阶段生成的实现计划执行开发。

**智能执行模式选择：**

HyperSpec 根据多因子分析选择最优执行模式：

| 因子 | 完整模式倾向 | 轻量模式倾向 |
|------|-------------|-------------|
| 任务数量 | ≥ 6 | ≤ 5 |
| 跨模块性 | ≥ 3 个模块 | 1-2 个模块 |
| 项目结构 | monorepo | single-module |

| 模式 | 方式 |
|------|------|
| 完整模式 | 调用 `subagent-driven-development`，子代理实现+审查 |
| 轻量模式 | 当前会话直接执行 |

**提交纪律：** 每个 task 完成后：编译检查 → 更新计划 checkbox → 更新状态文件 → commit。全程不做 push。

**流程：**
1. 执行实现（逐 Task 或子代理派发）
2. 调用 `verification-before-completion` 全量验证
3. 调用 `requesting-code-review` 全局审查
4. 修复审查问题后重新验证，循环直到通过

**硬门：** 本阶段禁止修改 `openspec/changes/` 下的规格文档。

### archive 阶段 — 验证一致、归档收尾

验证代码实现和规格文档的一致性，归档变更，完成开发周期。

**步骤：**

1. **规格一致性验证** — 逐项检查 design/specs/tasks 是否在代码中体现，生成验证清单
2. **处理不一致** — 改代码或改规格，重新验证直到通过
3. **调用 openspec-archive-change** — 通过 CLI 归档变更（含 artifact 完成、task 完成检查）
4. **分支收尾 + 总结** — 提交剩余文件，展示变更摘要

## 状态管理与断点恢复

### 结构化状态文件

HyperSpec 使用 `.hyperspec-state.yaml` 跟踪当前进度：

```yaml
version: 1
active_change: add-user-auth
phase: apply
checkpoint: task-3-complete
project_profile:
  languages: [java]
  frameworks: [spring-boot]
  build_tool: maven
  compile_command: mvn compile -q
  test_command: mvn test
  structure: single-module
  has_ci: true
  knowledge_graph:                   # 知识图谱可用性（可选增强；两态探测）
    codegraph_available: true        # = indexed && tool_reachable（计算字段）
    graphify_available: true         # = indexed && tool_reachable
    codegraph_indexed: true          # 项目属性：.codegraph/ 已生成
    codegraph_tool_reachable: true   # 会话属性：codegraph_explore MCP 真正可达（会失效，见运行时可达性规则）
    graphify_indexed: true           # 项目属性：graphify-out/ 已生成
    graphify_tool_reachable: true    # 会话属性：graphify CLI 可调
```

**安全策略**：状态文件用于快速路由，但在关键节点验证实际文件状态。两者冲突时以实际文件为准。

### 自动状态检测

重新运行 `/hyperspec` 时，skill 会自动检查状态文件和实际项目文件状态：

| 项目状态 | 进入阶段 |
|----------|----------|
| 无状态文件 + 无活跃变更 | propose 阶段（首次运行） |
| 有活跃变更但无计划文件 | propose 阶段（补生成计划） |
| 有计划文件但无 checkbox（plan 不完整） | propose 阶段（回到计划生成） |
| 有计划文件但未开始（无已勾选 checkbox） | apply 阶段（全新执行） |
| 有计划文件且部分 checkbox 勾选 | apply 阶段（断点恢复） |
| 有计划文件且全部 checkbox 勾选 | apply 阶段（验证→审查→自动进入 archive） |
| 有多个活跃变更 | 让用户选择 |

### 各阶段断点恢复

- **propose 阶段：** 通过 checkpoint 精确恢复到需求确认、openspec 生成、计划生成等具体步骤
- **apply 阶段：** 通过 checkpoint 精确恢复到具体 task，状态文件和 checkbox 双重验证
- **archive 阶段：** 通过 checkpoint 恢复到验证、归档、分支收尾等具体步骤

## 项目分析器

HyperSpec 首次运行时自动探测项目特征：

| 检测项 | 检测方式 | 影响 |
|--------|---------|------|
| 语言 | 源文件扩展名统计 | plan 生成、build 命令 |
| 框架 | 依赖配置文件 | spec 设计方案风格 |
| 构建工具 | 根目录配置文件名 | 编译/测试命令自动选择 |
| 测试框架 | 测试目录结构 + 依赖 | TDD 步骤中的具体工具 |
| 项目结构 | 子目录模式 | 执行模式选择 |
| CI 配置 | CI 配置文件是否存在 | 验证策略 |

**支持的构建工具自动检测：**

| 配置文件 | compile_command | test_command |
|----------|----------------|--------------|
| `pom.xml` | `mvn compile -q` | `mvn test` |
| `build.gradle` | `./gradlew compileJava` | `./gradlew test` |
| `package.json` | `npm run build` | `npm test` |
| `go.mod` | `go build ./...` | `go test ./...` |
| `Cargo.toml` | `cargo build` | `cargo test` |
| `pyproject.toml` | 跳过 | `pytest` |

## 产出的目录结构

一次完整的 HyperSpec 运行后，项目目录结构如下：

```
项目根目录/
├── .hyperspec-state.yaml           # 运行期间存在，完成后删除
├── .codegraph/                     # 知识图谱：CodeGraph 代码结构层（可选，gitignore 不提交）
├── graphify-out/                   # 知识图谱：Graphify 文档知识层（可选，gitignore 不提交）
├── openspec/
│   ├── specs/                      # 主规格库（archive阶段合并）
│   │   └── user-auth/
│   │       └── spec.md
│   └── changes/
│       └── archive/                # 已归档变更
│           └── 2026-05-14-add-user-auth/
│               ├── .openspec.yaml
│               ├── proposal.md
│               ├── design.md
│               ├── tasks.md
│               └── specs/
│                   └── user-auth/
│                       └── spec.md
└── superpowers/
    └── plans/
        └── 2026-05-14-add-user-auth.md  # 实现计划（带checkbox）
```

## 设计原则

- **轻量编排框架：** 以编排 OpenSpec/Superpowers 为主（项目感知、状态检测、阶段路由、commit 纪律），不重写原生 skill 功能；另含自有的规格一致性验证、计划质量审查等显式增强
- **规格与实现分离：** propose 阶段只产出文档，apply 阶段只写代码，各自有硬门禁止越界
- **项目感知自适应：** 根据项目技术栈自动调整编译命令、测试策略、执行模式
- **每个阶段有明确出口条件：** 不满足出口条件就不能进入下一阶段
- **可中断、可恢复：** 结构化状态文件 + 实际文件双重验证，支持从任何断点精确恢复
- **用户意图优先：** 自动检测只是默认行为，用户显式指定阶段时以用户意图为准
- **实际文件为 ground truth：** 状态文件是缓存，实际文件状态是权威，冲突时以实际文件为准
- **知识图谱是可选增强：** CodeGraph/Graphify 作为透明叠加的知识感知层，所有知识查询都有 grep/Read 兜底，核心流程不依赖知识图谱，不可用时优雅降级

## 常见问题

### 可以跳过某个阶段吗？

可以。用显式指令指定阶段，如「直接开始实现」。但前置条件必须满足（比如 apply 阶段需要有实现计划），否则 skill 会提示你先完成前置阶段。

### 如果实现过程中发现规格设计有问题怎么办？

apply 阶段的硬门禁止修改规格文档。你可以记录问题继续实现，等进入 archive 阶段后统一处理不一致。如果问题严重影响实现，可以主动回到 propose 阶段重新设计。

### 支持哪些编程语言和项目类型？

不限制语言和项目类型。HyperSpec 会自动探测项目技术栈并自适应调整编译/测试命令和执行策略。Java/Maven、Java/Gradle、Node.js、Go、Rust、Python 等主流技术栈都有内置支持。
