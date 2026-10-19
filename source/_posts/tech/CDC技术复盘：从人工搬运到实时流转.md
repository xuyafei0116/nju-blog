---
title: CDC技术复盘：从人工搬运到实时流转
date: 2026-02-27 15:00:00
categories:
  - 技术
  - 数据架构
tags:
  - CDC
  - Debezium
  - Canal
  - Flink
  - 数据同步
---

# CDC技术复盘：从"人工搬运"到"实时流转"

---

## TL;DR

- CDC捕获DB变更日志(binlog/WAL/Redo Log)，非轮询
- 三代演进：查询触发 → 日志解耦(Canal/Debezium) → 流批一体(Flink CDC)
- □ Canal：MySQL专属，轻量快速
- □ Debezium：多DB支持，三种部署模式(Kafka Connect/Server/Embedded)
- □ Flink CDC：同步即计算，Exactly-Once原生支持
- § 小团队解耦：CDC可直接替代MQ，无需Kafka

---

## 零、痛点：为什么需要CDC

### 传统数据同步的困境

10 个应用都需要知道数据库的变化，最笨的办法是什么？每个应用都去轮询数据库，或者写触发器、监听 API。结果：

- 数据库被 10 个应用轮询，压力暴增
- 触发器写死在数据库里，改一个逻辑要改所有表
- 不同数据库的 API 完全不一样，每换一个数据库就要重写一遍代码
- 传统的全量抽取（ETL）太慢、太笨重，且对业务数据库压力巨大
- **最要命的是双写问题**：应用先写数据库，再写缓存/搜索索引，两步之间没有事务保证，要么缓存丢了，要么索引慢了，数据永远不一致

### 小团队的解耦困境

在小型项目或初创团队中，经常会遇到这样的场景：

> **"业务逻辑需要解耦异步，但又不想引入 Kafka/RabbitMQ 这样的重量级消息中间件——部署成本高、运维复杂、团队学习曲线陡峭..."**

传统解决方案：要么硬编码调用（模块间强耦合），要么引入消息队列（架构变重）。

**CDC 提供了第三条路**：通过监听数据库变更，CDC 能把一次数据写入（如订单创建）自动转化为事件，供其他模块消费。不需要修改业务代码，不需要部署消息中间件，**数据库本身就是最轻量的事件源**。

```
订单服务写数据库 → CDC 捕获变更 → 库存服务/通知服务消费事件
```

这种方案特别适合：
- 不想维护消息队列的小团队
- 已有数据库，希望物尽其用的项目
- 从单体向微服务过渡的渐进式架构演进

---

## 一、CDC是什么

**定义**：CDC(Change Data Capture)捕获DB的Insert/Update/Delete变更。

**核心思想**：别去问数据库"改了什么"，而是让数据库自己"说"。

MySQL 有 binlog，PostgreSQL 有 WAL，MongoDB 有 Oplog，Oracle 有 Redo Log。这些日志本来就是为了数据恢复和复制设计的，记录了所有已提交的更改。CDC 工具就像一个监听器，实时读取这些日志，把每一条变更变成一个事件。

```
DB写入 → binlog/WAL/Redo Log → CDC工具 → 事件流
```

● 只捕获已提交变更，无需担心事务回滚
● 基于DB原生复制机制，非侵入式

---

## 二、三代技术演进

### ▷ 第一代：查询触发

**原理**：轮询SQL查询变更
```sql
SELECT * FROM table WHERE update_time > last_sync_time
```

○ 无法捕获DELETE
○ 需修改表结构(加时间戳字段)
○ 实时性差(秒级)

---

### ▷ 第二代：日志解耦

#### Canal — MySQL"死忠粉"

**原理**：伪装MySQL Slave，接收binlog
```
MySQL Master → binlog → Canal → 消息队列
```
![](./img/cannal架构.png)

Canal 通过伪装成 Slave，发送 Dump 协议，获取 Master 的 binlog 并翻译二进制字节，完成 CDC。这种设计让它对 MySQL 的支持极致高效，但也注定了它无法跨出 MySQL 的圈子。

● 性能极高，毫秒级延迟
● 无侵入，不改表结构
○ 仅支持MySQL/MariaDB

#### Debezium — "全能王"

**原理**：Kafka Connect Source Connector，读日志写Kafka
```
DB → binlog/WAL → Debezium → Kafka Topic → 消费者
```
![](./img/debezium+kafka架构.png)

**为何需要Kafka中转？**

无Kafka时：
```
DB → Debezium → 服务A
              → 服务B
              → 服务C
```
问题：Debezium维护10个连接，DB压力倍增。

有Kafka时：
```
DB → Debezium → Kafka Topic ← 服务A
                              ← 服务B
                              ← 服务C
```
● 解耦生产消费
● 削峰填谷
● 数据可回溯

**三种部署模式**：

| 模式 | 特点 | 场景 |
|-----|------|------|
| Kafka Connect | 高可用、分布式 | 企业级生产 |
| Debezium Server | 独立应用，支持Kinesis/PubSub/Pulsar/HTTP | 无Kafka环境 |
| Embedded | 库嵌入Java应用 | 微服务、轻量级 |

**Debezium Server 独立模式**：Debezium 不必须依赖 Kafka——Server 可直接发到云原生通道（Kinesis/PubSub）、消息队列（Pulsar/RabbitMQ/Kafka）或轻量通道（HTTP/Redis Streams）。

配置示例：
```yaml
debezium:
  source:
    connector.class: io.debezium.connector.mysql.MySqlConnector
    database.hostname: localhost
    table.include.list: inventory.orders
  sink:
    type: http
    http.url: http://order-service:8080/events
```

**Embedded模式**：

信奉"如非必要，勿增实体"？Embedded是最轻量选择。

```java
@Bean
public DebeziumEngine<ChangeEvent<String, String>> debeziumEngine() {
    Properties props = new Properties();
    props.setProperty("connector.class", "io.debezium.connector.mysql.MySqlConnector");
    props.setProperty("table.include.list", "inventory.orders");
    // props.setProperty配置db连接（略）

    return DebeziumEngine.create(Json.class)
        .using(props)
        .notifying(record -> processEvent(record))
        .build();
}
```

● 无需部署Kafka
● 低延迟，资源占用少
○ 无高可用，无数据堆积能力

---

### ▷ 第三代：流批一体(Flink CDC)

**原理**：CDC集成在计算引擎，DB即"流"。

**核心特性**：
1. 全量+增量自动处理(快照→binlog无缝切换)
2. ETL一体化(同步即Join/聚合/清洗)
3. Exactly-Once语义(基于Checkpoint)

```
传统：MySQL → Canal/Debezium → Kafka → Flink → 目标存储
Flink CDC：MySQL → Flink CDC → 目标存储
```

典型应用——实时关联查询：
```sql
-- 实时关联用户表和订单表，生成宽表
SELECT
    u.id, u.name, u.email,
    COUNT(o.id) as order_count,
    SUM(o.amount) as total_amount
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name, u.email;
```

● 极简架构，无Kafka中转
● 流批一体
○ 需Flink知识

---

## 三、全方位对比

### 核心对比

| 维度 | Canal | Debezium | Flink CDC |
|-----|-------|----------|-----------|
| 数据源 | MySQL/MariaDB | 极广(MySQL,PG,Oracle,SQLServer,MongoDB,DB2) | 极广(MySQL,PG,TiDB,Oracle,SQLServer,MongoDB) |
| 架构 | Server+Client | Kafka Connect/Server/Embedded | Flink Job(可无Kafka) |
| 全量同步 | ○需配合DataX | ●自动支持 | ●完美支持 |
| Exactly-Once | ○需自行实现 | ⚠️需Kafka事务 | ●原生支持 |
| 数据加工 | ○无 | ○无 | ●强(Join/聚合/清洗) |
| 部署复杂度 | 低 | 高 | 中 |

### 数据源支持

| 数据库 | Canal | Debezium | Flink CDC |
|--------|-------|----------|-----------|
| MySQL | ● | ● | ● |
| MariaDB | ● | ● | ● |
| PostgreSQL | ○ | ● | ● |
| Oracle | ○ | ● | ● |
| SQL Server | ○ | ● | ● |
| MongoDB | ○ | ● | ● |
| TiDB | ○ | ○ | ● |
| DB2 | ○ | ● | ○ |

### 功能特性对比

| 功能 | Canal | Debezium | Flink CDC |
|------|-------|----------|-----------|
| DDL 捕获 | 部分支持 | ● 完整支持 | ● 完整支持 |
| 断点续传 | ● | ● | ● |
| 多表订阅 | ● | ● | ● |
| 数据过滤 | 简单 | 中等 | 强大 |
| 格式转换 | JSON/Protobuf | JSON/Avro/Protobuf | 灵活自定义 |
| 监控指标 | 基础 | 完善 | 完善 |

![](./img/CDC方案.png)

---

## 四、数据一致性语义

| 语义 | 含义 | 场景 | 复杂度 |
|-----|------|------|--------|
| At-Most-Once | 可能丢失，不重复 | 日志、监控 | 低 |
| At-Least-Once | 不丢失，可能重复 | 大多数业务 | 中 |
| Exactly-Once | 不丢失，不重复 | 金融交易 | 高 |

**工程建议**：At-Least-Once + 消费者幂等，是最优解。绝大多数业务场景选择此方案即可。只有在极少数对重复零容忍的场景（如资金交易），才值得为 Exactly-Once 付出性能和复杂度的代价。

幂等实现：
```java
public void consume(OrderEvent event) {
    String key = event.getOrderId() + "_" + event.getEventType();
    if (redis.setnx(key, "1", Duration.ofHours(24))) {
        processOrder(event); // 首次执行
    }
}
```

---

## 五、选型金字塔

### 决策流程

```
                    开始选型
                       │
        ┌──────────────┴──────────────┐
        │                             │
   只有MySQL/MariaDB              多种数据库
        │                             │
   需复杂实时计算?               需复杂实时计算?
    是/    \否                   是/    \否
     /        \                  /        \
Flink CDC   Canal          Flink CDC   Debezium
```

### 场景化建议

**§ 场景一：只有MySQL，追求简单**
→ **Canal**
● 阿里出品，中文文档丰富
● 架构简单，部署成本低

**§ 场景二：多数据源，已有Kafka**
→ **Debezium(Kafka Connect)**
● 多DB支持，统一技术栈
● 与Kafka生态无缝集成

**§ 场景二(变体)：多数据源，不想引入Kafka**
→ **Debezium Server(独立模式)**
● 支持Kinesis/PubSub/Pulsar/RabbitMQ/HTTP
● 单个容器即可运行

**§ 场景三：需复杂实时处理**
→ **Flink CDC**
● 同步即计算
● Exactly-Once原生支持

**§ 场景四：小团队，不想维护MQ，需业务解耦**
→ **Debezium Embedded** 或 **Canal**——零侵入直连，后期可平滑迁 Kafka（需求见第零节，落地见实践三）

**何时升级到Kafka？**

| 信号 | 说明 |
|-----|------|
| 消费者>5个 | Embedded单点消费成瓶颈 |
| 消费延迟>1分钟 | 需Kafka堆积能力 |
| 需数据回溯 | 新服务消费历史数据 |
| 多语言消费 | 需标准协议 |

升级路径：
```
阶段1：MySQL → Debezium(Embedded) → 消费者
阶段2：MySQL → Debezium → Kafka → 多个消费者
```

对订单服务**完全透明**，依然只写DB。

---

## 六、工程实践

### 实践一：Redis与MySQL双写一致性

**传统问题**：
```java
userRepository.save(user);      // 写DB
cache.delete("user:" + id);     // 删缓存
// 问题：DB成功，Redis失败，数据不一致
```

**CDC方案**：
```
应用只写MySQL → Debezium Server(HTTP模式) → 调用缓存服务API → 删Redis
```

● 单向数据流
● 最终一致性
● 应用代码极简

### 实践二：CQRS架构数据同步

```
写侧(Command) → DB → CDC → MQ → 读侧(Query)更新
```

● 读写完全解耦
● 读侧自动更新
● 天然支持事件溯源

### 实践三：轻量级业务解耦(无MQ)

**场景**：订单创建后异步处理库存扣减和通知，不想维护Kafka。

**传统困境**：
```java
orderService.createOrder(order);
inventoryService.deductStock(order);  // 同步调用，强耦合
notificationService.sendSms(order);   // 阻塞主流程
```

**CDC方案**：

1. 订单服务只写DB：
```java
orderService.createOrder(order);  // 写完即返回
```

2. CDC监听orders表变更

3. 轻量消费者处理：
```java
@Component
public class OrderEventHandler {
    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        inventoryService.deductStock(event.getOrder());
        notificationService.sendSms(event.getOrder());
    }
}
```

**数据一致性**：At-Least-Once——基于 DB 事务不丢失、可能重复投递，幂等消费同第四节代码（同一个 key：orderId+eventType，换成 `setIfAbsent` 写法）。

### 实践四：实时数仓构建

**架构**：
```
业务数据库(MySQL/Oracle) → Flink CDC → 实时清洗/关联 → ClickHouse/Doris
                                    ↓
                              实时大屏/报表
```

**优势**：
- 分钟级延迟的实时分析
- 无需 T+1 等待
- 支持实时决策

---

## 七、结语

CDC本质是平衡**实时性**、**一致性**与**系统复杂性**。

- 只有MySQL，追求简单 → **Canal**
- 多数据源，依赖Kafka → **Debezium(Kafka Connect)**
- 多数据源，不想引入Kafka → **Debezium Server(独立模式)**
- 同步即计算，架构极简 → **Flink CDC**
- 小团队，不想维护MQ，需业务解耦 → **Debezium Embedded** 或 **Canal**

### 选型原则

1. 不要为了技术而技术
2. 考虑团队能力
3. 着眼未来，避免后期重构
4. 从简单开始，渐进式演进
5. CDC可作为MQ"前置方案"，避免过度设计

### CDC未来趋势

- **云原生化**：与 Kubernetes 深度集成，自动扩缩容
- **无服务器化**：Serverless CDC，按需付费
- **实时数仓标配**：CDC 将成为实时数仓的标准组件
- **Schema Registry**：统一的 Schema 管理，更好的兼容性

---

**CDC不是什么新技术，只是把DB本来就在做的事情(记录变更日志)，用更优雅的方式暴露出来。**

理解CDC的本质，比学会使用某个工具更重要。

当我们设计微服务时（服务间通过事件通信），当我们设计CQRS时（写侧和读侧通过事件连接），当我们设计事件溯源时（所有状态变更都是事件），依然在沿用这套伟大的设计哲学——**数据变更即事件**。