# Claude Code Session 与压缩机制原理

> 分析日期：2026-06-24  
> 源码路径：`src/utils/sessionStorage.ts` · `src/services/compact/` · `src/utils/messages.ts`

---

## 一、Session 的数据结构

### 1.1 消息类型体系

Session 里流通的消息统一用 `Message` 类型表示，共有以下几类：

| 类型 | 说明 | 是否持久化 |
|------|------|-----------|
| `UserMessage` | 用户输入、工具调用结果、压缩摘要 | ✅ |
| `AssistantMessage` | AI 的响应（含 `tool_use`、`thinking`、`text` 块） | ✅ |
| `SystemMessage` | 系统通知，包括 `compact_boundary`、`api_error` 等子类型 | ✅ |
| `AttachmentMessage` | 文件、技能、任务状态等附件 | ✅ |
| `ProgressMessage` | 工具执行进度（UI 专用） | ❌ 不落盘 |
| `TombstoneMessage` | 已删除消息的墓碑标记 | ✅ |

### 1.2 消息链：parentUuid 结构

每条消息包含两个核心标识符：

```
消息 A (uuid: "aaa", parentUuid: null)      ← 对话起始
    ↓
消息 B (uuid: "bbb", parentUuid: "aaa")
    ↓
消息 C (uuid: "ccc", parentUuid: "bbb")     ← 当前末端（leaf）
```

- 消息通过 `parentUuid` 构成**单向链表**，不可变
- 恢复时从 leaf 向前沿 `parentUuid` 回溯，重建完整对话链（`buildConversationChain()`）
- 分支对话天然支持：不同 leaf 共享同一条前缀链

### 1.3 持久化格式：JSONL

```
~/.claude/projects/<projectId>/<sessionId>.jsonl
```

每行一条 JSON 消息，增量追加。只有 Transcript Messages（`user` / `assistant` / `attachment` / `system`）写入文件；`progress` 消息仅在内存中流转。

---

## 二、压缩机制概览

Claude Code 有三种粒度的压缩策略，从粗到细：

```
完整压缩 (Full Compaction)
    └─ 压缩整段对话，生成摘要替换历史

部分压缩 (Partial Compaction)
    └─ 只压缩选定范围的消息，保留其余部分

微压缩 (Micro-Compaction / API Context Edit)
    └─ 不生成摘要，直接清除 API 负载中的冗余内容（工具结果、thinking 块等）
```

---

## 三、完整压缩（Full Compaction）

**文件**：`src/services/compact/compact.ts`，核心函数 `compactConversation()`（第 387 行）

### 3.1 触发条件

**自动触发**（`autoCompact.ts`）：

```
已用 token 数 > 模型上下文窗口 - 13,000 (AUTOCOMPACT_BUFFER_TOKENS)
```

**手动触发**：用户执行 `/compact` 命令

自动压缩有熔断机制：连续失败 3 次（`consecutiveFailures`）后停止重试，避免无效循环。

**重要阈值常量**：

| 常量 | 值 | 含义 |
|------|----|------|
| `AUTOCOMPACT_BUFFER_TOKENS` | 13,000 | 触发自动压缩的安全缓冲 |
| `WARNING_THRESHOLD_BUFFER_TOKENS` | 20,000 | 显示 token 警告的缓冲 |
| `MAX_OUTPUT_TOKENS_FOR_SUMMARY` | 20,000 | 摘要生成最大输出 token |
| `MANUAL_COMPACT_BUFFER_TOKENS` | 3,000 | 手动压缩缓冲 |

### 3.2 完整执行步骤

```
1. Pre-Compact Hook
       ↓
2. 生成对话摘要（调用 API）
       ↓  （失败则 PTL 重试，最多 3 次）
3. 清理缓存（readFileState、nestedMemoryPaths）
       ↓
4. 生成后置附件（文件恢复、异步 Agent 状态、计划、技能等）
       ↓
5. 执行 Session Start Hook（模拟新会话开始）
       ↓
6. 创建 compact_boundary 边界标记
       ↓
7. Post-Compact Hook
       ↓
8. 返回 CompactionResult
```

#### 步骤 2：摘要生成与 PTL 重试

PTL（Prompt Too Long）是指摘要请求本身超出上下文窗口的情况。处理策略：

```typescript
// truncateHeadForPTLRetry()（第 243 行）
// 逐步删除最早的 API 轮组，每次重试删一批，最多重试 3 次
for (let attempt = 0; attempt < 3; attempt++) {
  try {
    return await streamCompactSummary({ messages: truncatedMessages, ... })
  } catch (e) {
    if (isPTLError(e)) truncatedMessages = dropEarliestTurn(truncatedMessages)
    else throw e
  }
}
```

#### 步骤 4：后置附件生成

压缩后，Claude 需要重新"记住"一些关键上下文，通过附件注入：

| 附件类型 | 数量限制 | 说明 |
|---------|---------|------|
| 文件恢复附件 | 最多 5 个 | 压缩前最近访问的文件（`POST_COMPACT_MAX_FILES_TO_RESTORE = 5`） |
| 异步 Agent 状态 | — | 后台运行的 Agent 当前状态 |
| 计划文件附件 | — | 当前 Plan 模式下的计划内容 |
| 计划模式提醒 | — | 若处于 Plan 模式则注入提醒 |
| 技能附件 | — | 已调用的 Skill 信息 |
| 工具增量附件 | — | 新发现的工具差异 |

token 预算：`POST_COMPACT_TOKEN_BUDGET = 50,000`（总），单文件上限 `5,000`，技能总上限 `25,000`。

### 3.3 CompactionResult 结构

```typescript
interface CompactionResult {
  boundaryMarker: SystemCompactBoundaryMessage  // 边界标记（必须）
  summaryMessages: UserMessage[]               // 摘要消息（必须）
  attachments: AttachmentMessage[]             // 后置附件
  hookResults: HookResultMessage[]             // 钩子结果
  messagesToKeep?: Message[]                   // 部分压缩时保留的消息
  userDisplayMessage?: string                  // 给用户的展示文字
  preCompactTokenCount?: number                // 压缩前 token 数
  postCompactTokenCount?: number               // 摘要 API 调用消耗的 token
  truePostCompactTokenCount?: number           // 压缩后真实上下文 token 数
  compactionUsage?: TokenUsage                 // 详细 token 用量
}
```

---

## 四、compact_boundary 边界标记

压缩完成后，在消息链中插入一条 `SystemCompactBoundaryMessage`，作为"分水岭"：

```typescript
// messages.ts，createCompactBoundaryMessage()
{
  type: 'system',
  subtype: 'compact_boundary',
  content: 'Conversation compacted',
  level: 'info',
  uuid: '<新 UUID>',
  timestamp: '<ISO 8601>',
  compactMetadata: {
    trigger: 'auto' | 'manual',  // 触发方式
    preTokens: 128000,            // 压缩前的 token 数
    userContext?: string,          // 用户自定义说明（手动压缩时）
    messagesSummarized?: number,   // 被压缩的消息条数
    preservedSegment?: {           // 部分压缩时保留段的边界信息
      headUuid: UUID,   // 保留段第一条消息
      anchorUuid: UUID, // 锚点（边界前最后一条消息）
      tailUuid: UUID,   // 保留段最后一条消息
    },
    preCompactDiscoveredTools?: string[],  // 压缩前发现的工具列表
  }
}
```

**读取逻辑**：`getMessagesAfterCompactBoundary()` 找到最后一个 `compact_boundary`，只返回其后的消息发送给 API，之前的历史由摘要替代。

---

## 五、部分压缩（Partial Compaction）

**文件**：`compact.ts`，`partialCompactConversation()`（第 772 行）

部分压缩只压缩消息列表中的某个片段，由 `pivotIndex`（枢轴索引）和 `direction` 控制：

| direction | 含义 | 适用场景 |
|-----------|------|---------|
| `'from'` | 压缩 pivotIndex **之后**的消息，保留之前的 | 保留缓存前缀，节省 API 缓存命中 |
| `'up_to'` | 压缩 pivotIndex **之前**的消息，保留之后的 | 保留最新上下文，缓存失效 |

边界标记中会写入 `preservedSegment`，记录保留段的首尾 UUID，方便恢复时精确定位。

---

## 六、微压缩（Micro-Compaction / API Context Edit）

**文件**：`src/services/compact/apiMicrocompact.ts`

微压缩不生成摘要，而是在发送 API 请求前**直接清除消息体中的冗余内容**，两种策略：

### 策略一：`clear_tool_uses_20250919`

清除历史工具调用的输入/输出，减少上下文体积：

```typescript
{
  type: 'clear_tool_uses_20250919',
  trigger?: { type: 'input_tokens'; value: number },  // 超过阈值才触发
  keep?: { type: 'tool_uses'; value: number },          // 保留最近 N 次工具调用
  clear_tool_inputs?: boolean | string[],               // 清除工具输入
  exclude_tools?: string[],                             // 排除某些工具不清
  clear_at_least?: { type: 'input_tokens'; value: number }
}
```

**结果清理的工具**（输出大）：Bash、PowerShell、Glob、Grep、FileRead、WebFetch、WebSearch  
**使用清理的工具**（输入大）：FileEdit、FileWrite、NotebookEdit

### 策略二：`clear_thinking_20251015`

清除 Extended Thinking 产生的 `thinking` 块：

```typescript
{
  type: 'clear_thinking_20251015',
  keep: { type: 'thinking_turns'; value: 2 } | 'all'  // 保留最近 N 轮的 thinking
}
```

---

## 七、Session Memory（会话笔记）

**文件**：`src/services/SessionMemory/sessionMemory.ts`

Session Memory 是一个**自动维护的 Markdown 文件**，由后台 Agent 定期从对话中提取关键信息并更新，作为超长会话的"外部记忆"。

### 7.1 触发条件

```
shouldExtractMemory() = true  当且仅当：

  初始化阈值：token 数达到 minimumMessageTokensToInit  （首次触发前提）

  之后每次：
    (token 增量 >= minimumTokensBetweenUpdate)
    AND
    (自上次更新后的工具调用数 >= toolCallsBetweenUpdates)
```

### 7.2 提取流程

```
1. 判断是否需要提取（shouldExtractMemory）
       ↓
2. 设置 Session Memory 文件路径
       ↓
3. 读取现有 Memory 内容（currentMemory）
       ↓
4. 构建更新 Prompt（包含对话历史 + 旧 Memory）
       ↓
5. 启动 Fork Agent（querySource: 'session_memory'）
       ↓
6. Fork Agent 只能操作 Memory 文件（canUseTool 限制）
       ↓
7. 更新计数器（recordExtractionTokenCount, updateLastSummarizedMessageId）
```

### 7.3 配置项

```typescript
type SessionMemoryConfig = {
  minimumMessageTokensToInit: number   // 启动 Memory 的最低 token 阈值
  minimumTokensBetweenUpdate: number   // 两次更新的最小 token 间隔
  toolCallsBetweenUpdates: number      // 两次更新间的最小工具调用次数
}
```

---

## 八、Session 恢复机制

**文件**：`src/utils/sessionRestore.ts`，`src/utils/conversationRecovery.ts`

### 8.1 恢复入口：`restoreSessionStateFromLog()`（第 99 行）

```
1. 文件历史快照恢复 (fileHistorySnapshots)
2. Commit 归属状态恢复 (attributionSnapshots)
3. 上下文折叠状态恢复 (contextCollapseCommits)
4. Todos 提取 (extractTodosFromTranscript)
5. 消息链重建 (buildConversationChain)
```

### 8.2 消息链重建：`buildConversationChain()`

```typescript
// sessionStorage.ts，~第 1100 行
function buildConversationChain(
  messages: Map<UUID, TranscriptMessage>,
  leafMessage: TranscriptMessage,
): TranscriptMessage[] {
  const transcript = []
  let current = leafMessage
  while (current) {
    transcript.unshift(current)              // 向前插入，保证时序正确
    current = messages.get(current.parentUuid)
  }
  return transcript  // 完整的时间正序消息链
}
```

循环检测：通过 `seen: Set<UUID>` 防止 `parentUuid` 成环导致死循环。

### 8.3 压缩后的消息加载

JSONL 文件中 `compact_boundary` 之前的消息都会被加载到内存 Map，但发送给 API 时只取边界之后的部分：

```
JSONL 文件：[...历史消息...] [compact_boundary] [摘要消息] [...新消息...]
                    ↑                                            ↑
             恢复状态用                                       发送给 API
```

---

## 九、整体架构图

```
用户输入
    ↓
Session（JSONL 持久化，parentUuid 链表）
    ├─ 消息实时追加
    ├─ token 计数监控
    │
    ├─ [token 接近上限] ──→ Auto-Compact ──→ compactConversation()
    │                                              ├─ 调 API 生成摘要
    │                                              ├─ 插入 compact_boundary
    │                                              └─ 注入后置附件
    │
    ├─ [API 请求前] ──→ Micro-Compact ──→ 清除冗余工具结果/thinking 块
    │
    └─ [后台定期] ──→ Session Memory ──→ 外部 Markdown 笔记文件

Session 恢复：
    JSONL 文件 → buildConversationChain() → 重建消息链
              → getMessagesAfterCompactBoundary() → 只发边界后的消息给 API
```

---

## 十、关键函数速查

### 压缩

| 函数 | 文件 | 行号 | 说明 |
|------|------|------|------|
| `compactConversation()` | `compact/compact.ts` | 387 | 完整压缩入口 |
| `partialCompactConversation()` | `compact/compact.ts` | 772 | 部分压缩入口 |
| `truncateHeadForPTLRetry()` | `compact/compact.ts` | 243 | PTL 错误重试截断 |
| `buildPostCompactMessages()` | `compact/compact.ts` | 330 | 构建压缩后消息数组 |
| `createPostCompactFileAttachments()` | `compact/compact.ts` | 1415 | 文件恢复附件 |
| `getAutoCompactThreshold()` | `compact/autoCompact.ts` | 72 | 自动压缩阈值计算 |
| `isAutoCompactEnabled()` | `compact/autoCompact.ts` | 147 | 是否启用自动压缩 |

### 消息处理

| 函数 | 文件 | 行号 | 说明 |
|------|------|------|------|
| `createCompactBoundaryMessage()` | `utils/messages.ts` | ~4800 | 创建边界标记 |
| `annotateBoundaryWithPreservedSegment()` | `utils/messages.ts` | ~4820 | 写入保留段元数据 |
| `getMessagesAfterCompactBoundary()` | `utils/messages.ts` | ~5000 | 取边界后的消息 |
| `isCompactBoundaryMessage()` | `utils/messages.ts` | ~5100 | 判断是否为边界标记 |

### Session 存储与恢复

| 函数 | 文件 | 行号 | 说明 |
|------|------|------|------|
| `getTranscriptPath()` | `utils/sessionStorage.ts` | 202 | 获取 JSONL 文件路径 |
| `buildConversationChain()` | `utils/sessionStorage.ts` | ~1100 | 重建 parentUuid 消息链 |
| `loadTranscriptFile()` | `utils/sessionStorage.ts` | ~1150 | 完整加载 JSONL 文件 |
| `isTranscriptMessage()` | `utils/sessionStorage.ts` | 139 | 判断是否持久化消息 |
| `restoreSessionStateFromLog()` | `utils/sessionRestore.ts` | 99 | Session 恢复入口 |

### Session Memory

| 函数 | 文件 | 行号 | 说明 |
|------|------|------|------|
| `shouldExtractMemory()` | `SessionMemory/sessionMemory.ts` | 134 | 判断是否触发提取 |
| `extractSessionMemory()` | `SessionMemory/sessionMemory.ts` | 272 | 自动提取入口 |
| `manuallyExtractSessionMemory()` | `SessionMemory/sessionMemory.ts` | 387 | 手动触发提取 |
| `countToolCallsSince()` | `SessionMemory/sessionMemory.ts` | 108 | 统计工具调用次数 |
