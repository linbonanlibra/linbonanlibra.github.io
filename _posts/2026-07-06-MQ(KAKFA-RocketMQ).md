# Kafka & RocketMQ 技术笔记

---

## 一、核心差异总览

### 背景与定位

| 维度 | Kafka | RocketMQ |
|------|-------|----------|
| 出身 | LinkedIn → Apache | 阿里巴巴 → Apache |
| 核心定位 | **流处理平台**，高吞吐日志/事件流 | **业务消息中间件**，金融级可靠性 |
| 典型场景 | 日志收集、数据管道、实时分析 | 电商订单、交易、通知推送 |

### 关键功能差异

| 功能 | Kafka | RocketMQ |
|------|-------|----------|
| **事务消息** | 有（幂等/exactly-once 语义） | ✅ 更完善（半消息 + 二阶段提交） |
| **延迟消息** | ❌ 原生不支持 | ✅ 支持 18 个固定延迟等级（开源），商业版任意时间 |
| **死信队列** | ❌ 需自己实现 | ✅ 原生支持 DLQ |
| **消息重试** | ❌ 消费者自己处理 | ✅ 自动重试 + 退避策略 |
| **消息过滤** | Consumer 端过滤 | Broker 端 Tag / SQL92 过滤 |
| **消息回溯** | ✅ 按 offset/时间戳 | ✅ 按时间戳 |

### 吞吐量与延迟

| | Kafka | RocketMQ |
|--|-------|----------|
| **峰值吞吐** | 更高（百万级 msg/s） | 稍低（十万级） |
| **延迟** | 毫秒级（batch 优化后略高） | 微秒～毫秒（单条延迟更低） |
| **适合** | 大批量流式数据 | 低延迟业务消息 |

### 消费模型对比

| | Kafka | RocketMQ |
|--|-------|----------|
| 推/拉 | **Pure Pull** | **Push（封装了 Pull，见第二节）** |
| 消费位点 | 存在 Kafka 自身（`__consumer_offsets`） | 存在 Broker 端 |
| 广播消费 | 不原生支持 | ✅ 支持广播消费模式 |

### 运维与生态

| | Kafka | RocketMQ |
|--|-------|----------|
| 依赖 | 早期依赖 ZooKeeper，新版（KRaft）已去除 | 依赖 NameServer（轻量） |
| 生态 | **极其丰富**（Flink/Spark/Kafka Connect 等） | 较好，偏阿里云生态 |

### 选型建议

```
选 Kafka 当：
  ✅ 日志收集、埋点、数据管道
  ✅ 与 Flink/Spark 集成做流计算
  ✅ 极致吞吐，Topic 数量可控

选 RocketMQ 当：
  ✅ 电商、支付等需要事务消息/延迟消息
  ✅ 业务 Topic 数量多（上千个）
  ✅ 需要死信队列、自动重试等开箱即用的业务特性
  ✅ 国内业务场景，阿里云生态
```

> **一句话总结**：Kafka 是「高吞吐的数据高速公路」，RocketMQ 是「可靠的业务快递员」。

---

## 二、RocketMQ 消费模型：Push 是封装的 Pull

### 直觉区分

- **Pull（拉）**：消费者主动问 Broker "有新消息吗？给我一批"
- **Push（推）**：Broker 主动把消息推给消费者

RocketMQ 的 Push **并不是真正的服务端推送**，而是基于**长轮询（Long Polling）**实现的。

### 真实机制

```
表面上（业务代码视角）：
  consumer.registerMessageListener((msgs) -> {
      // 消息自动送到这里，感觉像 Broker 推过来的
  });

底层实际发生的事：
  1. Consumer 向 Broker 发送 Pull 请求
  2. 如果 Broker 没有新消息 → 不立即返回，挂起请求（默认最长 15s）
  3. 期间 Broker 有新消息 → 立即触发响应，返回给 Consumer
  4. Consumer 收到消息 → 交给 MessageListener 回调
  5. 回调处理完 → 立刻发下一次 Pull 请求（循环）
```

### 时序图

```
Consumer                          Broker
   |                                |
   |──── Pull 请求 ────────────────>|
   |                                |  (暂无消息，挂起 15s)
   |                                |  ....有新消息了！
   |<─── 响应（带消息）─────────────|
   |                                |
   | [执行 MessageListener 回调]    |
   |                                |
   |──── Pull 请求（立刻再拉）─────>|  ← 这一步对业务不可见
   |                                |
```

### 为什么这样设计？

| 问题 | 真 Push 的痛点 | 长轮询的解法 |
|------|----------------|--------------|
| **流量控制** | Broker 不知道 Consumer 处理速度，可能压垮消费者 | Consumer 自己控制何时发下一次 Pull |
| **连接管理** | Broker 需维护所有消费者的推送状态，复杂 | Consumer 主动来拿，Broker 无需维护推送队列 |
| **实时性** | 普通轮询延迟高 | 消息到达时立即触发响应，延迟低 |

### 与 Kafka Pure Pull 的区别

```
Kafka Pull：
  Consumer 自己写循环，自己决定 poll 时间间隔
  → 灵活，但业务代码需要管理拉取逻辑

RocketMQ Push（封装的 Pull）：
  SDK 内部自动维护拉取循环 + 长轮询 + 线程池
  → 业务只需写 MessageListener 回调，更简单
```

> **一句话总结**：RocketMQ 的 Push 模式 = SDK 帮你封装好的自动长轮询。对业务透明，底层是 Consumer 不断主动 Pull。

---

## 三、RocketMQ 存储架构：CommitLog + ConsumeQueue

### 整体分层结构

```
生产者发消息
     │
     ▼
┌─────────────────────────────────────────┐
│              CommitLog                   │  ← 所有消息的"原始仓库"
│  (一个 Broker 只有一个，顺序追加写入)    │
└─────────────────────────────────────────┘
     │
     │ 异步分发（ReputMessageService）
     ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ ConsumeQueue │  │ ConsumeQueue │  │ ConsumeQueue │  ← 每个 Topic 的每个 Queue
│  TopicA-0   │  │  TopicA-1   │  │  TopicB-0   │     各有一个索引文件
└──────────────┘  └──────────────┘  └──────────────┘
     │
     │ 消费者拉消息时
     ▼
  先查 ConsumeQueue（拿到偏移量）→ 再去 CommitLog（读消息体）
```

---

### CommitLog 详解

#### 物理结构

```
$ROCKETMQ_HOME/store/commitlog/
├── 00000000000000000000          ← 第 1 个文件（1GB）
├── 00000000001073741824          ← 第 2 个文件（文件名 = 起始全局偏移量）
├── 00000000002147483648
└── ...
```

- 每个文件固定 **1GB**
- 文件名就是该文件在整个 CommitLog 中的**起始全局偏移量**
- 写满就新建下一个，**永远只追加写末尾**，不修改已写内容

#### 单条消息存储格式

```
┌──────────────────────────────────────────────────────┐
│ totalSize       (4B)  消息总长度                      │
│ magicCode       (4B)  魔数，标记消息合法性            │
│ bodyCRC         (4B)  消息体 CRC 校验                 │
│ queueId         (4B)  属于哪个 Queue                  │
│ flag            (4B)  标志位                          │
│ queueOffset     (8B)  在对应 ConsumeQueue 中的逻辑偏移│  ← 关键
│ physicOffset    (8B)  自身在 CommitLog 中的物理偏移   │  ← 关键
│ sysFlag         (4B)  事务状态等系统标志              │
│ bornTimestamp   (8B)  消息产生时间                    │
│ bornHost        (8B)  发送方地址                      │
│ storeTimestamp  (8B)  存储时间                        │
│ storeHost       (8B)  存储的 Broker 地址              │
│ reconsumeTimes  (4B)  重试次数                        │
│ topic length    (1B)                                  │
│ topic           (NB)  Topic 名称                      │
│ propertiesLen   (2B)                                  │
│ properties      (NB)  扩展属性（Tag/Key/延迟级别等）  │
│ bodyLen         (4B)                                  │
│ body            (NB)  消息体（真实内容）               │
└──────────────────────────────────────────────────────┘
```

> CommitLog 里的一条记录，**包含完整消息内容 + 它属于哪个 Topic/Queue 的元信息**。

---

### ConsumeQueue 详解

#### 逻辑定位

```
CommitLog  = 一个巨大的混合日志，TopicA、TopicB 的消息混在一起
ConsumeQueue = 「按 Topic+QueueId 建的分类索引」，每条记录指向 CommitLog 中的位置
```

#### 物理存储路径

```
$ROCKETMQ_HOME/store/consumequeue/
└── TopicA/
│   ├── 0/                    ← QueueId=0
│   │   ├── 00000000000000000000   （每个文件 = 30w 条索引 ≈ 5.7MB）
│   │   └── 00000000000006000000
│   └── 1/                    ← QueueId=1
│       └── 00000000000000000000
└── TopicB/
    └── 0/
        └── 00000000000000000000
```

#### 每条记录内容（固定 20 字节）

```
┌──────────────────────────────────────────────────┐
│  commitLogOffset   (8B)  物理偏移量               │  ← 去 CommitLog 的哪里找
│  msgSize           (4B)  消息大小                 │  ← 读多少字节
│  tagsCode          (8B)  Tag 的 hashCode          │  ← 用于 Broker 端 Tag 过滤
└──────────────────────────────────────────────────┘
         总计 20 字节，定长！
```

> **定长的精妙之处**：消费第 N 条消息，直接用 `offset = N × 20` 计算文件位置，**O(1) 定位，无需遍历**。

---

### 消息写入全流程

```
① Producer 发消息到 Broker

② Broker 写入 CommitLog（同步/异步刷盘）
   └─ 追加到当前文件末尾
   └─ 返回写入成功的 physicOffset（全局偏移量）

③ ReputMessageService（异步分发线程）
   └─ 持续从 CommitLog 读取新写入的消息
   └─ 解析出 Topic、QueueId、physicOffset、msgSize、tagsCode
   └─ 写入对应的 ConsumeQueue 文件（也是顺序追加）
   └─ 同时写入 IndexFile（用于按 MessageKey 查询）

④ Consumer 拉消息
   └─ 告诉 Broker：我要消费 TopicA QueueId=0 的第 100 条
   └─ Broker 去 ConsumeQueue[TopicA][0] 的第 100×20 字节处
   └─ 读出 commitLogOffset + msgSize
   └─ 用 commitLogOffset 去 CommitLog 读出完整消息
   └─ 返回给 Consumer
```

**关键**：步骤 ② 和 ③ 是解耦的：

```
CommitLog 写入（热路径，同步）
     ↓ 异步
ConsumeQueue 写入（冷路径，微小延迟）

好处：消息写入延迟只取决于 CommitLog 的写入速度，不被 ConsumeQueue 写入阻塞
```

---

### 为什么「Topic 多时性能好」？

#### Kafka 的问题

```
Kafka：
  TopicA-Partition0 → 独立文件
  TopicA-Partition1 → 独立文件
  TopicB-Partition0 → 独立文件
  ...

1000 个 Topic × 平均 3 个 Partition = 3000 个文件同时被写入
→ 操作系统 PageCache 碎片化
→ 磁盘写入从「顺序 I/O」退化为「随机 I/O」
→ HDD 场景下性能断崖式下降
```

#### RocketMQ 的解法

```
无论多少 Topic、多少 Queue：
  CommitLog 始终只有 1 个写入点（当前文件末尾）

1000 个 Topic → 写 CommitLog 还是一个顺序追加
              → ConsumeQueue 文件虽多，但每条只有 20B，写入量极小
              → PageCache 压力集中在一处，顺序 I/O 保持
```

---

### 一条消息的完整「坐标」

| 坐标 | 含义 | 说明 |
|------|------|------|
| `physicOffset` | 在 CommitLog 中的全局字节偏移 | CommitLog 文件名 + 内部偏移 |
| `queueOffset` | 在某个 ConsumeQueue 中是第几条 | ConsumeQueue 的行号（逻辑序号） |
| `MessageId` | 对外暴露的 ID | Broker地址 + physicOffset 编码 |

> 消费者的**消费进度（Consumer Offset）** 记录的是 `queueOffset`，即「我消费到 TopicA-Queue0 的第几条」。

---

## 四、Broker 集群与 Topic 路由

### Broker 是什么单位？

```
Broker = 一个 RocketMQ 服务进程，负责存储和收发消息
         （通常一台物理机跑一个 Broker 进程）
```

生产环境是**集群部署**：

```
┌─────────────────────────────────────────────────┐
│                  RocketMQ 集群                   │
│                                                  │
│  ┌──────────────┐      ┌──────────────┐          │
│  │ BrokerA      │      │ BrokerB      │          │
│  │ (Master)     │      │ (Master)     │          │
│  └──────┬───────┘      └──────┬───────┘          │
│         │ 复制                │ 复制              │
│  ┌──────┴───────┐      ┌──────┴───────┐          │
│  │ BrokerA      │      │ BrokerB      │          │
│  │ (Slave)      │      │ (Slave)      │          │
│  └──────────────┘      └──────────────┘          │
│                                                  │
│  ┌────────────────────────────────────┐          │
│  │         NameServer 集群            │          │
│  │  （类似注册中心，存储路由信息）      │          │
│  └────────────────────────────────────┘          │
└─────────────────────────────────────────────────┘
```

---

### 一个 Topic 的消息可以存在多个 Broker

**是的，这是常规做法。** 创建 Topic 时可以指定它在哪些 Broker 上各有多少个 Queue：

```
Topic：OrderTopic
  └── BrokerA 上创建 4 个 Queue（QueueId: 0, 1, 2, 3）
  └── BrokerB 上创建 4 个 Queue（QueueId: 0, 1, 2, 3）

结果：OrderTopic 共 8 个 Queue，分散在 2 个 Broker 上
      每个 Broker 只存这个 Topic 的「一部分」消息（水平分片）
```

```
BrokerA 的 CommitLog：
  [OrderTopic msg1][UserTopic msg1][OrderTopic msg3]...
  ↓ 分发
  ConsumeQueue/OrderTopic/0/  （BrokerA 本机索引）

BrokerB 的 CommitLog：
  [OrderTopic msg2][OrderTopic msg4][PayTopic msg1]...
  ↓ 分发
  ConsumeQueue/OrderTopic/0/  （BrokerB 本机索引）
```

> **注意**：BrokerA 和 BrokerB 上 QueueId 相同的 Queue 是**相互独立的分片**，不是副本关系（副本是 Master/Slave 的事）。

---

### NameServer 的路由发现机制

#### NameServer 存储的路由表

```
Topic: OrderTopic
  ├── BrokerA (ip: 192.168.1.1:10911)
  │     ├── QueueId 0  (可读可写)
  │     ├── QueueId 1  (可读可写)
  │     ├── QueueId 2  (可读可写)
  │     └── QueueId 3  (可读可写)
  └── BrokerB (ip: 192.168.1.2:10911)
        ├── QueueId 0  (可读可写)
        ├── QueueId 1  (可读可写)
        ├── QueueId 2  (可读可写)
        └── QueueId 3  (可读可写)
```

#### 路由发现全流程

```
Broker 启动阶段：
  → 向所有 NameServer 注册自己 + 持有的 Topic 信息
  → 每隔 30s 向 NameServer 发心跳（更新路由）
  → NameServer 超过 120s 没收到心跳 → 剔除该 Broker

Producer 发消息：
  1. 启动时从 NameServer 拉取 Topic 路由表（缓存本地）
  2. 每隔 30s 定时更新路由缓存
  3. 发消息时按策略选一个 Queue（轮询 / 故障规避）
  4. 直连对应的 Broker 发送

Consumer 拉消息：
  1. 从 NameServer 拉取路由表
  2. 知道 Topic 的所有 Queue 分布在哪些 Broker
  3. 消费者组内做 Queue 分配（Rebalance）
  4. 每个消费者认领若干 Queue，直连对应 Broker 拉取
```

#### 消费者 Rebalance 示意

```
假设：OrderTopic 共 8 个 Queue，消费者组有 2 个实例

Consumer-1 负责：
  BrokerA/Queue-0, BrokerA/Queue-1
  BrokerB/Queue-0, BrokerB/Queue-1

Consumer-2 负责：
  BrokerA/Queue-2, BrokerA/Queue-3
  BrokerB/Queue-2, BrokerB/Queue-3

拉消息时：Consumer-1 同时向 BrokerA 和 BrokerB 发 Pull 请求
          各自维护各自 Queue 的消费进度（offset）
```

---

### Master / Slave 的两个维度

```
水平维度（分片）：BrokerA 和 BrokerB 存不同的消息 → 扩容吞吐量
垂直维度（副本）：BrokerA-Master 和 BrokerA-Slave 存相同的消息 → 高可用
```

```
BrokerA-Master  ──同步/异步复制──►  BrokerA-Slave
    │                                     │
  写入（Producer 写 Master）       读取（Consumer 在 Master 压力大时可读 Slave）
```

> RocketMQ 4.x 用 Master/Slave 模式；  
> RocketMQ 5.x 引入 **DLedger（Raft）**，支持自动选主，更像 Kafka 的 ISR 机制。

---

### 整体架构全景图

```
                    ┌─────────────┐
                    │  NameServer │ ← Broker 注册路由
                    │   (集群)    │ ← Client 查询路由
                    └──────┬──────┘
                           │ 路由信息
              ┌────────────┴────────────┐
              │                         │
         查路由                      查路由
              │                         │
       ┌──────▼──────┐           ┌──────▼──────┐
       │  Producer   │           │  Consumer   │
       └──────┬──────┘           └──────┬──────┘
              │ 直连 Broker 写           │ 直连 Broker 拉
    ┌─────────┴──────────┐    ┌─────────┴──────────┐
    ▼                    ▼    ▼                    ▼
┌───────┐           ┌───────┐ ┌───────┐       ┌───────┐
│Broker │           │Broker │ │Broker │       │Broker │
│   A   │           │   B   │ │   A   │       │   B   │
│Master │           │Master │ │Slave  │       │Slave  │
└───────┘           └───────┘ └───────┘       └───────┘
    │                   │
    ▼                   ▼
CommitLog           CommitLog     ← 各自独立，存不同的消息分片
```

---

## 五、核心概念速查

| 概念 | 说明 |
|------|------|
| **CommitLog** | 一个 Broker 上所有消息的原始存储，顺序追加，文件大小 1GB |
| **ConsumeQueue** | 按 Topic+QueueId 建的索引文件，每条固定 20B，指向 CommitLog |
| **NameServer** | 轻量级注册中心，存储 Broker 和 Topic 的路由信息 |
| **Broker** | 消息存储和收发的服务进程，集群部署，分 Master/Slave |
| **Queue** | Topic 的分区单位（类似 Kafka 的 Partition），可分散在多个 Broker |
| **physicOffset** | 消息在 CommitLog 中的全局字节偏移 |
| **queueOffset** | 消息在某个 ConsumeQueue 中的逻辑序号（Consumer 进度追踪用） |
| **Long Polling** | Push 模式的底层实现：Consumer Pull + Broker 挂起等待 |
| **ReputMessageService** | 异步将 CommitLog 新消息分发写入 ConsumeQueue 的内部线程 |

---

## 六、Kafka 水位（High Watermark）机制

### 基础概念：三个 Offset

一个 Partition 的日志文件中，同时存在几个关键位置指针：

```
Partition Log (Leader 副本)

Offset:   0    1    2    3    4    5    6    7    8
         [msg][msg][msg][msg][msg][msg][msg][msg][   ]
                                   ↑              ↑
                              High Watermark    LEO
                                 (HW = 6)    (LEO = 8)
                              消费者可见边界   下一条写入位置
```

| 概念 | 全称 | 含义 |
|------|------|------|
| **LEO** | Log End Offset | 下一条消息将写入的位置，即当前日志末尾 + 1 |
| **HW** | High Watermark | 所有 ISR 副本都已同步的最高 offset，消费者**只能看到 HW 之前**的消息 |

> **核心规则**：Consumer 无论怎么拉，最多只能拉到 HW 位置之前的消息，HW 之后的消息对消费者不可见。

---

### 为什么需要 HW？

**没有 HW 会怎样：**

```
场景：
  Leader LEO=8，Consumer 已经消费到 offset=7

  此时 Leader 宕机，Broker-2 成为新 Leader
  Broker-2 的 LEO=6，它根本没有 offset 6、7 的消息

  Consumer 再来消费 offset=6 → 新 Leader 找不到 → 数据丢失或错乱
```

**HW 的作用：**

```
HW = 所有 ISR 副本都确认写入的最高位置
   = 即使任意一个 ISR 副本成为新 Leader，也一定有这些消息

消费者只消费 HW 以内的数据 → 保证消费到的消息在任何副本切换后都不会丢失
```

---

### HW 是每个副本都维护的

每个副本（Leader 和所有 Follower）都各自维护一份 HW 和 LEO，但含义和权威性不同：

```
同一个 Partition 的三个副本各自的状态：

Leader  (Broker-1):  LEO=8, HW=6  ← 权威 HW，由 Leader 计算
Follower(Broker-2):  LEO=7, HW=5  ← 本地 HW，从 Leader 同步来，有滞后
Follower(Broker-3):  LEO=6, HW=4  ← 本地 HW，从 Leader 同步来，有更多滞后
```

- **Leader 的 HW**：由 Leader 根据所有 ISR 的 LEO 计算，是全局唯一的"真实水位"
- **Follower 的 HW**：从上一次 Fetch 响应里 Leader 带回来的，最多落后一个 Fetch 周期

---

### HW 的推进机制

ISR（In-Sync Replicas）= 与 Leader 保持同步的副本集合。Follower 超过 `replica.lag.time.max.ms`（默认 30s）没有同步 → 被踢出 ISR。

**一轮完整的 HW 传播流程：**

```
                  ① Follower 发 Fetch 请求（携带自己当前的 LEO）
Follower ──────────────────────────────────────────────► Leader
         fetchOffset=7, followerLEO=7

                                          Leader 收到后：
                                          - 更新对该 Follower 的 LEO 记录
                                          - 重新计算 HW = min(所有 ISR 的 LEO)
                                          - 准备响应

                  ② Leader 返回 Fetch 响应（携带数据 + 当前最新 HW）
Follower ◄────────────────────────────────────────────── Leader
         data=[msg7, msg8], leaderHW=7

Follower 收到后：
- 追加消息到本地日志，更新自己的 LEO=9
- 更新本地 HW = min(leaderHW=7, 自己的LEO=9) = 7
```

**HW 滞后时序示意：**

```
时刻   Leader HW   Follower HW   说明
──────────────────────────────────────────────────────
T0      5            3           初始状态
T1      6            3           Leader HW 推进，Follower 还不知道
T2      6            5           Follower Fetch 一轮，拿到 leaderHW=6
                                 但自身 LEO=5，所以 HW=min(6,5)=5
T3      7            6           再一轮后，Follower HW 继续追赶
T4      7            7           最终追平
```

> **结论：Follower 的 HW 始终比 Leader 的 HW 至少落后一个 Fetch 周期。**

---

### 消费者能从 Follower 拉取吗？

**Kafka 2.4 之前：只能从 Leader 读**

```
Consumer 永远只连 Leader 副本拉消息
Follower 只做一件事：从 Leader 同步数据（纯粹用于容灾）

缺点：多 AZ 部署时，Consumer 可能跨 AZ 读 Leader → 跨 AZ 流量费用 + 更高延迟
```

**Kafka 2.4+：KIP-392，支持从最近副本读**

```
引入 replica.selector.class 配置，可启用 RackAwareReplicaSelector
Consumer 优先读同机房 / 同 AZ 的 Follower 副本

好处：✅ 减少跨 AZ 流量费用  ✅ 降低读取延迟
代价：⚠️ 消费可见性略滞后（Follower 的 HW ≤ Leader 的 HW）
```

**从 Follower 读时 HW 不一致的影响：**

```
假设此刻：
  Leader HW = 7（消息 0~6 对外可见）
  Follower HW = 5（消息 0~4 对外可见）

Consumer-A 从 Leader 读：能拿到 offset 0~6 的消息
Consumer-B 从 Follower 读：只能拿到 offset 0~4 的消息
```

Kafka 的处理方式：Follower 以**自己本地的 HW** 为上限返回消息，保证返回的数据一定写入了本副本，不会出现幽灵数据。可见性延迟 ≈ Follower Fetch 周期（通常毫秒级）。

---

### HW 与 acks 可靠性配置的关系

```
acks=0：不等待任何确认就返回成功 → 可能丢消息

acks=1（默认）：Leader 写入即返回成功，不等 Follower 同步
               → Leader 挂掉而 Follower 未同步 → 消息丢失

acks=-1 / acks=all：等待所有 ISR 副本都写入才返回成功
                   → 返回成功时 HW 已推进过这条消息
                   → 消费者能看到 = 已被所有 ISR 持久化 = 不会因副本切换丢失
```

> **生产推荐配置**：`acks=all` + `min.insync.replicas=2`  
> `min.insync.replicas=2` 确保 ISR 里至少有 2 个副本，避免"ISR 只剩 Leader 自己"时的假安全。

---

### 经典问题：HW 机制下的数据丢失（与 Leader Epoch 修复）

Kafka 早期版本存在一个由 HW 机制引发的 bug：

```
初始状态：
  Leader(B1):   [0][1][2]  LEO=3, HW=3
  Follower(B2): [0][1][2]  LEO=3, 本地 HW 尚未更新=2

B2 宕机重启：
  B2 按本地 HW=2 截断日志 → 变成 [0][1]  LEO=2

B1 也同时宕机，B2 成为新 Leader：
  B2 LEO=2，HW=2

B1 恢复，向新 Leader(B2) 同步：
  B2 告诉 B1 当前 LEO=2，B1 把自己截断到 offset=2
  → 原来 offset=2 那条消息永久丢失
```

**解决方案：Leader Epoch（Kafka 0.11+ 引入）**

```
每次 Leader 切换，Epoch 编号 +1
Follower 重启后，先向 Leader 查询当前 Epoch 对应的起始 offset
再决定截断到哪里

→ 不再单纯依赖本地 HW 来截断，避免了上述数据丢失问题
```

---

### HW 机制总览

```
                     ┌─────────────────────┐
                     │  Leader (Broker-1)   │
                     │  LEO=8, HW=7 ◄──────┤  ← 唯一权威 HW（由 ISR LEO 计算）
                     └──────────┬──────────┘
                                │
              Fetch(LEO=7)      │           Fetch(LEO=6)
              ┌─────────────────┘           └──────────────────┐
              │ ← data + leaderHW=7                            │ ← data + leaderHW=7
              ▼                                                 ▼
  ┌───────────────────┐                           ┌────────────────────┐
  │ Follower(Broker-2)│                           │ Follower(Broker-3) │
  │ LEO=8, HW=7       │                           │ LEO=7, HW=6        │
  │ （已追上 Leader）  │                           │ （还落后一点）      │
  └───────────────────┘                           └────────────────────┘
           │                                                │
     Consumer-A 读                                    Consumer-B 读
     看到 offset 0~6                                  看到 offset 0~5
     （和 Leader 一致）                               （比 Leader 少看到 1 条）
```

**HW 一致性定性结论：**

| 问题 | 答案 |
|------|------|
| HW 是强一致的吗？ | 否，是最终一致；Follower HW ≤ Leader HW，会追赶但有滞后 |
| 谁是 HW 权威来源？ | 只有 Leader 计算真实 HW |
| Follower HW 从哪来？ | 上一次 Fetch 响应里 Leader 捎带的，有一个 Fetch 周期滞后 |
| 从 Follower 读安全吗？ | 安全，不会读到幽灵数据，但可见性比 Leader 慢几毫秒 |

---

### 水位相关核心概念速查

| 概念 | 一句话 |
|------|--------|
| **LEO** | 当前副本下一条要写的位置 |
| **HW** | 所有 ISR 都确认写入的最高位置，消费者的可见上限 |
| **ISR** | 与 Leader 保持同步的副本集合，决定 HW 的推进速度 |
| **acks=all** | 写入成功 = HW 已推进，最强可靠性保障 |
| **Leader Epoch** | 解决 HW 机制下 Follower 重启可能错误截断的问题 |
| **KIP-392** | Kafka 2.4+ 支持从最近副本读，减少跨 AZ 流量 |

---

## 七、RocketMQ 中的"水位"等价机制

### Kafka HW 解决了哪些问题

```
问题 1：消费可见性边界
        → 消费者不能读到"还没完成副本同步"的消息

问题 2：副本切换时的数据安全
        → 新 Leader 没有的消息，消费者不能提前消费

问题 3：读写解耦时的一致性
        → Producer ack 和 Consumer 可见是两件事，HW 是中间的桥
```

---

### 根本设计差异：把问题消灭在写入阶段

Kafka 是**先写入、后同步、用 HW 划边界**的思路：

```
Kafka 的路径：
  Producer 写入 → Leader ack（可能还没同步）→ 异步同步给 Follower
                                                    ↓
                                            HW 推进后，Consumer 才能看见

  HW 是事后补救：写入和可见性解耦，用 HW 来协调
```

RocketMQ 是**在写入阶段就决定是否等待副本**的思路：

```
SYNC_MASTER 模式：
  Producer 写入 → Master 写 CommitLog → 等 Slave 确认收到 → 才 ack Producer
                                                              ↓
                                                    Consumer 此时读到 = 一定安全

ASYNC_MASTER 模式：
  Producer 写入 → Master 写 CommitLog → 立即 ack Producer
                                              ↓（Slave 异步同步，可能滞后）
                                        Consumer 读到 = 可能存在丢失风险
```

> **结论：RocketMQ 4.x 没有显式的 HW 概念，它用"同步模式"在写入时就解决了 HW 要事后解决的问题。**

---

### 两种同步模式详解

**SYNC_MASTER（同步双写）：**

```
Producer                Master              Slave
   │                      │                  │
   │──── 发消息 ──────────►│                  │
   │                      │──── 复制 ────────►│
   │                      │◄─── 确认收到 ─────│
   │◄─── ack（成功）───────│                  │

✅ ack = 已写入 Master + 至少一个 Slave
✅ Consumer 读到的消息，Slave 一定也有
✅ Master 宕机切 Slave，不会丢已 ack 的消息
❌ 写入延迟更高（需等 Slave 确认）
```

**ASYNC_MASTER（异步复制）：**

```
Producer                Master              Slave
   │                      │                  │
   │──── 发消息 ──────────►│                  │
   │◄─── ack（成功）───────│                  │
   │                      │──── 异步复制 ────►│（稍后）

✅ 写入延迟低
❌ ack 时 Slave 可能还没同步
❌ Master 宕机 → Slave 切主 → 可能丢失最近几条消息
```

---

### Consumer 从 Slave 读时的边界控制

RocketMQ Consumer 默认从 Master 读。Master 压力过大时，Broker 会在响应中告知 Consumer "去 Slave 读"。此时 Slave 用 **`maxOffsetInQueue`**（自己本地 ConsumeQueue 的最大 offset）来限制返回范围：

```
Slave 本地 ConsumeQueue:
  [0][1][2][3][4][5][6]       maxOffsetInQueue = 7

Master ConsumeQueue:
  [0][1][2][3][4][5][6][7][8]  maxOffsetInQueue = 9

Consumer 从 Slave 读：最多只能读到 offset=6，消息 7、8 还没同步到 Slave
```

`maxOffsetInQueue` 与 Kafka Follower 本地 HW 类比：

```
Kafka Follower 的 HW：
  明确等于"Leader 通知我的已确认水位"，有严格的语义定义和传播协议

RocketMQ Slave 的 maxOffsetInQueue：
  就是"我本地同步到了哪里"，无专门水位协议，更粗放
```

---

### RocketMQ 5.x：DLedger 引入真正的"提交水位"

RocketMQ 5.x 引入基于 **Raft 协议**的 DLedger 存储层，有了真正类似 HW 的概念：

```
Raft 的 committedIndex（提交索引）：

  Leader 写入日志后，不立即"提交"
  等到多数派（Quorum）副本确认写入 → 才推进 committedIndex
  committedIndex 以内的日志才对 Consumer 可见

  committedIndex  ≈  Kafka HW
  多数派确认      ≈  min.insync.replicas 全员确认
```

```
DLedger 写入流程：

Producer ──► Leader 写入本地日志（uncommitted）
                │
                ├──► Follower-1 确认
                ├──► Follower-2 确认（多数派达成）
                ▼
         committedIndex 推进 → Consumer 现在可以看见这条消息
```

**DLedger 相比 4.x Master/Slave 的改进：**

| 维度 | 4.x Master/Slave | 5.x DLedger（Raft） |
|------|------------------|---------------------|
| 选主 | 需人工/运维介入 | ✅ 自动选主 |
| 可见性边界 | 无明确水位 | ✅ committedIndex 提供明确边界 |
| 数据安全 | SYNC 模式保障 | ✅ 多数派写入保证 |
| 性能 | ASYNC 模式更快 | ❌ 多数派确认有额外开销 |

---

### 整体对比总结

```
Kafka HW 要解决的三个问题，RocketMQ 怎么应对：

┌───────────────────┬────────────────────────┬──────────────────────────┐
│ 问题               │ Kafka 的解法            │ RocketMQ 的解法           │
├───────────────────┼────────────────────────┼──────────────────────────┤
│ 消费可见性边界     │ HW 限制 Consumer 可读   │ SYNC 模式：ack=已复制     │
│                   │ 范围，事后协调          │ 写入时前置解决，无需水位  │
├───────────────────┼────────────────────────┼──────────────────────────┤
│ 副本切换数据安全   │ Consumer 只读 HW 内数据 │ SYNC 模式：切主不丢数据   │
│                   │ 新 Leader 一定有        │ ASYNC 模式：有丢失风险    │
│                   │                        │ DLedger：committedIndex  │
│                   │                        │ 保证多数派持久化          │
├───────────────────┼────────────────────────┼──────────────────────────┤
│ 从副本读的边界     │ Follower 本地 HW        │ Slave 的 maxOffsetInQueue│
│                   │ 有严格协议保证          │ 较粗放，无专门水位协议    │
└───────────────────┴────────────────────────┴──────────────────────────┘
```

### 设计哲学差异一句话概括

```
Kafka：
  写入和可见性解耦，灵活（acks=0/1/all 可调）
  用 HW 在事后划出"安全可见"边界
  代价：需要一套复杂的水位传播协议

RocketMQ 4.x：
  在写入阶段就决定可靠性等级（SYNC / ASYNC）
  用同步模式把"写成功 = 已复制"绑在一起
  代价：灵活性稍差，SYNC 延迟高，ASYNC 无保护

RocketMQ 5.x DLedger：
  走向 Raft，committedIndex ≈ HW
  自动选主 + 多数派确认，向 Kafka 的机制靠拢
  代价：性能开销，架构更复杂
```

---

## 八、RIP-44：DLedger Controller 架构升级

> 原始文档：https://github.com/apache/rocketmq/wiki/RIP-44-Support-DLedger-Controller  
> 状态：已接受（Accepted）｜作者：RongtongJin, ZhangHeng Huang

---

### 原始 DLedger 模式的问题

RocketMQ 4.5 将 Raft 直接嵌入 Broker 存储层，导致三个严重缺陷：

```
原始 DLedger 模式的结构：

┌──────────────────────────────────┐
│           Broker                  │
│  ┌────────────────────────────┐  │
│  │  DLedger CommitLog (Raft)  │  │  ← Raft 和存储完全耦合
│  │  （替换了原生 CommitLog）   │  │
│  └────────────────────────────┘  │
└──────────────────────────────────┘

问题 1：副本数必须 ≥ 3（Raft 多数派要求，2 副本无法使用）
问题 2：写入需多数派 Ack，性能受限（3 副本必须 2 个 Ack）
问题 3：抛弃了 RocketMQ 原生存储能力（zero-copy、transientStorePool 等全部失效）
```

---

### 核心思路：把 Raft 上移到独立 Controller 层

```
RIP-44 之后的结构：

┌────────────────────────────────────────────┐
│       Controller 集群（DLedger Raft）        │  ← Raft 只负责元数据共识
│  维护：谁是 Master、SyncStateSet 是谁        │    不碰消息存储
└───────────────────┬────────────────────────┘
                    │ 指令（选主 / 切换 / 更新成员）
        ┌───────────┼──────────────┐
        ▼           ▼              ▼
┌─────────────┐ ┌─────────┐ ┌─────────┐
│  Broker-A   │ │Broker-B │ │Broker-C │  ← 用回原生 CommitLog + HA 复制
│ (Master)    │ │(Slave)  │ │(Slave)  │    性能、特性全部恢复
└─────────────┘ └─────────┘ └─────────┘
```

**关键思想**：Raft 只做元数据强一致协商（谁是主），消息数据复制仍用原生 Master→Slave HA，两件事彻底解耦。

---

### 核心概念

#### SyncStateSet（类比 Kafka ISR）

```
SyncStateSet = 当前与 Master 保持同步的副本集合（含 Master 本身）

判断"是否同步"：Slave 落后时间是否超过 haMaxTimeSlaveNotCatchUp（默认 15s）

动态变化：
  Slave 落后太多 → 从 SyncStateSet 移除（Shrink）
  Slave 追上来了 → 重新加入 SyncStateSet（Expand）

Master 宕机时：只从 SyncStateSet 内的存活副本中选新 Master
              → 保证新 Master 一定有最新的已提交消息
```

#### Active Controller（元数据的唯一权威）

```
Controller 集群本身用 DLedger（Raft）保证强一致，选出 Active Controller

职责：
  - 维护所有 Broker 副本组的元数据（replicaInfoTable + syncStateSetInfoTable）
  - 接受 Master 上报的 SyncStateSet 变更请求
  - 检测 Master 心跳超时，触发选主
  - 将选主结果通知给 Broker 副本组

部署方式（二选一）：
  A. 嵌入 NameServer（通过开关启用，至少 3 个 NameServer）
  B. 独立部署（与 NameServer 完全解耦）
```

#### Epoch（任期版本号，类比 Raft 的 Term）

```
每次选出新 Master → Epoch +1（由 Controller 分配，全局唯一递增）
每个 Epoch 只有一个 Master

Epoch 文件（epochFile）存储在每个 Broker 本地：
  [(epoch=1, startOffset=0),
   (epoch=2, startOffset=1000),
   (epoch=3, startOffset=1800), ...]

startOffset = 该 Epoch 第一条消息在 CommitLog 中的物理偏移
```

---

### 选主与切换的完整流程

```
Step 1：Controller 检测到 Master 心跳超时

Step 2：从 SyncStateSet 中选一个存活副本作为新 Master
        通过 DLedger Raft 共识写入 ElectMaster 事件
        → Epoch +1，更新 masterEpoch

Step 3：Controller 通知副本组所有 Broker（NOTIFY_BROKER_ROLE_CHANGED）
        新 Master → ChangeToMaster
        其他 Slave → ChangeToSlave

Step 4：Slave 向新 Master 发起连接，进入 HandShake 阶段（日志截断对齐）

Step 5：Slave 追上新 Master → Master 上报 AlterSyncStateSet 扩容
        Controller 共识后更新 SyncStateSet

Step 6：恢复正常复制
```

---

### 日志截断对齐算法（保证切换不丢消息）

主从切换后，新 Slave（原 Master）需要截断与新 Master 不一致的日志：

```
握手阶段，Slave 拿到 Master 的 epochFile，与自己的对比：

从最新的 epoch 向前遍历 Slave 自己的 epochFile：

  for 每个 (epoch, startOffset) in Slave 的 epochFile（从新到旧）:
      masterOffset = Master epochFile 中同 epoch 的 startOffset

      if masterOffset 存在 且 startOffset 相同:
          truncateOffset = min(slave.endOffset[epoch], master.endOffset[epoch])
          break  // 找到对齐基准，截断
      else:
          continue  // 继续找上一个 epoch
```

**示例：**

```
Slave epochFile:  [(1, start=0), (2, start=1000)]
                   Slave 在 epoch=2 写到了 offset=1500
Master epochFile: [(1, start=0), (2, start=1000), (3, start=1201)]
                   Master 在 epoch=2 写到了 offset=1201，然后切换到 epoch=3

比对 epoch=2：startOffset 均为 1000 → 找到基准
truncateOffset = min(1500, 1201) = 1201

Slave 截断到 1201，然后从 Master 同步 epoch=3 的消息
```

**一致性保证**：同一个 `(epoch, startOffset)` 全局唯一，只由一个 Master 写入，因此截断后的日志与新 Master 完全一致。

---

### ConfirmOffset —— 显式的"水位"概念

```
问题：截断前 ConsumeQueue 可能已 dispatch 了截断区间的消息
     → Consumer-A 消费了，Consumer-B 没消费，截断后 B 找不到 → 不一致

解决：引入 ConfirmOffset，ConsumeQueue 只 dispatch 到 ConfirmOffset

ConfirmOffset 计算：
  Master 侧：SyncStateSet 中所有副本 MaxOffset 的最小值
  Slave 侧： min(Master 传输 Header 中的 confirmOffset, 自身最大 offset)

与 Kafka HW 的直接对应：
  Kafka HW          ≈  RocketMQ ConfirmOffset
  ISR               ≈  SyncStateSet
  Follower 本地 HW  ≈  Slave 本地的 confirmOffset
  HW 通过 Fetch 响应传播  ≈  confirmOffset 通过复制消息 Header 传播
```

---

### Async Learner（异步学习者）

```
新角色（配置 isAsyncLearner=true）：
  ✅ 使用 AutoSwitchHA 协议从 Master 复制消息
  ❌ 不在 SyncStateSet 中（Master 不等它 Ack）
  ❌ 不参与选举

适用场景：跨数据中心异步备份（另一个 IDC 的副本，只做灾备）
```

---

### 新旧方案对比

| 维度 | 原 DLedger 模式 | RIP-44 Controller 模式 |
|------|----------------|------------------------|
| Raft 的位置 | 存储层（CommitLog） | 独立 Controller 层 |
| Raft 管理的内容 | 消息数据本身 | 元数据（谁是主 / 成员） |
| 消息数据复制 | 通过 Raft 日志 | 原生 Master→Slave HA |
| 最小副本数 | 3（Raft 多数派） | 2（主从即可） |
| 原生存储特性 | 不可用 | ✅ 全部恢复 |
| 写入 Ack 策略 | 严格多数派 | 可配置（SYNC / ASYNC） |
| 自动选主 | ✅ | ✅（Controller 驱动） |
| 跨 IDC 异步副本 | ❌ | ✅ Async Learner |
| 迁移成本 | — | 主从可直升；原 DLedger 需清空重建 |

---

### 架构全景图

```
┌──────────────────────────────────────────────────┐
│           Controller 集群（DLedger Raft）          │
│  replicaInfoTable + syncStateSetInfoTable         │
│  职责：选主、管理 SyncStateSet、分配 brokerId      │
└───────────────┬──────────────────────────────────┘
                │  心跳 + 指令
     ┌──────────┼──────────────┐
     ▼          ▼              ▼
┌─────────┐ ┌─────────┐ ┌──────────────┐
│Broker-A │ │Broker-B │ │  Broker-C    │
│Master   │─►Slave    │ │Async Learner │
│epoch=3  │ │epoch=3  │ │（不参与选举） │
└─────────┘ └─────────┘ └──────────────┘
     │
     │ ConfirmOffset = SyncStateSet 所有副本 MinOffset
     ▼
ConsumeQueue（只 dispatch 到 ConfirmOffset）
     │
     ▼
Consumer（只能消费安全已提交的消息）
```

---

### 一句话总结

> **RIP-44 做了一件事：把 Raft 从存储层「搬」到协调层。**  
> Raft 只做它最擅长的事——元数据共识（谁是主）；  
> 消息复制回归 RocketMQ 原生 HA——性能和特性全部恢复；  
> ConfirmOffset 作为显式水位，填补了主从模式下"可见性边界"的空白。
