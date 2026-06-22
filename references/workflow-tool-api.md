# Workflow 工具 API 参考

Workflow 是 Claude Code 内置的编排工具（类型  `Workflow`），用于执行 JS 脚本来编排多个子代理的协作。

## 调用参数

```javascript
Workflow({
  script: "export const meta = {...}; ...",  // JS 脚本内容（二选一，与 scriptPath 互斥）
  name: "task-name",                          // 预定义工作流名称（可选）
  scriptPath: "/path/to/script.js",           // 从文件加载脚本（二选一，与 script 互斥）
  args: ["file1.txt", "file2.txt"],           // 传给脚本 args 变量的数据（可选）
  resumeFromRunId: "wf_xxx",                 // 恢复之前的运行（可选）
  description: "简短描述",                     // 仅用于展示
})
```

### 参数详解

| 参数 | 类型 | 说明 |
|------|------|------|
| `script` | `string` | 完整的 JS 脚本内容。必须以 `export const meta = {...}` 开头。最大 512KB。 |
| `name` | `string` | 预定义工作流名称（来自 .claude/workflows/）。不与 script/scriptPath 同时使用。 |
| `scriptPath` | `string` | 已存在磁盘上的脚本文件路径。优先于 `script`。 |
| `args` | `any` | 传给脚本的 `args` 全局变量。**必须是实际值，不是 JSON 字符串**——传数组就是数组，不是 `"[\"a\"]"`。 |
| `resumeFromRunId` | `string` | 同一会话中之前 Workflow 运行的 ID。已完成的 agent() 调用返回缓存结果，只重新运行修改后的调用。 |
| `description` | `string` | 显示用，不在脚本中可用。 |

## 返回值

Workflow 工具返回一个对象，其中 `result` 字段是脚本中 `return` 语句返回的值：

```
{
  report: "...",       // 脚本 return 的任意值
  fileCount: 14,
  totalSizeKB: "792.3",
  summary: "...",
  // 脚本 return 的任何字段都在 result 中
}
```

此外，工具结果还提供：
- `agentCount`: 脚本中启动的子代理总数
- `logs`: 脚本中 log() 输出的消息数组
- 如果脚本返回的对象中包含 `report` 字段，工具会额外在 `report` 字段中返回（便于展示）

## 运行机制

1. 脚本在安全的 JS 沙箱中执行（无 Node.js API）
2. agent() 调用启动子代理，子代理拥有自己的工具集
3. 并发上限 ~10 个 agent（超出排队）
4. 单个 Workflow 最多 1000 个 agent 调用
5. 单个 parallel/pipeline 最多 4096 个元素
6. 脚本内禁止 `Date.now()` / `Math.random()` / 无参 `new Date()` 保证可重放
7. 子代理失败返回 null（不抛异常到脚本）

## 恢复执行（Resume）

用 `resumeFromRunId` 可以在脚本修改后恢复之前的运行：

```
Workflow({
  scriptPath: "script.js",        // 修改后的脚本
  resumeFromRunId: "wf_xxx",      // 前一次运行的 ID
})
```

已完成的 agent() 调用（prompt + opts 完全一致）返回缓存结果，只有新增/修改的调用重新执行。这避免了重跑整个工作流。

**约束**：仅限同一会话内；需要先停掉之前的运行（TaskStop）。
