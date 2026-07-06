# RocketMQ 特色功能实现原理

---

## 一、事务消息（半事务消息）

**解决的问题**：本地业务操作与消息发送的原子性（例如：扣库存 + 发订单消息，两件事要么都成功要么都失败）。

### 实现：两阶段提交 + 回查

```
Phase 1：发送半消息（Half Message）
  Producer ──► Broker
              Broker 把消息存到内部 Topic：RMQ_SYS_TRANS_HALF_TOPIC
              消费者此时看不到这条消息
              Broker 返回 ack 给 Producer

Phase 2：执行本地事务
  Producer 执行本地业务（如扣库存、写 DB）
  根据结果决定：COMMIT / ROLLBACK / UNKNOW

Phase 3：二次确认
  COMMIT   → Broker 把消息从 Half Topic 转到真实 Topic → Consumer 可见
  ROLLBACK → Broker 删除 Half Topic 中的消息
  UNKNOW   → 等待 Broker 回查
```

### 回查机制

```
Producer 崩溃 / 网络超时 → Broker 一直没收到二次确认

Broker 的 TransactionCheckService（默认每 60s 扫描一次）：
  扫描 RMQ_SYS_TRANS_HALF_TOPIC 中超时未确认的消息
  向 Producer 发起回查请求：这条消息的本地事务到底成功了吗？
  Producer 查询本地 DB 状态 → 返回 COMMIT / ROLLBACK

默认：最多回查 15 次，超过后强制 ROLLBACK
```

```
时序图：

Producer            Broker              DB
   │                  │                  │
   │── 发半消息 ──────►│                  │
   │◄─ ack ───────────│                  │
   │── 执行本地事务 ───────────────────────►│
   │◄─ 结果 ─────────────────────────────│
   │── COMMIT ────────►│                  │
   │                  │── 转到真实 Topic ──►（Consumer 可见）

（如果 Producer 挂了，Broker 定时回查）
   │◄─ 回查请求 ────────│
   │── 查 DB，返回结果 ─►│
```

---

## 二、延迟消息

**解决的问题**：消息在未来某个时间才被消费（如：下单 30 分钟未支付则取消）。

### 实现：内部中转 Topic + 定时调度

开源版 RocketMQ 支持 **18 个固定延迟等级**：

```
等级：  1    2    3    4    5    6    7    8    9   10   11   12   13   14   15   16   17   18
延迟： 1s   5s  10s  30s   1m   2m   3m   4m   5m   6m   7m   8m   9m  10m  20m  30m   1h   2h
```

```
发送延迟消息的内部流程：

① Producer 发消息，设置 delayLevel=5（即 1 分钟后投递）

② Broker 收到消息，发现有 delayLevel：
   → 把消息的真实 Topic 和 QueueId 存入消息属性
   → 将消息重定向写入：SCHEDULE_TOPIC_XXXX / Queue[4]（等级 5 对应 Queue 4）

③ ScheduleMessageService（Broker 内部定时线程）：
   → 每个延迟等级对应一个独立的定时器
   → 定时扫描 SCHEDULE_TOPIC_XXXX 各队列
   → 发现到期消息 → 恢复真实 Topic/QueueId → 重新投递到真实 Topic

④ Consumer 从真实 Topic 消费，完全无感知延迟机制的存在
```

```
存储结构：

SCHEDULE_TOPIC_XXXX
  ├── Queue 0（对应延迟等级 1，即 1s）
  ├── Queue 1（对应延迟等级 2，即 5s）
  ├── Queue 4（对应延迟等级 5，即 1m）  ← 延迟消息暂存于此
  └── ...

等 1 分钟后：
ScheduleMessageService 扫到到期消息 → 转投到 real_topic/queue
```

> **商业版 / RocketMQ 5.x** 支持任意时间精度的延迟，原理变为时间轮（TimeWheel）。

---

## 三、顺序消息

**解决的问题**：同一业务实体的消息必须按发送顺序消费（如：同一订单的创建→支付→发货）。

### 两种粒度

```
全局顺序（了解即可，几乎不用）：
  Topic 只有 1 个 Queue，Producer 单线程发送
  → 严格全局有序，但吞吐量极低，无容错能力

分区顺序（生产常用）：
  同一业务键（如 orderId）的消息发到同一个 Queue
  该 Queue 的消费者单线程消费
  → 同一业务实体有序，不同实体并行，吞吐量正常
```

### 实现细节

```java
// Producer 端：实现 MessageQueueSelector
producer.send(msg, (queues, msg, arg) -> {
    String orderId = (String) arg;
    int index = Math.abs(orderId.hashCode()) % queues.size();
    return queues.get(index);  // 同 orderId → 同 Queue
}, orderId);

// Consumer 端：注册 MessageListenerOrderly
consumer.registerMessageListener((MessageListenerOrderly) (msgs, context) -> {
    // RocketMQ SDK 对每个 Queue 加锁，保证单线程消费
    // 消费失败时阻塞等待重试，不跳过（保证顺序不破坏）
    return ConsumeOrderlyStatus.SUCCESS;
});
```

```
顺序保证的边界：

orderId=1001 的消息 → 全部在 Queue-2 上顺序排列 → Consumer 单线程消费 ✅
orderId=1002 的消息 → 全部在 Queue-5 上顺序排列 → 另一个 Consumer 并发消费 ✅
1001 和 1002 之间无顺序保证（也不需要）
```

---

## 四、消息重试与死信队列

**解决的问题**：消费失败时自动补偿，彻底失败后隔离处理。

### 重试机制

```
Consumer 消费失败（抛异常 / 返回 RECONSUME_LATER）：

  消息被发回 Broker，存入重试 Topic：
  %RETRY%<ConsumerGroup>   （每个消费者组独立）

  重试间隔按延迟等级递增（借用延迟消息机制）：
  第  1 次重试：10s 后
  第  2 次重试：30s 后
  第  3 次重试：1m 后
  ...
  第 16 次重试：2h 后

  重试 Topic 对业务代码透明：
  Consumer 注册的 Listener 也会收到重试消息，无需特殊处理
```

### 死信队列（DLQ）

```
超过最大重试次数（默认 16 次）→ 消息转入死信 Topic：
%DLQ%<ConsumerGroup>

特点：
  - 死信 Topic 默认不消费，需要人工订阅处理
  - 消息保留时间与普通消息一致（默认 72h）
  - 通常配合告警系统：DLQ 有消息 → 触发报警 → 人工排查
  - 可通过 mqadmin 或 Dashboard 手动重发 DLQ 消息

完整生命周期：

发送 → 消费失败 → 进 %RETRY% → 重试 16 次 → 进 %DLQ% → 人工处理
                    ↑_________重试间隔递增_________|
```

---

## 五、广播消费

**解决的问题**：消息需要被所有消费者实例各消费一次（如：配置变更通知所有服务实例刷新缓存）。

### 集群消费 vs 广播消费

```
集群消费（CLUSTERING，默认）：
  同一 ConsumerGroup 内，消息只被其中一个实例消费
  ConsumerOffset 存在 Broker 端，组内共享
  → 横向扩展消费能力

广播消费（BROADCASTING）：
  同一 ConsumerGroup 内，每个实例都收到并消费全部消息
  ConsumerOffset 存在 Consumer 本地（~/.rocketmq_offsets）
  → 广播通知，类似 pub-sub
```

### 实现关键

```
广播模式的本质：
  每个 Consumer 实例有自己独立的 offset，互不影响
  Rebalance 时：每个实例认领所有 Queue（而不是分配部分 Queue）

存储位置变化：
  集群模式：offset 在 Broker 的 consumerOffset.json 中集中管理
  广播模式：offset 在 Consumer 本地文件中各自管理

注意：
  广播模式下 Consumer 重启，从本地 offset 续读
  如果本地文件丢失，从最新位置开始消费（可能漏消费）
```

---

## 六、消息过滤

**解决的问题**：一个 Topic 下有多种消息类型，Consumer 只消费自己关心的。

### Tag 过滤（最常用）

```java
// Producer 发送时打标签
msg.setTags("PaySuccess");

// Consumer 订阅时指定 Tag
consumer.subscribe("OrderTopic", "PaySuccess || Refund");
```

```
过滤发生在 Broker 端：
  ConsumeQueue 每条记录存有 tagsCode（Tag 的 hashCode）
  Broker 先用 hashCode 过滤（粗过滤）
  Consumer 收到后再用字符串精确比对（细过滤，防 hash 冲突）

好处：大量不匹配消息在 Broker 就被拦截，不占网络带宽
```

### SQL92 过滤

```java
// Producer 设置自定义属性
msg.putUserProperty("region", "Beijing");
msg.putUserProperty("price", "100");

// Consumer 用 SQL 表达式订阅
consumer.subscribe("OrderTopic",
    MessageSelector.bySql("region = 'Beijing' AND price > 50"));
```

```
过滤在 Broker 端执行 SQL 表达式：
  需要 Broker 开启 enablePropertyFilter=true
  相比 Tag 过滤，CPU 开销更大，但表达能力更强
```

---

## 七、消息轨迹（Message Trace）

**解决的问题**：生产环境排查"消息去哪了"。

```
开启后，RocketMQ SDK 自动采集每条消息的轨迹数据：
  - Producer 发送时间、Broker 地址、发送耗时
  - Broker 存储时间戳
  - Consumer 拉取时间、消费耗时、消费结果

轨迹数据发到内部 Topic：RMQ_SYS_TRACE_TOPIC（异步发送，不影响主链路性能）

查询：通过 RocketMQ Dashboard 输入 MessageId 即可看到完整轨迹

生产价值：
  "这条消息发出去了吗？" → 看 Producer 轨迹
  "Broker 收到了吗？"    → 看 Store 轨迹
  "消费者消费了吗？"     → 看 Consumer 轨迹
```

---

## 八、请求-回复模式（RPC over MQ）

**解决的问题**：基于消息队列做同步 RPC，解耦调用方和被调用方，同时保留请求-响应语义。（RocketMQ 4.6+ 引入）

```java
// Producer（请求方）：发送请求并阻塞等待回复
Message req = new Message("RequestTopic", body);
req.putUserProperty("orderId", orderId);
// SDK 自动注入 correlationId 和 replyTo
Message reply = producer.request(req, 3000);  // 超时 3s

// Consumer（处理方）：处理后发回响应
consumer.registerMessageListener((MessageListenerConcurrently) (msgs, ctx) -> {
    Message replyMsg = MessageUtil.createReplyMessage(msgs.get(0), replyBody);
    producer.send(replyMsg);
    return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
});
```

```
底层实现：
  请求方在本地用 correlationId 建一个 Future，阻塞等待
  回复消息发到 ReplyTopic，请求方 Consumer 接收后
  通过 correlationId 找到对应 Future，唤醒等待线程

适用场景：
  跨服务异步解耦，但某些场景需要等待结果
  例：发短信服务，既要解耦又要知道是否发送成功
```

---

## 九、事务消息的业务适配指南

### SDK 接口要求

RocketMQ SDK 要求业务实现 `TransactionListener`，只有两个方法：

```java
public interface TransactionListener {

    /**
     * 方法 1：执行本地事务
     * 半消息发送成功后，SDK 自动回调此方法
     */
    LocalTransactionState executeLocalTransaction(Message msg, Object arg);

    /**
     * 方法 2：回查本地事务状态
     * Broker 长时间未收到二次确认时主动回查
     * 判断逻辑完全由业务自己决定，SDK 不维护任何数据库表
     */
    LocalTransactionState checkLocalTransaction(MessageExt msg);
}

enum LocalTransactionState {
    COMMIT_MESSAGE,    // 提交，Consumer 可见
    ROLLBACK_MESSAGE,  // 回滚，消息作废
    UNKNOW,            // 不确定，等 Broker 下次再来回查
}
```

### 完整业务适配示例

```java
// 发送端
TransactionMQProducer producer = new TransactionMQProducer("order-producer-group");
producer.setTransactionListener(new OrderTransactionListener());
producer.start();

// 发半消息（Consumer 此时看不到）
Message msg = new Message("order-topic", JSON.toBytes(orderDTO));
msg.putUserProperty("orderId", orderDTO.getOrderId());  // 回查时需要
producer.sendMessageInTransaction(msg, orderDTO);        // arg 透传给 Listener
```

```java
public class OrderTransactionListener implements TransactionListener {

    @Autowired
    private OrderService orderService;
    @Autowired
    private OrderRepository orderRepository;

    /** 半消息发送成功后执行：做真正的本地事务 */
    @Override
    public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        OrderDTO orderDTO = (OrderDTO) arg;
        try {
            orderService.createOrder(orderDTO);  // 扣库存、写订单记录
            return LocalTransactionState.COMMIT_MESSAGE;
        } catch (Exception e) {
            return LocalTransactionState.ROLLBACK_MESSAGE;
        }
    }

    /** Broker 回查时执行：查询本地事务是否成功（判断逻辑业务自己实现） */
    @Override
    public LocalTransactionState checkLocalTransaction(MessageExt msg) {
        String orderId = msg.getUserProperty("orderId");
        Order order = orderRepository.findById(orderId);

        if (order != null && order.getStatus() == OrderStatus.CREATED) {
            return LocalTransactionState.COMMIT_MESSAGE;    // 订单存在，提交
        } else if (order == null) {
            return LocalTransactionState.ROLLBACK_MESSAGE;  // 订单不存在，回滚
        } else {
            return LocalTransactionState.UNKNOW;            // 状态不明，等待
        }
    }
}
```

### 回查判断方案对比

#### 方案 A：直接查业务表（最常见）

```
前提：本地事务成功 = 业务表里有那条记录

回查逻辑：
  查订单表，orderId 存在 → COMMIT
  查订单表，orderId 不存在 → ROLLBACK

优点：无额外开销，复用现有数据
缺点：需确保"记录存在"与"事务提交"在同一 DB 事务内
```

#### 方案 B：维护专门的事务状态表（更严谨）

```sql
-- 业务自己建表，SDK 完全不参与
CREATE TABLE local_transaction_log (
    msg_id      VARCHAR(64) PRIMARY KEY,
    biz_id      VARCHAR(64),
    status      TINYINT,      -- 0: 进行中, 1: 成功, 2: 失败
    create_time DATETIME,
    update_time DATETIME
);
```

```
executeLocalTransaction 中：
  1. 先插入 status=0 的记录
  2. 执行业务逻辑
  3. 更新 status=1（在同一个 DB 事务里）

checkLocalTransaction 中：
  查 local_transaction_log 的 status，返回对应状态

优点：状态清晰，回查逻辑简单，与业务表解耦
缺点：多一张表，多一次写入
适合：业务表判断不够明确，或事务逻辑复杂的场景
```

#### 方案 C：调用内部接口（微服务场景）

```
当 Producer 和本地事务不在同一个服务时：

checkLocalTransaction 中调用内部 HTTP/RPC 接口查询状态
接口返回业务自定义的状态 → 转换为 COMMIT / ROLLBACK / UNKNOW

缺点：多一次网络调用，需处理超时等异常情况
```

### 完整流程（含业务视角）

```
                    业务代码视角                      RocketMQ 内部
                         │
① producer.sendMessageInTransaction()
                         │──── 发半消息 ─────────────► Broker 暂存到 HALF_TOPIC
                         │◄─── ack ──────────────────
                         │
② executeLocalTransaction() 被 SDK 回调
   └─ 执行 DB 事务（扣库存、写订单）
   └─ 返回 COMMIT
                         │──── COMMIT ───────────────► Broker 转发到真实 Topic
                                                        Consumer 可以消费了 ✅

（异常场景：Producer 崩溃，Broker 没收到确认）
                                    Broker（60s 后）──► checkLocalTransaction() 回调
                                                        └─ 查订单表
                                                        └─ 返回 COMMIT / ROLLBACK
```

### 业务适配注意事项

```
1. executeLocalTransaction 和 checkLocalTransaction 必须幂等
   回查可能被调用多次，判断逻辑要稳定

2. checkLocalTransaction 不要做耗时操作
   建议纯 DB 查询，不要调用外部服务

3. UNKNOW 要谨慎使用
   只在事务真的"进行中"时返回
   一直返回 UNKNOW → 超过最大回查次数（默认 15 次）→ 强制 ROLLBACK

4. 消息属性里存的 key 要足够精确
   回查时只有消息本身，确保能唯一定位到本地事务记录

5. 本地 DB 事务和消息发送的顺序
   ✅ 正确：先发半消息 → 在 executeLocalTransaction 里执行本地 DB 事务
   ❌ 错误：先执行 DB 事务 → 再发消息（失败时无法回滚 DB）
```

### 职责边界总结

```
SDK 负责：
  ✅ 发半消息、等 ack
  ✅ 回调 executeLocalTransaction
  ✅ 触发 Broker 回查时，回调 checkLocalTransaction
  ✅ 按返回值执行 COMMIT / ROLLBACK

业务负责：
  ✅ 在 executeLocalTransaction 里写本地事务逻辑
  ✅ 在 checkLocalTransaction 里实现状态查询逻辑（查自己的 DB）
  ✅ 保证消息属性里有足够信息供回查使用
  ✅ 建不建"事务状态表"完全自己决定，SDK 不干涉
```

---

## 十、功能速查与选型对照

| 功能 | 核心原理 | 典型场景 |
|------|---------|---------|
| **事务消息** | 半消息 + 二阶段提交 + 回查 | 下单扣库存、转账 |
| **延迟消息** | 中转到 SCHEDULE_TOPIC + 定时调度 | 订单超时取消、定时提醒 |
| **顺序消息** | 同 key 路由同 Queue + 单线程消费锁 | 订单状态流转、Binlog 同步 |
| **消息重试** | %RETRY% Topic + 延迟等级递增 | 消费失败自动补偿 |
| **死信队列** | 超限后转入 %DLQ% Topic | 异常消息隔离 + 人工处理 |
| **广播消费** | 每实例独立 offset，认领全量 Queue | 配置刷新、本地缓存更新 |
| **Tag 过滤** | Broker 端 hashCode 粗过滤 | 同 Topic 多业务类型分流 |
| **SQL92 过滤** | Broker 端 SQL 表达式 | 多属性复合条件订阅 |
| **消息轨迹** | 异步采集到 Trace Topic | 生产排查消息丢失 / 延迟 |
| **请求-回复** | correlationId + Future 阻塞 | 异步解耦但需等待结果 |
