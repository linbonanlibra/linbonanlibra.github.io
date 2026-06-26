# Agent Planning

> Agent 工程中的规划机制：从思维链到多智能体编排

---

## 一、什么是 Planning

Planning（规划）是 Agent 将复杂目标分解为可执行步骤序列的能力。

没有规划能力的 LLM 是**反应式**的——给一个问题，直接输出一个答案。有规划能力的 Agent 是**前瞻式**的——先思考"要做什么、分几步、每步用什么工具"，再行动。

```
无规划：
User → [LLM] → Answer

有规划：
User → [Plan: step1→step2→step3] → Execute(step1) → Observe → Execute(step2) → ... → Answer
```

**规划解决的核心问题：**
- 任务太复杂，一步无法完成
- 需要按依赖顺序执行（先读文件，才能改文件）
- 中途需要根据观察结果调整方向
- 需要协调多个工具/子 Agent 并行工作

---

## 二、ReAct：最基础的规划范式

ReAct（Reasoning + Acting）是目前使用最广泛的 Agent 规划模式，由 2022 年的论文提出。

### 核心循环

```
Thought → Action → Observation → Thought → Action → Observation → ... → Final Answer
```

每一步都由三个元素组成：

| 元素 | 含义 | 示例 |
|---|---|---|
| **Thought** | 当前推理/计划 | "我需要先读取 config.json 确认数据库配置" |
| **Action** | 调用工具执行 | `read_file("config.json")` |
| **Observation** | 工具返回的结果 | `{"db_host": "localhost", "port": 5432}` |

### 在 Claude 里的体现

Claude 的 Tool Use 天然实现了 ReAct 循环：

```
[Thought]      Claude 输出 text block："我先看看项目结构..."
[Action]       Claude 输出 tool_use block：{name: "bash", input: {command: "find . -name '*.ts'"}}
[Observation]  用户返回 tool_result block：{content: "src/index.ts\nsrc/api/..."}
[Thought]      Claude 继续推理下一步...
```

```typescript
// ReAct 循环的代码实现
while (true) {
  const response = await client.messages.create({
    model: "claude-opus-4-8",
    max_tokens: 16000,
    tools,
    messages,
  });

  // Thought + Final Answer → 结束
  if (response.stop_reason === "end_turn") break;

  // Action → 执行工具（Observation）→ 继续循环
  const toolResults = await executeTools(response.content);
  messages.push(
    { role: "assistant", content: response.content },
    { role: "user", content: toolResults },
  );
}
```

### ReAct 的局限

- **串行**：每次只能做一件事，等结果，再决定下一步
- **短视**：每步只看当前状态，没有全局计划
- **无回溯**：走错了不会回头，只能往前修正

---

## 三、规划的四个层次

从简单到复杂，Agent 规划有四个递进的层次：

### Level 1：直接回答（无规划）

```
适用：分类、摘要、单轮问答
特征：一次 API 调用，直接输出结果
```

### Level 2：链式推理（CoT）

让模型在回答前先"想一想"，输出推理过程：

```
适用：需要多步推导的问题（数学、逻辑、代码分析）
机制：Extended Thinking / Adaptive Thinking
```

```typescript
// Claude 的 Adaptive Thinking = 自动 CoT
const response = await client.messages.create({
  model: "claude-opus-4-8",
  max_tokens: 32000,
  thinking: { type: "adaptive" },     // 模型自己决定是否思考、思考多久
  output_config: { effort: "high" },
  messages: [{ role: "user", content: "分析这段代码的性能问题..." }],
});

// response.content 可能包含：
// [{ type: "thinking", thinking: "这个循环复杂度是 O(n²)，因为..." }]
// [{ type: "text", text: "主要问题在于第 23 行的嵌套循环..." }]
```

### Level 3：工具调用循环（ReAct）

上一节描述的模式，适用于需要与外部世界交互的任务。

### Level 4：显式规划 + 执行（Plan & Execute）

先生成完整计划，再按计划执行。适用于复杂的长周期任务：

```
阶段一（规划）：
  输入：目标 + 可用工具
  输出：[step1, step2, step3, ...]  ← 完整的执行计划

阶段二（执行）：
  按顺序执行每个 step
  每步结束后可选择：继续 / 修正计划 / 提前终止
```

---

## 四、Claude Code 的规划机制

Claude Code 实现了一套完整的规划体系，从轻量到重量级：

### 4.1 EnterPlanMode：显式规划模式

当任务复杂、影响范围大时，Claude Code 会进入规划模式：

```
用户：帮我重构认证模块

Claude：（调用 EnterPlanMode）
  1. 探索代码库（只读，不修改）
  2. 理解现有架构
  3. 设计实现方案
  4. 向用户展示计划，等待审批
  5. 用户确认后，调用 ExitPlanMode，开始实现
```

**触发规划模式的条件：**
- 新功能实现（影响多个文件）
- 有多种有效实现方案
- 修改现有行为
- 需要架构决策
- 需求不清晰，需先探索

**不触发的情况：**
- 单行修复（typo、明显 bug）
- 用户给出了极其详细的具体指令
- 纯研究/探索性任务

### 4.2 TaskCreate / TaskUpdate：任务追踪

对于多步骤任务，Claude Code 使用显式任务列表追踪进度：

```
TaskCreate → TaskUpdate(in_progress) → 实际工作 → TaskUpdate(completed)
```

```
任务列表示例：
[in_progress] 分析现有认证代码
[pending]     设计新的 JWT 方案
[pending]     实现 TokenService
[pending]     更新路由中间件
[pending]     编写单元测试
```

这本质上是把**隐式的规划**变成**显式可追踪的状态**，解决了两个问题：
- 长任务不会"忘记"剩余步骤
- 用户可以实时看到进度

### 4.3 EnterWorktree：隔离执行环境

对于可能破坏现有代码的操作，规划时可以指定在独立 git worktree 中执行：

```bash
# Claude Code 内部：为当前任务创建隔离工作区
git worktree add .claude/worktrees/feature-branch -b feature/auth-refactor

# 在这个隔离环境里实验性地修改代码
# 不影响主工作区，用户可以随时放弃
```

---

## 五、多智能体规划：Workflow 编排

单 Agent 的规划能力受上下文窗口限制。当任务需要：
- 并行处理大量文件
- 独立的多视角验证
- 超出单个上下文的工作量

就需要**多智能体规划**——把工作分解给多个 Agent 并行执行。

### 5.1 两种基础并发模式

**pipeline（流水线）：** 每个 item 独立穿越所有阶段，无屏障

```javascript
// 5 个文件，每个独立经历：分析 → 验证 → 修复
// 文件A可以在验证，文件B还在分析，文件C已经在修复
const results = await pipeline(
  files,
  file => agent(`分析 ${file} 中的安全漏洞`, { schema: FINDINGS }),
  findings => agent(`验证这些发现是否真实存在`, { schema: VERDICT }),
  verdict => verdict.isReal ? agent(`生成修复方案`) : null,
);
// 总耗时 = 最慢单个文件的全链路时间（而不是所有文件的总和）
```

**parallel（并行）：** 所有任务同时启动，等全部完成再继续（有屏障）

```javascript
// 需要汇总所有结果再做下一步决策时使用
const allFindings = await parallel(
  DIMENSIONS.map(d => () => agent(`从 ${d} 维度扫描漏洞`, { schema: FINDINGS }))
);
// 去重、合并
const dedupedFindings = dedup(allFindings.flat());
// 再统一验证
const verified = await parallel(dedupedFindings.map(f => () => agent(`验证: ${f}`)));
```

**关键原则：默认用 pipeline，只有真正需要全部结果才用 parallel（屏障）。**

### 5.2 规划 + 执行的多 Agent 范式

```javascript
export const meta = {
  name: 'plan-and-execute',
  description: '先规划再并行执行',
  phases: [
    { title: '规划', detail: '分析任务，生成执行计划' },
    { title: '执行', detail: '并行执行每个子任务' },
    { title: '汇总', detail: '合并结果，生成报告' },
  ],
}

// Phase 1: 规划（单 Agent，全局视野）
phase('规划')
const plan = await agent(
  `分析这个代码库，生成重构计划。每个子任务应该独立可并行执行。`,
  { schema: PLAN_SCHEMA }
)

// Phase 2: 执行（多 Agent 并行，每个负责一个子任务）
phase('执行')
const results = await pipeline(
  plan.subtasks,
  task => agent(`执行子任务: ${task.description}`, {
    label: task.name,
    isolation: 'worktree',  // 每个子任务在独立 git worktree 中执行
  }),
)

// Phase 3: 汇总
phase('汇总')
const report = await agent(`合并以下执行结果，生成总结报告: ${JSON.stringify(results)}`)
```

### 5.3 对抗性验证（Adversarial Verify）

规划中的一个重要模式——让多个 Agent 从不同角度质疑同一个发现：

```javascript
// 找到一个 bug 后，让 3 个独立 Agent 尝试反驳它
const votes = await parallel(
  Array.from({ length: 3 }, (_, i) => () =>
    agent(`尝试反驳这个 bug：${bug.description}
           如果不确定，默认返回 refuted=true。
           视角${i}: ${['安全角度', '性能角度', '逻辑角度'][i]}`,
      { schema: VERDICT_SCHEMA }
    )
  )
)

// 只有超过半数认为真实存在，才保留
const isReal = votes.filter(v => !v.refuted).length >= 2
```

---

## 六、规划的核心权衡

### 规划深度 vs 执行成本

```
浅规划（ReAct）：
  + 低延迟，边走边看
  + 适应性强，能根据中途发现调整
  - 可能走弯路
  - 无法并行

深规划（Plan & Execute）：
  + 可并行执行子任务
  + 全局最优路径
  - 规划本身有成本
  - 计划可能因中途发现而过时
```

**选择原则：**

| 场景 | 推荐策略 |
|---|---|
| 目标明确、步骤清晰 | Plan & Execute + 并行 |
| 需要探索、未知因素多 | ReAct（边做边看） |
| 简单任务 | 直接回答，无需规划 |
| 需要验证正确性 | Plan & Execute + 对抗验证 |

### 什么时候需要规划 Agent

满足以下条件才值得用 Agent 规划，否则保持简单：

1. **复杂性**：任务多步骤，难以事先完全描述
2. **价值**：结果质量值得更高的成本和延迟
3. **可行性**：模型在这类任务上有能力
4. **可恢复性**：错误可被发现和纠正（有测试、有 Review、可回滚）

---

## 七、Claude 的 Adaptive Thinking 与规划的关系

Claude 的 Adaptive Thinking（自适应思考）是**模型内置的规划机制**——在生成最终答案前，模型在一个独立的"思考"空间里进行推演。

```
用户看到的：
  [最终回答]

模型实际做的：
  [thinking block（用户默认看不到）]
    "这道题需要先...
     如果用方法A，那么...
     但方法A有个问题...
     换方法B，步骤是：1. 2. 3.
     验证：..."
  [text block]
    "这道题的解法是..."
```

**Adaptive Thinking 的特点：**
- `thinking: { type: "adaptive" }` — 模型自己决定是否思考、思考多久
- 与 `effort` 配合：effort 越高，思考越深
- 思考内容默认不返回给用户（`display: "omitted"`）
- 设置 `display: "summarized"` 可以看到摘要版思考内容

```typescript
// 开启 Adaptive Thinking，让 Claude 充分规划后再回答
const response = await client.messages.create({
  model: "claude-opus-4-8",
  max_tokens: 32000,
  thinking: {
    type: "adaptive",
    display: "summarized",  // 可选：返回思考摘要
  },
  output_config: { effort: "xhigh" },  // 复杂任务用 xhigh
  messages: [{ role: "user", content: "设计一个分布式限流系统..." }],
});
```

**Adaptive Thinking vs 显式 Tool Use 规划的区别：**

| 维度 | Adaptive Thinking | Tool Use 规划 |
|---|---|---|
| 位置 | 模型内部 | 外部可观察 |
| 与环境交互 | 不能（纯推理） | 能（调用工具） |
| 适用 | 推理密集型任务 | 需要外部信息的任务 |
| 用户可见性 | 默认不可见 | 完全可见 |
| 成本 | 增加 output token | 增加 API 调用次数 |

---

## 八、Task Budget：控制 Agent 的规划深度

对于长周期 Agent，规划可能无限延伸，导致 token 失控。**Task Budget** 是告诉模型"你总共有 N 个 token 来完成整个任务"的机制：

```typescript
// 模型看到一个倒计时，会自我调节规划深度
const response = await client.beta.messages.create({
  betas: ["task-budgets-2026-03-13"],
  model: "claude-opus-4-8",
  max_tokens: 64000,        // 每次响应的硬上限（模型不感知）
  thinking: { type: "adaptive" },
  output_config: {
    effort: "high",
    task_budget: {
      type: "tokens",
      total: 200000,        // 整个任务的 token 预算（模型感知并自我调节）
    },
  },
  messages: [...],
});
```

**`max_tokens` vs `task_budget` 的区别：**

```
max_tokens：硬限制，模型不知道，超出直接截断
task_budget：软提示，模型知道剩余预算，主动调整规划粒度
             预算充足时 → 深度探索
             预算紧张时 → 快速收敛，聚焦核心
```

---

## 九、规划失败的常见原因

### 1. 过度规划（Over-planning）

```
症状：花大量 token 生成详细计划，但计划在执行第一步就过时了
原因：规划时对环境假设太多，实际情况与预期不符
修复：用 ReAct 代替 Plan & Execute，或规划粒度更粗
```

### 2. 规划与执行脱节

```
症状：生成了完美计划，但执行时忘记按计划行事
原因：计划没有显式传递给执行阶段
修复：把计划写入 TaskCreate，或在每次执行前重新引用计划
```

### 3. 并行滥用（Barrier 滥用）

```
症状：parallel() 到处用，实际上没有减少总时间
原因：把"概念上独立"等同于"需要同步"
修复：默认用 pipeline()，只有真正需要全部结果才用 parallel()
```

### 4. 无限循环

```
症状：Agent 一直规划、一直行动，不知道何时停止
原因：没有明确的终止条件
修复：
  - 设置 maxTurns 上限
  - 明确定义"完成"的标准（Done Definition）
  - 使用 Outcome 机制（user.define_outcome + rubric）
```

---

## 总结

```
任务复杂度 ↑

简单问答
  └─ 直接回答（无规划）

需要推理
  └─ Adaptive Thinking（模型内置规划）

需要外部信息
  └─ ReAct（Thought → Action → Observation 循环）

复杂多步骤
  └─ Plan & Execute
       ├─ 显式规划阶段（EnterPlanMode / TaskCreate）
       ├─ 顺序执行（pipeline）
       └─ 并行执行（parallel，仅在需要全量结果时）

超大规模 / 需要验证
  └─ 多 Agent Workflow
       ├─ 扇出搜索（Explore Agent）
       ├─ 对抗验证（Adversarial Verify）
       └─ 循环直到收敛（Loop-until-dry）
```

**核心思想：用最简单的规划机制完成任务。只有当更简单的方式不够用时，才升级到更复杂的规划策略。**

---

## 相关

- [[KV Cache 与 Prompt Caching]] — 规划过程中的上下文缓存优化
- [[ReAct]] — ReAct 模式的详细介绍
- [[AI 记忆]] — Agent 跨 Session 的记忆机制
