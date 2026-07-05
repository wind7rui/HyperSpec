# 更新日志

本项目遵循 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)，版本号遵循 [语义化版本](https://semver.org/lang/zh-CN/)。

## [v1.1.0] — 2026-07-01

### Added

- **知识图谱感知层（可选增强）**：在三阶段工作流上叠加两层知识图谱做结构化检索，替代原有扁平 grep/Read。未安装或不可用时各阶段自动回退 grep/Read，核心流程不依赖知识图谱，v1.0 行为不受影响。
  - **CodeGraph（代码结构层）**：基于 AST 静态解析，提供符号 / 调用链 / 依赖图 / 影响面（blast radius），零 LLM 依赖；覆盖 grep 追不到的跨模块调用与 `@Component` 等注解驱动的运行时分派。
  - **Graphify（文档知识层）**：对 `openspec/` 构建语义图谱，检索历史规格与归档经验（上次类似需求怎么设计、某接口踩过什么坑）。
- **三阶段知识图谱集成点**：
  - **propose**：① 规格生成前用 Graphify 按语义检索历史规格当设计参考（`query` / `explain` / `path`，支持词表扩展提升中文小语料相关性）② 规格生成后增量合并进文档图谱（`build_merge`）③ API 验证用 CodeGraph 一次拿全调用方与影响面，替代 grep + 多次 Read（优先单次 `codegraph_explore`，控制在 1–2 次调用）④ KG 上下文传入实现计划
  - **apply**：① 每个 task 实现前用 `codegraph_impact` 评估影响范围作为实现约束 ② 修复后 `codegraph_explore` 验证调用链完整性
  - **archive**：① 基于代码结构做交叉验证，用 `search` / `callers` / `callees` / `explore` 核验 design.md 提到的符号与调用链是否与代码一致，替代逐文件 Read ② 归档后触发 Graphify 增量更新（`build_merge` 增量并入 + `prune_sources` 按 `source_file` 精确清理过期节点）
- **可用性两态探测**：将"可用性"拆为 `indexed`（项目属性，目录/索引在不在）与 `tool_reachable`（会话属性，工具是否注入当前会话、会随时失效），`available = 两者皆真`。纠正"用目录在推断工具通"的范畴错误——目录在是项目属性，工具是否注入当前会话是会话属性，装了 CLI 没配 MCP 时目录在、工具却不可达。
- **运行时可达性规则**：任一 KG 工具调用若返回 *Unknown tool* / 工具不存在 / 连接错误，立即判本次不可达、回退 grep/Read 完成当前步骤、并把对应 `tool_reachable` 置 `false`。堵死"目录在但 MCP 未注入 → 知识图谱层静默失败、流程却假通过"的口子。
- **强制 sync**：MCP stdio 部署下 CodeGraph 文件观察器实测不运行（"自动增量同步"不生效），apply 每个 task/fix 后及进入 archive 交叉验证前必须手动 `codegraph sync`，否则 explore/impact 返回过期调用链。每个 fix 前**先 sync 再 impact**，确保影响面基于含本次改动的最新索引。
- **状态文件 `knowledge_graph` 块**：新增 6 字段（`indexed` / `tool_reachable` × CodeGraph/Graphify + 2 个 `available` 计算字段）。断点恢复新增知识图谱一致性校验：`*_indexed == true` 但对应目录不存在则修正标记为 `false`（仅修正可用性标记，不回退 checkpoint——知识图谱是可选增强，不构成阶段节点）。
- **工具面说明**（SKILL.md「CodeGraph / Graphify 工具面说明」）：
  - CodeGraph MCP 服务默认只暴露 `codegraph_explore` 一个工具（单次调用内联 search + callers + callees + impact + 源码）；`search` / `callers` / `callees` / `impact` / `node` 需设 `CODEGRAPH_MCP_TOOLS` 启用或用 CLI 等价命令；索引/同步是 CLI 命令。
  - Graphify 构建通过 `/graphify` Skill（非 MCP 工具），默认用 Claude Code 子代理做语义抽取、不读取任何外部 LLM key；查询/维护用 CLI，增量合并/清理用 Python API `build_merge`。

### Changed

- 无。

### Fixed

- 无。

### Notes

- **兼容性**：向后兼容。未安装 CodeGraph / Graphify 时 HyperSpec 检测到不可用即标记 `available: false` 并回退 grep/Read，行为与 v1.0 一致。安装方式不变（见 README「安装」）。`.codegraph/` 与 `graphify-out/` 已 gitignore，不进版本库，新克隆环境需重新索引。
- **已知限制**：
  - MCP stdio 部署下须手动 `codegraph sync`（文件观察器不运行）。
  - MCP 默认只暴露 `codegraph_explore`，其余工具需环境变量或 CLI。
  - greenfield 首次运行（openspec 为空）时 Graphify 无法构建，推迟到 propose 规格生成后首次构建。
  - 本次集成仅经功能验证（三阶段集成点实跑通过、招牌能力成立），未做基准测试；日常简单需求的投入产出比暂无数据。

---

## [v1.0.0] — 2026-05-17

HyperSpec 首次发布。规格驱动 + 工程纪律的完整开发工作流 Skill，协调 OpenSpec（规格管理）与 Superpowers（TDD + 子代理审查），三阶段（propose / apply / archive）从需求到实现到归档一条流程走完。

### Added

- 项目感知（自动探测语言/框架/构建工具/测试框架）、状态检测、阶段路由、commit 纪律
- 结构化状态文件（`.hyperspec-state.yaml`）+ 实际文件双重验证的断点恢复
- 智能执行模式选择（完整模式 / 轻量模式）、多语言构建工具自动适配（Maven / Gradle / npm / Go / Cargo / pytest）
- 自有的规格一致性验证、计划质量审查
