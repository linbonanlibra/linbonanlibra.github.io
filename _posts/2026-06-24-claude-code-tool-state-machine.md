# Claude Code 工具执行状态机分析

> 分析日期：2026-06-24  
> 源码路径：`src/services/tools/StreamingToolExecutor.ts`

---

## 状态定义

```typescript
// StreamingToolExecutor.ts:19
type ToolStatus = 'queued' | 'executing' | 'completed' | 'yielded'
```

每个被跟踪的工具（`TrackedTool`）携带完整的执行上下文：

```typescript
// StreamingToolExecutor.ts:21-32
type TrackedTool = {
  id: string
  block: ToolUseBlock
  assistantMessage: AssistantMessage
  status: ToolStatus                // 当前状态
  isConcurrencySafe: boolean        // 是否允许并发
  promise?: Promise<void>
  results?: Message[]               // 收集到的执行结果
  pendingProgress: Message[]        // 实时进度消息
  contextModifiers?: Array<(context: ToolUseContext) => ToolUseContext>
}
```

---

## 状态转换流程

```
addTool() 调用
     ↓
  [queued]  ──→ 等待并发条件满足 ──→ [executing] ──→ [completed] ──→ [yielded]
  已入队列                           工具运行中       执行结束        结果已送出（终态）
```

---

## 四个状态详解

### 1. `queued` — 排队等待

| 项目 | 说明 |
|------|------|
| **触发位置** | `addTool()`（第 76–124 行） |
| **触发条件** | 工具定义存在、输入通过验证、`isConcurrencySafe` 已确定 |
| **含义** | 工具已登记入队列，尚未获得执行许可 |
| **后续行为** | 等待 `processQueue()` 调度，检查并发许可 |

---

### 2. `executing` — 正在执行

| 项目 | 说明 |
|------|------|
| **触发位置** | `executeTool()`（第 265 行） |
| **触发条件** | `canExecuteTool()` 返回 `true` |
| **含义** | 工具正在主动执行，结果通过 `runToolUse()` 异步生成 |

#### 并发控制规则（核心设计）

```
canExecuteTool(isConcurrencySafe):
  ├─ 无任何工具在执行中         → ✅ 可执行
  ├─ 有工具执行中 && 全部是并发安全 && 本工具也安全 → ✅ 可并行执行
  └─ 其他情况                  → ❌ 等待
```

- **并发安全工具**（`isConcurrencySafe: true`）：可与同类工具**并行运行**
- **非并发安全工具**（如 Bash）：必须**独占**执行槽，等所有工具结束才能跑

---

### 3. `completed` — 执行完毕

| 项目 | 说明 |
|------|------|
| **触发位置** | 第 289 行、第 385 行、第 83 行 |
| **触发条件** | 工具执行完成，或被中止（错误 / 用户中断） |
| **含义** | 结果已收集到 `tool.results`，但尚未返回给外层调用方 |

#### 关键边界逻辑

```typescript
const collectResults = async () => {
  // 若在启动前已收到中止信号 → 生成合成错误消息（第 153–205 行）
  if (initialAbortReason) { /* 生成合成 ToolResult */ }

  // 正常执行
  for await (const update of generator) {
    if (isErrorResult) {
      // Bash 工具报错 → 级联中止队列中的其他工具（第 359–363 行）
    }
    // 收集进度消息和最终结果
  }

  tool.status = 'completed'  // 第 385 行
}
```

#### 特殊处理

- **Bash 工具报错**：会级联中止其他排队工具，后者收到合成错误 `ToolResult`
- **用户中断**：根据 `interruptBehavior`（`'cancel'` / `'block'`）决定处理方式（第 233–240 行）
- **进度消息**：存入 `pendingProgress`，与最终结果分离，优先产出

---

### 4. `yielded` — 结果已送出（终态）

| 项目 | 说明 |
|------|------|
| **触发位置** | `getCompletedResults()`（第 412–440 行） |
| **触发条件** | `tool.status === 'completed' && tool.results` |
| **含义** | 结果已 yield 给消费者，标记为终态防止重复产出 |

```typescript
// 第 428–433 行
if (tool.status === 'completed' && tool.results) {
  tool.status = 'yielded'           // 立刻标记，防止重入
  for (const message of tool.results) {
    yield { message, newContext: this.toolUseContext }
  }
  markToolUseAsComplete(this.toolUseContext, tool.id)
}
```

---

## 进度消息的特殊处理

进度消息（`type === 'progress'`）不与最终结果混合，走独立通道优先产出：

```
执行期间        →  存入 tool.pendingProgress
getCompletedResults() 调用时：
  1. 先 yield 所有 pendingProgress（第 418–422 行）
  2. 再 yield tool.results（第 429–435 行）
```

---

## 设计亮点总结

| 问题 | 设计方案 |
|------|---------|
| 进度消息要实时显示 | `pendingProgress` 单独存，优先于最终结果 yield |
| 工具并发但要保安全 | `isConcurrencySafe` 标志控制哪些工具可以并行 |
| 避免重复消费结果 | `completed → yielded` 这一跳做幂等保护 |
| 错误级联取消 | Bash 报错后，其他 `queued` 工具收到合成错误，跳过执行 |
| 中止信号传播 | `initialAbortReason` 在工具启动前检查，早失败早返回 |

---

## 附：任务状态机（Task State Machine）

与工具状态机不同，`Task.ts` 中还有一套**任务级**状态机，用于后台任务和 Agent：

```typescript
// Task.ts:15-20
export type TaskStatus =
  | 'pending'    // 任务已创建，等待启动
  | 'running'    // 任务正在执行
  | 'completed'  // 任务成功完成
  | 'failed'     // 任务执行失败
  | 'killed'     // 任务被用户停止

export function isTerminalTaskStatus(status: TaskStatus): boolean {
  return status === 'completed' || status === 'failed' || status === 'killed'
}
```

两套状态机职责分离：
- **ToolStatus**：管理单次工具调用的执行生命周期
- **TaskStatus**：管理后台长任务 / Agent 的整体生命周期
