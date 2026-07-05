# HyperSpec:propose — 规格阶段

## 目标

把用户的需求从模糊想法变成可执行的任务清单。通过项目感知 + 调用原生 skill 产出完整的规格文档和实现计划。

## 阶段流程

### 1. 项目分析（仅首次运行）

如果 `.hyperspec-state.yaml` 不存在或 `project_profile` 为空，执行项目分析器：

1. **探测语言和框架**：扫描源码目录的文件扩展名 + 读取依赖配置文件
2. **识别构建工具**：检测根目录的 `pom.xml`、`build.gradle`、`package.json`、`go.mod`、`Cargo.toml`、`pyproject.toml` 等
3. **推导编译/测试命令**：根据构建工具映射（见 SKILL.md "构建命令映射"表）
4. **判断项目结构**：单模块 vs 多模块/monorepo
5. **检测 CI 配置**：是否有 `.github/workflows/`、`.gitlab-ci.yml`、`Jenkinsfile` 等

分析完成后将 `project_profile` 写入 `.hyperspec-state.yaml`，并更新 `phase: propose`、`checkpoint: profiler-done`。

**命令运行时验证**：推导出 compile_command 后，必须实际运行一次验证其可用性。具体规则见 SKILL.md "命令运行时验证"段落。如果验证失败且为环境问题，立即阻塞并报告，等待用户提供正确的命令后再继续。

**空白项目（greenfield）处理：**
如果是空项目（无源码、无配置），跳过分析，project_profile 保持默认空值。在需求确认步骤中一并确定技术栈，分析完成后补充 project_profile。

**知识图谱初始化（可选增强）：**
project_profile 写入后，检测并初始化知识图谱工具。任一失败都标记不可用并继续，不影响主流程：

1. **CodeGraph（代码结构层）**：检测项目根目录是否有 `.codegraph/`
   - 不存在 → 运行 CLI `codegraph init` 执行首次索引；成功则 `knowledge_graph.codegraph_available: true`、`codegraph_indexed: true`
   - 存在 → 已就绪，标记可用（**注意：CodeGraph 在 MCP stdio 下不自动同步，apply 阶段每个 task 后须手动 `codegraph sync`**）
   - 工具不可用/失败 → `codegraph_available: false`，后续回退 grep/Read
2. **Graphify（文档知识层）**：检测项目根目录是否有 `graphify-out/`
   - 不存在 → 先检查 `openspec/` 是否有规格文档：
     - 有规格 → 调用 `/graphify openspec` Skill 构建知识图谱（无需外部 API key，默认用 Claude Code 子代理做语义抽取）；成功则 `graphify_available: true`、`graphify_indexed: true`
     - **为空（greenfield，首次运行）→ 无法构建（无内容可索引），标记 `graphify_available: false`、`graphify_indexed: false`，推迟到 Step 3 规格生成后首次构建**
   - 存在 → 已就绪，标记可用
   - 工具不可用/失败 → `graphify_available: false`，后续回退文件读取
3. 将检测结果写回 `.hyperspec-state.yaml` 的 `project_profile.knowledge_graph`

> 详见 SKILL.md「知识图谱感知」段落。各后续步骤通过 `knowledge_graph.<tool>_available` 标记决定走 MCP 工具还是回退。

### 2. 需求确认

与用户交互确认需求。规则：

- 一次只问一个问题
- 优先使用选择题
- 关注：目的、约束、成功标准
- 从用户描述中提取变更名（英文短横线格式，如 `add-user-auth`）

**空白项目（greenfield）额外确认：**
如果 Step 1 中检测到是空项目（project_profile 为空），在此步骤中一并确认技术栈（语言、框架、构建工具）。确认后补充 `project_profile` 并更新 `.hyperspec-state.yaml`。

**如果用户提供了需求链接（如 JIRA 任务、GitHub Issue）：**
先用 web reader 或其他方式获取需求详情，将完整需求作为本次需求的输入。

**自主模式：**
如果用户声明了自主执行意图（如「完全自主」「你决定」「不用确认」），跳过交互式需求确认，直接根据用户最初的需求描述进入 Step 3。空白项目自主模式下根据需求描述推断技术栈。

需求确认完成后，更新 `.hyperspec-state.yaml`：`active_change: <变更名>`、`checkpoint: requirements-confirmed`。

### 3. 调用 openspec-propose 生成规格文档

调用原生 `openspec-propose` skill，通过 CLI 创建变更并生成所有 artifacts。

**历史规格检索（如果 `project_profile.knowledge_graph.graphify_available == true`）：**
规格生成前，先从文档知识图谱检索相关历史经验，作为设计参考上下文：
- 使用 `graphify query "<需求语义>"` 按语义遍历与本次需求相关的历史规格节点/关系（BFS/DFS，可 `--budget` 限 token）
- **词表扩展（提升相关性）**：`graphify query` 的起点是词项匹配，中文小语料下相关性受词项重叠影响——查询前先用同义词/英文术语扩展问题（如"慢操作"补"latency/timeout/耗时阈值"），必要时加 `--dfs` 深度遍历；社区说明用 `graphify explain` 通常比 `query` 更稳
- 使用 `graphify explain "<关键节点>"` 获取该节点及其邻居（聚类社区）的说明；或 `graphify path "A" "B"` 查两概念间最短路径
- 将检索到的历史设计经验整理为「历史规格参考」，在调用 openspec-propose 时作为上下文附加到描述中，让规格文档复用既有设计经验

**知识图谱不可用时回退：** 跳过历史检索，直接调用 openspec-propose（现有逻辑）。

**做法：**

1. 宣布："调用 openspec-propose 生成规格文档"
2. 使用 Skill 工具调用 `openspec-propose`，args 格式：
   ```
   Change name: <变更名>. Description: [项目: {project_profile.languages} + {project_profile.frameworks}, 构建: {project_profile.build_tool}] <需求描述>
   ```
   将 project_profile 信息作为上下文前缀附加到描述中，让 openspec-propose 能生成贴合项目技术栈的规格文档。
3. `openspec-propose` 会自动执行：
   - `openspec new change "<name>"` — 创建变更目录
   - `openspec status --change "<name>" --json` — 获取 artifact build order
   - 循环 `openspec instructions <artifact>` — 按依赖顺序生成每个 artifact
   - 产出：proposal.md、design.md、specs/、tasks.md
4. 等待 skill 完成后，验证 `openspec/changes/<变更名>/` 下所有 artifacts 已生成且**非空**（检查每个文件 size > 0）。如果某个 artifact 文件为空或不存在，重新调用 openspec-propose 补充缺失的 artifact。
5. **覆盖 openspec-propose 的执行建议**：openspec-propose 完成后会建议 "Run /opsx:apply"。在 HyperSpec 上下文中忽略此建议，宣布 "回到 HyperSpec 流程，继续生成实现计划"

**降级方案：** 如果 `openspec-propose` skill 不可用（Skill 工具返回 "Unknown skill"），手动执行等效操作：
1. 运行 `npx @fission-ai/openspec new change "<变更名>"` 创建变更目录
2. 运行 `npx @fission-ai/openspec status --change "<变更名>" --json` 获取 artifact build order
3. 对每个 artifact，运行 `npx @fission-ai/openspec instructions <artifact>` 获取生成指令
4. 按照指令内容，使用 Write 工具直接创建对应的 artifact 文件（proposal.md、design.md、specs/ 下的 .md 文件、tasks.md）
5. 确保每个 artifact 文件非空且内容完整

完成后更新 `.hyperspec-state.yaml`：`checkpoint: openspec-generated`。

**规格生成后的知识增量（Graphify）：**
- **greenfield 首次构建**：若 Step 1 因 openspec 为空标记 `graphify_indexed: false`（但 Graphify 工具已安装），此时 openspec 已有本次规格 → 调用 `/graphify openspec` 完成首次构建，置 `graphify_available: true`、`graphify_indexed: true`
- **常规增量合并**：若 `graphify_available == true` 且已建图 → 调用 `graphify.build.build_merge([本次规格的抽取片段], graph_path)` 将本次新增的规格文档（proposal/design/specs/tasks）**只增不减**地增量合并到文档知识图谱，使后续步骤和历史变更能语义检索到本次规格

失败不影响主流程。

**知识图谱不可用时回退：** 跳过增量合并。

### 4. 调用 writing-plans 生成实现计划

调用 Superpowers 的 `writing-plans` skill，传入 openspec artifacts + project_profile 作为上下文。

**做法：**

1. 宣布："调用 writing-plans 生成实现计划"
2. **准备上下文**：读取以下文件并构造上下文字符串：
   - `openspec/changes/<变更名>/proposal.md` — 需求背景和目标
   - `openspec/changes/<变更名>/design.md` — 技术方案
   - `openspec/changes/<变更名>/specs/` — 规格增量（所有 .md 文件）
   - `openspec/changes/<变更名>/tasks.md` — 任务清单（**作为权威任务分解，writing-plans 应基于此展开**）
3. **准备 API 验证上下文**：提取关键框架 API 的实际签名，作为 writing-plans 的额外约束。重点关注：
   - 框架提供的工具方法是否存在多个重载（参数个数/类型不同），确认计划中应使用哪个签名
   - 第三方库的 import / require 路径是否与直觉不同（如包名重组后的新路径）
   - 构建工具在项目实际环境中的可用命令（如私有仓库可能限制增量编译，必须全量编译）
   - 将验证结果整理为"框架 API 注意事项"列表，附加到 writing-plans 的 args 中

   **优先使用 CodeGraph（如果 `project_profile.knowledge_graph.codegraph_available == true`）：**
   a. 使用 `codegraph_search` 搜索计划中涉及的框架类名/工具方法，获取所有使用位置
   b. 使用 `codegraph_callers` 验证目标方法的调用签名和重载情况，确认计划应使用哪个签名
   c. 使用 `codegraph_explore` 检查跨模块依赖关系，确认 import 路径与模块结构

   > CodeGraph MCP 默认只暴露 `codegraph_explore`（单次调用已内联 search/callers/impact 的结果）。`search`/`callers` 需 `CODEGRAPH_MCP_TOOLS` 启用或用 CLI 等价命令（`codegraph query`/`callers`）；未启用时统一用 `codegraph_explore` 即可。**优先用单次 `codegraph_explore`** 一次性拿全 search+callers+impact，把 Step 4.3 控制在 1–2 次调用（§12 目标）；分查（多次 search/callers）实测约 3 次。详见 SKILL.md「CodeGraph 工具面说明」。

   **知识图谱不可用时回退（现有逻辑）：**（运行时工具不可达/Unknown tool 同样回退，见 SKILL「运行时可达性规则」）
   a. 在项目中 grep 找同类文件（同模块的 Controller、Service、Handler 等）
   b. Read 提取其中对框架工具类、第三方库的实际调用签名
4. 使用 Skill 工具调用 `superpowers:writing-plans`，args 传入上下文：
   ```
   变更名: <name>
   项目技术栈: {project_profile.languages} + {project_profile.frameworks}
   构建工具: {project_profile.build_tool}
   测试框架: {project_profile.test_command}
   项目结构: {project_profile.structure}
   编译命令: {project_profile.compile_command}
   计划保存路径: superpowers/plans/YYYY-MM-DD-<变更名>.md
   编译约束:
   - 本项目的编译命令为 {compile_command}，计划中每个 Task 完成后必须使用此命令验证编译
   - 接口层和实现层分开定义在不同的 Task 中会导致单独编译失败，必须在同一 Task 中同时修改接口和实现
   checkbox 唯一性约束:
   - 计划中每个 Step 的 checkbox 描述必须全局唯一，不能仅靠步骤编号区分（如"Step 2: 编译验证"会在多个 Task 中重复）
   - 推荐格式：`- [ ] **Step N: <动作描述>（Task <编号>）**` 或在步骤描述中包含 Task 特有的上下文（如文件名、类名）
   - 这确保 apply 阶段使用 Edit 工具更新 checkbox 时，old_string 在文件中唯一匹配
   框架 API 注意事项:
   <Step 3 中验证的实际 API 签名列表>
   知识图谱上下文（如果对应工具可用，否则省略对应行）:
   - 项目模块结构: <从 codegraph_explore 获取的模块依赖关系>
   - API 验证结果: <从 codegraph_search/callers 获取的签名确认>
   - 历史类似变更: <从 graphify query 获取的相关归档规格>
   openspec artifacts:
   <将上述文件内容拼接>
   ```
   将完整 project_profile 信息、编译约束、API 验证结果和知识图谱上下文传入，让 writing-plans 从一开始就生成正确的任务分解，避免事后合并。
5. writing-plans 会读取上下文，生成实现计划（File Structure 表 + 带 checkbox 的 TDD 微步骤），保存到 `superpowers/plans/YYYY-MM-DD-<变更名>.md`
6. **跳过执行移交**：writing-plans 完成后会 offer 执行选择（Subagent-Driven / Inline）。在 HyperSpec 上下文中，跳过此 offer，直接回到本阶段 Step 5。
7. 确认 `superpowers/plans/` 下有计划文件且包含至少 1 个 checkbox。如果没有 checkbox，说明 writing-plans 未能基于 tasks.md 展开步骤，需要重新执行并更明确地指定 "按 tasks.md 中的每个 Task 展开为 TDD 微步骤"。
8. **绑定变更名**：在计划文件开头添加一行注释 `<!-- hyperspec change: <变更名> -->`，用于 SKILL.md 状态检测时确认计划文件与活跃变更的对应关系。
9. **计划质量审查（必须完成）**：逐项检查 writing-plans 生成的计划，确认以下维度全部通过。此步骤不可跳过，即使自主模式也必须执行：
   - **编译约束遵守**：接口+实现是否在同一 Task 中。如果 writing-plans 仍将接口和实现拆分为不同 Task，手动合并并标注原因
   - **import 语句正确性**：检查计划中代码块的 import 语句是否与项目实际路径一致（对比 Step 3 中扫描的已有代码的 import 路径）
   - **无效代码检查**：检查代码块中是否有调试残留、空行占位、无意义的方法调用（如 `.getClass()` 仅用于触发泛型推断）
   - **类型一致性**：计划中后出现的 Task 引用的类型、方法签名是否与前面 Task 定义的一致
   - 如果发现问题，直接在计划文件中修复，修复后重新确认编译依赖
10. **立即更新 checkpoint**：writing-plans 完成且 Step 7-9 验证通过后，**立即**更新 `.hyperspec-state.yaml`：`checkpoint: plan-generated`。这一步**不可跳过**，即使用户声明自主模式也必须先写 `plan-generated`，再在 Step 5 中更新为 `plan-generated-and-confirmed`。确保每个 checkpoint 都是原子性推进，避免中断时状态文件与实际文件不一致。

### 5. 用户确认

展示产出摘要，请用户确认「规格 OK，可以开始实现」：

- 变更名和位置
- 项目 profile 信息（语言、框架、构建工具）
- 生成的 artifacts 列表
- 实现计划中的 Task 数量和关键步骤
- 预计涉及修改的文件列表

## 出口条件

- `openspec/changes/<变更名>/` 下有完整的 artifacts（由 openspec-propose 生成）
- `superpowers/plans/` 下有对应的实现计划文件（由 writing-plans 生成）
- 计划文件中至少有 1 个 checkbox
- 用户明确确认可以进入构建阶段，或用户已声明自主执行模式

出口时更新 `.hyperspec-state.yaml`：`phase: apply`、`checkpoint: plan-generated-and-confirmed`。

## 断点恢复

重新运行时，先读取 `.hyperspec-state.yaml` 的 checkpoint，然后检查实际文件状态：

| checkpoint | 实际文件状态 | 恢复到 |
|-----------|-------------|--------|
| `profiler-done` | 无活跃变更目录 | Step 2（需求确认） |
| `requirements-confirmed` | 变更目录不存在 | Step 3（调用 openspec-propose） |
| `requirements-confirmed` | 变更目录存在但 artifacts 不完整或有空文件 | Step 3（openspec-propose 会自动补充） |
| `openspec-generated` | artifacts 完整但 `superpowers/plans/` 无计划文件 | Step 4（调用 writing-plans） |
| `plan-generated` | 计划文件存在但无 checkbox | Step 4（重新调用 writing-plans） |
| `plan-generated` | 计划文件存在且有 checkbox | Step 5（用户确认） |
| `plan-generated-and-confirmed` | 计划文件存在且有 checkbox | 出口到 apply 阶段（`phase: apply, checkpoint: plan-generated-and-confirmed`） |
| `plan-generated-and-confirmed` | 计划文件不存在或无 checkbox | 回退到 Step 4（重新生成计划） |

**状态文件缺失时的降级**：如果没有 `.hyperspec-state.yaml`，回退到文件扫描方式（按上述表格右列的实际文件状态判断）。

## 硬门

本阶段**禁止写任何代码或创建分支**。任何实现行为必须在 apply 阶段进行。
