# Kafka 学习笔记

## 1. Kafka 是什么？

Kafka 是一个分布式事件流平台，也常作为高吞吐量的消息系统使用。主要用于：

-   **异步解耦**：生产者发布消息，不需要直接调用或等待所有下游服务。
-   **流量削峰**：利用消息缓冲，将瞬时流量转换为消费者可以逐步处理的积压。
-   **消息持久化与回放**：消息按配置保留，消费者通过 Offset
    管理消费位置，可重新消费历史数据。
-   **实时数据流处理**：用于日志采集、用户行为分析、监控、数据同步和实时计算。

典型架构：

``` text
Producer
   |
   v
Kafka Cluster
   |
   +--> Consumer A：执行业务
   +--> Consumer B：记录日志
   +--> Consumer C：统计指标
```

Kafka
不会凭空提高下游处理能力；流量削峰是将瞬时压力转化为排队等待，需要监控积压量和消费延迟。

## 2. Topic、Partition、Broker 的关系

  概念             作用               理解方式
  ---------------- ------------------ -------------------------------
  Topic            消息的逻辑分类     消息主题
  Partition        Topic 的数据分片   独立的有序消息日志
  Broker           Kafka 服务节点     存储和管理 Partition 的服务器
  Producer         消息生产者         发送消息
  Consumer         消息消费者         读取和处理消息
  Consumer Group   消费者组           组内消费者分担 Partition
  Offset           消息偏移量         某个 Partition 内的消息位置
  Replication      副本机制           为 Partition 保存多个副本

一个 Kafka 集群由多个 Broker 组成；一个 Topic 可以拆分成多个
Partition；一个 Broker 可以存储多个 Topic 的多个 Partition。

示例：一个集群有 3 个 Broker，Topic `agent-task` 有 6 个
Partition，暂不考虑副本：

``` text
Kafka Cluster
|
+-- Broker 1
|     +-- Partition 0
|     +-- Partition 3
|
+-- Broker 2
|     +-- Partition 1
|     +-- Partition 4
|
+-- Broker 3
      +-- Partition 2
      +-- Partition 5
```

以上是示意分配，实际分配由 Kafka 的分区分配机制决定。

**Topic 是逻辑概念，Partition 是实际存储消息的基本单位。** 每个
Partition 都有自己的消息日志和 Offset：

``` text
Partition 0:
Offset 0 -> task_001
Offset 1 -> task_002
Offset 2 -> task_003

Partition 1:
Offset 0 -> task_004
Offset 1 -> task_005
Offset 2 -> task_006
```

Offset 是 Partition 内部的位置，不是整个 Topic 的全局消息编号。

### 2.1 Broker 的职责

1.  接收 Producer 发送的消息。
2.  将消息写入自己管理的 Partition。
3.  向 Consumer 提供消息读取服务。
4.  管理 Partition 副本。
5.  参与集群协调和故障恢复。

Producer 通常向目标 Partition 的 Leader 所在 Broker 发送消息。

### 2.2 Partition 与副本

多个 Partition 分布在不同 Broker 上，可以提高读写并行度。Partition
可以设置多个副本：Leader 负责处理读写，Follower 跟随复制。Leader 所在
Broker 故障时，Kafka 可以从符合条件的同步副本中选举新的 Leader。

## 3. Consumer Group 与 Partition

假设 Topic 有 3 个 Partition：

``` text
Consumer Group A:
Consumer A1 --> Partition 0
Consumer A2 --> Partition 1
Consumer A3 --> Partition 2
```

不同 Consumer Group 可以独立消费同一个 Topic：

-   Group A：业务任务执行
-   Group B：运行日志记录
-   Group C：执行情况统计

关键规则：

1.  同一个 Consumer Group 中，一条消息由其中一个消费者处理。
2.  不同 Consumer Group 可以分别消费同一条消息。
3.  同一个 Partition 在同一个 Consumer Group
    中，同一时刻只分配给一个消费者。
4.  一个组的并行消费数量受 Partition 数量限制。3 个 Partition，最多 3
    个消费者同时获得 Partition 分配；多余消费者会空闲。

## 4. Producer 如何选择 Partition？

假设 Topic `agent-task` 有 3 个 Partition。Producer
发送消息时，需要确定消息写入哪个 Partition。常见方式有三种。

### 4.1 指定 Partition

``` python
producer.produce(
    topic="agent-task",
    partition=1,
    value='{"task_id":"task_1001"}'
)
```

消息会发送到 Partition 1。优点是路由可控；缺点是业务代码需要了解
Partition 分布，可能导致负载不均衡。

### 4.2 根据 Message Key 选择 Partition

``` python
producer.produce(
    topic="agent-task",
    key="task_1001",
    value='{"task_id":"task_1001"}'
)
```

Producer 的分区器会根据 Key 计算目标 Partition。概念上可以理解为：

``` text
Partition = Hash(Key) % Partition数量
```

示意（Hash 数值仅用于说明，不代表 Kafka 实际算法结果）：

  Message Key     假设 Hash 结果   Partition 数量   目标 Partition
  ------------- ---------------- ---------------- ----------------
  task_1001                  100                3                1
  task_1002                  101                3                2
  task_1003                  102                3                0
  task_1004                  103                3                1

**相同的 Key，在分区数量和分区器配置不变的情况下，会被路由到同一个
Partition。**

例如订单 1001 有三个事件：

``` text
OrderCreated
OrderPaid
OrderShipped
```

使用订单 ID 作为 Key：

``` python
key = "order_1001"
```

这些消息会进入同一个 Partition：

``` text
Partition 1
Offset 0 -> OrderCreated
Offset 1 -> OrderPaid
Offset 2 -> OrderShipped
```

Kafka 保证同一个 Partition 内的消息顺序。因此可以利用业务
Key，将同一业务对象的事件放在同一个 Partition
中。注意：这只是分区内顺序保证，不代表跨 Partition
有序，也不自动保证业务处理完成顺序。

### 4.3 不指定 Key，也不指定 Partition

``` python
producer.produce(
    topic="agent-task",
    value='{"task_id":"task_1001"}'
)
```

此时 Producer 客户端的分区器自行选择 Partition。

现代 Kafka Java Producer 默认分区策略中：

-   有 Key：根据 Key 的哈希值选择 Partition。
-   没有 Key：使用 Sticky Partitioning（粘性分区策略）。

Sticky Partitioning 的思路是：在一段时间内持续向同一个 Partition
发送消息，之后再切换到其他
Partition。这样可以让消息积累成批次发送，减少网络请求，提高吞吐量。

**注意：不同 Kafka 客户端的默认分区器实现可能不同。** Python 的
`confluent-kafka` 底层使用
librdkafka，具体策略需要结合客户端配置确认，不能直接认为所有客户端都使用
Java Producer 的默认实现。

### 4.4 三种分区方式对比

  方式             分区依据         适用场景
  ---------------- ---------------- ------------------------
  指定 Partition   人工指定         特殊路由需求
  指定 Key         Hash(Key)        需要业务对象分区内顺序
  不指定 Key       客户端分区策略   普通事件、高吞吐场景

### 4.5 增加 Partition 对 Key 路由的影响

假设 Topic 原来有 3 个 Partition：

``` text
Hash("order_1001") % 3 = 1
```

扩展到 4 个 Partition 后：

``` text
Hash("order_1001") % 4 = 0
```

同一个 Key 可能被路由到不同
Partition。这意味着扩容前后的消息可能分散在不同 Partition，两个
Partition 之间没有顺序保证。

如果业务非常依赖 Key 的消息顺序，不要随意增加 Partition 数量。某个 Key
的消息量特别大，也可能造成热点 Partition 和负载不均衡。

## 5. Kafka 使用时的注意事项

### 5.1 消息顺序

Kafka 只保证同一个 Partition 内的消息有序，不保证多个 Partition
之间有序。需要业务顺序时，使用业务 ID 作为 Message
Key，让同一业务对象的消息进入同一个 Partition。

### 5.2 消息重复消费与幂等

不要默认认为消息只会被消费一次。典型场景：

``` text
1. Consumer 获取消息
2. 执行业务逻辑
3. 数据库写入成功
4. Consumer 还没提交 Offset 就宕机
5. 重启后重新消费这条消息
```

这样会出现重复消费。解决思路是业务幂等，例如：

``` sql
UPDATE orders
SET status = 'PAID'
WHERE order_id = '1001'
  AND status = 'CREATED';
```

也可以使用消息 ID 建立唯一约束，避免同一消息重复产生业务副作用。

### 5.3 Offset 提交时机

  方式       特点
  ---------- -----------------------------------------
  自动提交   简单，但可能出现业务未完成就提交 Offset
  手动提交   由业务控制提交时机，更容易保障可靠性

需要可靠业务处理时，可以采用手动提交：

``` text
获取消息 -> 处理业务 -> 业务成功 -> 提交 Offset
```

如果处理失败，不要提前提交
Offset。手动提交不能自动解决所有重复消费问题，仍需要幂等设计。

### 5.4 消费失败、重试与 DLQ

``` text
消费消息
   |
   v
执行业务
   |
   +---- 成功 ----> 提交 Offset
   |
   +---- 失败 ----> 重试
                    |
                    +---- 成功 --> 提交 Offset
                    |
                    +---- 多次失败 --> DLQ
```

DLQ（Dead Letter
Queue，死信队列）用于存放多次处理仍然失败的消息，供后续排查和人工处理。不能无限快速重试，否则可能造成消费积压和下游系统压力。

### 5.5 Partition 数量设计

    Partition 数量   同组消费者数量   最大分区并行度
  ---------------- ---------------- ----------------
                 1                1                1
                 3                3                3
                 6                3                3
                 6                6                6
                 6               10                6

Partition 不是越多越好。过多会增加 Broker
元数据管理压力、文件句柄和资源消耗，以及副本复制和故障恢复成本。需要结合吞吐量、消息大小、消费者数量和集群规模设计。

### 5.6 消息保留和磁盘容量

消息被消费后，不代表消息立即删除。Kafka 根据 Topic 的保留策略清理消息。

-   `retention.ms`：按时间保留。
-   `retention.bytes`：按分区数据大小限制。
-   `cleanup.policy=delete`：按照保留策略删除。
-   `cleanup.policy=compact`：按照 Key 保留最新状态记录。

需要关注磁盘容量和数据保留时间。

### 5.7 消费积压监控

Consumer Lag 表示最新消息位置与消费者消费位置之间的差距。

``` text
最新消息 Offset：100,000
消费者 Offset：   85,000
积压消息：       15,000
```

如果积压持续增长，说明生产速度超过消费处理能力，或者消费者发生故障。

建议监控：

-   Consumer Lag
-   消息生产速率
-   消息消费速率
-   消费失败率
-   消费处理耗时
-   Broker 磁盘使用率
-   副本同步状态

## 6. Python 使用示例：消息收发

使用 `confluent-kafka` Python 客户端。

### 6.1 启动 Kafka

本地学习可以使用 Docker Compose：

``` yaml
services:
  kafka:
    image: apache/kafka:latest
    ports:
      - "9092:9092"
```

启动：

``` bash
docker compose up -d
```

这是本地学习用的单节点示例，不是生产集群配置。实际部署应固定镜像版本，并配置持久化、监听地址及安全策略。

### 6.2 安装 Python 客户端

``` bash
pip install confluent-kafka
```

### 6.3 Producer：生产消息

创建 `producer.py`：

``` python
import json
from confluent_kafka import Producer

producer = Producer({
    "bootstrap.servers": "localhost:9092"
})

def delivery_report(err, msg):
    if err:
        print("发送失败:", err)
    else:
        print(
            f"发送成功: topic={msg.topic()}, "
            f"partition={msg.partition()}, "
            f"offset={msg.offset()}"
        )

task = {
    "task_id": "task_1001",
    "type": "nl2sql",
    "question": "查询今天的活跃告警数量"
}

producer.produce(
    topic="agent-task",
    key=task["task_id"],
    value=json.dumps(task, ensure_ascii=False),
    callback=delivery_report
)

# 等待消息发送完成
producer.flush()
```

关键方法：

-   `Producer()`：创建生产者。
-   `produce()`：发送消息。
-   `topic`：指定消息发送到哪个主题。
-   `key`：消息 Key，用于分区路由。
-   `value`：实际消息内容。
-   `flush()`：等待待发送消息完成处理。

### 6.4 Consumer：消费消息

创建 `consumer.py`：

``` python
import json
from confluent_kafka import Consumer

consumer = Consumer({
    "bootstrap.servers": "localhost:9092",
    "group.id": "agent-worker-group",
    "auto.offset.reset": "earliest",
    "enable.auto.commit": False
})

consumer.subscribe(["agent-task"])

try:
    while True:
        msg = consumer.poll(1.0)

        if msg is None:
            continue

        if msg.error():
            print("消费异常:", msg.error())
            continue

        try:
            task = json.loads(msg.value().decode("utf-8"))

            print("收到任务:", task)

            # 执行业务逻辑
            print("执行任务:", task["task_id"])

            # 业务处理成功后提交 Offset
            consumer.commit(
                message=msg,
                asynchronous=False
            )

        except Exception as e:
            print("任务处理失败:", e)

finally:
    consumer.close()
```

### 6.5 运行

先启动消费者：

``` bash
python consumer.py
```

再启动生产者：

``` bash
python producer.py
```

消费者会收到类似消息：

``` json
{
  "task_id": "task_1001",
  "type": "nl2sql",
  "question": "查询今天的活跃告警数量"
}
```

然后执行对应业务逻辑。

以上是学习示例，省略了异常重试、死信队列、Topic
自动创建策略及数据库幂等处理。正式使用时需要补齐。

## 7. Kafka 在 NOEMate / Agent 项目中的应用

结合 NL2SQL、命令 Agent、RAG 服务和 UI Agent，可以设计异步任务架构：

``` text
用户请求
   |
   v
Agent Gateway
   |
   | 鉴权、创建任务、分配 Task ID
   v
Kafka
   |
   +--> agent-task
   +--> agent-event
   +--> agent-result
   |
   +--> NL2SQL Agent Worker
   +--> Command Agent Worker
   +--> RAG Agent Worker
   |
   v
结果汇总 / 状态管理
   |
   v
数据库持久化任务状态
   |
   v
SSE / WebSocket
   |
   v
前端展示任务进度和结果
```

设计建议：

1.  用户提交任务后立即返回 Task ID，不需要一直等待 Agent 执行。
2.  Kafka 负责任务事件的异步传递和缓冲。
3.  不同 Agent Worker 可以独立扩容。
4.  任务状态建议持久化到数据库，不能只依赖 Kafka。
5.  前端实时展示进度，可以继续使用 SSE / WebSocket。

Kafka 和 SSE 不是替代关系：

  技术        主要作用
  ----------- ----------------------------
  Kafka       服务与服务之间的消息通信
  SSE         服务端向浏览器推送实时数据
  WebSocket   浏览器与服务端双向实时通信

不要为了使用 Kafka 而把所有 HTTP 调用都替换掉：

-   NL2SQL 查询需要立即返回结果，且执行时间较短，可以保留同步调用。
-   大批量数据处理、Agent 长任务、后台任务编排，适合引入 Kafka。
-   高频日志和事件采集，也适合 Kafka。

## 8. Kafka 与 HTTP 同步调用的区别

  对比           HTTP 同步调用        Kafka 异步消息
  -------------- -------------------- ----------------------
  通信方式       请求-响应            发布-订阅
  是否等待结果   通常等待             不需要等待
  服务耦合       相对较高             相对较低
  流量削峰       需要额外机制         支持消息缓冲
  消息持久化     需要额外实现         原生支持
  消息回放       需要额外实现         支持通过 Offset 回退
  适用场景       实时查询、即时响应   异步任务、事件驱动

## 9. 建议掌握的知识点

-   [ ] Topic、Partition、Broker 的关系
-   [ ] Producer 如何选择 Partition
-   [ ] Consumer Group 如何实现并行消费
-   [ ] Offset 和手动提交机制
-   [ ] 消息重复消费与幂等设计
-   [ ] 消息顺序保证
-   [ ] 消费失败、重试和 DLQ
-   [ ] Partition 副本与高可用
-   [ ] Consumer Lag 和监控
-   [ ] Kafka 的事务与 Exactly-once 语义

## 10. 核心知识总结

1.  Kafka 通过消息机制实现系统解耦。
2.  Partition 实现数据分片、并行消费和分区内有序。
3.  Consumer Group 实现消费负载均衡，同时支持多个独立订阅者。
4.  Producer 可以通过指定 Partition、指定 Key 或客户端分区策略选择目标
    Partition。
5.  使用业务 Key 可以让同一业务对象的消息进入同一个
    Partition，从而获得分区内顺序保证。
6.  Offset
    提交、幂等、重试、死信队列和消费积压监控，是生产环境可靠性设计的重要组成部分。

## 11. 官方资料

-   Apache Kafka Documentation：https://kafka.apache.org/documentation/
-   Confluent Kafka Python
    Client：https://docs.confluent.io/kafka-clients/python/current/overview.html
