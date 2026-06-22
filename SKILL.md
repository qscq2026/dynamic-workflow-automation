---
name: dynamic-workflow-automation
description: |
  Workflow 脚本自动生成引擎 — 把自然语言任务描述自动转换为 Workflow 工具可执行的
  JS 脚本（agent/parallel/pipeline/phase/loop），然后立即执行。核心价值是把
  "我需要先理解你的任务，然后手动设计工作流"变成"你只管描述任务，我自动生成脚本并运行"。

  ALWAYS 触发此 skill 当任务涉及以下任一特征（即使用户没提 workflow）：
  - 并行处理多个文件/数据源/网站
  - 流水线处理（多阶段，后阶段依赖前阶段输出）
  - 循环发现（不确定要做多少轮）
  - 多视角分析验证（从多个角度分析同一件事再综合）
  - 大工作量（明显超过单次处理极限）
  - 用户说"用工作流/用并行/帮我编排一下"

  对于符合以上特征的任务，必须使用此 skill 自动生成 Workflow 脚本并执行，
  不得直接串行处理。

  不适用于：单步简单任务、需要实时用户交互的任务。
---

# Dynamic Workflow Automation

## 核心逻辑

当此 skill 被触发时，你的任务只有一个：
**分析用户的需求 → 选择合适的 Workflow 模式 → 自动生成完整的 JS 脚本 → 调用 Workflow 工具执行它。**

不要教用户怎么做，而是直接做出来。

## Step 1：任务分析

分析用户描述，提取以下信息填入任务分析表：

| 维度 | 提取结果 |
|------|---------|
| **输入** | 用户要处理什么？（文件列表、URL、搜索查询、代码等） |
| **对每个单元做什么操作** | 分析/转换/提取/验证？ |
| **单元之间有无依赖** | 独立？需要先做完 A 才知道 B 做什么？ |
| **需要汇聚吗** | 所有结果合成一份报告？还是各自独立输出？ |
| **有无循环/迭代需求** | 不确定数量？需要找到没有新发现为止？ |
| **子代理能直接处理吗** | 需要主会话先准备数据吗？ |

## Step 2：模式选择

根据分析结果选择脚本模板：

### 模式 1：Fan-Out + 汇聚（最常见）

```
输入：N 个独立单元（文件/URL/数据项）
操作：对每个单元执行相同分析
汇聚：所有结果交给一个 agent 综合生成报告
```

```javascript
export const meta = {
  name: 'task-fanout',
  description: '并行处理 N 个单元并汇聚结果',
  phases: [
    { title: '并行处理', detail: '处理所有单元' },
    { title: '综合', detail: '汇聚结果生成报告' },
  ],
}

phase('并行处理')
const results = await parallel(items.map((item, i) => () =>
  agent(`【任务描述】处理这个单元: ${item}`, {
    label: `处理 ${i+1}/${total}`,
    phase: '并行处理',
  })
))

phase('综合')
const output = await agent(
  `【综合任务】以下是对 ${total} 个单元的处理结果，请综合：\n${
    JSON.stringify(results.filter(Boolean))
  }`,
  { label: '综合报告', phase: '综合' }
)

return output
```

### 模式 2：Fan-Out 独立输出（不需要汇聚）

```
输入：N 个独立单元
操作：对每个执行相同操作
汇聚：不需要，每个输出自己的结果
```

```javascript
export const meta = {
  name: 'task-fanout-individual',
  description: '并行处理 N 个单元，各自输出',
  phases: [{ title: '并行处理', detail: '处理所有单元' }],
}

phase('并行处理')
const results = await parallel(items.map((item, i) => () =>
  agent(`【任务描述】处理这个单元并直接输出结果: ${item}`, {
    label: `处理 ${i+1}/${total}`,
    phase: '并行处理',
  })
))

return { results: results.filter(Boolean) }
```

### 模式 3：Pipeline 流水线（无屏障）

```
输入：N 个单元
阶段 1：对每个单元做操作 A
阶段 2：对每个单元的结果做操作 B（不等所有完成阶段 1）
```

```javascript
export const meta = {
  name: 'task-pipeline',
  description: '流水线处理',
  phases: [
    { title: '阶段1', detail: '阶段 1 处理' },
    { title: '阶段2', detail: '阶段 2 处理' },
  ],
}

phase('阶段1')
const results = await pipeline(
  items,
  item => agent(`阶段1: 处理 ${item}`, {phase: '阶段1'}),
  prev => agent(`阶段2: 处理上一步结果: ${prev}`, {phase: '阶段2'}),
)

return results
```

### 模式 3b：Pipeline with Barrier

```
输入：N 个单元
阶段 1：并行处理所有单元
Barrier：在所有结果上去重/过滤/排序
阶段 2：在结果上做下一阶段处理
```

```javascript
export const meta = {
  name: 'task-pipeline-barrier',
  description: '流水线带屏障处理',
  phases: [
    { title: '阶段1', detail: '并行处理所有单元' },
    { title: '汇聚', detail: '在全局结果上操作' },
    { title: '阶段2', detail: '继续处理' },
  ],
}

phase('阶段1')
const stage1 = await parallel(items.map(i => () =>
  agent(`阶段1: ${i}`, {phase: '阶段1'})
))
const valid = stage1.filter(Boolean)
// 在这里做全局操作（去重/过滤等）
const processed = globalOp(valid)
phase('阶段2')
const stage2 = await parallel(processed.map(r => () =>
  agent(`阶段2: ${r}`, {phase: '阶段2'})
))
return stage2
```

### 模式 4：循环直到枯竭（Loop-Until-Dry）

```
输入：不知道有多少发现
循环：不断寻找直到连续 N 轮没有新发现
输出：所有发现汇总
```

```javascript
export const meta = {
  name: 'task-loop-dry',
  description: '循环直到没有新发现',
  phases: [{ title: '循环发现', detail: '迭代搜索直到枯竭' }],
}

phase('循环发现')
const all = new Set()
let dryRuns = 0
while (dryRuns < 2) {
  const batch = await agent(
    `【循环搜索任务】继续搜索新发现。已有结果: ${[...all].join(', ')}`,
    { phase: '循环发现' }
  )
  if (!batch || batch.trim().length === 0) { dryRuns++; continue }
  dryRuns = 0
  all.add(batch.trim())
  log(`已找到 ${all.size} 个结果`)
}
return { results: [...all] }
```

### 模式 5：对抗验证

```
输入：一个或多个发现/结论
验证：多个独立子代理尝试反驳
输出：只有通过验证的结果
```

```javascript
export const meta = {
  name: 'task-adversarial-verify',
  description: '对抗验证模式',
  phases: [
    { title: '发现', detail: '生成初始结果' },
    { title: '验证', detail: '并行验证每个结果' },
    { title: '输出', detail: '输出通过验证的结果' },
  ],
}

phase('发现')
const findings = await agent('【发现任务】', {phase: '发现'})

phase('验证')
const verified = await parallel(findings.split('\n').filter(Boolean).map(f => () =>
  agent(`【验证任务】尝试反驳这个发现。如果无法明确反驳则接受：\n${f}`, {
    phase: '验证',
  })
))

phase('输出')
const passed = verified.filter(Boolean).filter(v => !v.includes('确认错误'))
return { findings, passed }
```

### 模式 6：评判小组（多方案生成+打分）

```
输入：问题描述
第一步：从多个角度生成方案
第二步：并行打分
第三步：选最优
```

```javascript
export const meta = {
  name: 'task-judge-panel',
  description: '多方案评判',
  phases: [
    { title: '方案生成', detail: '多角度生成方案' },
    { title: '评审', detail: '独立评审每个方案' },
    { title: '综合', detail: '选最优方案' },
  ],
}

phase('方案生成')
const approaches = await parallel(perspectives.map(p => () =>
  agent(`从 ${p} 角度设计解决方案`, {phase: '方案生成'})
))

phase('评审')
const scores = await parallel(approaches.map(a => () =>
  agent(`评审这个方案的优缺点并打分：\n${a}`, {phase: '评审'})
))

phase('综合')
const best = await agent(
  `以下是多个方案的评审结果，请综合选出最优方案：\n${JSON.stringify(scores)}`,
  {phase: '综合'}
)

return { approaches, scores, best }
```

## Step 3：脚本生成

将选择的模板适配到用户的具体需求：

1. **替换占位文本** — `【任务描述】`、`【综合任务】` 等用用户的实际需求替换
2. **确定输入** — 从用户描述中提取 items 列表，或让 agent 在脚本内部去发现
3. **添加 schema** — 如果子代理需要有结构化的输出，加 schema 参数
4. **设置 label** — 给每个 agent 起有意义的 label，方便跟踪进度
5. **确定 effort** — 简单操作用 `low`，复杂分析/验证用 `medium` 或 `high`

## Step 4：执行

脚本生成完毕后，**立即调用 Workflow 工具执行**：

```javascript
Workflow({
  script: "上一步生成的完整 JS 脚本",
  description: "简短的中文描述",
  args: 如果有需要传入脚本的参数,
})
```

---

## 重要提醒

### 子代理能力约束
- 子代理默认**不能**用 Read 工具读主会话的本地文件
- 子代理可以跑 Bash 命令
- 如果子代理需要读本地文件，用 Bash cat/grep 代替 Read，或在 prompt 中传入文件内容
- 子代理不能 Edit/Write 到主会话的文件系统

### 脚本约束
- 不能用 `new Date()` / `Date.now()` / `Math.random()`
- 不是 TypeScript，不能用类型注解
- 脚本内没有 Node.js 文件系统 API
- 每个 Workflow 调用最多 1000 个子代理

### 关于返回结果
- `return { report, fileCount, ... }` — 用对象返回多个值
- 工作流完成后，**自动将结果摘要展示给用户**（结果在 tool result 的 result 字段中）
- 如果结果是长篇报告，直接展示在对话中给用户看
