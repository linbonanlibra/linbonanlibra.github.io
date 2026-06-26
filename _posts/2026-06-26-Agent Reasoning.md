# Agent Reasoning

> 推理机制：LLM 如何"思考"，以及如何在 Agent 工程中利用它

---

## 一、什么是 Reasoning

**Reasoning（推理）** 是模型在给出最终答案之前，进行中间步骤推导的能力。

```
无推理（直接映射）：
  输入 → [黑盒] → 输出
  "2+2=" → "4"（直接记忆，无推理）

有推理（显式中间步骤）：
  输入 → [步骤1] → [步骤2] → [步骤3] → 输出
  "如果火车速度 60km/h，距离 180km..." →
    "时间 = 距离/速度" →
    "= 180/60" →
    "= 3小时" →
    "需要 3 小时"
```

推理的本质是：**将复杂问题分解为若干个模型"有把握"的子问题，逐步求解。**

---

## 二、为什么推理能提升效果

这背后有一个直觉上的模型理论：

**Transformer 的推理容量受限于前向传播的"深度"。**

对于足够简单的问题，单次前向传播就能给出正确答案。但对于复杂问题，需要多个推理步骤，而每个 token 的生成就是一次前向传播——**生成中间步骤 = 给模型更多"计算空间"**。

```
直接回答：
  问题 token → [1次前向传播] → 答案 token
  计算量有限，复杂问题容易出错

Chain-of-Thought：
  问题 → [前向传播] → 步骤1 token
       → [前向传播] → 步骤2 token（能看到步骤1）
       → [前向传播] → 步骤3 token（能看到步骤1+2）
       → [前向传播] → 答案 token（能看到所有步骤）
  每一步都能"站在前一步的肩膀上"继续推理
```

实验证明，对于需要多步推导的问题（数学、逻辑、代码分析），加入中间推理步骤能显著提升正确率。

---

## 三、推理的主要范式

### 3.1 Chain-of-Thought（CoT，思维链）

最基础、最广泛的推理方式——让模型逐步推导：

```
提示方式：
  ① "Let's think step by step"（零样本 CoT）
  ② 提供有推理过程的示例（少样本 CoT）
  ③ 直接训练模型产生推理链（Claude 的做法）

输出示例：
  Q: 一个商店有 23 个苹果。如果他们卖出 5 个，又进了 7 个，现在有多少？
  A: 先计算卖出后：23 - 5 = 18 个
     再加上新进货：18 + 7 = 25 个
     所以现在有 25 个苹果。
```

**特点：**
- 线性，从头到尾一条路
- 简单有效
- 一旦某步出错，后续全错（无纠错机制）

### 3.2 Tree of Thoughts（ToT，思维树）

将推理过程组织为树状结构，在每个节点可以探索多个分支：

```
                   问题
                    │
          ┌─────────┼─────────┐
        方案A      方案B      方案C
          │          │          │
       评估A      评估B      评估C
        差❌       好✅        差❌
                   │
              继续深化B
                   │
              最终答案
```

**特点：**
- 可以探索、比较、回溯
- 计算代价更高（多次采样）
- 适合有多种解法的开放问题

### 3.3 Scratchpad（草稿本推理）

给模型一个"草稿本"空间，让它随意思考，最终只输出答案：

```
[Scratchpad（不给用户看）]
  让我想想... 这道题可以用两种方法
  方法一：暴力枚举... 但时间复杂度是 O(n²)，太慢了
  方法二：哈希表... 一次遍历，O(n)
  应该用方法二，但要注意边界情况...
  如果数组为空... 返回 -1
  如果有重复...
  好，思路清楚了

[最终输出（给用户看）]
  使用哈希表，时间复杂度 O(n)：...
```

这正是 Claude 的 **Thinking Block** 机制。

### 3.4 Reflection（自我反思）

让模型检查自己的输出，发现错误后修正：

```
第一次输出：答案是 42
反思：等等，我检查一下... 步骤3有个算术错误
修正后：答案应该是 48
```

Claude 在高 effort 模式下会自动进行这类反思。

---

## 四、Claude 的 Thinking 机制

Claude 将推理实现为一个独立的 **Thinking Block**——在生成最终回答之前的内部思考空间。

### 4.1 Thinking Block 的结构

```
API 响应的 content 数组：

[
  {
    type: "thinking",
    thinking: "让我分析这段代码...
               首先看第 23 行，这里有个潜在的空指针...
               再看调用链...
               根本原因是上游没有做 null 检查..."
  },
  {
    type: "text",
    text: "代码有一个空指针问题，根本原因在于..."
  }
]
```

**关键特性：**
- Thinking Block 的内容**不计入后续对话上下文**（每次都重新思考）
- 思考内容对用户**默认不可见**（`display: "omitted"`）
- 思考内容**无法被用户注入或篡改**（安全隔离）
- 思考内容**消耗 output token**（有成本）

### 4.2 三代 Thinking 实现

**第一代：Extended Thinking（固定 budget）**

```typescript
// 旧方式，已在 Opus 4.7+ 废弃
thinking: { type: "enabled", budget_tokens: 8000 }
// 问题：固定预算，简单问题浪费，复杂问题不够用
```

**第二代：Adaptive Thinking（自适应）**

```typescript
// 当前推荐方式（Opus 4.6+ / Sonnet 4.6+）
thinking: { type: "adaptive" }
// 模型自己决定：是否需要思考 + 思考多久
// 结合 effort 参数控制整体深度
```

**第三代：Interleaved Thinking（交错思考）**

```typescript
// 不只在最开始思考，在每次工具调用之后也思考
// Adaptive Thinking 自动启用此特性，无需额外配置

[thinking] → 决定调用工具A
[tool_use] → 调用工具A
[tool_result] → 收到结果
[thinking] → 分析结果，决定下一步   ← 关键：工具调用之间也思考
[tool_use] → 调用工具B
...
```

### 4.3 Thinking 的可见性控制

```typescript
// 默认：思考内容不返回（节省带宽，Opus 4.7+ 默认）
thinking: { type: "adaptive", display: "omitted" }

// 返回摘要版思考（适合调试，或流式展示"正在思考"进度）
thinking: { type: "adaptive", display: "summarized" }
```

**什么时候需要 `display: "summarized"`？**
- 在 UI 里展示"AI 正在思考..."的实时反馈
- 调试 Agent 的推理过程
- 需要记录 Agent 决策逻辑的审计场景

```typescript
// 流式展示思考过程
const stream = client.messages.stream({
  model: "claude-opus-4-8",
  thinking: { type: "adaptive", display: "summarized" },
  // ...
});

for await (const event of stream) {
  if (event.type === "content_block_delta") {
    if (event.delta.type === "thinking_delta") {
      process.stdout.write("💭 " + event.delta.thinking);  // 实时思考内容
    } else if (event.delta.type === "text_delta") {
      process.stdout.write(event.delta.text);              // 最终回答
    }
  }
}
```

---

## 五、Effort：控制推理深度的旋钮

`effort` 参数是控制 Claude 推理深度（和成本）的核心旋钮：

```typescript
output_config: { effort: "low" | "medium" | "high" | "xhigh" | "max" }
```

### 各级别的含义

| Effort | 推理深度 | 工具调用 | 适用场景 |
|---|---|---|---|
| `low` | 最浅，快速回答 | 少而合并 | 分类、简单问答、低延迟场景 |
| `medium` | 适中 | 正常 | 多数日常任务的平衡点 |
| `high` | 深度推理 | 充分 | 智力密集型任务（默认推荐） |
| `xhigh` | 深度 + 充分探索 | 多而细粒度 | 复杂编程、长链路 Agent（Claude Code 默认） |
| `max` | 穷举探索 | 最多 | 极端复杂问题，不在意成本 |

**关键洞察：effort 不是线性的。**

对于 Agent 任务：
```
effort: "low"  → 快速但可能走弯路，反而需要更多轮次纠错
effort: "xhigh" → 每步深度推理，前期多花 token，但总轮次少，往往更经济
```

**更高 effort 有时反而更便宜**（总 token 数更少，因为少走弯路）。

### Effort 与 Thinking 的关系

```
effort 控制"整体深度"（包括推理 + 行动）
thinking 控制"是否进行内部推理"

最佳实践（复杂 Agent 任务）：
thinking: { type: "adaptive" } + output_config: { effort: "xhigh" }

不需要深度推理的任务：
thinking: { type: "disabled" } + output_config: { effort: "low" }
```

---

## 六、推理在 Agent 中的特殊作用

### 6.1 工具选择推理

Agent 需要推理"现在该用哪个工具"——这是 ReAct 中 Thought 步骤的核心：

```
[thinking]
  用户问的是最新的 API 文档，我的训练数据可能过时了
  需要用 web_search 工具先查一下
  查询关键词应该是...

[tool_use: web_search]
  query: "Claude API 2026 最新文档"

[thinking]
  搜索结果显示文档在 platform.claude.com
  需要 web_fetch 获取具体内容
  还是直接回答？先看看搜索摘要够不够用...
```

没有推理，Agent 的工具选择会变成盲目触发（每次都用，或从不用）。

### 6.2 多步计划的动态调整

推理让 Agent 能够在执行过程中**根据新信息调整计划**：

```
初始计划：修改 auth.ts

[thinking] 读取文件后发现...
  原来 auth.ts 依赖了 session.ts 里的一个类型定义
  我需要先修改 session.ts，否则类型会不匹配
  调整计划：先改 session.ts → 再改 auth.ts

[调整后执行]
```

这是"在线规划"（Online Planning）——边执行边更新计划，而不是严格按预设计划执行。

### 6.3 不确定性处理

推理帮助 Agent 识别自己的不确定性，决定是"猜测执行"还是"先询问"：

```
[thinking]
  用户说"删除旧文件"，但没说哪些算"旧"
  如果按文件时间戳 > 30天算旧，会删掉 config/ 下的配置文件
  这个风险很高
  应该先问用户确认，而不是自己决定
```

---

## 七、Chain-of-Thought 在提示工程中的应用

即使不用 Thinking Block，也可以通过提示工程激发推理：

### 7.1 零样本 CoT

```
# 效果最差（直接要求）
问题文本

# 效果较好（引导思考）
请一步步思考：问题文本

# 效果最好（Claude 4.x 自动推理，通常不需要显式引导）
直接提问，配合 thinking: { type: "adaptive" }
```

### 7.2 Few-Shot CoT（提供推理示例）

```
以下是处理类似问题的示例：

问题：[示例问题]
思路：首先... 然后... 因此...
答案：[示例答案]

---

现在请处理：[实际问题]
```

### 7.3 Self-Consistency（多路推理取众数）

对同一问题进行多次推理，取最频繁的答案：

```python
# 对同一个问题采样 5 次
answers = []
for _ in range(5):
    response = client.messages.create(
        model="claude-opus-4-8",
        thinking={"type": "adaptive"},
        temperature=None,  # 新模型不支持 temperature
        messages=[{"role": "user", "content": question}]
    )
    answers.append(extract_answer(response))

# 取众数
from collections import Counter
final_answer = Counter(answers).most_common(1)[0][0]
```

适用于：高风险决策、答案不确定、需要高置信度的场景。

---

## 八、推理的成本与权衡

### 8.1 Token 成本

```
无推理：
  input: 1000 tokens → output: 200 tokens
  成本 ≈ input × $5 + output × $25（Opus 4.8 价格/MTok）

有推理（Thinking）：
  input: 1000 tokens → thinking: 2000 tokens + output: 200 tokens
  thinking token 按 output 价格计费
  成本增加约 10x（如果 thinking 比 output 多 10 倍）
```

### 8.2 延迟成本

Thinking 是串行的——模型必须先完成思考，再生成回答。对延迟敏感的场景（实时对话、语音交互），高推理深度可能不合适。

### 8.3 选择策略

```
推理什么时候值得投入：

✅ 多步数学/逻辑推导
✅ 代码 bug 分析（需要理解调用链）
✅ 复杂决策（需要权衡多个因素）
✅ 安全敏感的操作（需要仔细确认）
✅ 长链路 Agent 任务（减少走弯路的总成本）

❌ 不需要推理的场景：
  简单事实查询（"Python 怎么打印 Hello World"）
  格式转换（JSON → CSV）
  纯信息检索（用工具查比推理更准）
  高频低价值任务（成本不合算）
```

---

## 九、推理 vs 检索：一个重要的设计决策

Agent 工程中有一个经典问题：**让模型推理，还是调用工具检索？**

```
推理（Reasoning）：
  依赖训练数据中的知识
  + 无需外部调用，低延迟
  - 知识有截止日期，可能过时
  - 复杂计算容易出错

检索（Retrieval）：
  调用工具获取最新/精确信息
  + 信息准确，可追溯
  - 需要工具调用，有延迟
  - 依赖工具的质量
```

**最佳实践（Claude 的 Interleaved Thinking 实现了这个）：**

```
推理用于：决定检索什么 + 解读检索结果
检索用于：获取事实性、实时性、精确性数据

[thinking] 用户问的是某个 API 的具体参数
           我的训练数据里有，但可能已经更新了
           先搜索一下确认

[tool: web_search] → 搜索最新文档

[thinking] 搜索结果确认了参数名，但有一个新增参数我不知道
           结合文档和我对这个 API 的理解来回答

[text] 该 API 的参数包括...
```

---

## 十、Claude Code 中 Reasoning 的体现

Claude Code 在以下场景主动使用推理：

### 代码理解
```
读取文件后：
[thinking] 这个函数接受一个 callback，
           但在第 34 行直接修改了外部状态...
           这会导致如果 callback 抛出异常，状态仍然被修改
           这是个 bug
```

### 修改决策
```
用户要求修改 A：
[thinking] 修改 A 会影响调用它的 B 和 C
           B 在测试里直接依赖 A 的返回格式
           需要同时修改 B 的测试
           是否应该询问用户这个影响范围？
           → 修改影响较大，先告知用户
```

### 工具调用时序
```
[thinking] 我需要读取 5 个文件，但这些文件相互独立
           可以并发读取，减少延迟
           → 在一个响应里发出 5 个并行 tool_use
```

---

## 总结

```
Reasoning 的本质：
  给模型更多"计算空间"解决复杂问题
  通过生成中间步骤，让每步都站在前一步的基础上

Claude 的实现：
  Thinking Block = 安全隔离的推理草稿本
  Adaptive Thinking = 模型自己判断是否推理、推理多久
  Interleaved Thinking = 工具调用之间也持续推理
  Effort 参数 = 控制推理深度的旋钮

Agent 工程的关键决策：
  推理深度（effort）× 是否开启推理（thinking） × 推理 vs 检索
  → 根据任务复杂度、延迟要求、成本预算综合权衡

一句话总结：
  Reasoning 是 Agent 的"元认知"——
  不仅做事，还思考"怎么做"、"做得对不对"、"下一步做什么"。
```

---

## 相关

- [[Agent Planning]] — Reasoning 是 Planning 在单步层面的实现
- [[Agent Spec]] — Effort 和 Thinking 参数是 Spec 的一部分
- [[KV Cache 与 Prompt Caching]] — Thinking Block 的 token 如何影响缓存策略
- [[ReAct]] — Reasoning 在 ReAct 中以 Thought 步骤的形式出现
