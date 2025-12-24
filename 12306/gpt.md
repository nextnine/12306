````md
# 高并发火车票购票系统：需求分析与系统设计（互联网工程设计版）
> 适用：百万级 QPS（读）/ 十万级 TPS（写，经排队削峰后）  
> 版本：v1.0（2025-12-24）  
> 关键词：削峰填谷、区间库存、Redis 原子预扣、订单状态机、幂等、最终一致、热点隔离、可观测

---

## 0. 文档概述

### 0.1 背景与挑战
火车票开售存在“极端洪峰 + 强一致库存约束 + 外部支付依赖”：
- 读：余票/车次查询可达百万级 QPS，热点车次集中。
- 写：下单在短时间爆发，需避免 DB 行锁风暴。
- 一致性：区间票需对多段库存同时扣减，严禁超卖。
- 可靠性：支付回调乱序/重复，出票失败需补偿。

### 0.2 目标（工程化）
- **正确性**：0 超卖；订单状态机严格；资金/库存最终一致。
- **高可用**：查询 ≥99.99%，下单核心 ≥99.95%（排队+降级保障）。
- **低延迟**：查询 P95 <100ms（缓存命中）；出队后下单处理 P95 <1s。
- **可扩展**：水平扩容；按日期/车次分片；热点隔离。
- **可观测**：指标+日志+链路追踪，支持压测与演练。

### 0.3 范围与边界
包含：查询、下单、库存/选座、支付、出票、退改、候补、风控、通知、运营后台。  
不包含：铁路调度系统（视为上游运行图/时刻表数据源）、支付机构内部实现。

---

## 1. 术语
- **Segment（区间段）**：相邻站点之间的最小库存扣减单元（S1→S2、S2→S3…）。
- **区间票**：购买 [i, j) 覆盖 segments i..j-1，需“全段同时成功”。
- **预扣库存**：在 Redis 中原子扣减以抗高并发，再异步落库对账。
- **锁库存**：支付前冻结库存/座位，超时自动释放。
- **幂等**：同一业务请求重复提交/重复回调只生效一次。

---

## 2. 需求分析

### 2.1 角色（Actors）
- 乘客用户：查询、下单、支付、订单查询、退票/改签、候补。
- 运营/客服：车次/票价配置、库存校准、异常处理、风控策略。
- 外部系统：实名校验、支付网关、短信/邮件、上游运行图/时刻表。

### 2.2 核心用例（Use Cases）
1. 查询车次/余票/票价  
2. 下单（可能排队）→ 锁库存/锁座 → 支付 → 出票  
3. 支付超时取消 → 释放库存  
4. 退票/改签 → 库存回补/新单锁票  
5. 候补 → 有票自动触发下单/通知

### 2.3 功能需求（FR）

#### A. 查询域（读多写少）
- 站点查询（联想、热门）
- 车次列表（起终点/日期/时间段筛选）
- 余票查询（车次+日期+区间+席别）
- 票价与规则查询（优惠/学生票/退改规则）
- 订单/行程查询（列表/详情/电子凭证）

#### B. 下单域（高并发核心）
- 乘客管理：实名信息、证件校验、常用乘客
- 规则校验：限购、同乘客行程冲突、黑名单/频控、时效规则
- 排队：返回 queue_token，支持进度/结果查询
- 锁库存/锁座：原子扣减、锁记录、超时释放
- 幂等：重复点击不重复下单、不重复扣库存

#### C. 支付与出票
- 支付发起、支付单管理
- 支付回调（强幂等，支持乱序/重复）
- 出票：票号/检票码/电子凭证
- 出票失败补偿：自动退款/重试/人工介入

#### D. 退票/改签/候补
- 退票：手续费计算、库存回补、退款跟踪
- 改签：新单锁票 + 旧单撤销（两阶段，防票丢）
- 候补：候补队列，来票自动尝试下单并通知

#### E. 运营/风控
- 风控策略：验证码开关、限流阈值、黑名单、设备指纹（可选）
- 运营配置：车次/票价/规则、降级开关
- 监控看板：库存、订单、支付、出票、队列堆积

---

## 3. 非功能需求（NFR）与 SLO

| 维度 | 指标 | 目标 |
|---|---|---|
| 可用性 | 查询链路 | ≥ 99.99%（可降级） |
| 可用性 | 下单链路 | ≥ 99.95%（排队模式） |
| 延迟 | 余票查询 P95 | < 100ms（缓存命中） |
| 延迟 | 出队后下单处理 P95 | < 1s |
| 正确性 | 超卖 | 0 |
| 一致性 | 资金/订单/库存 | 最终一致（分钟级纠偏） |
| 扩展性 | 水平扩容 | 近线性提升 |

容量假设（可调整）：
- 查询峰值：1,000,000 QPS（热点集中）
- 下单入口：100,000 TPS（网关限流+排队后实际处理 20k~50k TPS）
- 支付回调：10,000 QPS（含重试）

---

## 4. 总体架构

### 4.1 架构原则
- 读写分离：查询强缓存；写入排队削峰。
- 热点隔离：按车次/日期分区队列与库存 Key。
- 异步解耦：支付、出票、通知、对账用 MQ。
- 全链路幂等：下单/回调/消费三层去重。
- 最终一致：Redis 预扣 + DB 账本 + 对账补偿。

### 4.2 逻辑组件（微服务）
- API Gateway：鉴权、限流、灰度、WAF、路由
- Query Service：站点/车次/余票/票价
- Order Service：订单、状态机、幂等、排队
- Inventory Service：区间库存扣减/回滚（核心一致性）
- Seat Service：座位图、选座/自动分配
- Payment Service：支付单、回调、对账、退款
- Ticketing Service：出票、凭证生成
- User/Passenger Service：实名与乘客档案
- Risk Service：黑名单/频控/验证码策略
- Notification Service：短信/邮件/站内信
- Admin/OPS：运营后台、配置中心

基础设施建议：
- Redis Cluster（缓存/原子扣减/限流/幂等/队列进度）
- Kafka/RocketMQ（削峰、异步、补偿、DLQ）
- MySQL 分库分表（订单/锁记录/支付账本），读写分离
- 可选：ES（检索）、对象存储（凭证/对账文件）

### 4.3 架构图（Mermaid）
```mermaid
flowchart LR
  U[Client] --> G[API Gateway]
  G --> Q[Query Service]
  G --> O[Order Service]
  O --> MQ[(MQ)]
  MQ --> W[Order Worker]
  W --> I[Inventory Service]
  W --> S[Seat Service]
  W --> DB[(DB)]
  G --> P[Payment Service]
  P --> MQ2[(MQ: Ticketing)]
  MQ2 --> T[Ticketing Service]
  T --> DB
  R[Risk Service] --> G
  N[Notification] <-- MQ
````

---

## 5. 核心设计：库存一致性（区间票）

### 5.1 区间库存模型

将一趟车按站序拆分为相邻 segments：

* segments = (S1→S2, S2→S3, …, Sk-1→Sk)
* 对每个 segment、seat_type 维护 `available_count`

购买区间 [i, j) 时需：

* **检查** segments i..j-1 均 `available_count >= qty`
* **原子扣减**所有这些 segment（要么全成功，要么全失败）

### 5.2 扣库存方案（推荐：Redis Lua 原子预扣 + DB 账本）

**Why**：热点车次下 DB 行锁不可承受；Redis Lua 原子操作吞吐更高。

流程（简化）：

1. Worker 执行 Lua：多 key 检查 + 扣减（原子）
2. 成功：写 `inventory_lock`（含 expire_at），订单→LOCKED
3. 支付成功：锁转实（或锁即实，失败回滚）
4. 支付失败/超时：Lua 回滚释放 + 订单取消

Redis Key 示例：

* `inv:{date}:{train}:{seatType}:{segmentIdx} -> int`
* `lock:{orderId} -> lock_payload (ttl=pay_timeout)`
* `idem:{idempotencyKey} -> orderId (ttl)`

### 5.3 锁库存与过期释放

* 支付倒计时（如 10 分钟）到期自动取消：

  * 订单状态→CANCELLED
  * Lua 释放 segments 库存
  * 删除锁记录（幂等）

### 5.4 对账与纠偏

* 定时任务对账 Redis 与 DB（分钟/小时级）
* 发现偏差：

  * 以 DB 锁记录/订单状态为准重放修正
  * 输出告警与审计记录

---

## 6. 排队削峰（百万级必选）

### 6.1 网关限流

* 维度：用户、IP、设备、车次热点、接口
* 策略：令牌桶/漏桶 + 动态阈值（配置中心热更新）
* 行为：直接拒绝（429）或要求验证码（风控）

### 6.2 排队模型

* 下单接口快速返回：`queue_token`
* MQ 按 `(train_date, train_no)` 分区：

  * 同热点局部有序（降低并发竞争）
  * 非热点可多分区并行
* 用户端：轮询 `queue/result` 或 WebSocket 推送

---

## 7. 订单：状态机与幂等

### 7.1 状态机

* INIT：已创建
* QUEUED：排队中
* LOCKED：已锁库存/锁座，待支付（expire_at）
* PAID：已支付
* TICKETING：出票中
* SUCCESS：出票成功
* CANCELLED：取消/支付超时
* FAILED：出票失败（含已退款/待处理）

迁移约束：只允许合法迁移；写时校验“当前状态 + version”。

### 7.2 幂等策略（必须）

* 下单幂等：`Idempotency-Key`（客户端生成或服务端发放）
* 支付回调幂等：按 `txn_id` 去重
* MQ 消费幂等：按 `message_id` 或 `order_id + step` 去重（可 Redis set / DB 去重表）

---

## 8. 座位分配（Seat Service）

### 8.1 数据结构

* 车厢座位图（carriage, seat_no）
* 每座位占用用 bitmap 表示 segment 占用情况：

  * 可用检查：区间 mask 与 occupied_bitmap 做位运算
  * 占用：bitmap OR 更新（带 version）

### 8.2 策略与降级

* 正常：连座优先、同车厢优先、偏好（窗/过道）
* 峰值降级：关闭选座/连座，仅自动分配（快且冲突少）

---

## 9. 数据设计（核心表 & 分片）

### 9.1 核心表（示例）

* `train_schedule(train_no, date, station_idx, arr, dep, ...)`
* `inventory_segment(train_no, date, segment_idx, seat_type, available, version)`
* `inventory_lock(order_id, train_no, date, seat_type, segments_json, qty, expire_at, status)`
* `order(order_id, user_id, status, amount, expire_at, idempotency_key, version, created_at)`
* `order_item(order_id, passenger_id, from_idx, to_idx, seat_type, seat_no, carriage)`
* `payment(payment_id, order_id, txn_id, status, amount, created_at)`
* `refund(refund_id, order_id, status, amount, created_at)`
* `queue_task(queue_token, order_id, status, result, created_at)`

### 9.2 分库分表建议

* 库存域：按 `date(月/日)` + `hash(train_no)` 分片（热点隔离）
* 订单域：按 `hash(user_id)` 分片（用户查询友好）
* 支付账本：按 `order_id` 分片，支持对账

---

## 10. API 设计（节选）

### 10.1 查询类

* `GET /stations?q=` 站点联想
* `GET /trains?from=&to=&date=` 车次列表
* `GET /availability?train=&date=&from=&to=` 余票
* `GET /price?train=&date=&from=&to=&seatType=` 票价

### 10.2 下单/排队

* `POST /orders` 创建订单（返回 order_id + queue_token）

  * Header: `Idempotency-Key: xxx`
* `GET /queue/result?token=` 排队结果
* `GET /orders/{id}` 订单详情
* `POST /orders/{id}/cancel` 取消（幂等）

### 10.3 支付/出票

* `POST /payments` 发起支付
* `POST /payments/callback` 支付回调（幂等）
* `GET /tickets/{orderId}` 电子凭证

### 10.4 退改/候补

* `POST /refunds` 退票
* `POST /reschedule` 改签
* `POST /waitlist` 候补

错误码建议：

* `42901` 限流/系统繁忙
* `40902` 库存不足
* `40901` 幂等冲突（已存在订单）
* `40903` 规则不满足（限购/冲突）
* `50020` 出票失败（进入补偿）

---

## 11. 缓存、消息与热点治理

### 11.1 缓存策略

* 站点/运行图：长 TTL + 版本号
* 余票：短 TTL（1~5s）+ 主动刷新（热点）
* 订单列表：短 TTL 或读从库

### 11.2 热点治理

* Redis Cluster 分片；热点 key 打散（必要时引入热点隔离节点）
* 热点车次独立 MQ 分区或专用 Topic
* 余票展示可降级为“有/紧张/无”（可选）

---

## 12. 安全与风控

* 鉴权：JWT/Session；关键操作二次校验
* 隐私：证件号/手机号字段级加密；访问审计
* 防刷：频控、验证码开关、黑名单、（可选）设备指纹/行为风控
* 运营后台：RBAC 权限与操作留痕

---

## 13. 可观测性（Observability）

* Metrics：QPS/TPS、排队长度、扣减成功率、回滚次数、支付成功率、出票耗时、MQ 堆积
* Tracing：order_id 贯穿 Query→Order→Inventory→Payment→Ticketing
* Logging：结构化日志（user_id、train_no、date、token、状态迁移、错误码）
* 告警：SLO 违约、MQ 堆积阈值、Redis/DB P95 飙升、错误码激增

---

## 14. 容量规划与压测（要点）

### 14.1 容量规划思路（简版）

* 查询：缓存命中率为王（>95%）；水平扩展 Query/Gateway
* 下单：入口限流 + 排队；Worker 按分区扩容
* Redis：预扣库存与幂等键为关键负载；分片/热点隔离
* MQ：按热点分区，确保消费能力 ≥ 出队处理能力

### 14.2 压测场景

1. 开售 0 点：余票查询 1,000,000 QPS
2. 下单洪峰：入口 100,000 TPS（限流+排队后处理 30,000 TPS）
3. 热点车次：单车次占 30% 流量（验证热点隔离）
4. 外部依赖抖动：支付延迟 5~10s、回调重复/乱序
5. 故障注入：Redis/MQ/DB 抖动、服务重启、网络丢包

---

## 15. 风险与权衡（简表）

* Redis 预扣带来最终一致复杂度 → 用锁记录账本 + 对账补偿
* 精细选座计算耗时与冲突 → 峰值降级关闭
* 热点车次造成 key/分区热点 → 专用分区/Topic + 资源隔离

---

## 附录：关键时序（Mermaid）

### A. 下单（排队 + 预扣库存）

```mermaid
sequenceDiagram
  participant C as Client
  participant G as Gateway
  participant O as OrderSvc
  participant MQ as MQ
  participant W as Worker
  participant R as Redis
  participant DB as DB

  C->>G: POST /orders (Idempotency-Key)
  G->>O: create INIT + queue_token
  O->>MQ: enqueue(order_id)
  O-->>C: queue_token

  MQ->>W: consume(order_id)
  W->>R: Lua multi-segment check&decr
  alt success
    W->>DB: write inventory_lock + order->LOCKED(expire)
  else fail
    W->>DB: order->FAILED(no ticket)
  end
```

### B. 支付回调（幂等）与出票

```mermaid
sequenceDiagram
  participant Pay as PaymentGW
  participant P as PaymentSvc
  participant DB as DB
  participant MQ as MQ
  participant T as TicketingSvc

  Pay->>P: callback(txn_id, order_id)
  P->>DB: idem check txn_id + order state
  alt first time
    P->>DB: order->PAID
    P->>MQ: enqueue(ticketing)
  else duplicate
    P-->>Pay: OK (idempotent)
  end

  MQ->>T: consume(ticketing)
  T->>DB: order->SUCCESS / FAILED + compensation
```

---

## 结尾

本设计以“**限流 + 排队削峰**”保障洪峰可用性，以“**Redis 原子预扣 + DB 账本 + 对账补偿**”实现库存一致性与吞吐兼顾，并通过“**状态机 + 全链路幂等 + 可观测**”保证线上可控与可运营。

```
```
