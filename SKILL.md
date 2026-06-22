---
name: dynamic-workflow-automation
description: |
  Workflow 编排引擎 — 用 Workflow 工具的 JS 脚本（agent/parallel/pipeline/phase/loop）
  将多步骤任务编排为自动化流水线。核心价值是让 Claude 同时做 N 件事（并行子代理），
  再汇聚结果。

  ALWAYS 使用此 skill 当任务涉及以下任一特征：
  - "并行处理多个文件/数据源/网站" — 需要同时对 N 个目标执行相同操作再汇总
  - "流水线处理" — 数据需要经过多个阶段，后阶段依赖前阶段输出
  - "循环发现" — 不确定要做多少次（持续找 bug、迭代优化、直到没有新发现）
  - "多视角验证" — 从多个角度分析同一件事再综合判断
  - "大工作量" — 明显超过单次处理极限的任务（用 ultracode/workflow 的标配）

  即使用户没有明确说"工作流"或"并行"，只要任务明显有以上特征，就应该用此 skill。
  对于 FAN-OUT 类型的任务，Workflow 脚本是实现并行的唯一方式——不要尝试串行处理。

  不适用于：单步简单任务（问个问题、改一行代码）、需要实时交互（用户必须中途输入）。
---

# Dynamic Workflow Automation

## 哲学：为什么用 Workflow 而不是直接做

你在单次对话中只能做**一件事**（虽然可以用 Agent 工具启动子代理，但启动后你就得等）。
Workflow 脚本打破了这堵墙：

- **并行** — 同时让 10+ 个子代理干各自的活
- **编排** — 精确控制谁先干、谁后干、谁等谁
- **循环** — "直到没有新发现"这种动态终止条件
- **预算感知** — 用户说"+500k"时自动扩大搜索范围

Workflow 不适合的场景：
- 只需要 1-2 步就能完成的任务 → 直接干
- 用户需要在过程中输入 → 用普通方式
- 你不知道要做什么，需要先探索 → 先探索再决定要不要上 Workflow

## 架构模式：四步设计法

每次写 Workflow 脚本前，先过这四步：

```
Step 1: 列出所有"可并行的独立单元"
  → "我要对 14 个文件分别做分析" → 14 个独立单元
  → "我要从 5 个维度审阅这段代码" → 5 个独立单元
  → "我要搜索 3 个不同的数据源" → 3 个独立单元

Step 2: 确定有没有阶段依赖
  → 所有单元独立 → 一个 phase + parallel() 搞定
  → 需要先发现再处理 → 两个 phase（发现 → 处理）
  → 需要发现 → 处理 → 验证 → 三个 phase（发现 → 处理 → 验证）
  → 多个阶段带汇聚 → 混合流水线（见下文模式库）

Step 3: 选择汇聚方式
  → 全部结果收集起来给一个 agent 分析 → agent(综合)
  → 每个结果独立输出（不需要汇聚）→ 每个 agent 直接输出文件
  → 需要多轮迭代 → 循环模式

Step 4: 确定约束
  → 子代理能否直接完成任务？还是需要主会话先准备数据？
  → 有没有 token 预算限制？——用 budget.remaining() 控制深度
```

## Workflow 脚本结构

### 模板骨架

```javascript
export const meta = {
  name: 'my-workflow',
  description: '一句话描述这个 workflow 做什么',
  phases: [
    { title: '发现', detail: '扫描/搜索/发现所有待处理项' },
    { title: '处理', detail: '并行处理每项' },
    { title: '综合', detail: '汇聚结果生成输出' },
  ],
}

// Phase 1: 发现（通常是串行的前置步骤）
phase('发现')
const items = await agent('扫描/搜索/发现任务...', {schema: ITEMS_SCHEMA})
log(`发现 ${items.length} 个待处理项`)

// Phase 2: 并行处理
phase('处理')
const results = await parallel(items.map(item => () =>
  agent(`处理这个项: ${item.name}...`, {label: `处理:${item.name}`, phase: '处理', schema: RESULT_SCHEMA})
))

// Phase 3: 汇聚综合
phase('综合')
const final = await agent(`综合以下结果...\n${JSON.stringify(results.filter(Boolean))}`, {label: '综合分析'})
return final
```

### 必记约束

| 约束 | 说明 | 绕过方式 |
|------|------|---------|
| `new Date()` / `Date.now()` / `Math.random()` | 脚本内禁止使用（破坏可重放） | 在主会话中获取时间/随机数，通过 `args` 传入 |
| 子代理不能读主会话本地文件 | agent() 启动的子代理没有主会话的 Read 权限 | 用 Bash 先提取内容传给 agent；或用本地脚本读取后通过 agent prompt 传入 |
| 脚本是 JS 不是 TS | 不能用类型注解 | 用 JSDoc 注释代替 |
| 最大并发 ~10 | 超过的排队等待 | 正常，不需要手动控制 |
| 单次 parallel/pipeline 最多 4096 项 | 超大集合需分片 | 分多次 parallel 调用 |
| 总 agent 数上限 1000 | 防止 runaway | 用 budget + loop-until-dry 控制 |

## 模式库

### 模式 1：Fan-Out 并行（最常用）

**适用场景**：有一批同质化数据要处理——多个文件、多个 URL、多个维度。

**结构**：
```
phase('处理')
results = await parallel(items.map(item => () =>
  agent(`处理 ${item}`, {label: `proc:${item}`})
))
phase('综合')
output = await agent(`综合 ${results.length} 个结果...`)
```

**真实例子**（来自 rq 目录分析）：
```javascript
phase('内容读取')
const fileContents = await parallel(files.map(f => () =>
  agent(`读取文件 ${f.name} 的标题和摘要信息`, {label: f.name, phase: '内容读取'})
))
phase('综合分析')
const summary = await agent(`综合以下 ${files.length} 个文件的分析...`)
```

**关键决策**：结果要不要汇聚？→ 要就用两个 phase，不要就直接 return。

---

### 模式 2：Pipeline 流水线（无屏障）

**适用场景**：每个数据项需要经过 N 个阶段，但不需要等所有项完成阶段 1 才进入阶段 2。

**结构**：
```javascript
const results = await pipeline(
  items,
  item => step1(item),    // 每个 item 独立走 stage1
  prev => step2(prev),    // 完成后立即走 stage2，不等其他 item
  prev => step3(prev),    // stage2 完成后立即走 stage3
)
```

**为什么用 pipeline 而不是 parallel + parallel**：
- pipeline 在每项内部串行，在项之间无屏障——A 可以在 stage3 而 B 还在 stage1
- 总耗时 ≈ 最慢的一项的链式总时间，不是各阶段最慢项的总和
- 只有当你需要**在所有结果上做汇聚/去重**时才用 parallel（加屏障）

---

### 模式 3：Pipeline with Barrier（有屏障）

**适用场景**：需要在所有项完成阶段 1 后，在全局结果上做一些操作（去重、排序、过滤），然后再进入阶段 2。

**结构**：
```javascript
// Barrier: 等 ALL 完成阶段 1
const stage1Results = await parallel(items.map(i => () => agent(`阶段1: ${i}`)))
// 在全局结果上操作后才能决定阶段 2 做什么
const deduped = dedupeByKey(stage1Results.filter(Boolean))
// 然后并行阶段 2
const stage2Results = await parallel(deduped.map(r => () => agent(`阶段2: ${r}`)))
```

**何时必须用 barrier**：
- 去重/合并后才做昂贵的下游工作
- 总数太少或太多需要调整策略（"0 个 bug→跳过验证"）
- 阶段 2 的 prompt 需要引用"其他发现"做比较

---

### 模式 4：循环发现（Loop-Until-Dry）

**适用场景**：不知道有多少东西要发现——持续找 bug、持续搜索直到没有新发现。

**结构**：
```javascript
const all = new Set()
let dryRuns = 0
while (dryRuns < 2) {        // 连续 2 轮没有新发现才停
  const batch = await agent('找新东西...', {schema: BATCH})
  const fresh = batch.filter(b => !all.has(key(b)))
  if (fresh.length === 0) { dryRuns++; continue }
  dryRuns = 0
  fresh.forEach(b => all.add(key(b)))
  // 处理这批新发现...
}
```

**关键参数**：
- `dryRuns < 2` — 连续 2 轮空就跑完。太高浪费 token，太低可能遗漏
- 如果已知上限（"找 10 个 bug"），用 `while (all.size < 10)` 更简单

---

### 模式 5：预算感知循环（Loop-Until-Budget）

**适用场景**：用户设了 token 预算（"+500k"），你想把预算花光来找到尽可能多的结果。

**结构**：
```javascript
const found = []
while (budget.total && budget.remaining() > 50_000) {
  const batch = await agent('继续找...', {schema: BATCH})
  found.push(...batch)
  log(`${found.length} found, ${Math.round(budget.remaining()/1000)}k remaining`)
}
// 无预算时用普通数量限制兜底
while (!budget.total && found.length < 20) {
  // ...常规数量上限循环
}
```

### 模式 6：对抗验证（Adversarial Verify）

**适用场景**：某个 agent 产生了结果（找到 bug、得出结论），你想确认它是不是真的。

**结构**：
```javascript
// 并行启动多个"反驳者"，每个独立尝试拒绝这个发现
const votes = await parallel(Array.from({length: 3}, () => () =>
  agent(`尝试反驳这个发现: ${claim}。不确定时就认为它是假的。`, {schema: VERDICT})))
const survives = votes.filter(Boolean).filter(v => !v.refuted).length >= 2
```

**变体：视角多样化验证**——不是 N 个相同的"反驳者"，而是 N 个不同视角的验证者：
```javascript
const lenses = ['正确性', '安全性', '可复现性']
const judged = await parallel(lenses.map(lens => () =>
  agent(`从 ${lens} 视角判断这个发现是否真实`, {schema: VERDICT})))
const real = judged.filter(Boolean).filter(v => v.real).length >= 2
```

---

### 模式 7：评判小组（Judge Panel）

**适用场景**：需要从多个角度独立尝试解决问题，再选最优方案。

**结构**：
```javascript
// 独立生成多个方案
const approaches = await parallel([
  () => agent('从 MVP 优先的角度设计解决方案...'),
  () => agent('从风险最小化的角度设计...'),
  () => agent('从用户体验最优的角度设计...'),
])
// 独立评判各方案
const scores = await parallel(approaches.map(a => () =>
  agent(`评估以下方案的优缺点并打分...\n${a}`)))
// 选择最优 + 融合
const best = synthesizeBest(approaches, scores)
```

## 脚本编写指南

### 1. 如何传数据给子代理

子代理通过 agent() 的 prompt 参数接收上下文。不要试图传文件路径让子代理去读：

```javascript
// ❌ 子代理读不了本地文件
agent(`分析 /path/to/file.html 的内容`)

// ✅ 用 Bash 先提取，把内容传给 agent
agent(`分析以下 HTML 文件的内容：\n标题: ${title}\n大小: ${size}\n内容摘要: ${content}`)

// ✅ 或者让子代理自己跑 Bash（如果它有 Bash 权限）
agent(`用 cat 读取 /path/to/file.html，分析其标题和 H1 标签`)
```

### 2. 如何从子代理拿结构化数据

用 `schema` 参数让 agent 返回结构化对象（JSON Schema）：

```javascript
const result = await agent('分析这个文件', {
  schema: {
    type: 'object',
    properties: {
      title: { type: 'string' },
      score: { type: 'number' },
      tags: { type: 'array', items: { type: 'string' } },
    },
    required: ['title', 'score'],
  }
})
// result 是 { title: 'xxx', score: 85, tags: ['a', 'b'] }
```

不加 schema 时 agent() 返回子代理的最终文本输出。

### 3. 如何控制子代理的行为

```javascript
agent(prompt, {
  label: '显示名',          // 覆盖进度显示
  phase: '阶段名',         // 显式分组（用于避免 race condition）
  schema: {...},           // 结构化输出
  model: 'sonnet',         // 模型覆盖（默认继承主会话）
  effort: 'low',           // 推理力度：low/medium/high
  isolation: 'worktree',   // 需要隔离的工作目录
  agentType: 'Explore',    // 使用特定子代理类型
})
```

**effort 选择**：
- `low` — 便宜机械任务（提取标题、分类、简单判断）
- `medium`（默认）— 大多数情况
- `high` / `xhigh` — 最难的分析/验证/评判工作

### 4. 如何组织 phase 分组

```javascript
phase('扫描')     // 之后的 agent() 显示在"扫描"组下
agent(...)        // → 扫描组
agent(...)        // → 扫描组
phase('分析')     // 切换分组
agent(...)        // → 分析组
```

在 `meta.phases` 中声明的 phase 会有专门的进度分组；没声明的也有，只是没有独立进度条。
phase 标题在 meta.phases 和 phase() 调用中必须一致匹配。

### 5. 错误的优雅处理

```javascript
// agent() 失败返回 null（不抛异常）
const results = await parallel(items.map(i => () => agent(`处理 ${i}`)))
// 用 filter(Boolean) 跳过失败项
const valid = results.filter(Boolean)

// pipeline() 某阶段失败 → 该项变为 null，后续阶段跳过
const pipelineResults = await pipeline(items,
  i => step1(i),
  prev => prev ? step2(prev) : null,  // 防止 null 传播
)
```

### 6. log() 的正确用法

`log()` 在进度树上输出一行消息，用于：
- 报告中间进度（"找到 5/10 个 bug"）
- 提示关键决策点
- 警告异常情况

不要用 log() 输出原始数据，那是 return 做的事。

## 常见陷阱

### 陷阱 1：子代理不继承主会话的工具权限

这是最常见的失败模式。子代理的可用工具有自己的限制——默认不能 Read 本地文件、不能 Edit/Write 到主会话的文件系统。**不要假设子代理能做什么**，要么在 prompt 里明确让它跑 Bash，要么由主会话准备好数据传给 agent。

### 陷阱 2：忘记 filter(Boolean)

`agent()` 可能返回 null（用户取消、API 错误后重试耗尽）。如果不 filter，后面的综合 agent 会读到一堆 null，影响结果质量。

### 陷阱 3：parallel 的 barrier 用错

```javascript
// ❌ 不需要 barrier 却用了 parallel
const a = await parallel([agent('X'), agent('Y')])
const b = transform(a)  // 只是 flatten/filter，不需要等 ALL
const c = await parallel(b.map(...))

// ✅ 用 pipeline 消除不必要的 barrier
const c = await pipeline(items, stage1, r => transform([r]).flat(), stage2)
```

判断标准：阶段 N 是否需要"看其他所有阶段 N-1 的结果"来判断做什么？
- 去重 → 需要 barrier
- 计数 → 需要 barrier（"0 个→跳过"）
- 比较结果 → 需要 barrier
- 简单的过滤/映射 → 不需要 barrier

### 陷阱 4：没有 meta.phases 定义

每个 workflow 脚本**必须**以 `export const meta = { name, description, phases }` 开头。
`phases` 数组是可选的但强烈推荐——没有它进度显示不分组。

### 陷阱 5：在 pipeline 内用 phase() 的 Race Condition

```javascript
// ❌ 可能出问题：pipeline 内多个 agent 同时执行，phase() 切换不原子
const results = await pipeline(items,
  r => { phase('验证'); return agent(`验证 ${r}`) },
)

// ✅ 用 label 的 phase 参数显式指定
const results = await pipeline(items,
  r => agent(`验证 ${r}`, {phase: '验证'}),
)
```

## 决策速查表

```
任务开始
│
├─ 只需 1-2 步？──────→ 直接做，不需要 Workflow
│
├─ 有 N 个独立单元要处理？
│   ├─ 各单元互不依赖，一次处理完 ──→ 模式 1: Fan-Out
│   ├─ 各单元需经多阶段，阶段间不等待 ──→ 模式 2: Pipeline
│   └─ 各单元需经多阶段，阶段间有全局操作 ──→ 模式 3: Pipeline+Barrier
│
├─ 不知道有多少东西要处理？
│   ├─ 有 token 预算上限 ──→ 模式 5: Budget-Loop
│   └─ 没有预算上限 ──→ 模式 4: Loop-Until-Dry
│
├─ 需要验证发现是否真实？
│   ├─ 同一角度反复验证 ──→ 模式 6: Adversarial
│   └─ 多角度验证 ──→ 模式 6 变体: 多维验证
│
└─ 需要从多个方案中选最优？─→ 模式 7: Judge Panel
```

## 完整示例：文件目录分析

这是你之前成功跑过的例子，作为参考模板：

```javascript
export const meta = {
  name: 'analyze-directory',
  description: '扫描目录中的所有文件，并行分析内容，生成汇总报告',
  phases: [
    { title: '文件扫描', detail: '获取文件元数据和初步分类' },
    { title: '内容分析', detail: '并行读取和分析每个文件' },
    { title: '综合报告', detail: '汇聚所有结果生成最终报告' },
  ],
}

// Step 1: 文件扫描（硬编码或用 Bash 获取）
phase('文件扫描')
log('扫描目录...')
const files = [
  // ...从 Bash ls 输出提取，或硬编码
]

// Step 2: 并行分析每个文件
phase('内容分析')
const results = await parallel(files.map(f => () =>
  agent(`分析文件 ${f.name}...\n大小: ${f.size}`, {
    label: f.name,
    phase: '内容分析',
    schema: { type: 'object', properties: {
      title: { type: 'string' },
      type: { type: 'string' },
      summary: { type: 'string' },
    }, required: ['title', 'type'] }
  })
))

// Step 3: 综合报告
phase('综合报告')
const report = await agent(
  `以下是对 ${files.length} 个文件的分析结果，请综合汇总：\n` +
  JSON.stringify(results.filter(Boolean)),
  { label: '生成汇总报告' }
)

return { report, totalFiles: files.length, results: results.filter(Boolean) }
```

## 与其它工具的关系

- **Agent 工具**（非 Workflow 中的 agent()）：适合启动一个独立的后台任务。和 Workflow 脚本的区别是，Workflow 让你在 JS 中编排多个 agent 的协作。
- **TaskCreate/TaskUpdate**：适合在常规对话中跟踪步骤进度。Workflow 的 phase() 是其工作流版的等价物。
- **CronCreate**：适合定时任务。Workflow 是一次性编排，不适合"每天早上 9 点运行"的场景。
