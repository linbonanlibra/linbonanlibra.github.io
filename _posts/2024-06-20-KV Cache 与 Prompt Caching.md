# KV Cache 与 Prompt Caching

> Agent 工程中的核心优化机制

---

## 一、KV Cache 的本质：Transformer 层面的机制

在 Transformer 架构里，每次生成 token 时，模型需要对序列中**所有之前的 token** 计算 Attention（Key-Value 矩阵）。

$$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^\top}{\sqrt{d}}\right) \cdot V$$

**没有 KV Cache 时：**

```
输入：[system][tools][history][user_turn]
                                   ↑
每次推理，前面所有 token 的 K/V 都要重新计算
计算量 = O(n²)，n 是全部 token 数
```

**有 KV Cache 时：**

```
第一次（写缓存）：
[system][tools][history] → 计算并缓存这部分的 K/V 矩阵
                 ↑
              cache 写入

后续请求（读缓存）：
[cached K/V ✓][user_turn] → 只需计算新增部分
                 ↑
              直接复用，不重新计算
```

**核心：把不变的前缀的 K/V 矩阵存下来，后续请求直接复用。**

---

## 二、Claude API 的实现：Prompt Caching

Anthropic 将 KV Cache 暴露为 **Prompt Caching** 特性，通过 `cache_control` 标记来控制缓存边界。

### 最关键的一个不变量

> **缓存是前缀匹配。前缀中任何位置的任何字节变化，都会使该位置之后的所有内容失效。**

渲染顺序固定为：`tools → system → messages`

### 基本用法

```typescript
// 最简单：顶层自动缓存（缓存最后一个可缓存的块）
const response = await client.messages.create({
  model: "claude-opus-4-8",
  max_tokens: 16000,
  cache_control: { type: "ephemeral" },  // 自动放在最后一个可缓存块上
  system: "你是一个代码分析助手...(大量系统提示)",
  messages: [{ role: "user", content: "分析这段代码" }],
});

// 精细控制：手动打标记
const response = await client.messages.create({
  model: "claude-opus-4-8",
  max_tokens: 16000,
  system: [
    {
      type: "text",
      text: LARGE_STABLE_SYSTEM_PROMPT,      // 稳定内容放前面
      cache_control: { type: "ephemeral" },  // 在这里打断点
    },
  ],
  messages: [{ role: "user", content: userQuestion }],  // 可变内容不缓存
});
```

### 验证是否命中缓存

```typescript
console.log(response.usage.cache_creation_input_tokens); // 写入缓存的 token（花了 1.25x）
console.log(response.usage.cache_read_input_tokens);     // 从缓存读取的 token（只花 0.1x）
console.log(response.usage.input_tokens);                // 正常计费的 token
```

---

## 三、成本模型

| 操作 | 费率 | TTL |
|---|---|---|
| 正常输入 token | 1x | — |
| 缓存写入 | **1.25x**（5分钟 TTL）/ **2x**（1小时 TTL） | 5min / 1h |
| 缓存读取 | **0.1x** | — |

**盈亏平衡分析：**
- 5分钟 TTL：第 2 次请求开始盈利（1.25 + 0.1 = 1.35 < 2x 不缓存）
- 1小时 TTL：第 3 次请求开始盈利（2 + 0.2 = 2.2 < 3x 不缓存）

---

## 四、Agent 工程中的关键应用模式

### 难题 1：System Prompt 中途被修改 → 前缀失效

**错误做法：**

```typescript
// 每次都修改 top-level system，破坏缓存前缀
system: `你是代码助手。当前模式：${currentMode}。时间：${new Date()}`
//                                                          ↑
//                                               每次请求都不同！缓存永远失效
```

**正确做法：把动态内容注入 messages，而不是 system**

```typescript
// system 冻结，打上缓存标记
system: [
  {
    type: "text",
    text: FROZEN_SYSTEM_PROMPT,               // 永远不变
    cache_control: { type: "ephemeral" },
  },
],
// 动态内容放进 messages（不影响已缓存的前缀）
messages: [
  ...history,
  { role: "user", content: userMessage },
  // beta 功能：用 role:"system" 注入 operator 指令，而不是污染 user 内容
  { role: "system", content: `当前模式：${currentMode}` },
]
```

### 难题 2：工具列表在 Session 中途变化 → 位置 0 失效

**工具在渲染顺序中处于第一位。** 添加/删除/重排任何工具，整个缓存都失效。

```typescript
// ❌ 错误：每次根据上下文动态生成工具集
tools: buildToolsForUser(user)  // 每个用户的工具集不同 → 无跨用户缓存

// ✅ 正确：使用 Tool Search 动态发现，而不是替换工具集
// Tool Search 是追加（append）而不是替换，已缓存的前缀保持不变
tools: [
  { type: "tool_search", ... },  // 固定的工具搜索能力
  // Claude 在需要时自己搜索并加载特定工具的 schema
]
```

### 难题 3：多轮对话 → 随对话增长缓存断点移动

```typescript
const messages: Anthropic.MessageParam[] = [];

async function chat(userMessage: string) {
  messages.push({ role: "user", content: userMessage });
  
  const response = await client.messages.create({
    model: "claude-opus-4-8",
    max_tokens: 16000,
    cache_control: { type: "ephemeral" },  // 自动放在最后一个可缓存块
    system: STABLE_SYSTEM,
    messages,
  });
  
  // ⚠️ 必须 append 完整的 response.content，不能只取 text！
  // 如果用了 Compaction，compaction block 必须保留！
  messages.push({ role: "assistant", content: response.content });
  
  return response;
}
```

每轮对话后，新一轮的缓存命中量递增：

```
第1轮：[system✓缓存] + [user1]               → system 缓存命中
第2轮：[system✓] + [user1+a1✓缓存] + [user2] → system + turn1 命中
第3轮：[system✓] + [turn1✓] + [turn2✓缓存] + [user3] → 更多命中
```

---

## 五、Claude Code 自身的 KV Cache 架构

这是 Agent 工程的真实案例。

```
主 Claude（Opus 4.8）
├── 保持不变的 system prompt（缓存稳定）
├── 工具列表固定（不随任务变化）
│
├─ Explore 子 Agent（Haiku 4.5）
│   └── 独立的上下文窗口，不影响主 Agent 缓存
│
└─ general-purpose 子 Agent
    └── 用于复杂子任务，结论返回主 Agent
```

**为什么 Explore 用 Haiku？**
- Haiku 更便宜、更快
- 独立的上下文窗口，扫描大量文件不污染主 Agent 的 KV Cache
- 结论（几行文字）返回主 Agent，主 Agent 的缓存前缀完全不受影响

**核心策略：不在同一模型中途切换模型** —— 切换模型会使缓存失效，因为缓存是模型级别隔离的。动态子任务改为派给子 Agent，主 Agent 始终保持同一模型和稳定前缀。

---

## 六、常见的"静默失效"陷阱（Silent Invalidators）

```typescript
// ❌ 时间戳注入 system → 每秒不同
system: `当前时间：${new Date().toISOString()} 你是助手...`

// ❌ 非确定性 JSON 序列化
system: JSON.stringify(config)  // 对象 key 顺序不固定

// ❌ 工具集因用户而异
tools: buildTools(userId)  // 每个用户独立前缀，无共享缓存

// ❌ Set 迭代顺序不保证
tools: [...new Set(userTools)]  // 顺序随机 → 每次不同
```

**排查方法：** 如果 `cache_read_input_tokens` 一直为 0，说明有静默失效器——对比两次请求的渲染字节找差异。

| 模式 | 原因 | 修复方式 |
|---|---|---|
| `datetime.now()` 在 system 中 | 每次请求前缀不同 | 移到 messages 末尾 |
| `uuid4()` 在 system 中 | 同上 | 同上 |
| `json.dumps(d)` 无 sort_keys | 序列化不确定 | `json.dumps(d, sort_keys=True)` |
| 工具集因用户变化 | 前缀因人而异 | 用 Tool Search |
| Session 中途切换模型 | 缓存按模型隔离 | 固定主 Agent 模型 |

---

## 七、缓存预热（Cache Pre-warming）

对于交互式 Agent，冷启动延迟很明显。可以在流量到来前预热：

```typescript
// max_tokens: 0 → 只做 prefill，写入缓存，不生成任何输出
// 计费：只有 cache_creation 费用，零 output token
await client.messages.create({
  model: "claude-opus-4-8",
  max_tokens: 0,
  system: [
    {
      type: "text",
      text: SYSTEM_PROMPT,
      cache_control: { type: "ephemeral" },
    },
  ],
  messages: [{ role: "user", content: "warmup" }],
});
```

**何时预热？**
- 应用启动时（流量未到之前）
- Worker boot 时
- 两次流量间隔超过 TTL（否则正常请求自动维持缓存）

**何时不需要预热？**
- 流量连续（请求间隔 < TTL），实际请求自动维持缓存
- 前缀较小或低于缓存门槛
- 前缀因请求而异，无法共享

---

## 八、各模型的最小缓存门槛

| 模型 | 最小可缓存 token 数 |
|---|---|
| Opus 4.8 / 4.7 / 4.6 / Haiku 4.5 | **4096** |
| Fable 5 / Sonnet 4.6 | **2048** |
| Sonnet 4.5 / 4.1 / 4.0 | **1024** |

> 低于门槛的前缀**静默不缓存**——不报错，只是 `cache_creation_input_tokens: 0`。

---

## 九、失效层级速查

并非所有参数变化都让所有缓存失效，了解层级可以更精细地设计：

| 变化类型 | tools 缓存 | system 缓存 | messages 缓存 |
|---|:---:|:---:|:---:|
| 工具定义（增/删/改顺序） | ❌ 失效 | ❌ 失效 | ❌ 失效 |
| 切换模型 | ❌ 失效 | ❌ 失效 | ❌ 失效 |
| system prompt 内容变化 | ✅ 保留 | ❌ 失效 | ❌ 失效 |
| `tool_choice` / `thinking` 开关 | ✅ 保留 | ✅ 保留 | ❌ 失效 |
| messages 内容变化 | ✅ 保留 | ✅ 保留 | ❌ 失效 |

---

## 总结：Agent 工程的 KV Cache 设计原则

```
稳定性 ↑ → 越靠前                    稳定性 ↓ → 越靠后

[tools 固定] → [system 冻结] → [缓存断点] → [messages 可变内容]
                                    ↑
                              cache_control 放这里
```

1. **冻结 system prompt**，动态内容通过 `role:"system"` 消息或 user turn 注入
2. **工具列表不变**，动态发现用 Tool Search（追加而非替换）
3. **不切换模型**，动态子任务派给子 Agent（可用更便宜的模型）
4. **多轮对话**断点跟着最新 turn，用顶层 `cache_control` 自动处理
5. **验证命中**：看 `cache_read_input_tokens` 是否大于 0
6. **交互式场景**考虑预热，批处理场景不需要
