# Redis listpack

---

## 一、ziplist 用在哪些上层数据结构中

ziplist（压缩列表）是 Redis 的一种紧凑型内存编码，在**元素数量少且元素体积小**时作为底层实现使用。

| 上层数据结构 | 触发条件（使用 ziplist） |
|---|---|
| **Hash（哈希）** | 键值对数量 ≤ `hash-max-ziplist-entries`（默认128）且所有键值长度 ≤ `hash-max-ziplist-value`（默认64字节）|
| **List（列表）** | 元素数量 ≤ `list-max-ziplist-size`（Redis 3.2 之前）|
| **Zset（有序集合）** | 元素数量 ≤ `zset-max-ziplist-entries`（默认128）且所有元素长度 ≤ `zset-max-ziplist-value`（默认64字节）|

> **注意**：List 在 Redis **3.2** 之后改用 `quicklist`（由多个 ziplist 节点组成的双向链表）替代，不再直接使用单独的 ziplist。
>
> Redis **7.0** 引入 `listpack` 作为 ziplist 的替代，Hash 和 Zset 的底层编码逐步从 ziplist 迁移到 listpack。

---

## 二、ziplist 的连锁更新问题

### ziplist entry 结构

每个 entry 包含三个字段：

```
+-------------------+----------+---------+
| previous_entry_len|  encoding|  content|
+-------------------+----------+---------+
```

`previous_entry_len` 存储的是**前一个节点的长度**：

| 前一节点长度 | `previous_entry_len` 占用字节数 |
|---|---|
| < 254 字节 | **1 字节** |
| ≥ 254 字节 | **5 字节** |

### 连锁更新的触发过程

假设有一串节点，每个节点大小都是 **253 字节**（卡在临界值以下）：

```
[e1: 253字节] → [e2: 253字节] → [e3: 253字节] → ...
                  prev_len=1字节  prev_len=1字节
```

此时在 e1 之前插入一个大于 254 字节的新节点 e0：

```
e0插入（300字节）
  → e1.prev_len 需要从 1B 扩展到 5B，e1 体积 253 → 257B（≥254）
    → e2.prev_len 需要从 1B 扩展到 5B，e2 体积 253 → 257B（≥254）
      → e3.prev_len 需要从 1B 扩展到 5B ...
        → 💥 每个节点都要 realloc + memmove，一路扩展到尾部
```

**一次插入，引发 N 次内存重分配，最坏 O(N²) 复杂度。**

> 类比：就像一排紧挨着的多米诺骨牌，推倒第一块，后面全部连锁倒下。

---

## 三、listpack 的数据结构

### 整体布局

```
+-------------+--------------+--------+--------+-----+------+
| total_bytes | num_elements | entry1 | entry2 | ... | 0xFF |
+-------------+--------------+--------+--------+-----+------+
    4字节           2字节                              结束符
```

| 字段 | 大小 | 含义 |
|---|---|---|
| `total_bytes` | 4 字节 | 整个 listpack 占用的总字节数 |
| `num_elements` | 2 字节 | 元素个数（超出65535需遍历统计）|
| `entries` | 变长 | 实际数据节点 |
| `0xFF` | 1 字节 | 结束标志 |

### Entry 结构

```
+----------+----------+--------------+
| encoding | content  | element-len  |
+----------+----------+--------------+
  变长        变长        变长（1~5字节）
                          ↑
                   记录【当前 entry 的总长度】
                   用于反向遍历
```

### encoding 字段：同时编码类型和长度/值

encoding 的高位前缀决定数据类型，剩余位直接编码长度或值：

```
前缀 0xxxxxxx  → 7位无符号整数，低7位直接就是值（0~127，content为空！）
前缀 10xxxxxx  → 字符串，低6位是 content 的字节长度（0~63字节）
前缀 110xxxxx  → 13位有符号整数，低5位 + 下一字节
前缀 11110001  → 16位有符号整数，后跟2字节 content
前缀 11110010  → 24位有符号整数，后跟3字节 content
前缀 11110011  → 32位有符号整数，后跟4字节 content
前缀 11110100  → 64位有符号整数，后跟8字节 content
```

> 对于 0~127 的整数，只需 1字节 encoding + 1字节 element-len，共 **2 字节**，极度紧凑。

### element-len 的变长编码

每个字节最高位是标志位，**从左到右读**：

```
最高位 = 1 → 还有后续字节，继续读
最高位 = 0 → 这是最后一个字节，停止
```

例如长度 200，需要 2 字节：

```
10001000  00000001
↑还有后续   ↑最后字节

拼合（去掉标志位）：0000001 | 0001000 = 200 ✓
```

---

## 四、正向遍历

读完 encoding 之后，**不能直接跳到 entry 末尾**，需要三段逐步走过：

```
第1步：读 encoding
       → 得知数据类型 + content 长度（或整数值）
       → encoding 本身占 1~2 字节（取决于类型）

第2步：跳过 content
       → 已知长度，直接跳过（不需要逐字节扫描）

第3步：扫描 element-len
       → 逐字节检查最高位，直到 MSB=0
       → 跳过这几个字节

→ 到达下一个 entry 的起点
```

### 具体示例

存储 `["hi", 42]`，内存布局：

```
偏移:  0     1     2     3     4     5
      [0x82]['h']['i'][0x04][0x2A][0x02]...
      |<---entry1(4字节)--->|<-e2->|
```

- `0x82` = `10000010`：高两位=10（字符串），低六位=2（content长度=2）
- `0x04`：element-len，本 entry 共4字节（1+2+1）
- `0x2A` = `00101010`：高位=0（7位整数），值=42，无 content
- `0x02`：element-len，本 entry 共2字节（1+0+1）

正向遍历 entry1：

```
ptr=0：读 0x82 → 字符串，content=2字节，ptr=1
ptr=1：跳过2字节 content，ptr=3
ptr=3：读 0x04，MSB=0，element-len 结束，ptr=4

→ 到达 entry2 起点 ✓
```

---

## 五、反向遍历

反向遍历利用 element-len 存储的**当前 entry 总长度**往回跳：

```
当前站在某个 entry 的起点（ptr=N）
  → 读 ptr-1 位置的字节
  → 如果 MSB=0，这是 element-len 的最后一个字节（最靠近当前位置）
  → 如果 MSB=1，继续往左读，直到 MSB=0
  → 拼合得到上一个 entry 的总长度 L
  → 新 ptr = N - L → 上一个 entry 的起点
```

### 具体示例

从 entry2（ptr=4）回到 entry1：

```
偏移:  0     1     2     3     4
      [0x82]['h']['i'][0x04][0x2A]...
                        ↑     ↑
                      偏移3   ptr=4

读 ptr-1 = 偏移3 的字节 = 0x04 = 00000100
MSB=0 → 只有这1个字节，element-len = 4
新 ptr = 4 - 4 = 0 → entry1 的起点 ✓
```

### 对比 ziplist 的反向遍历

```
ziplist:  直接读 prev_entry_len（存前一节点长度）→ 简单，但修改邻居会连锁更新
listpack: 读 element-len（存自身长度）→ 略复杂，但完全独立，无连锁更新
```

---

## 六、底层实现：连续字节数组 + 精确 realloc

### 底层就是一块连续内存

```c
unsigned char *lp = zmalloc(size);  // 本质是 unsigned char 数组
```

初始分配仅 **7 字节**：6字节头部 + 1字节结束符，不预留任何额外空间。

### 每次写入都触发 realloc

```c
// 插入元素时（伪代码）
size_t add = encoding_size + content_size + backlen_size;
lp = zrealloc(lp, current_size + add);  // 精确按新大小分配
memmove(dst, src, bytes_to_move);       // 腾出空间
// 写入新 entry
```

**每次插入/删除都会 realloc，可能引发内存拷贝。**

### 为什么不像 vector 一样预留空间？

```
vector:   [  占用  |    预留空间    ]  ← 容量翻倍，均摊 realloc 成本
listpack: [  占用  ]                 ← 精确分配，零内存浪费
```

listpack 的首要目标是**内存紧凑**，不是高频写入性能：

- 只在元素数量少时启用 → 每次 memmove 移动的字节本身就少，拷贝成本可接受
- 预留空间反而浪费内存，违背设计初衷

### 对比 Redis SDS

| | listpack | SDS |
|---|---|---|
| 分配策略 | 精确 realloc | 预分配额外空间 |
| 内存占用 | 极小 | 略有浪费 |
| 写入性能 | 每次可能拷贝 | 均摊 O(1) |
| 适用场景 | 小数据量紧凑存储 | 频繁追加的字符串 |

---

## 七、总体对比

| 特性 | ziplist | listpack |
|---|---|---|
| 节点记录 | 前一节点长度 | 当前节点长度 |
| 连锁更新 | ✅ 存在，最坏 O(N²) | ❌ 不存在 |
| 正向遍历 | encoding → 直接算出 entry 大小 | encoding → content → element-len 三段走 |
| 反向遍历 | 直接用 prev_len | 解析 element-len 推算 |
| 底层存储 | 连续字节数组，精确 realloc | 连续字节数组，精确 realloc |
| 引入版本 | 早期 Redis | Redis 7.0 |
| 小整数优化 | 有 | 有（0~127 仅2字节，更极致）|
