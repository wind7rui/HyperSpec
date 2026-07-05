# HyperSpec:apply — 构建阶段

## 目标

用 Superpowers 的工程纪律执行 propose 阶段生成的实现计划。本阶段是编排层，实际执行和审查由原生 skill 完成。

## 前置检查

1. 确认 `superpowers/plans/` 下有计划文件（`YYYY-MM-DD-<变更名>.md`）
2. 如果没有计划文件但有 `openspec/changes/` 下的活跃变更 → 提示用户：`回到 /hyperspec 进入 propose 阶段完成计划生成`
3. 读取 `.hyperspec-state.yaml` 的 `project_profile`，获取 `compile_command` 和 `test_command`

## 环境检查

在开始实现前，验证编译环境是否可用：

1. **验证编译**：运行 `compile_command`（来自 project_profile）
   - 如果 compile_command 为 null（如纯 Python/前端项目），跳过编译检查
   - 如果编译失败且原因是环境问题（如 JDK 版本不匹配、Node 版本不匹配），**立即阻塞并报告**，等待用户确认
   - 如果编译失败是代码问题，正常进入实现阶段（TDD 会逐步修复）

## 智能执行模式选择

根据多因子分析选择最优执行模式，不再使用简单的任务数量阈值：

| 因子 | 权重 | 检测方式 |
|------|------|---------|
| 任务数量 | 高 | 读取计划文件 checkbox 数量 |
| 任务独立性 | 高 | 分析 tasks.md 中任务间的依赖关系 |
| 跨模块性 | 中 | 分析计划文件中 File Structure 涉及的模块/目录数量 |
| 项目结构 | 中 | `project_profile.structure`：monorepo 倾向完整模式 |
| 变更复杂度 | 低 | 预估涉及修改的文件数量 |

**决策规则：**

- **完整模式**（subagent-driven-development）：满足以下任一条件
  - 任务 ≥ 6
  - 涉及 ≥ 3 个不同模块/目录的修改
  - monorepo 结构 + 任务 ≥ 3
  - 任务间无依赖关系（独立性高）且任务 ≥ 4（适合并行）
- **轻量模式**（inline 执行）：满足以下任一条件
  - 任务 ≤ 5 且集中在 1-2 个模块
  - 单模块小功能
  - 任务间有强依赖关系（需要顺序执行）
  - **独立性否决**：即使任务 ≥ 6 或跨 ≥ 3 模块，如果任务间存在强编译依赖（如 Java 接口+实现必须同时编译），优先使用轻量模式

## 阶段流程

**重要：每个 task/fix 完成后自动 commit。** 遵循"一个 task/fix 一个 commit"原则，commit 前必须执行编译检查且编译通过。全程不做自动 push 操作。

**断点检测（阶段流程入口）：**

从 `.hyperspec-state.yaml` 读取 checkpoint，然后验证实际状态：

| checkpoint | 实际状态验证 | 路由到 |
|-----------|-------------|--------|
| `plan-generated-and-confirmed` | 计划文件有 checkbox 且全部未勾选 | Step 1（全新执行） |
| `plan-generated-and-confirmed` | 计划文件有 checkbox 且部分已勾选 | Step 1（断点恢复，从第一个未勾选继续） |
| `task-N-complete` | 计划文件中 task N 的 checkbox 确实已勾选 | Step 1（从 task N+1 继续） |
| `task-N-complete` | 计划文件中 task N 的 checkbox 未勾选（状态文件过期） | Step 1（回退到上一个确认一致的 task） |
| `verified` | 所有 checkbox 勾选 + 测试通过 | Step 3（审查） |
| `verified` | 测试未通过 | Step 1（从失败点修复，重新走提交流程） |
| `reviewed` | 所有 checkbox 勾选 + 审查通过 | Step 4（最终确认） |
| `reviewed` | 审查仍有 Critical 问题未修复 | Step 1（修复审查问题） |

### 1. 执行实现

**完整模式**：
调用 Superpowers 的 `subagent-driven-development` skill 执行实现：
- 使用 `superpowers/plans/` 下的计划文件作为输入
- 每个任务派发给独立的实现者子代理
- 每个任务完成后派发审查者子代理做独立审查
- **子代理只负责写代码和 commit**，**主会话负责进度跟踪**：每个子代理返回后，主会话用 Edit 工具将计划文件中该 Task 的所有 `- [ ]` 改为 `- [x]`，并更新 `.hyperspec-state.yaml` 的 checkpoint 为 `task-N-complete`
- 如果跳过了 worktree：subagent-driven-development 中 `using-git-worktrees` 的 REQUIRED 可忽略

**轻量模式**：
在当前会话中直接执行：
- 按 `superpowers/plans/` 下的计划文件逐个 Task 执行
- 严格遵循计划文件中每个 Step 的 checkbox 顺序
- 每个 Task 完成后执行收尾流程

**执行前知识感知（可选增强，两种模式通用）：**
每个 Task 开始实现前，若 `project_profile.knowledge_graph.codegraph_available == true`，通过 CodeGraph 获取精确代码上下文：
1. `codegraph_explore("目标文件/模块")` → 理解当前代码结构和目标位置
2. `codegraph_impact("要修改的方法/符号")` → 评估本次 Task 的影响范围，将影响面信息作为实现约束纳入执行
3. `codegraph_callers`/`codegraph_callees("要修改的方法")` → 了解所有调用者和方法行为，避免遗漏调用点

轻量模式额外用 `codegraph_explore` + `codegraph_impact` 做精确代码导航，替代逐文件 Read。

**知识图谱不可用时回退：** 使用现有 grep/Read 逐文件理解代码上下文。**运行时工具不可达（调用返回 *Unknown tool*/报错，如装了 CLI 但 MCP 未注入）同样触发回退**，并把 `codegraph_tool_reachable` 置 `false`——见 SKILL「运行时可达性规则」。

> **CodeGraph 索引必须手动同步。** 实测在 MCP stdio 部署下，CodeGraph 的文件观察器**不会运行**——apply 修改代码后，索引仍是旧状态（`codegraph query` 查不到新符号，`codegraph_impact`/`callers` 返回过期调用链）。因此：**每个 Task 完成后、每个 fix 后、以及进入 archive 交叉验证前，都必须运行 CLI `codegraph sync`** 刷新索引。这是 apply 阶段的强制步骤，不是可选项（之前文档所称"默认自动增量同步、通常无需手动 sync"在 MCP stdio 下不成立）。`codegraph_explore` 是默认可用的 MCP 工具（内联 impact/callers/callees 结果），其余工具需 `CODEGRAPH_MCP_TOOLS` 启用或用 CLI 等价命令，详见 SKILL.md「CodeGraph 工具面说明」。

**每个任务完成后的提交流程：**

无论哪种执行模式，每个 task 完成后必须执行以下流程：

1. **编译检查**：运行 `compile_command`，编译必须通过（如果 compile_command 为 null 则跳过）
   - 若 `codegraph_available == true`：同步运行 CLI `codegraph sync` 刷新 CodeGraph 索引（MCP stdio 下观察器不运行，apply 改动不会自动入索引；不同步则后续 impact/callers 查询过期）
2. **更新 checkbox**：用 Edit 工具将计划文件中**该 Task 下的所有** `- [ ]` 改为 `- [x]`（包括 Verify、Commit 等非实现步骤，不能遗漏）
3. **更新状态文件**：更新 `.hyperspec-state.yaml` 的 `checkpoint: task-N-complete`
4. **自动 commit**：将代码改动 + 计划文件变更 + 状态文件变更一起提交到本地仓库
   - commit message 格式：`<类型>(<范围>): <task描述>`
   - 一个 task 对应一个 commit（包含计划文件和状态文件更新）
   - **全程不做 push 操作**

**原子性保证**：先更新 checkbox 和状态文件再 commit，确保三者一致。如果 commit 前中断，重跑时该 Task 会被重新执行（但代码已写入，TDD 测试会直接通过）；如果 commit 后中断，checkpoint 和 checkbox 已更新，重跑时会跳过该 Task。

### 2. 验证完成

调用 Superpowers 的 `verification-before-completion` skill：
- 运行 `test_command`（来自 project_profile），如果 test_command 为 null 则跳过自动化测试
- 确认结果符合预期

验证通过后更新 `.hyperspec-state.yaml`：`checkpoint: verified`。

### 3. 全局审查

全部任务完成后，调用 Superpowers 的 `requesting-code-review` skill 做一轮全局代码审查。

如果审查发现 Critical 或 Important 问题，逐项修复。每个 fix 的提交流程：
1. **（如 `codegraph_available == true`）fix 前 sync**：运行 CLI `codegraph sync` 刷新索引——**必须在下面的 impact 之前**，确保影响面判断基于含本 fix 之前全部改动的最新索引。否则 fix-2 会在 fix-1 改动前、尚未 sync 的脏索引上做决策，使"验证调用链完整性"给出**虚假通过**。
2. 编译检查 → 必须通过
3. 自动 commit → 格式：`fix(<范围>): <fix描述>`

**修复前影响面分析（如果 `project_profile.knowledge_graph.codegraph_available == true`）：**
- 每个 fix 前：**先 `codegraph sync`**，再 `codegraph_impact` 评估修复的影响范围（sync 必须在 impact 前，否则 impact 基于过期索引）
- 所有 fix 后：`codegraph_explore` 验证调用链完整性（若验证前刚 commit 了改动，同样先 sync 再 explore）

**知识图谱不可用时回退：** 按现有方式逐项修复（凭经验判断影响面）。

修复完成后回到 Step 2 重新验证，再重新审查，直到审查通过。

**重试上限**：如果全局审查循环超过 3 次仍有 Critical 问题，提示用户：「审查反复不通过，可能需要回到 propose 阶段重新审视设计方案。」让用户决定是否继续。

审查通过后更新 `.hyperspec-state.yaml`：`checkpoint: reviewed`。

### 4. 最终确认

Step 2 验证通过 + Step 3 审查通过后：

1. **同步 tasks.md 状态**：将 `openspec/changes/<变更名>/tasks.md` 中所有任务的完成状态标记为已完成（与 superpowers plan 的 checkbox 状态对齐）
2. 向用户展示变更摘要（改动文件列表、commit 列表、验证结果、审查结论）

此步骤是 apply 阶段的最终出口，完成后更新 `.hyperspec-state.yaml`：`phase: archive`、`checkpoint: apply-done`，然后自动进入 archive 阶段。

## 出口条件

- `superpowers/plans/` 下的计划文件中所有 checkbox 已勾选
- 全量编译通过（或 compile_command 为 null）
- 测试全部通过（或 test_command 为 null）
- 代码审查无 Critical 级别问题
- `openspec/changes/<变更名>/tasks.md` 中所有任务标记为已完成（在 Step 4 统一同步）

## 断点恢复

1. **计划文件完整性**：如果计划文件存在但不完整（缺少头部或 checkbox），提示用户回到 propose 阶段重新生成
2. **状态文件过期**：如果 checkpoint 声称某个 task 完成，但实际 checkbox 未勾选，回退到上一个确认一致的 checkpoint
3. **环境检查**：断点恢复时跳过环境检查（已在首次运行时验证），直接进入执行

## 审查循环

如果审查者打回某个任务：
1. 实现者根据反馈修改
2. 重新提交审查
3. 循环直到通过

如果同一任务修改 3 次以上仍不通过，提示用户：「这个任务反复审查不通过，可能需要回到 propose 阶段重新审视设计方案。」

## 硬门

本阶段**禁止修改规格设计文档**（openspec/changes/ 下的 proposal.md、design.md、specs/）。如果实现过程中发现规格有问题，记录下来留到 archive 阶段处理。

**例外：** `tasks.md` 中的任务完成状态可在 Step 4 同步更新（仅限状态标记，不修改任务内容）。
