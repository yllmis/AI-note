## 架构设计类

### Q1: 为什么选择微服务架构而不是单体？如何划分服务边界？

**答：** 传统单体 IM 架构存在扩展性瓶颈与单点风险。采用微服务后，按业务域划分为四个服务：

| 服务 | 职责 |
|---|---|
| user（用户服务） | 注册、登录、用户信息查询 |
| social（社交服务） | 好友关系、群组管理 |
| im（即时通讯服务） | 会话管理、消息拉取、WebSocket 网关 |
| task（异步任务服务） | Kafka 消费、消息持久化、已读回执处理 |

**划分原则：** 每个服务只关注自己的领域，user 不知道消息，im 不知道好友关系，通过 gRPC 跨服务调用。这样可以独立部署、独立扩缩容，比如 WebSocket 网关压力大就单独扩容 im-ws。

---

### Q2: 服务间如何通信？为什么选 gRPC？

**答：**

- **对外：** HTTP REST API（user-api、social-api、im-api）通过 APISIX 网关暴露给客户端
- **对内：** gRPC 服务间通信，通过 Etcd 做服务发现（`user.rpc`、`social.rpc`、`im.rpc`）

**为什么选 gRPC：**
1. Protobuf 二进制序列化，比 JSON 体积小、解析快
2. HTTP/2 多路复用，连接效率高
3. 强类型，proto 文件定义接口，减少沟通成本
4. go-zero 的 zrpc 对 gRPC 封装完善，天然支持服务发现、拦截器、负载均衡

---

### Q3: 配置中心用的 Etcd，配置热更新是怎么实现的？

**答：** 通过 Sail + Etcd 实现远程配置管理：

1. Sail 将 YAML 配置存储在 Etcd 中
2. 每个服务启动时通过 `pkg/configserver` 从 Etcd 拉取配置
3. 注册回调监听配置变更，变更时触发服务重启

**热更新机制：** 配置变更后，通过 `proc.Shutdown()` 或 `grpcSvr.GracefulStop()` 优雅停机，然后重新加载配置启动。这是进程级重启，不是原地热替换。

**局限性：** 重启期间有短暂不可用窗口，但对于配置变更频率低的场景可以接受。

---

## 消息系统核心类

### Q4: 消息发送的完整链路是什么？

**答：**

```
客户端 -> WebSocket 网关 (im-ws)
         -> Kafka (msgChatTransfer topic)
         -> task-mq 消费者
            -> 写入 MongoDB (chat_log)
            -> 更新 Conversation 全局计数 ($inc: total+1)
            -> WebSocket push 给在线用户
```

**离线消息：** 用户上线后通过 HTTP API 调用 `GetConversations`，对比 `Conversation.Total`（全局消息数）和 `Conversations.Total`（用户已读位置），计算未读数后拉取。

---

### Q5: 为什么用 Kafka 而不是直接同步写数据库？

**答：**

1. **削峰填谷：** 高并发发送消息时，Kafka 作为缓冲层，消费者按自己的节奏写入 MongoDB，避免数据库被打爆
2. **异步解耦：** WebSocket 网关收到消息后立即写 Kafka 返回，不阻塞发送方，降低响应延迟
3. **消息持久化：** Kafka 本身持久化消息，即使 MongoDB 写入失败，消息也不会丢失，可以重放

---

### Q6: Kafka 消费者是如何工作的？有哪些 topic？

**答：** 两个 topic：

| Topic | 用途 | 生产者 | 消费者 |
|---|---|---|---|
| `msgChatTransfer` | 聊天消息转发 | im-ws | task-mq |
| `msgReadTransfer` | 已读回执处理 | im-ws | task-mq |

**消费者逻辑：**
- `MsgChatTransfer.Consume`：反序列化消息 -> 插入 MongoDB ChatLog -> 更新 Conversation 计数和最后消息 -> WebSocket push
- `MsgReadTransfer.Consume`：更新 ChatLog 的 bitmap -> 更新用户已读位置 -> 推送已读状态

**已知问题：** 没有指定 partition key，同一会话的消息可能分布在不同 partition，导致乱序。修复方案：按 `conversationId` 作为 key。

---

### Q7: 消息同步模型（Timeline + 读扩散）是怎么设计的？

**答：** 采用两级计数的拉模型：

- **Conversation.Total（全局）：** 每个会话的消息总数，每发一条消息通过 MongoDB `$inc` 原子递增
- **Conversations.Total（用户级）：** 每个用户在该会话中的已读位置

**未读计算：** `ToRead = Conversation.Total - Conversations.Total`

**流程：**
1. 新消息到来，Conversation.Total++，存入 ChatLog
2. 用户上线，拉取会话列表，对比两个 Total 得到未读数
3. 用户标记已读，Conversations.Total 更新为 Conversation.Total

**优势：** 不需要维护每条消息的已读状态列表，只维护一个计数器，空间和计算开销都很小。

---

### Q8: 已读回执的 Bitmap 方案是怎么实现的？为什么用 Bitmap？

**答：** 每条 ChatLog 存储一个 `[]byte` 类型的 `readRecords` 字段：

- 使用自定义 Bitmap 实现（`pkg/bitmap/bitmap.go`），通过 BKDRHash 把 userId 映射到 bit 位
- 默认 250 字节 = 2000 bit，支持最多 2000 人
- 单聊：`[1]` 表示对方已读
- 群聊：每个成员独立设置自己的 bit

**为什么用 Bitmap 而不是其他方案：**

| 方案 | 空间 | 问题 |
|---|---|---|
| 存用户列表 | O(n) | 群聊 500 人时每条消息存 500 个 ID |
| 存 count | O(1) | 无法区分谁读了谁没读 |
| Bitmap | O(n/8) | 空间最优，500 人只需 63 字节 |

**发送者默认已读：** 消息插入时自动标记发送者为已读。

---

### Q9: 群聊已读回执的批量处理是怎么做的？

**答：** 群聊场景下，如果每个成员的已读都立即推送，网络开销巨大。采用批量合并机制：

- **计数触发：** 同一会话累计 10 条已读回执后触发一次推送
- **定时触发：** 最长等待 1 秒后强制推送，无论是否达到计数阈值

**实现：** 用状态机（`apps/task/mq/internal/handler/msgTransfer/groupMsgRead.go`）按 conversationId 合并已读更新，到阈值时统一推送。这样 500 人群聊，500 人的已读可以合并成少量推送。

---

## 幂等与限流类

### Q10: 幂等控制是怎么实现的？

**答：** 采用 gRPC 拦截器 + Redis 的方案：

1. **客户端拦截器** (`NewIdempotentClient`)：为每个请求生成 UUID，通过 gRPC metadata 传递
2. **服务端拦截器** (`NewIdempotentServer`)：提取 UUID，用 `Redis.SetnxEx`（TTL 1 小时）检查是否已处理
3. 如果 key 已存在，返回 `DeadlineExceeded` 错误；否则写入 key 并继续处理

**白名单机制：** 只有写操作（好友请求、群组操作）才需要幂等，读操作跳过。

**如果 Redis 挂了：** 可以降级为不去重，保证可用性（CAP 选择 AP）。

---

### Q11: 限流有几层？分别是什么算法？

**答：** 两层限流：

| 层级 | 算法 | 实现 | 配置 |
|---|---|---|---|
| HTTP API 层 | 令牌桶 | `limit.TokenLimiter` + Redis | rate=1, burst=100 |
| gRPC 层 | 信号量 | `syncx.Limit` 内存级 | maxCount=200 |

**令牌桶：** 以固定速率往桶里放令牌，请求需要拿到令牌才能处理，桶满时丢弃新令牌，允许突发（burst）。

**信号量：** 限制同时处理的请求数，超过 200 个并发时直接返回 `codes.Unavailable`。

**区别：** 令牌桶控制的是速率（每秒多少请求），信号量控制的是并发度（同时处理多少请求）。两者配合使用效果更好。

---

### Q12: 令牌桶和漏桶算法的区别？为什么选令牌桶？

**答：**

- **漏桶：** 以固定速率处理请求，流入速率大于流出速率时排队等待。流量平滑但无法应对突发。
- **令牌桶：** 以固定速率生成令牌，请求消耗令牌。桶中有剩余令牌时允许突发处理。

**选令牌桶的原因：** API 场景需要应对突发流量（比如用户同时打开多个聊天窗口），令牌桶的 burst 能力更适合。而漏桶的严格匀速会增加正常用户的延迟。

---

## WebSocket 类

### Q13: WebSocket 服务是怎么管理连接的？

**答：** 自定义实现的 WebSocket 框架（`apps/im/ws/websocket/`），核心设计：

1. **双向映射：** `connToUser`（连接 -> 用户）和 `userToConn`（用户 -> 连接），O(1) 查找
2. **方法路由：** JSON 帧格式，通过 `method` 字段分发（类似 RPC-over-WebSocket）：
   - `user.online`：用户上线
   - `conversation.chat`：发送消息
   - `conversation.markRead`：标记已读
   - `push`：服务端推送
3. **新设备踢人：** 新连接到来时查找并关闭旧连接
4. **心跳检测：** 定时器跟踪空闲时间，超时断开

---

### Q14: 心跳检测是怎么实现的？

**答：** 借鉴 gRPC keepalive 机制（`apps/im/ws/websocket/connection.go`）：

- 每个连接维护一个 `idleTimer`，记录最后活跃时间
- 定时检查空闲时长，超过 `maxConnectionIdle` 阈值则关闭连接
- WebSocket 层也处理 `FramePing` 帧，直接 echo 回去

**生产配置：** `maxConnectionIdle` 设为 `math.MaxInt64`（相当于关闭超时），开发环境 1000s。实际生产中依赖客户端主动发心跳包保活。

---

### Q15: 消息投递的 ACK 机制有几种？为什么设计三种？

**答：** 三种 ACK 级别，平衡性能和可靠性：

| 级别 | 流程 | 可靠性 | 延迟 |
|---|---|---|---|
| NoAck | 发送后不管 | 低 | 最低 |
| OnlyAck | 服务端确认后处理 | 中 | 中 |
| RigorAck | 双向确认 + 重试 | 高 | 最高 |

**RigorAck 重试策略：**
- 线性退避：每次重试间隔增加 200us，最大 1s
- 最多重试 5 次（`sendErrCount`）
- 超过重试次数放弃投递

**设计思路：** 不同业务场景用不同级别。普通消息可以用 OnlyAck，重要消息（如转账通知）用 RigorAck。

---

### Q16: 多设备登录怎么处理的？

**答：**

1. 新 WebSocket 连接建立时，通过 `userToConn` 查找该用户已有的连接
2. 如果存在旧连接，发送关闭信号并清理
3. 更新 `userToConn` 映射为新连接
4. 同时更新 Redis Hash（`online:users`）中的在线状态

**保证：** 同一时刻同一用户只有一个活跃连接。

---

## 并发与性能类

### Q17: 如何保证高并发下消息不丢失？

**答：**

1. **Kafka 持久化：** 消息先写入 Kafka（持久化到磁盘），消费者异步处理
2. **MongoDB 原子操作：** `$inc` 更新计数器，避免并发写覆盖
3. **Redis 原子操作：** `SetnxEx` 保证幂等，防止重复处理
4. **ACK 机制：** 客户端确认收到才算投递成功

**已知风险：** MongoDB 写入成功但 WebSocket push 失败时，消息已持久化但用户未收到，上线后可通过拉取补偿。

---

### Q18: 如果让你优化这个系统，你会怎么做？

**答：**

1. **Kafka 分区策略：** 加 partition key（conversationId），保证同会话消息顺序
2. **推送重试：** WebSocket push 失败后加入重试队列
3. **消息分库分表：** MongoDB 按 conversationId 分片，避免单集合过大
4. **已读 Bitmap 优化：** 超过 2000 人的大群可以用 Redis Bitmap 或分片存储
5. **消费积压监控：** 增加 Kafka consumer lag 监控告警

---

## 工程实践类

### Q19: 为什么用 go-zero 而不是直接用 gRPC/gin？

**答：** go-zero 是一个微服务全家桶框架：

1. **一站式：** 内置 zrpc（gRPC）、rest（HTTP）、缓存、限流、服务发现
2. **代码生成：** goctl 根据 proto/api 文件生成 model、logic、handler，减少重复代码
3. **最佳实践内置：** 拦截器、中间件、优雅停机开箱即用
4. **社区活跃：** 国内用的多，遇到问题容易找到解决方案

**对比：** 直接用 gRPC + gin 需要自己组装服务发现、限流、缓存等组件，开发成本高。

---

### Q20: ID 生成方案为什么选 WUID 而不是 UUID/Snowflake？

**答：**

| 方案 | 特点 | 问题 |
|---|---|---|
| UUID | 全局唯一、无序 | 128 位太长，MySQL 索引不友好 |
| Snowflake | 趋势递增、64 位 | 依赖时钟，时钟回拨会出问题 |
| WUID | 趋势递增、16 位 hex | 依赖 MySQL，但无时钟问题 |

**选 WUID 的原因：**
- 16 位 hex 简短且趋势递增，MySQL 索引友好
- 基于 MySQL 的 H28 算法，不依赖外部组件（如 Zookeeper）
- 会话 ID 通过 `wuid.CombineId`（排序后拼接）生成，保证同一对用户的会话 ID 一致

---

### Q21: CI/CD 流程是怎样的？

**答：**

**CI（持续集成）：** PR 或 push 到 main 时触发
- Lint & Vet：go mod tidy -> go vet -> go build
- Test：go test ./...

**CD（持续部署）：** 打 tag（v*）时触发
1. 矩阵构建 8 个 Docker 镜像
2. 推送到阿里云容器镜像仓库
3. SSH 远程部署：拉取新镜像 -> 停止旧容器 -> 启动新容器
4. 容器运行在 `im-grpc_yllmis-im` Docker 网络中

---

## 高频追问

### Q22: 如何支持万人群聊？

**答：**

- **已读 Bitmap 限制：** 当前 250 字节只支持 2000 人，万人群需要分片或改用 Redis Bitmap
- **消息扇出：** task-mq 需要查群成员列表再逐个推送，万人群的推送开销大
- **优化方向：** 用 Redis Pub/Sub 做消息分发，或引入消息聚合（类似 Timeline 模型）减少推送次数

---

### Q23: 已读状态的最终一致性怎么保证？

**答：** 当前方案是强一致（MongoDB 原子更新 Bitmap 和计数器），但 Bitmap 更新和 WebSocket push 不在同一事务中，可能出现短暂不一致（用户看到已读但推送还没到）。

**可接受的原因：** 已读状态本身就是最终一致性场景，短暂延迟不影响用户体验。如果要求强一致，可以将 Bitmap 更新和推送放在同一个事务中，但会增加延迟。

---

### Q24: 系统的监控和告警怎么做的？

**答：**

- **Prometheus + Grafana：** 指标采集和可视化（QPS、延迟、错误率）
- **ELK（Logstash + Filebeat + Kibana）：** 日志收集和检索
- **关键指标：**
  - API 层 QPS 和延迟
  - Kafka 消费者 lag（积压量）
  - MongoDB 连接数和慢查询
  - WebSocket 在线连接数
  - Redis 内存使用率

---

### Q25: Kafka 消费者积压了怎么办？

**答：**

1. **增加消费者实例：** 扩容 task-mq，同一 consumer group 内多个实例并行消费
2. **优化消费逻辑：** 批量写 MongoDB（减少网络往返）
3. **增加 partition 数：** 更多 partition 支持更多消费者并行
4. **监控告警：** 设置 lag 阈值，提前发现积压趋势
5. **紧急处理：** 临时停止非关键消费，优先处理消息转发

---

### Q26: 你在这个项目中遇到的最大挑战是什么？

**建议回答方向：**

> **已读回执的 Bitmap 设计：** 群聊场景下，如何高效存储"谁读了这条消息"。最初考虑存用户列表，但空间开销大；存 count 无法区分具体谁读了。最终用 Bitmap + BKDRHash，250 字节支持 2000 人，空间效率提升了几十倍。同时设计了批量合并机制，避免频繁推送。

或者：

> **Kafka 消息顺序性问题：** 发现没有指定 partition key，导致同一会话的消息可能乱序。通过分析 go-zero 的 kq 封装，确认需要按 conversationId 作为 key，保证同一会话的消息进入同一 partition。

---

### Q27: 如果要做零停机部署，你会怎么做？

**答：** 当前方案是停止旧容器再启动新容器，有短暂不可用。零停机方案：

1. **滚动更新：** 先启动新实例，健康检查通过后再停止旧实例
2. **gRPC 优雅停机：** 调用 `GracefulStop()` 等待现有请求处理完再关闭
3. **Kafka 消费者：** 等待当前消息消费完再关闭（commit offset）
4. **WebSocket 连接：** 通知客户端重连到新实例（通过负载均衡器实现）

---

### Q28: 为什么消息存 MongoDB 而不是 MySQL？

**答：**

1. **写入性能：** 消息量大、写入频繁，MongoDB 的写入性能优于 MySQL
2. **文档模型：** 消息结构灵活（不同消息类型、扩展字段），MongoDB 的 BSON 更适合
3. **水平扩展：** MongoDB 支持分片，可以按 conversationId 分片存储
4. **读写分离：** MongoDB 的副本集天然支持读写分离，消息拉取可以走从节点
5. **无事务需求：** 消息存储不需要跨文档事务，MongoDB 的优势更明显

---

### Q29: 系统的整体可用性如何保证？

**答：**

| 措施 | 实现 |
|---|---|
| 服务发现 | Etcd，节点故障自动摘除 |
| 负载均衡 | APISIX 网关 + gRPC 内置负载均衡 |
| 限流 | 令牌桶 + 信号量，防止过载 |
| 幂等 | Redis + 拦截器，防止重复处理 |
| 异步解耦 | Kafka 缓冲，避免级联故障 |
| 优雅停机 | proc.Shutdown() + GracefulStop() |
| 监控告警 | Prometheus + Grafana + ELK |

**单点风险：** Redis 和 Kafka 是潜在单点，生产环境应部署集群模式。

---

### Q30: 总结：这个项目的技术亮点

1. **完整的微服务架构：** 4 个业务域、8 个服务，职责清晰
2. **消息可靠投递：** 三级 ACK + Kafka 持久化 + 幂等控制
3. **高效已读回执：** Bitmap 设计，空间效率极高
4. **异步消息处理：** Kafka 削峰 + 消费者批量处理
5. **完善的基础设施：** Etcd 服务发现、APISIX 网关、Prometheus 监控、CI/CD 流水线
6. **自定义 WebSocket 框架：** 方法路由、连接管理、心跳检测、ACK 协议完整实现
