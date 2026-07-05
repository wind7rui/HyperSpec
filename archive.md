# HyperSpec:archive — 收尾阶段

## 目标

验证代码实现和规格文档的一致性，通过原生 skill 归档变更，完成整个开发周期。

## 前置检查

确认 apply 阶段已完成：

- `superpowers/plans/` 下的计划文件所有步骤已勾选
- 测试全部通过（或 test_command 为 null）
- 代码审查无 Critical 问题
- `.hyperspec-state.yaml` 的 checkpoint 为 `apply-done`、`consistency-verified`、`archived` 或 `done` 之一

如果前置条件不满足，提示：「apply 阶段尚未完成，请先运行 /hyperspec」

## 阶段流程

### 1. 规格一致性验证

逐项检查代码实现是否和规格文档一致。这是 HyperSpec 特有的验证环节，确保实现没有偏离设计。

**做法：**
- 读取 `openspec/changes/<变更名>/design.md`，逐一确认其中定义的技术方案是否在代码中体现
- 读取 `openspec/changes/<变更名>/specs/` 下的规格增量文件，确认每个 ADDED/MODIFIED/REMOVED 标记对应的代码变更确实存在
- 读取 `openspec/changes/<变更名>/tasks.md`，确认所有任务已完成
- 将验证结果整理为一份清单，逐项标注「通过」或「不一致」

**基于代码结构的交叉验证（如果 `project_profile.knowledge_graph.codegraph_available == true`）：**
逐项验证每个技术方案点时，用 CodeGraph 做精确的交叉引用验证，替代/补充逐文件 Read：
- `codegraph_search` 搜索 design.md 中提到的代码符号，定位其实现位置
- `codegraph_callers`/`codegraph_callees` 验证调用链是否符合设计方案（如设计要求 A 调用 B，验证代码中确实存在该调用关系）
- `codegraph_explore` 验证两个关键符号之间的调用路径（含跨模块/动态分派跳转）是否符合设计预期
- 每项标注「通过」（调用链符合设计）或「不一致」（调用链缺失或偏离设计）

> CodeGraph MCP 默认只暴露 `codegraph_explore`（内联 search/callers/callees/impact 结果与调用路径）；`search`/`callers`/`callees` 需 `CODEGRAPH_MCP_TOOLS` 启用或用 CLI 等价命令，未启用时统一用 `codegraph_explore`。详见 SKILL.md「CodeGraph 工具面说明」。

**知识图谱不可用时回退：** 逐文件 Read 对比设计文档和代码（现有方式）。运行时工具不可达（Unknown tool/报错）同样回退，见 SKILL「运行时可达性规则」。

验证维度：
- design.md 中定义的技术方案是否全部实现
- specs/ 中标记的 ADDED/MODIFIED/REMOVED 是否在代码中体现
- tasks.md 中的任务是否全部完成

**持久化验证状态**：验证完成后，在 `openspec/changes/<变更名>/` 下创建 `.close-verification-done` 文件（内容为验证结果清单），用于断点恢复时判断是否需要重跑验证。

完成后更新 `.hyperspec-state.yaml`：`checkpoint: consistency-verified`。

### 2. 处理不一致

如果验证发现不一致，逐项列出并让用户选择：

**选项 A：改代码**
- 标记哪些代码需要修改
- 询问用户是否在本阶段直接修复（简单修复）或回到 apply 阶段（复杂修复）
- 如果用户选择回到 apply 阶段：更新 `.hyperspec-state.yaml` 为 `phase: apply, checkpoint: reviewed`，然后结束 archive 阶段

**选项 B：改规格**
- 标记哪些规格文档需要更新
- 更新 `openspec/changes/` 下的对应文件
- 重新走验证流程

**无论选择哪种修复方式，修复完成后必须：**
1. 删除 `.close-verification-done` 文件（使验证状态失效）
2. 更新 `.hyperspec-state.yaml`：`checkpoint: apply-done`（回退到验证前）
3. 重新执行 Step 1 验证
4. 重新生成 `.close-verification-done`

验证结果全部通过后（或不一致项已修复并重新验证通过），进入 Step 3。

### 3. 调用 openspec-archive-change 归档

调用原生 `openspec-archive-change` skill，通过 CLI 执行归档。

**做法：**

1. 宣布："调用 openspec-archive-change 归档变更"
2. 使用 Skill 工具调用 `openspec-archive-change`，args 格式：`<变更名>`
3. `openspec-archive-change` 会自动执行：
   - 检查 artifact 完成状态（通过 CLI）
   - 检查 task 完成状态（读 tasks.md）
   - 评估 spec sync 状态（对比 delta specs 和主规格库）
   - 执行归档：`mv openspec/changes/<name> openspec/changes/archive/YYYY-MM-DD-<name>`
   - 展示归档摘要
4. 等待 skill 完成后，确认归档成功：检查 `openspec/changes/archive/` 下是否有对应目录，且原活跃变更目录已移除

**注意：** `openspec-archive-change` 内部已包含用户确认环节（它用 AskUserQuestion 确认是否继续），不需要 HyperSpec 重复确认。但 Step 1 的验证结果应在调用 archive 前展示给用户。

**降级方案：** 如果 `openspec-archive-change` skill 不可用（Skill 工具返回 "Unknown skill"），**或 `openspec archive` CLI 卡在交互式确认（Y/n）、或因 spec 未用 `## ADDED/MODIFIED Requirements` delta 头被判"无 delta"无法自动合并规格**，手动执行归档操作：
1. 确认 `openspec/changes/<变更名>/tasks.md` 中所有任务已标记为完成
2. 运行 `mkdir -p openspec/changes/archive` 确保归档目录存在
3. 运行 `mv openspec/changes/<变更名> openspec/changes/archive/$(date +%Y-%m-%d)-<变更名>` 执行归档
4. 确认归档成功：活跃变更目录已移除，archive 目录已创建

> **注**：openspec 1.3.1 的 `archive` 是交互式的，并要求 spec 用 `## ADDED/MODIFIED/REMOVED Requirements` delta 头（而非 `### Requirement:`）；不符会提示"无 delta"（非阻塞警告）。若规格未自动合并到 `openspec/specs/`，归档仍可经手动 `mv` 完成（知识图谱仅依赖归档目录的文件，不依赖 specs 合并）。

完成后更新 `.hyperspec-state.yaml`：`checkpoint: archived`。

**归档后知识更新（如果 `project_profile.knowledge_graph.graphify_available == true`）：**
归档完成、变更已移入 `archive/` 后，触发 Graphify 增量更新，把本次变更沉淀到文档知识图谱，供后续变更语义检索复用：
1. `graphify.build.build_merge([归档规格的抽取片段], graph_path)` — 将归档的新规格知识**只增不减**地增量合并到图谱（也可先 `/graphify openspec --update` 重抽取变更文件再合并）
2. `build_merge(..., prune_sources=[<被移除的 source_file>])` — 按 `source_file` 精确清理活跃变更目录移除后产生的过期节点（缩图需 `to_json(..., force=True)` 确认；或 `graphify.build.prune_repo_from_graph` 按仓库标签清理）

失败不影响归档结果（归档已完成，知识更新是可选增强）。

**知识图谱不可用时回退：** 跳过知识更新。

### 4. 分支收尾 + 总结

**如果使用了 worktree：**
提示用户选择分支处理方式：

1. **合并到主分支** — 在 worktree 中合并，清理 worktree
2. **创建 PR** — 推送到远程，创建 Pull Request
3. **保持现状** — 保留分支和 worktree，稍后处理
4. **丢弃** — 放弃所有改动（需输入 "discard" 确认）

**如果未使用 worktree（直接在当前分支开发）：**
跳过分支处理，提示用户是否需要提交代码。

**总结报告：**
向用户报告本次变更的完整信息：
- 完成了哪些任务
- 涉及哪些文件的改动
- 规格更新了什么
- 归档位置

**清理状态文件：**
更新 `.hyperspec-state.yaml`：`checkpoint: done`。使用 `git rm .hyperspec-state.yaml` 将其从 git 追踪中移除，然后 commit（message: `chore: hyperspec <变更名> 完成`）。确保工作区干净。

## 出口条件

- 实现和规格一致（验证通过）
- 变更已归档（由 openspec-archive-change 完成）
- 如使用了 worktree：用户已选择分支处理方式且 worktree 已处理
- 如未使用 worktree：代码已提交或用户已确认稍后处理
- `.hyperspec-state.yaml` 已删除

## 断点恢复

archive 阶段可能通过两种方式进入：
1. apply 阶段完成后自动进入（正常路径）
2. 用户显式指定「归档收尾」进入（用户意图覆盖）

重新运行时，读取 `.hyperspec-state.yaml` 的 checkpoint 并验证：

| checkpoint | 实际状态验证 | 恢复到 |
|-----------|-------------|--------|
| `apply-done` | `.close-verification-done` 不存在 | Step 1（验证） |
| `consistency-verified` | `.close-verification-done` 存在且归档目录不存在 | Step 3（归档） |
| `consistency-verified` | `.close-verification-done` 不存在（被手动删除） | Step 1（重跑验证） |
| `apply-done` | `.close-verification-done` 存在（与 checkpoint 不一致，以文件为准） | 检查归档目录是否存在，存在则 Step 4，不存在则 Step 3 |
| `archived` | 归档目录存在但分支未处理 | Step 4（分支收尾） |
| `archived` | 归档目录不存在（归档中断） | Step 3（重做归档） |
| `archived` | 归档目录存在且分支已处理 | 展示变更摘要（同 Step 4 总结报告），然后清理状态文件完成 |

**异常状态检测：**
- 活跃变更目录已删除但 archive 目录不存在 → 归档过程中断，提示用户可尝试 `openspec list --json` 检查状态
- `.hyperspec-state.yaml` 存在但活跃变更目录不存在且未归档 → 可能是手动删除，提示用户确认状态

## 硬门

本阶段如果发现需要改代码，简单的修复可以直接做（不影响规格的 bug 修复）；但如果改动会影响规格定义，必须回到 apply 阶段。
