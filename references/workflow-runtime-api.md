# Workflow 脚本运行时 API 参考

Workflow 脚本运行在 JS 沙箱中，不需要 import 任何东西——所有 API 都是全局可用的。

---

## `meta`（模块导出）

每个脚本**必须**以 `export const meta = {...}` 开头。这是 Workflow 工具识别和展示脚本的元数据。

```javascript
export const meta = {
  name: 'my-workflow',                    // 必须：唯一标识
  description: '一句话描述做什么',          // 必须：显示在工作流列表中
  phases: [                                // 可选：进度分组定义
    { title: '扫描', detail: '扫描文件' },
    { title: '处理', detail: '处理数据' },
    { title: '综合', detail: '生成报告' },
  ],
}
```

`meta` 必须是纯字面量对象——不能用变量、函数调用、模板字符串、展开运算符。

---

## `agent(prompt, opts?)`

启动一个子代理执行任务。

### 参数

| 参数 | 类型 | 必需 | 默认 | 说明 |
|------|------|------|------|------|
| `prompt` | `string` | 是 | — | 子代理的任务描述文本 |
| `opts.label` | `string` | 否 | 自动 | 进度显示名 |
| `opts.phase` | `string` | 否 | 当前 phase | 显式指定所属分组（用于 pipeline 内避免 race） |
| `opts.schema` | `JSON Schema` | 否 | 无 | 强制子代理输出结构化数据 |
| `opts.model` | `string` | 否 | 继承主会话 | 模型覆盖：`'sonnet'`/`'opus'`/`'haiku'`/`'fable'` |
| `opts.effort` | `string` | 否 | 继承主会话 | 推理力度：`'low'`/`'medium'`/`'high'`/`'xhigh'`/`'max'` |
| `opts.isolation` | `string` | 否 | 无 | `'worktree'` — 在独立 git worktree 中运行（昂贵） |
| `opts.agentType` | `string` | 否 | 通用 | 子代理类型，如 `'Explore'`、`'code-reviewer'` |

### 返回值

- **无 schema**：返回子代理的最终输出文本（`string`）
- **有 schema**：返回经过 schema 验证的结构化对象（`object`）
- **失败**：返回 `null`（用户取消或 API 错误耗尽重试）

### 示例

```javascript
// 简单文本输出
const text = await agent('分析这段代码的问题')

// 结构化输出
const result = await agent('分析这个文件', {
  schema: {
    type: 'object',
    properties: {
      title: { type: 'string' },
      score: { type: 'number' },
      issues: { type: 'array', items: { type: 'string' } },
    },
    required: ['title', 'score'],
  }
})
// result → { title: 'xxx', score: 85, issues: ['a', 'b'] }

// 控制模型和推理力度
const deep = await agent('深入分析这段代码的安全漏洞', {
  effort: 'high',
  model: 'opus',
})
```

---

## `parallel(thunks)`

并发执行多个函数，等**所有**完成后才继续（屏障）。

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `thunks` | `Array<() => Promise<any>>` | 要并发执行的函数数组 |

### 返回值

`Promise<Array<any>>` — 每个 thunk 的返回值按顺序排列。失败的返回 `null`。

### 约束

- 最大 4096 个元素
- 并发上限 ~10（超出自动排队）
- 单一 thunk 失败不影响其他 thunk
- **不会 reject** — 失败项为 null

### 示例

```javascript
// 基本用法
const results = await parallel([
  () => agent('任务1'),
  () => agent('任务2'),
  () => agent('任务3'),
])
// results → [result1, result2, result3]

// 动态生成
const files = ['a.txt', 'b.txt', 'c.txt']
const results = await parallel(files.map((f, i) => () =>
  agent(`分析 ${f}`, { label: `文件 ${i+1}/${files.length}` })
))
```

---

## `pipeline(items, ...stages)`

让每个 item 依次流过所有 stage，stage 之间**无屏障**。

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `items` | `Array<any>` | 要处理的输入数组 |
| `...stages` | `Array<(prev, orig, idx) => Promise<any>>` | 处理函数链 |

### stage 回调参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `prev` | `any` | 上一个 stage 的返回值（stage1 时等于原 item） |
| `orig` | `any` | 原始输入 item |
| `idx` | `number` | 在输入数组中的索引 |

### 返回值

`Promise<Array<any>>` — 每个 item 的最终 stage 返回值。某 stage 失败则该项为 `null`。

### 示例

```javascript
// 三阶段处理
const results = await pipeline(
  items,
  item => stage1(item),                    // 阶段1
  prev => stage2(prev),                    // 阶段2
  (prev, orig, idx) => stage3(prev, orig), // 阶段3，可访问原始 item
)

// 真实场景：搜索→下载→分析
const results = await pipeline(
  searchQueries,
  query => agent(`搜索: ${query}`, {phase: '搜索'}),
  url => agent(`抓取: ${url}`, {phase: '抓取'}),
  content => agent(`分析内容摘要`, {phase: '分析'}),
)
```

---

## `phase(title)`

切换到指定的阶段分组。后续的 agent() 调用在进度显示中归入此组。

```javascript
phase('扫描')
agent(...)  // 显示在"扫描"组

phase('分析')
agent(...)  // 显示在"分析"组
```

**注意**：标题必须与 `meta.phases` 中定义的一致才能正确分组。在 pipeline 内用 phase() 切换可能有 race——应改用 agent() 的 `phase` 参数：

```javascript
// ✅ 推荐：在 pipeline 中显式指定
agent(`验证`, {phase: '验证'})

// ❌ 避免：在 pipeline 中切换 phase
phase('验证')
```

---

## `log(message)`

在进度树上输出一行消息。用于报告进度和提示。

```javascript
log('文件扫描完成，共 14 个文件')
log('已找到 5/10 个 bug')
log('警告：部分结果可能不完整')
```

**注意**：log 是"报进度"用的，不是"输出数据"用的。要输出数据用 `return`。

---

## `args`（全局变量）

Workflow 调用时传入的 `args` 参数原样暴露给脚本。

```javascript
// Workflow({ args: ["file1.txt", "file2.txt"] })
// 脚本中:
args  // → ["file1.txt", "file2.txt"]
```

---

## `budget`（全局变量）

Token 预算控制。

```javascript
budget.total       // 总预算（number 或 null——null 表示无限制）
budget.spent()     // 已消耗 token 数
budget.remaining() // 剩余 token 数（无限制时为 Infinity）
```

```javascript
// 预算感知循环
while (budget.total && budget.remaining() > 50_000) {
  const batch = await agent('继续搜索...')
  found.push(...batch)
  log(`${found.length} found, ${Math.round(budget.remaining()/1000)}k remaining`)
}
```

---

## 约束总结

| 约束 | 说明 |
|------|------|
| 禁止 `Date.now()` | 破坏可重放性 |
| 禁止 `Math.random()` | 破坏可重放性 |
| 禁止无参 `new Date()` | 破坏可重放性；可用 `args` 传入时间戳 |
| 禁止 Node.js API | 无 `fs`、`path`、`process` 等 |
| 纯 JavaScript | 不是 TypeScript（无类型注解） |
| meta 是纯字面量 | 不能用变量、函数调用、模板字符串 |
| 最多 1000 个 agent | 防止 runaway 循环 |
| agent 失败返回 null | 不会抛异常到脚本 |
