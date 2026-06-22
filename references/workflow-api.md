# Workflow 脚本 API 参考

本文档是 `dynamic-workflow-automation` skill 的 API 参考。当需要精确的 API 细节时读取此文。

## 核心 API

### `agent(prompt, opts?)`

启动一个子代理执行任务。

| 参数 | 类型 | 说明 |
|------|------|------|
| `prompt` | `string` | 子代理的任务描述 |
| `opts.label` | `string` | 进度显示名 |
| `opts.phase` | `string` | 所属 phase 分组（避免 pipeline 中 race） |
| `opts.schema` | `JSON Schema` | 强制结构化输出 |
| `opts.model` | `'sonnet'\|'opus'\|'haiku'\|'fable'` | 模型覆盖 |
| `opts.effort` | `'low'\|'medium'\|'high'\|'xhigh'\|'max'` | 推理力度 |
| `opts.isolation` | `'worktree'` | 是否需要隔离 git worktree |
| `opts.agentType` | `string` | 子代理类型（Explore等） |

**返回值**：`Promise<any>`
- 无 schema → 子代理最后输出的文本（字符串）
- 有 schema → 验证后的结构化对象
- 用户取消或 API 错误耗尽重试 → `null`

### `parallel(thunks)`

并发执行多个函数，等所有完成后再返回。

```javascript
const results = await parallel([
  () => agent('任务1'),
  () => agent('任务2'),
  () => agent('任务3'),
])
// results: [result1, result2, result3]（失败项为 null）
```

**并行上限**：~10（超过的排队等待）
**最大项数**：4096
**注意**：这是 BARRIER——等所有完成才继续。

### `pipeline(items, ...stages)`

让每个 item 依次流过所有 stage，stage 之间无屏障。

```javascript
const results = await pipeline(
  items,                           // 输入数组
  item => stage1(item),            // 每个 item 走 stage1
  prev => stage2(prev),            // 完成后立即走 stage2
  (prev, orig, idx) => stage3(prev, orig, idx),  // stage3 可访问原始 item
)
```

- 每个 stage 回调参数：`(prevResult, originalItem, index)`
- 某 stage 抛异常 → 该项变为 null，跳过后续 stage

### `phase(title)`

切换当前 phase 分组。之后的 agent() 调用显示在此分组下。

```javascript
phase('扫描')   // → 后续 agent 在"扫描"组
agent(...)
phase('分析')   // → 后续 agent 在"分析"组
agent(...)
```

### `log(message)`

在进度树上输出一行消息。用于报告进度，不是输出数据。

## 全局变量

### `args`

用户在 Workflow 工具调用时传入的 `args` 参数，原样传入。

```javascript
// Workflow 调用： { args: ["file1.txt", "file2.txt"] }
// 脚本中：
args  // → ["file1.txt", "file2.txt"]
```

### `budget`

Token 预算控制。

```javascript
budget.total       // 预算总数（null 表示无限制）
budget.spent()     // 已消耗 token 数
budget.remaining() // 剩余 token 数（无限制时为 Infinity）
```

## 约束

- **禁止**：`Date.now()`, `Math.random()`, `new Date()`（无参）
- **语言**：纯 JavaScript（无 TypeScript 类型注解）
- **文件系统**：无 Node.js API（不能 `fs.readFileSync` 等）
- **子代理权限**：不继承主会话的全部工具权限
