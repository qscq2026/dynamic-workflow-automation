---
name: dynamic-workflow-automation
description: |
  Workflow 脚本自动生成引擎 — 分析自然语言任务描述，自动选择合适的脚本模式，
  生成完整的 JS 脚本，然后立即用 Workflow 执行。
  核心价值是把"我需要先理解你的任务，再手动设计工作流脚本"变成
  "你只管描述任务，我自动生成脚本并运行"。

  ALWAYS 使用此 skill 当任务涉及以下任一特征：
  - 并行处理多个文件/数据源/网站
  - 流水线处理（多阶段，后阶段依赖前阶段）
  - 循环发现（不确定做多少轮）
  - 多视角验证（同一件事多角度验证再综合）
  - 大工作量（明显超过单次处理极限）
  - 用户说"用工作流/用并行/编排一下"

  对于符合以上特征的任务，必须使用此 skill 自动生成 Workflow 脚本并执行，
  不得直接串行处理。
  
  不适用于：单步简单任务、需要实时用户交互的任务。
---

# Dynamic Workflow Automation — 自动生成引擎

## 核心指令

收到用户请求后，按以下 5 步自动处理：

```
Step 1: 分析 → 提取任务特征（处理对象、数量、依赖关系、是否需要汇聚）
Step 2: 选择 → 匹配最适合的模式模板（见下方 7 种模式）
Step 3: 生成 → 把模板适配为用户任务的具体 JS 脚本
Step 4: 执行 → 用 Workflow 运行脚本
Step 5: 展示 → 把结果清晰呈现给用户
```

**不得解释你要怎么写脚本，不得让用户确认脚本内容。** 直接生成并执行。
如果 Workflow 报错，根据错误信息修正脚本后重试，不要放弃使用 Workflow 模式。

Workflow 是 Claude Code 内置的编排工具（就像 Write/Bash/Agent 一样），不是外部依赖。

详细的 API 参考放在 `references/` 目录中：
- **`references/workflow-tool-api.md`** — Workflow() 的调用参数、返回值、恢复机制
- **`references/workflow-runtime-api.md`** — 脚本内 agent/parallel/pipeline/phase/log/budget 的完整 API

---

## Step 1：任务分析

在脑中快速提取以下信息（不需要输出给用户看）：

| 维度 | 要回答的问题 |
|------|------------|
| 处理对象 | 文件？URL？代码？数据？搜索查询？ |
| 操作类型 | 分析/转换/提取/搜索/验证/打分？ |
| 单元数量 | 已知明确的 N 个？还是未知需循环发现？ |
| 单元间依赖 | 独立并行？需要先做完 A 才知道 B？ |
| 汇聚方式 | 所有结果合成一份报告？各自独立输出？ |
| 子代理能力 | 子代理能直接完成任务吗？需要主会话提前准备数据吗？ |

---

## Step 2：模式选择 — 7 种模板

根据分析结果选择模板，直接跳到对应的模板处复制适配。

### 模板 1：Fan-Out + 汇聚

**适用条件**：N 个独立对象各自处理后汇聚成一份综合结果。
**最常用模式**，适用于文件分析、批量搜索、多源数据提取等。

```javascript
export const meta = {
  name: 'task-name',
  description: '任务描述',
  phases: [
    { title: '并行处理', detail: '处理所有单元' },
    { title: '综合', detail: '汇聚结果' },
  ],
}

phase('并行处理')
const results = await parallel(items.map((item, i) => () =>
  agent(`【操作描述】\n输入: ${item}`, {
    label: `处理 ${i+1}/${items.length}`,
    phase: '并行处理',
  })
))

phase('综合')
const final = await agent(
  `【综合任务描述】\n\n以下是 ${items.length} 个结果:\n${
    JSON.stringify(results.filter(Boolean), null, 2)
  }`,
  { label: '综合报告', phase: '综合' }
)
return final
```

---

### 模板 2：Fan-Out 各自输出

**适用条件**：N 个独立对象各自处理并输出，不需要汇聚。

```javascript
export const meta = {
  name: 'task-fanout-individual',
  description: '任务描述',
  phases: [{ title: '并行处理', detail: '处理所有单元' }],
}

phase('并行处理')
const results = await parallel(items.map((item, i) => () =>
  agent(`【操作描述】\n输入: ${item}`, {
    label: `处理 ${i+1}/${items.length}`,
    phase: '并行处理',
  })
))
return { results: results.filter(Boolean) }
```

---

### 模板 3：Pipeline（无屏障）

**适用条件**：每个单元需经多阶段处理，且各阶段间**不需要跨单元聚合**（无去重、无排序、无过滤阈值）。选 T3 还是 T4 的判据：阶段切换时是否需要拿到**所有单元的结果**才能继续？不需要 → T3，需要 → T4。

```javascript
export const meta = {
  name: 'task-pipeline',
  description: '任务描述',
  phases: [
    { title: '阶段1', detail: '阶段一' },
    { title: '阶段2', detail: '阶段二' },
  ],
}

phase('阶段1')
const results = await pipeline(
  items,
  item => agent(`阶段1: ${item}`, {phase: '阶段1'}),
  prev => agent(`阶段2: ${prev}`, {phase: '阶段2'}),
)
return results
```

---

### 模板 4：Pipeline with Barrier

**适用条件**：阶段间存在**跨单元的聚合操作**——去重、排序、按阈值过滤、取 Top-N 等，这些操作必须拿到所有单元的阶段 1 结果才能执行。如果只是每个单元各自处理完继续下一阶段，用 T3。

```javascript
export const meta = {
  name: 'task-pipeline-barrier',
  description: '任务描述',
  phases: [
    { title: '阶段1', detail: '并行处理' },
    { title: '阶段2', detail: '继续处理' },
  ],
}

phase('阶段1')
const stage1 = await parallel(items.map(i => () =>
  agent(`阶段1: ${i}`, {phase: '阶段1'})
))
const valid = stage1.filter(Boolean)
log(`阶段1完成，${valid.length} 个有效结果`)

// 全局操作：去重/过滤/排序等
// const deduped = dedupe(valid)

phase('阶段2')
const stage2 = await parallel(valid.map(r => () =>
  agent(`阶段2: ${r}`, {phase: '阶段2'})
))
return stage2.filter(Boolean)
```

---

### 模板 5：循环发现（Loop-Until-Dry）

**适用条件**：不知道有多少发现，持续寻找直到连续 N 轮无新发现。

```javascript
export const meta = {
  name: 'task-loop-dry',
  description: '任务描述',
  phases: [{ title: '循环发现', detail: '搜索直到枯竭' }],
}

phase('循环发现')
const found = []        // 存结构化对象，用于去重和传给下一轮
const seen = new Set()  // 对关键字段去重，不对整段文本去重
let dryRuns = 0
while (dryRuns < 2 && (!budget.total || budget.remaining() > 50_000)) {
  const batch = await agent(
    `【搜索任务描述】\n已发现 ${found.length} 个: ${found.map(x => x.key).join(', ')}\n\n返回本轮新发现，若无新发现返回空数组。`,
    {
      phase: '循环发现',
      schema: {
        type: 'object',
        properties: {
          items: {
            type: 'array',
            items: {
              type: 'object',
              properties: {
                key: { type: 'string' },   // 用于去重的唯一标识（名称/ID/URL）
                data: { type: 'string' },  // 其余详情
              },
              required: ['key'],
            },
          },
        },
        required: ['items'],
      },
    }
  )
  const newItems = (batch?.items ?? []).filter(x => x.key && !seen.has(x.key))
  if (newItems.length === 0) { dryRuns++; continue }
  dryRuns = 0
  newItems.forEach(x => { seen.add(x.key); found.push(x) })
  log(`已找到 ${found.length} 个`)
}
return { results: found }
```

---

### 模板 6：对抗验证

**适用条件**：需要验证结论/发现的真实性，多个独立验证者尝试反驳。

```javascript
export const meta = {
  name: 'task-verify',
  description: '对抗验证',
  phases: [
    { title: '发现', detail: '生成结论' },
    { title: '验证', detail: '并行验证' },
  ],
}

phase('发现')
const findings = await agent('【提取/生成结论】', {phase: '发现'})

phase('验证')
const votes = await parallel(Array.from({length: 3}, (_, i) => () =>
  agent(`验证者 ${i+1}，尝试反驳以下结论。若能找到有效反驳，verdict 填 "reject"；若无法反驳，verdict 填 "pass"。\n\n结论:\n${findings}`, {
    phase: '验证',
    schema: {
      type: 'object',
      properties: {
        verdict: { type: 'string', enum: ['pass', 'reject'] },
        reason: { type: 'string' },
      },
      required: ['verdict', 'reason'],
    },
  })
))
const passed = votes.filter(Boolean).filter(v => v.verdict === 'pass').length >= 2
return { findings, verdict: passed ? '通过' : '驳回' }
```

---

### 模板 7：评判小组

**适用条件**：需从多个角度设计方案，然后打分选最优。

```javascript
export const meta = {
  name: 'task-judge-panel',
  description: '多方案评判',
  phases: [
    { title: '方案生成', detail: '多角度生成' },
    { title: '评审', detail: '独立评审' },
    { title: '综合', detail: '选最优' },
  ],
}

const perspectives = ['【角度1】', '【角度2】', '【角度3】']

phase('方案生成')
const approaches = await parallel(perspectives.map(p => () =>
  agent(`从 ${p} 视角设计方案:\n需求:【需求】`, {phase: '方案生成'})
))

phase('评审')
const scores = await parallel(approaches.map(a => () =>
  agent(`评审以下方案优缺点并打分:\n${a}`, {phase: '评审'})
))

phase('综合')
const best = await agent(
  `综合选最优:\n${JSON.stringify(scores.filter(Boolean))}`,
  { phase: '综合' }
)
return { approaches, scores, best }
```

---

## Step 3：适配生成

选取模板后，做以下替换：

1. **替换 `【操作描述】` / `【综合任务描述】` / `【搜索任务描述】`** 为用户的具体任务描述
2. **确定 items 列表** — 如果用户给了具体列表，硬编码在脚本内；否则让 agent 去发现
3. **确定是否需要 schema** — 需要结构化输出时加 `schema` 参数（格式见 `references/workflow-runtime-api.md`）
4. **确定 model 和 effort** — 复杂推理 `effort: 'high'` / `model: 'opus'`；简单任务 `effort: 'low'`
5. **确定 args** — 主会话数据通过 `args` 传给脚本

**⚠️ 子代理不能读本地文件**：如果涉及本地文件，先跑 Bash 提取内容，然后把内容放进 prompt 传给 agent，或让 agent 自己跑 Bash。

---

## Step 4：执行

脚本生成后立即执行（详细参数见 `references/workflow-tool-api.md`）：

```javascript
Workflow({
  script: "上一步生成的完整 JS 脚本",
  description: "中文描述",
  // 如果 items 列表由主会话获取，用 args 传入
  args: items列表或需要传的参数,
})
```

---

## Step 5：展示结果

Workflow 返回的 `result` 字段包含脚本的 return 值。
**直接把结果清晰展示给用户**，不要说"工作流执行完成"就结束。

---

## 约束备忘

| 类别 | 内容 |
|------|------|
| **脚本内禁止** | `new Date()` / `Date.now()` / `Math.random()`（破坏可重放）|
| **语言** | 纯 JS，不是 TS，无 Node.js API |
| **子代理权限** | 不能 Read 主会话本地文件；可以跑 Bash |
| **最大并发** | ~10 个 agent 同时运行 |
| **每 Workflow** | 最多 1000 个 agent 调用 |
| **budget** | T5 等开放循环必须加 budget guard：`while (dryRuns < 2 && (!budget.total \|\| budget.remaining() > 50_000))`；budget 耗尽时已收集结果仍可 return，不会丢失 |
| **失败处理** | agent 返回 null，脚本需 filter(Boolean)；Workflow 报错则修正重试 |
| **详细 API 参考** | 见 `references/workflow-runtime-api.md` 和 `references/workflow-tool-api.md` |
