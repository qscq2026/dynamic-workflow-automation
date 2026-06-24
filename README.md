# dynamic-workflow-automation

**Claude Code skill** · 自然语言 → Workflow 脚本 → 自动执行

---

## 是什么

一个运行在 Claude Code 里的 skill，核心概念是**动态工作流**。

传统工作流工具要求你预先设计好流程、写好脚本、再触发执行。动态工作流反过来：脚本在运行时根据你的任务描述即时生成，编排模式由 skill 分析任务特征后自动选择，整个过程对你透明。「动态」体现在两个层面：

- **脚本动态生成**：不是调用预先写好的固定脚本，而是每次根据具体任务即时生成对应的 Workflow JS 脚本
- **模式动态路由**：七种编排模板不需要你指定，skill 通过分析任务的处理对象、依赖关系、汇聚方式等特征自动匹配

你只需描述任务，skill 自动完成剩下三件事：

1. 分析任务结构，匹配最合适的编排模式
2. 生成对应的 Workflow JS 脚本
3. 立即执行，把结果呈现给你

不需要你写脚本，不需要你确认脚本，描述任务就够了。

![Dynamic Workflow Automation 架构图](assets/dynamic_workflow_automation_architecture.svg)

---

## 适用场景

满足以下任一条件时触发：

- 并行处理多个文件 / 数据源 / URL
- 流水线处理（多阶段，后阶段依赖前阶段结果）
- 循环发现（不确定要做多少轮，直到枯竭为止）
- 多视角验证（同一结论由多个独立 agent 交叉验证）
- 工作量明显超过单次对话的处理极限
- 你说了「用并行」「编排一下」「用工作流跑」

**不适用：** 单步简单任务、需要实时用户交互的任务。

---

## 环境要求

**仅限 Claude Code**。本 skill 依赖 Claude Code 内置的 `Workflow` 编排工具（`parallel()` / `pipeline()` / `agent()` 等运行时原语），在 claude.ai 或 API 环境中无法执行。

---

## 七种编排模式

skill 会自动分析任务并选择其中一种：

| 模式 | 适用场景 | 核心原语 |
|------|---------|---------|
| **T1 Fan-Out + 汇聚** | N 个独立对象各自处理，最后合成一份综合报告 | `parallel` + 汇聚 `agent` |
| **T2 Fan-Out 各自输出** | N 个独立对象各自处理，各自输出，不汇聚 | `parallel` |
| **T3 Pipeline 无屏障** | 每个单元经多阶段处理，阶段间无跨单元聚合 | `pipeline` |
| **T4 Pipeline + Barrier** | 阶段间需要全局去重 / 排序 / 过滤后再继续 | `parallel` × 2 + 中间聚合 |
| **T5 循环发现（3 种变体）** | 数量未知持续搜索，或已知目标数，或需独立评估完成 | `while` + budget guard |
| **T6 验证（2 种变体）** | 对抗验证（驳斥结论），或多视角验证（不同维度评估） | `parallel` + schema 投票 |
| **T7 评判小组** | 多视角生成方案，独立评审，综合选最优 | `parallel` × 2 + 综合 `agent` |

### T1 — Fan-Out + 汇聚

![T1 Fan-Out + 汇聚](assets/t1_fanout_merge.svg)

N 个独立单元同时处理，全部完成后由一个汇聚 agent 合成一份综合报告。最常用模式。

### T2 — Fan-Out 各自输出

![T2 Fan-Out 各自输出](assets/t2_fanout_individual.svg)

N 个独立单元同时处理，结果各自保留，无汇聚步骤。与 T1 的唯一区别是省略了最后的合并 agent。

### T3 — Pipeline（无屏障）

![T3 Pipeline 无屏障](assets/t3_pipeline_nobarrier.svg)

每个单元独立穿过所有阶段，前一单元完成第 N 阶段后即可进入第 N+1 阶段，不等其他单元。阶段间无跨单元操作。

### T4 — Pipeline + Barrier

![T4 Pipeline + Barrier](assets/t4_pipeline_barrier.svg)

阶段 1 并行完成后设屏障，在全部结果上做去重 / 排序 / 过滤等跨单元操作，再进入阶段 2。判据：阶段切换时是否必须拿到所有单元结果才能继续？是 → T4，否 → T3。

### T5 — 循环发现（3 种变体）

![T5 循环发现](assets/t5_loop_dry.svg)

**变体 A — Loop-Until-Dry**（如图）：数量未知，持续搜索直到连续 2 轮无新 key。
```
┌──────┐  搜索   ┌───────┐  结果   ┌──────────┐
│ agent │ ─────→ │ 去重   │ ─────→ │ 空转检查  │ ── 空转2轮 ──→ 退出
└──────┘        └───────┘         └──────────┘
    ↑                                      │
    └────────── 还有新发现？继续 ────────────┘
```

**变体 B — Loop-Until-Count**：明确知道目标数量（如"找 10 个案例"），数量够了就停。
```
┌──────┐  搜索   ┌───────────┐
│ agent │ ─────→ │ found < N │ ── 已满 ──→ 退出
└──────┘        └───────────┘
    ↑                │ 未满
    └────────────────┘
```

**变体 C — Loop-Until-Evaluator**：由独立的评估 agent 判断工作是否完成，适合完成标准不单是数量而是质量的场景。
```
┌──────┐  结果   ┌──────────┐  done?  ┌──────────┐
│ agent │ ─────→ │ evaluator │ ─────→ │ 通过？    │ ── 是 ──→ 退出
└──────┘        └──────────┘         └──────────┘
    ↑                │                     │ 否
    │                └── feedback ──────────┘
    └────────────────── 继续迭代 ────────────┘
```
所有变体均内置 budget guard，预算耗尽时优雅退出，已有结果不丢失。

### T6 — 验证（2 种变体）

![T6 对抗验证](assets/t6_adversarial_verify.svg)

**变体 A — 对抗验证**（如图）：多个独立验证者以相同标准尝试反驳同一结论。通过 schema 强制输出 `verdict: "pass"|"reject"`，在结构化字段上计票，≥2 票 pass 则通过。
```
                  ┌────────────┐
                  │ 验证者 1    │ ──→ pass / reject
结论 ──→ parallel ┼────────────┤
                  │ 验证者 2    │ ──→ pass / reject
                  ├────────────┤
                  │ 验证者 3    │ ──→ pass / reject
                  └────────────┘
                         ↓
                   ≥2 pass → 通过
```

**变体 B — 多视角验证**：每个验证者负责不同视角（正确性 / 安全性 / 性能 / 可复现性等），各自评分并指出问题，最后综合各视角判断。
```
                  ┌──────────────┐
                  │ 正确性视角    │ ──→ score + issues
结论 ──→ parallel ┼──────────────┤
                  │ 安全性视角    │ ──→ score + issues
                  ├──────────────┤
                  │ 性能视角      │ ──→ score + issues
                  ├──────────────┤
                  │ 可复现性视角  │ ──→ score + issues
                  └──────────────┘
                         ↓
                  平均分 + 各视角 issues 汇总
```

### T7 — 评判小组

![T7 评判小组](assets/t7_judge_panel.svg)

第一轮并行从多个视角生成方案，第二轮并行对各方案独立评审打分，最后由综合 agent 根据所有评审选出最优方案。

---

## 文件结构

```
dynamic-workflow-automation/
├── SKILL.md                          # skill 主文件，Claude Code 读取此文件
└── references/
    ├── workflow-tool-api.md          # Workflow() 调用参数与返回值
    └── workflow-runtime-api.md       # 脚本内运行时 API（agent/parallel/pipeline 等）
```

---

## 安装

将整个目录放入你的 Claude Code skill 路径下，确保 `SKILL.md` 可被 Claude Code 读取即可。具体路径取决于你的 Claude Code 配置。

---

## 使用示例

```
帮我并行分析这 20 份合同，提取每份的关键条款和风险点，最后汇成一份对比报告
```

```
爬取这个论坛的所有帖子 URL，不知道有多少页，一直找到没有新的为止
```

```
用三个不同视角评审这份技术方案，找出最优方案
```

```
把这批图片先分类，再对每类做风格统一，不同类别的处理互不依赖
```

---

## 设计说明

### 执行原则

- **不解释、不确认，直接执行。** skill 生成脚本后立即调用 Workflow 运行，不等用户确认。
- **报错自动修正。** Workflow 执行出错时，skill 根据错误信息修正脚本后重试，不放弃 Workflow 模式。
- **结果直接呈现。** 执行完成后直接展示结果内容，不以「执行完成」结尾。

### 脚本约束

运行时沙箱的硬性限制，已内置在模板中：

| 约束 | 说明 |
|------|------|
| 禁止 `Date.now()` / `Math.random()` / 无参 `new Date()` | 破坏脚本可重放性 |
| 纯 JavaScript | 不是 TypeScript，无 Node.js API（无 `fs` / `path` / `process`） |
| 子代理无法读取主会话本地文件 | 本地文件须由主会话预读后通过 `args` 传入 |
| 最大并发 ~10 个 agent | 超出自动排队 |
| 每个 Workflow 最多 1000 个 agent 调用 | — |
| 开放循环必须内置 budget guard | T5 模板已内置，防止预算耗尽时无法优雅退出 |

### 质量设计说明（与原始版本的差异）

本 skill 在以下三处做了加固，解决了模板原始设计中的已知问题：

**T6 投票判定**：验证者 agent 通过 `schema` 强制输出 `{ verdict: "pass" | "reject", reason }` 结构化字段，投票判定基于 `v.verdict === 'pass'` 而非字符串匹配，消除了对自然语言「通过」二字的依赖。

**T5 去重机制**：去重从对整段返回文本做 Set 匹配，改为对结构化 `key` 字段去重。agent 通过 schema 返回 `{ items: [{ key, data }] }`，`key` 是语义唯一标识（名称 / ID / URL），避免同一发现因表述不同被重复计入。

**T5 budget guard**：循环条件从 `while (dryRuns < 2)` 改为 `while (dryRuns < 2 && (!budget.total || budget.remaining() > 50_000))`，budget 耗尽时自然退出并 return 已有结果，不丢数据。

---

## 局限

- 强依赖 Claude Code 环境，无法跨平台运行
- 子代理数量接近 1000 时需提前规划分批策略
- T5 的 `key` 字段含义需在适配时根据具体任务明确定义（名称？URL？ID？）
- T6 / T7 的验证质量取决于 prompt 对「验证角度」的描述是否具体

---

## 附1：与 loop-engine 的比较

两个 skill 解决同一个问题域的不同层面，定位不互斥。

[loop-engine](https://github.com/qscq2026/loop-engine/) 是 prompt 模板形式的完整系统——从需求分析、YAML 配置、用户确认到 10 步执行流程和汇总报告全链路覆盖，内置质量门禁（Gate，独立 subagent 校验）、循环回退和 best-of 收敛，由模型本身作为调度者执行，无外部运行时依赖，可在任何 agent 环境下使用。

dynamic-workflow-automation 是**执行层**工具：跳过设计阶段，直接从自然语言任务描述生成可运行的 Workflow 脚本并立即执行，依赖 Claude Code 原生并发运行时，强调零摩擦的自动化。

| | dynamic-workflow-automation | loop-engine |
|---|---|---|
| **定位** | 执行层，自动生成脚本并运行 | 全链路系统，设计 + 执行 + 报告 |
| **交互模式** | 静默执行，不与用户确认 | 协作设计，问卷 + 草稿确认 |
| **并发实现** | `parallel()` 原语，真并发 | 模型启动多个 subagent，声明式并发 |
| **质量控制** | T6 对抗验证（可选模板） | Gate 作为一等公民，内置 double_check |
| **循环语义** | T5 搜索枯竭（发现型） | loop 质量不达标回退（改进型） |
| **平台依赖** | 仅限 Claude Code | 任意 agent 环境 |
| **适合场景** | 批量处理、并行爬取、多源分析 | 多阶段流程、质量驱动的迭代任务 |

两者可以组合使用：用 loop-engine 设计和把关整体流程，在需要大规模并行处理的阶段内嵌 dynamic-workflow-automation 的编排模式。

---

## 附二：与 Boris Loop / Loop Engineering 的关系

2026 年 6 月，Claude Code 创建者 **Boris Cherny** 宣告了从"写 prompt"到"写循环"的范式转变——agent 不再由人逐轮提示，而是在循环中自主运行，由独立的 evaluator 判断何时完成。Google 工程师 **Addy Osmani** 将其系统化为 **Loop Engineering**，提炼出 6 个构建模块（Automations、Worktrees、Skills、Connectors、Sub-agents、Memory）。

本 skill 对应于 Boris 三层架构中的 **Layer 3（/batch + 动态 Workflow 集群）**，是 Workflow 运行时原语（`agent()` / `parallel()` / `pipeline()`）的模板化封装。其核心设计原则与 Boris Loop 一脉相承：

| 本 skill 的概念 | 对应的 Boris / Loop Engineering 概念 |
|:---------------|:-----------------------------------|
| **T5 变体 C：Loop-Until-Evaluator** | 递归式终止——子 agent 判断完成与否，不靠硬编码条件 |
| **T6 变体 B：多视角验证** | 写查分离——独立验证者从不同维度评估，不自我打分 |
| **T5/T6 的 schema 结构化输出** | 可验证性——输出必须可被程序化检查，不依赖自然语言判断 |
| **budget guard（预算保护）** | 成本意识——无限制循环比有限循环危险得多 |

相比 Boris 的完整三层架构（本地 `/loop` + 云端 Routines + `/batch` Workflow），本 skill 仅聚焦第三层——集群级的动态 Workflow 脚本生成与执行。前两层的调度能力（时间驱动 / 云端持久化）依赖 Claude Code 原生命令（`/loop`、`/schedule`、Routines），与本 skill 互补，配合使用即可复现 Boris 三层飞轮。

---

## License

MIT
