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

## Step 2：模式选择 — 7 个基础模板 + 变体

根据分析结果选择模板。每个模板下可能包含多个变体，根据任务特征选对应变体。
直接跳到对应的模板处复制适配。

### 模板 1：Fan-Out + 汇聚

**适用条件**：N 个独立对象各自处理后汇聚成一份综合结果。
**最常用模式**，适用于文件分析、批量搜索、多源数据提取等。

**完整性批评变体**：如果担心遗漏了什么维度/来源，在汇聚前加一个 agent 查漏补缺：
```javascript
// 完整性批评（可选，加在汇聚前）
const critique = await agent(
  `检查以下结果是否完整。缺失了什么维度？哪个来源没覆盖？\n${JSON.stringify(results.filter(Boolean), null, 2)}`,
  { schema: { type: 'object', properties: { missing: { type: 'array', items: { type: 'string' } }, complete: { type: 'boolean' } }, required: ['missing', 'complete'] } }
)
if (!critique.complete) {
  // 根据 critique.missing 补充搜索，然后重新汇聚
}
```

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
    // 推荐加 schema 让子代理返回结构化数据，方便汇聚阶段处理
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
    // 需要结构化输出时加 schema 参数
  })
))
return { results: results.filter(Boolean) }
```

---

### 模板 3：Pipeline（无屏障）

**适用条件**：每个单元需经多阶段处理，且各阶段间**不需要跨单元聚合**（无去重、无排序、无过滤阈值）。选 T3 还是 T4 的判据：阶段切换时是否需要拿到**所有单元的结果**才能继续？不需要 → T3，需要 → T4。

**屏障（barrier）正确的 3 种场景**（选 T4）：① 在昂贵下游工作前对整个结果集去重/合并；② 总数为零时提前退出（"0 个结果 → 跳过全部"）；③ Stage 的 prompt 需要引用"其他发现"做比较。

**屏障不合理的 3 种场景**（应选 T3）：① 只是先 flatten/map/filter 再继续——在 pipeline 的 stage 内部做就行；② 觉得阶段"概念上是分开的"——分开的阶段 ≠ 同步的阶段；③ 觉得"代码更干净"——屏障延迟真实存在，5 个任务最慢比最快慢 3 倍，屏障浪费 2/3。

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

### 模板 5：循环发现（3 种变体）

**适用条件**：需要循环搜索/发现/迭代，直到某个条件满足。

根据 Step 1 分析的"单元数量"判断用哪个变体：
- **已知目标数量**（如"找 10 个案例"）→ 变体 B Loop-Until-Count
- **需要 agent 自行判断完成与否**（如"把所有安全漏洞找出来"）→ 变体 C Loop-Until-Evaluator
- **其他情况**（如"搜到没有新发现为止"）→ 变体 A Loop-Until-Dry

**⚠️ budget guard 强制要求**：所有循环变体都必须加 budget guard——`(!budget.total \|\| budget.remaining() > 50_000)`。预算耗尽时已收集结果仍可 return，不会丢失。

---
#### 变体 A：Loop-Until-Dry

**适用**：不知道有多少发现，连续空转后停止。

```javascript
export const meta = {
  name: 'task-loop-dry',
  description: '任务描述',
  phases: [{ title: '循环发现', detail: '搜索直到枯竭' }],
}

phase('循环发现')
const found = []
const seen = new Set()
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
                key: { type: 'string' },
                data: { type: 'string' },
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
#### 变体 B：Loop-Until-Count

**适用**：明确知道要收集多少个（如"找 10 个客户案例"），收集够了就停。

```javascript
export const meta = {
  name: 'task-loop-count',
  description: '任务描述',
  phases: [{ title: '循环发现', detail: '搜索直到满足目标数' }],
}

phase('循环发现')
const found = []
const seen = new Set()
const TARGET = 10  // 改为目标数量
while (found.length < TARGET && (!budget.total || budget.remaining() > 50_000)) {
  const batch = await agent(
    `【搜索任务描述】\n已找到 ${found.length}/${TARGET} 个\n\n返回本轮新发现。`,
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
                key: { type: 'string' },
                data: { type: 'string' },
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
  newItems.forEach(x => { seen.add(x.key); found.push(x) })
  log(`已找到 ${found.length}/${TARGET}`)
}
return { results: found }
```

---
#### 变体 C：Loop-Until-Evaluator

**适用**：需要让独立的评估 agent 判断"做完了没有"（如"把所有安全漏洞找出来"——完成标准不是数量而是质量）。

**⚠️ budget guard 不可省略**——没有它会无限循环。

```javascript
export const meta = {
  name: 'task-loop-evaluator',
  description: '任务描述',
  phases: [{ title: '循环迭代', detail: '迭代直到评估通过' }],
}

phase('循环迭代')
const results = []
// evaluator 的 schema：必须含 done（是否完成）和 feedback（反馈给下一轮的改进方向）
const EVAL_SCHEMA = {
  type: 'object',
  properties: {
    done: { type: 'boolean' },
    feedback: { type: 'string' },
    coverage: { type: 'string' },
  },
  required: ['done', 'feedback'],
}
while (!budget.total || budget.remaining() > 50_000) {
  // 1. 工作 agent
  const batch = await agent(
    `【任务描述】${results.length > 0 ? '\n已轮到的反馈：' + results[results.length-1].feedback : ''}`,
    { phase: '循环迭代', schema: { /* 任务对应的 schema */ } }
  )
  if (!batch) continue
  results.push(batch)

  // 2. 独立评估 agent——判断是否完成
  const evalResult = await agent(
    `评估以下结果是否已完成任务目标。若未完成，说明还需要做什么：\n${JSON.stringify(batch, null, 2)}`,
    { phase: '循环迭代', schema: EVAL_SCHEMA }
  )
  if (!evalResult) continue

  // 3. 检查终止
  if (evalResult.done) {
    log(`评估通过：${evalResult.coverage || ''}`)
    break
  }
  // 未完成：把 feedback 传给下一轮的工作 agent
  log(`需继续：${evalResult.feedback}`)
}
return { results }
```

---

### 模板 6：验证（2 种变体）

**适用条件**：需要验证结论/发现的真实性。

根据验证目的选变体：
- **验证结论是否可信**，多个同视角验证者尝试反驳 → **变体 A 对抗验证**
- **从不同维度评估**（正确性/安全性/性能等） → **变体 B 多视角验证**

**验证者数量**：普通任务 3 个，高风险/重要任务 5 个。

---
#### 变体 A：对抗验证

**适用**：多个独立验证者以相同标准尝试反驳同一结论。

```javascript
export const meta = {
  name: 'task-adversarial',
  description: '对抗验证',
  phases: [
    { title: '发现', detail: '生成结论' },
    { title: '验证', detail: '并行验证' },
  ],
}

phase('发现')
const findings = await agent('【提取/生成结论】', {phase: '发现'})

phase('验证')
const VERIFIER_COUNT = 3  // 高风险任务改为 5
const votes = await parallel(Array.from({length: VERIFIER_COUNT}, (_, i) => () =>
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
#### 变体 B：多视角验证

**适用**：需要从不同维度评估同一产出——每个验证者负责一个独立视角（正确性/安全性/性能/可复现性等），不同视角覆盖不同失败模式。

```javascript
export const meta = {
  name: 'task-multi-lens',
  description: '多视角验证',
  phases: [
    { title: '发现', detail: '生成结论' },
    { title: '验证', detail: '多视角并行验证' },
  ],
}

phase('发现')
const findings = await agent('【提取/生成结论】', {phase: '发现'})

phase('验证')
const LENSES = ['正确性', '安全性', '性能', '可复现性']  // 根据任务调整视角
const votes = await parallel(LENSES.map(lens => () =>
  agent(`以「${lens}」视角评估以下结论：是否存在问题？请给出评分（1-10）和说明。\n\n${findings}`, {
    phase: '验证',
    schema: {
      type: 'object',
      properties: {
        score: { type: 'number', minimum: 1, maximum: 10 },
        issues: { type: 'array', items: { type: 'string' } },
        pass: { type: 'boolean' },
      },
      required: ['score', 'issues', 'pass'],
    },
  })
))
const avgScore = votes.filter(Boolean).reduce((s, v) => s + v.score, 0) / votes.filter(Boolean).length
const allPass = votes.filter(Boolean).every(v => v.pass)
return { findings, avgScore, allPass, details: votes.filter(Boolean) }
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
  `综合选最优，可参考其他方案的优点进行融合:\n${JSON.stringify(scores.filter(Boolean))}`,
  { phase: '综合' }
)
return { approaches, scores, best }
```

---

## 模式组合指南

以下是一些常用的组合模式。组合时直接从对应模板复制代码并在同一个 Workflow 脚本中拼接即可。

### 组合 1：Multi-modal Sweep（多模态扫描）

**场景**：同一件事需要从多个不同角度搜索，单一搜索角度可能遗漏。
**实现**：T1 Fan-Out + 每个 agent 用不同搜索策略。

```javascript
phase('多模态搜索')
const STRATEGIES = ['按容器搜索', '按内容搜索', '按实体搜索', '按时间搜索']  // 根据任务调整
const results = await parallel(STRATEGIES.map((s, i) => () =>
  agent(`以「${s}」策略执行:\n【任务描述】`, {
    label: `策略 ${i+1}/${STRATEGIES.length}`,
    phase: '多模态搜索',
  })
))
phase('汇聚')
const final = await agent(`综合各策略结果:\n${JSON.stringify(results.filter(Boolean), null, 2)}`)
return final
```

### 组合 2：发现 + 验证（从 T5 变体 C + T6 变体 A）

**场景**：循环发现（如找漏洞）找到后需要独立验证真实性。

```javascript
phase('循环发现')
// ... T5 变体 C 的 evaluator 循环 ...
return { findings, verified }
```

在 Workflow 脚本中，可以在 T5 的每轮 evaluator 通过后，直接跟 T6 的验证阶段。将 T5 的 return 改为把 findings 传给 T6 作为输入即可。

### 组合 3：完整性批评 + 对抗验证

**场景**：先检查是否遗漏了什么，再验证已找到的是否真实。

```javascript
// 用 T1 的完整性批评变体检查遗漏
// 用 T6 对抗验证确认已发现内容的真实性
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
| **无静默上限** | 若限制了覆盖范围（top-N、采样、只处理前 K 项等），必须用 `log()` 说明丢弃了什么、为什么、可能遗漏什么 |
| **失败处理** | agent 返回 null，脚本需 filter(Boolean)；Workflow 报错则修正重试 |
| **详细 API 参考** | 见 `references/workflow-runtime-api.md` 和 `references/workflow-tool-api.md` |
