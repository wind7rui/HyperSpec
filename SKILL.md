---
name: hyperspec
description: 规格驱动+工程纪律的完整开发工作流。协调OpenSpec（规格管理）和Superpowers（TDD+子代理审查），从需求到实现到归档一条流程走完。当用户说「用hyperspec」「规格驱动开发」「完整流程开发功能」时触发。前提：需已安装Superpowers和OpenSpec。
---

# HyperSpec

规格驱动 + 工程纪律的完整开发工作流。协调 OpenSpec（规格管理）和 Superpowers（TDD + 子代理审查），从需求到实现到归档一条流程走完。

OpenSpec 管「做什么和为什么」，Superpowers 管「怎么做和做得对不对」。HyperSpec 是**轻量编排框架**：以编排 OpenSpec/Superpowers 为主（状态检测、阶段路由、commit 纪律），不重写原生 skill 的功能；另含自有的规格一致性验证、计划质量审查等显式增强（非纯透传）。

## 编排协议

HyperSpec 是**轻量编排框架**，以编排原生 skill 为主，核心做以下四件事：

1. **项目感知** — 自动探测语言/框架/构建工具，生成 `project_profile` 驱动后续阶段的自适应行为
2. **状态检测** — 通过结构化状态文件 + 实际文件验证确定当前阶段和断点位置
3. **阶段路由** — 加载对应 prompt 文件，按其中的流程调用原生 skill
4. **Commit 纪律** — 每个 task/fix 完成后自动 commit，编译前置，不做 push

HyperSpec **不做**：

- 不手动创建 openspec artifacts（由 `openspec-propose` 负责）
- 不手动转 tasks → plan（由 `superpowers:writing-plans` 负责）
- 不手动执行归档操作（由 `openspec-archive-change` 负责）
- 不重申 TDD 规则（由 `superpowers:subagent-driven-development` 负责）
- 不重申审查规则（由 `superpowers:requesting-code-review` 负责）

### 各阶段委托的原生 Skill

| 阶段 | 委托 Skill | 职责 |
|------|-----------|------|
| propose | `openspec-propose` | 通过 CLI 创建变更目录 + 生成所有 artifacts（proposal、design、specs、tasks） |
| propose | `superpowers:writing-plans` | 读取 openspec artifacts，生成 superpowers 格式实现计划 |
| apply | `superpowers:subagent-driven-development` 或 inline | 按计划执行实现，TDD + 子代理审查 |
| apply | `superpowers:verification-before-completion` | 全量验证 |
| apply | `superpowers:requesting-code-review` | 全局代码审查 |
| archive | `openspec-archive-change` | 通过 CLI 归档变更（含 spec sync） |

## 前提检查

触发后先检查两项：

**检查1：Superpowers** — 看 brainstorming 等 skill 是否可用。不可用则提示：`/plugin install superpowers@claude-plugins-official`

**检查2：OpenSpec** — 看项目根目录是否有 `openspec/`。没有则提示：`npx @fission-ai/openspec init`

注意：`openspec init` 除了创建 `openspec/` 目录外，还会在 `.claude/` 下生成配置文件和 slash commands。这些都是正常的副作用，不需要特别处理。

两项都通过后才继续。

## 项目分析器（Project Profiler）

前提检查通过后，**在首次运行或状态文件不存在时**，自动扫描项目生成 project_profile。结果保存到 `.hyperspec-state.yaml` 的 `project_profile` 字段。

### 检测逻辑

| 检测项 | 检测方式 | 默认值 |
|--------|---------|--------|
| **languages** | 统计 `src/`、`lib/`、`app/` 等源码目录下的文件扩展名，取占比最高的 1-2 种 | `[]` |
| **frameworks** | 读取 `package.json` 的 dependencies/devDependencies、`pom.xml` 的 `<dependencies>`、`go.mod` 的 require 等提取关键框架名 | `[]` |
| **build_tool** | 检测根目录配置文件：`pom.xml` → maven，`build.gradle`/`build.gradle.kts` → gradle，`package.json` → npm/yarn/pnpm（看 lock 文件），`go.mod` → go，`Cargo.toml` → cargo，`pyproject.toml` → python | `unknown` |
| **compile_command** | 根据 build_tool 自动推导，见下方"构建命令映射" | `null` |
| **test_command** | 根据 build_tool + 测试目录推导，见下方"构建命令映射" | `null` |
| **structure** | 检查子目录模式：有 `modules/`/`packages/`/多个 `pom.xml`/`go.work` → `monorepo`；否则 → `single-module` | `single-module` |
| **has_ci** | 检查 `.github/workflows/`、`.gitlab-ci.yml`、`Jenkinsfile`、`azure-pipelines.yml` 等 | `false` |

### 构建命令映射

下表为默认推导值。项目分析器会在推导后执行**运行时验证**（见下方），根据实际环境校准命令。

| build_tool | compile_command（默认） | test_command（默认） |
|------------|----------------|--------------|
| maven | `mvn compile` | `mvn test` |
| gradle | `./gradlew compileJava` | `./gradlew test` |
| npm | `npm run build`（如果 script 存在） | `npm test`（如果 script 存在） |
| go | `go build ./...` | `go test ./...` |
| cargo | `cargo build` | `cargo test` |
| python | `null`（跳过编译） | `pytest`（如果安装了） |
| unknown | `null` | `null` |

### 命令运行时验证

默认推导完成后，按以下顺序验证（先检测参数，再执行验证）：

1. **项目特有参数检测**（在执行命令之前）：检查项目配置文件中是否有需要额外传入的参数：
   - Maven：检查根目录是否有 `settings.xml`，有则 compile_command 附加 `-gs ./settings.xml`
   - Gradle：检查是否有自定义 `gradle.properties` 或 `local.properties`
   - npm：检查 `.npmrc` 是否有 registry 配置
2. **执行编译命令**：运行调整后的 compile_command（如果非 null）
3. **成功**：确认命令可用，写入 project_profile
4. **失败且为环境问题**（JDK/Node 版本不匹配、依赖下载失败等）：**先尝试 scoped 降级**——若是 monorepo 且失败来自无关模块的依赖解析（如某子模块第三方依赖在私有仓库缺失，与本次改动无关），改用受影响模块的 scoped 编译（Maven：`mvn compile -pl <变更涉及模块> -am`；Gradle：`./gradlew :<模块>:compileJava`）。scoped 编译通过则命令可用，写入 project_profile 并标注 scoped 模式（apply/archive 的全量编译验证同样降级为 scoped）；scoped 仍失败再**立即阻塞并报告**，等待用户确认正确的命令
5. **失败且为代码问题**（已有代码编译错误）：命令本身可用，写入 project_profile，编译错误留到 apply 阶段处理
6. **test_command**：不强制运行（可能依赖外部服务），但检查 test framework 是否安装（如 `which pytest`、检查 `package.json` 的 devDependencies）

### 项目 profile 的传递方式

project_profile 不会直接传给 openspec-propose 或 writing-plans（它们有自己的上下文读取机制），但 HyperSpec 在调用它们之前，会将 profile 信息作为**上下文前缀**拼接到 args 中：

- 调用 `openspec-propose` 时，在 description 前附加 `[项目: {languages} + {frameworks}, 构建: {build_tool}]`
- 调用 `writing-plans` 时，在 args 中附加 `项目技术栈: {languages} + {frameworks}，构建工具: {build_tool}，测试框架: {test_command}`
- apply 阶段使用 `compile_command` 和 `test_command` 驱动编译检查和验证

## 知识图谱感知（可选增强）

项目分析完成后，检测知识图谱工具的可用性。知识图谱是**可选增强**：不可用时 HyperSpec 仍能完整运行，各阶段自动回退到 grep/Read。

检测在项目分析器产出 `project_profile` 之后进行，结果写入 `project_profile.knowledge_graph`：

**关键：区分"项目属性"与"会话属性"。** `.codegraph/` 目录存在是项目属性（commit 后永久存在），而 `codegraph_explore` MCP 工具是否注入当前会话是会话属性（会随会话重启/换环境消失）。用前者推断后者是范畴错误——装了 CLI 但没配 MCP 时，目录在、工具却不可达。因此分别检测：

1. **CodeGraph 检测**（代码结构层）：
   - `codegraph_indexed`（项目属性）：`.codegraph/` 不存在 → 运行 CLI `codegraph init` 首次索引；存在或 init 成功 → `true`；CLI 未装/init 失败 → `false`
   - `codegraph_tool_reachable`（会话属性）：初始化时试调一次 `codegraph_explore`（最小查询）探活，成功 → `true`。**此值会随时失效**，不可"检一次管全程"
   - `codegraph_available = codegraph_indexed && codegraph_tool_reachable`
2. **Graphify 检测**（文档知识层）：同理拆 `graphify_indexed`（`graphify-out/` 存在）/ `graphify_tool_reachable`（`graphify` CLI 可调），`graphify_available = 两者皆真`。
   - `graphify-out/` 不存在且 openspec 非空 → `/graphify openspec` 构建（**无需外部 API key**，见下方工具面说明）
   - greenfield（openspec 为空）→ `graphify_indexed: false`，推迟到 propose Step 3 规格生成后首次构建
3. 任一检测失败 → 标记对应工具为不可用，**不影响主流程**，后续步骤自动回退到 grep/Read。

> ⚠️ **运行时可达性规则（贯穿所有 prompt 的 KG 分支）**：`*_tool_reachable` 是会话属性、会随时失效。**任何 KG 工具调用若返回 *Unknown tool* / 工具不存在 / 连接错误，立即视为本次不可达**——回退 grep/Read 完成当前步骤，并把对应 `*_tool_reachable` 置 `false`，后续 KG 分支不再尝试直到下次显式探活。这堵死了"目录在但 MCP 未注入 → KG 分支静默失败"的口子；**不要只靠初始化探活一次**（中途失效仍会静默错误）。

各阶段 prompt 中的知识图谱相关步骤都会检查 `project_profile.knowledge_graph.<tool>_available`：
- `true` → 使用对应知识图谱工具（但每次调用若返回 Unknown tool/报错，按上方"运行时可达性规则"立即降级）
- `false` → 回退到现有 grep/Read 方式

**CodeGraph 工具面说明**：

- `codegraph` MCP 服务**默认只暴露 `codegraph_explore` 一个工具**——一次调用即返回相关符号的源码 + 调用链（含跨模块/动态分派跳转）+ 影响面（blast radius），已内联 callers/callees/impact/search 的结果。
- `codegraph_search`/`callers`/`callees`/`impact`/`node` 也是真实工具，但**默认未列出**，需设置环境变量 `CODEGRAPH_MCP_TOOLS=explore,node,search,callers,callees,impact` 启用，或直接用 CLI 等价命令（`codegraph query`/`callers`/`callees`/`impact`/`node`）。
- 因此后续 prompt 中的 `codegraph_search`/`codegraph_callers` 等均理解为：**优先用单次 `codegraph_explore`**（一次内联 search+callers+callees+impact+源码，把 API 验证控制在 1–2 次调用），或对应的 CLI/已启用工具。
- 索引/同步是 **CLI 命令**（`codegraph init`/`sync`），不是 MCP 工具。⚠️ **MCP stdio 部署下文件观察器实测不运行**（"自动增量同步"不生效）——apply 每个 task/fix 后、archive 交叉验证前，**必须手动 `codegraph sync`**，否则 explore/impact/callers 返回过期调用链。
- `codegraph_path` 不是真实工具——两符号间调用路径用 `codegraph_explore` 获取。

**Graphify 工具面说明**：

- Graphify 的「构建」通过 `/graphify` **Skill** 完成（不是某个 MCP 工具，也没有 `graphify build` 子命令）。Skill 默认用 **Claude Code 子代理**做语义抽取——**不读取 `ANTHROPIC_API_KEY`/`OPENAI_API_KEY` 等任何外部 key**；仅当设了 `GEMINI_API_KEY`/`GOOGLE_API_KEY` 时才改走 Gemini API。**因此环境无任何 LLM key 也能完整构建**（之前的"缺 key"失败是对 Skill 的误读）。
- 查询/维护用 CLI：`graphify query "<问题>"`（BFS/DFS 遍历，对应设计中的 query_graph）、`graphify explain "<节点>"`（节点+邻居说明，对应 get_community）、`graphify path "A" "B"`（最短路径）、`graphify update`（增量重抽取，无 LLM）、`graphify merge-graphs`（合并多图）。
- 增量合并/清理用 Python API（`graphify.build.build_merge`）：`build_merge(new_chunks, graph_path, prune_sources=...)` **只增不减**地并入新规格抽取；`prune_sources` 按 `source_file` **精确**清理过期节点（缩图需 `to_json(..., force=True)` 确认）。对应设计中的 build_merge / prune。
- 因此后续 prompt 中的假设名映射为：`graphify_build`→`/graphify` Skill、`graphify_query_graph`→`graphify query`、`graphify_get_community`→`graphify explain`、`graphify_build_merge`→`build_merge()`、`graphify_prune`→`build_merge(prune_sources=)`。
- **何时该开 Graphify（决策指引）**：Graphify 的价值依赖语料积累——**单变更 / 小语料 / 一次性项目价值有限**（query 相关性受词项重叠限制，且每次构建都要跑子代理抽取、有 token/时间成本），可不开；**多变更积累、或需要历史规格语义回溯时**才值得开启。

**编排层约束**：知识图谱逻辑只在 propose.md/apply.md/archive.md 中以「优先使用 XX 工具，不可用时回退」的形式出现，SKILL.md 只做可用性检测与状态记录，不引入复杂 KG 逻辑。HyperSpec 自有的逻辑（archive 规格一致性验证、propose 计划质量审查）是**显式增强**，不伪装为「纯透传」——定位是「轻量编排框架」而非「零自有逻辑的纯编排」。

## 状态管理

### 状态文件：`.hyperspec-state.yaml`

位于项目根目录，HyperSpec 的**快速路由入口**。结构：

```yaml
version: 1
active_change: add-user-auth
phase: propose | apply | archive
checkpoint: profiler-done | requirements-confirmed | openspec-generated | plan-generated | plan-generated-and-confirmed | task-N-complete | verified | reviewed | apply-done | consistency-verified | archived | done
project_profile:
  languages: [java]
  frameworks: [spring-boot]
  build_tool: maven
  compile_command: mvn compile
  test_command: mvn test
  structure: single-module
  has_ci: true
  knowledge_graph:
    codegraph_available: true        # = indexed && tool_reachable（计算字段）
    graphify_available: true         # = indexed && tool_reachable
    codegraph_indexed: true          # 项目属性：.codegraph/ 已生成
    codegraph_tool_reachable: true   # 会话属性：codegraph_explore MCP 真正可达（会失效，见运行时可达性规则）
    graphify_indexed: true           # 项目属性：graphify-out/ 已生成
    graphify_tool_reachable: true    # 会话属性：graphify CLI 可调
```

### 安全策略：状态文件 vs 实际文件

状态文件用于快速路由（避免每次扫描所有目录），但在**关键节点**必须验证实际文件状态：

| 时机 | 验证内容 | 不一致处理 |
|------|---------|-----------|
| 阶段入口 | 状态文件声称的 phase 对应的文件是否存在 | 修正状态文件，路由到实际状态对应的阶段 |
| 断点恢复 | checkpoint 声称的进度是否与实际文件一致 | 回退到最近一个确认一致的状态 |
| 阶段出口 | 出口条件对应的实际文件是否满足 | 不满足则不退出，继续当前阶段 |

**原则**：实际文件状态是 ground truth，状态文件只是缓存。两者冲突时，以实际文件为准并修正状态文件。

## 状态检测

检查 `.hyperspec-state.yaml` 和实际文件状态，确定应该进入哪个阶段：

### Step 1：读取状态文件

- 状态文件不存在 → 这是首次运行，执行项目分析器，然后进入 propose 阶段
- 状态文件存在 → 读取 phase 和 checkpoint

### Step 2：验证状态文件与实际文件一致性

根据状态文件中的 phase，验证对应的前提条件：

**phase = propose 时验证：**
- `openspec/changes/` 下是否有活跃（非 archive）变更目录？ → 无则一致（propose 刚开始）
- **通用规则（适用于所有 checkpoint）**：如果有活跃变更目录且 `superpowers/plans/` 下有对应的计划文件（含 `<!-- hyperspec change: <active_change> -->`），说明实际进度已进入 apply 阶段，修正 `phase: apply`，根据 checkbox 状态判定 checkpoint
- 如果有活跃变更目录但无计划文件：根据变更目录下 artifacts 的完整程度判定 checkpoint（artifacts 完整 → `openspec-generated`，不完整 → `requirements-confirmed`）
- 如果 checkpoint ≥ `openspec-generated`：变更目录下的 artifacts 是否存在且非空？ → 有空文件则回退 checkpoint 到 `requirements-confirmed`
- **知识图谱一致性校验**（如果 `knowledge_graph` 字段存在）：
  - 如果 `codegraph_indexed == true`：验证 `.codegraph/` 目录是否存在 → 不存在则标记 `codegraph_indexed: false`（CodeGraph 不可用，后续回退 grep/Read）
  - 如果 `graphify_indexed == true`：验证 `graphify-out/` 目录是否存在 → 不存在则标记 `graphify_indexed: false`
  - 此校验只修正知识图谱可用性标记，**不回退 checkpoint**（知识图谱是可选增强，不构成阶段节点）

**phase = apply 时验证：**
- `superpowers/plans/` 下是否有包含 `<!-- hyperspec change: <active_change> -->` 的计划文件？ → 无则回退到 propose 阶段
- 计划文件中是否至少有 1 个 checkbox？ → 无则回退到 propose 阶段（plan 不完整）
- 如果 checkpoint 声称 `task-N-complete`：计划文件中对应的 checkbox 是否确实已勾选？ → 未勾选则回退到上一个确认一致的 checkpoint

**phase = archive 时验证：**
- 所有计划 checkbox 是否已勾选？ → 未全勾选则修正状态文件为 `phase: apply, checkpoint: reviewed`，路由到 apply
- 如果 checkpoint ≥ `consistency-verified`：`openspec/changes/archive/` 下是否有对应目录？ → 无则回退到 `consistency-verified`（重做归档）
- 如果 checkpoint ≥ `archived`：`openspec/changes/archive/` 下是否有对应目录？ → 无则回退到 `consistency-verified`（归档中断）
- **注意**：`.close-verification-done` 的存在性由 archive.md 内部处理（断点恢复表），SKILL.md 不据此覆盖 checkpoint，避免基于过期文件做错误路由

### Step 3：路由

| 状态文件 phase | 验证结果 | 路由到 |
|---------------|---------|--------|
| propose | 一致 | propose 阶段（从 checkpoint 恢复） |
| propose | 不一致（artifacts 缺失） | propose 阶段（回退 checkpoint） |
| apply | 一致 | apply 阶段（从 checkpoint 恢复） |
| apply | plan 不存在或不完整 | propose 阶段 |
| archive | 一致 | archive 阶段（从 checkpoint 恢复） |
| archive | apply 未完成 | apply 阶段 |

### Step 4：无状态文件时的降级检测

如果状态文件不存在（可能是旧项目或手动删除），降级为文件扫描检测：

| 检查项 | 怎么查 | 结果 |
|--------|--------|------|
| 有活跃变更？ | `openspec/changes/` 下是否有非 archive 子目录 | 有 → 继续检查 |
| 有多个活跃变更？ | 活跃变更数量 > 1 | 是 → 让用户选择继续哪个 |
| 有计划文件？ | `superpowers/plans/` 下是否有与活跃变更对应的计划文件（文件含 `<!-- hyperspec change: <name> -->`） | 有 → 看实现状态 |
| 计划文件有效？ | 计划文件中是否至少有 1 个 checkbox | 无 → propose 阶段 |
| checkbox 状态？ | checkbox 全部未勾选 | → apply 阶段（全新执行） |
| checkbox 状态？ | checkbox 部分已勾选 | → apply 阶段（断点恢复） |
| checkbox 状态？ | checkbox 全部已勾选 | → apply 阶段（验证→审查→自动进入 archive） |

## 路由

根据检测结果，用 Read 工具加载对应文件并执行：

- **propose 阶段** → 读取 skill 目录下的 `propose.md`，按其中流程执行
- **apply 阶段** → 读取 skill 目录下的 `apply.md`，按其中流程执行
- **archive 阶段** → 读取 skill 目录下的 `archive.md`，按其中流程执行

加载后严格按照文件中定义的流程、出口条件和硬门执行，不要跳步。

## 用户意图覆盖

如果用户明确指定了阶段（如「先做规格」「直接开始实现」「归档收尾」），以用户意图为准，跳过状态检测直接加载对应文件。但如果前置条件不满足（如用户说「直接实现」但 `superpowers/plans/` 下没有计划文件），需要提醒用户先完成前置阶段。

## 状态文件更新时机

| 时机 | 更新内容 |
|------|---------|
| 项目分析器完成 | 写入 `project_profile`（含 `knowledge_graph` 可用性检测结果），`phase: propose`，`checkpoint: profiler-done` |
| propose 阶段需求确认完成 | `checkpoint: requirements-confirmed`，`active_change: <变更名>` |
| openspec-propose 完成 | `checkpoint: openspec-generated` |
| writing-plans 完成 | `checkpoint: plan-generated` |
| propose 用户确认进入 apply | `phase: apply`，`checkpoint: plan-generated-and-confirmed` |
| apply 阶段每个 task 完成 | `checkpoint: task-N-complete` |
| apply 验证通过 | `checkpoint: verified` |
| apply 审查通过 | `checkpoint: reviewed` |
| apply 最终确认完成 | `phase: archive`，`checkpoint: apply-done` |
| archive 一致性验证通过 | `checkpoint: consistency-verified` |
| archive 归档完成 | `checkpoint: archived` |
| archive 全部完成 | `checkpoint: done`，删除状态文件 |
