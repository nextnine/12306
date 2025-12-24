# 高并发火车票购票系统：需求分析与系统设计（互联网工程设计版）
> 适用规模：百万级 QPS（读）/ 十万级 TPS（写，经排队削峰后）  
> 版本：v1.1（2025-12-24）  
> 关键词：削峰填谷、区间库存、Redis 原子预扣、订单状态机、幂等、最终一致、热点隔离、可观测

---

## 0. 文档概述

### 0.1 背景与挑战
火车票开售存在“极端洪峰 + 强一致库存约束 + 外部依赖（支付/实名/出票）”的典型互联网高并发场景：
- 读：余票/车次查询可达百万级 QPS，且热点车次高度集中。
- 写：下单瞬时爆发，若直接落 DB 容易引发行锁风暴与级联雪崩。
- 一致性：区间票需对多段库存同时扣减，严禁超卖。
- 可靠性：支付回调可能乱序/重复，出票失败需要补偿与对账纠偏。

### 0.2 建设目标
- 正确性：库存 0 超卖；订单状态机严格；资金-库存-订单最终一致。
- 高可用：查询 ≥ 99.99%，下单核心 ≥ 99.95%（排队模式保障）。
- 低延迟：查询 P95 < 100ms（缓存命中）；出队后下单处理 P95 < 1s。
- 可扩展：水平扩容；按日期/车次分片；热点隔离与弹性伸缩。
- 可观测：指标、日志、链路追踪、审计与对账可落地。

### 0.3 范围与边界
包含：查询、下单、库存/选座、支付、出票、退改、候补、风控、通知、运营后台。  
不包含：铁路运行调度系统本体（视为上游数据源）、支付机构内部实现细节。

---

## 1. 术语与定义
- Segment（区间段）：相邻站点之间的最小库存扣减单元（S1->S2、S2->S3…）。
- 区间票：购买区间 [i, j) 覆盖 segments i..j-1，必须“全段同时成功”。
- 预扣库存：在 Redis 进行原子扣减以抗高并发，再异步落库对账。
- 锁库存：支付前冻结库存/座位，超时自动释放。
- 幂等：同一请求/回调重复到达只生效一次。

---

## 2. 需求分析

### 2.1 角色（Actors）
- 乘客用户：查询、下单、支付、订单查询、退票/改签、候补。
- 运营/客服：车次与票价配置、库存校准、异常订单处理、风控策略。
- 外部系统：实名校验、支付网关、短信/邮件、上游运行图/时刻表。

### 2.2 核心业务流程（用户视角）
1) 查询车次与余票  
2) 选择日期/车次/区间/席别/乘客（可选偏好：靠窗、连座）  
3) 下单（可能进入排队）  
4) 锁库存/锁座成功 -> 待支付（倒计时）  
5) 支付回调确认 -> 出票 -> 订单成功  
6) 支付超时/失败 -> 自动取消并释放库存  
7) 退票/改签/候补（扩展）

### 2.3 功能需求（FR）

#### A. 查询域（读多写少）
- 站点查询：模糊搜索、联想、热门站点。
- 车次查询：起终点/日期/时间段筛选，支持多种排序。
- 余票查询：按车次+日期+区间+席别返回余票信息。
- 票价查询：席别、优惠、规则、手续费说明。
- 订单/行程查询：列表、详情、电子凭证展示。

#### B. 下单域（高并发核心）
- 乘客管理：实名信息、证件校验、常用乘客。
- 规则校验：限购、同乘客行程冲突、黑名单、频控、时效规则。
- 排队：生成 queue_token；支持进度/结果查询。
- 锁库存/锁座：扣减与锁记录；过期释放。
- 防重与幂等：重复点击不重复扣票；同一幂等键返回同一订单。

#### C. 支付与出票
- 支付发起：创建支付单并返回支付参数。
- 支付回调：强幂等，处理乱序/重复。
- 出票：生成票号/检票码/电子凭证。
- 失败补偿：出票失败自动退款/重试/人工介入。

#### D. 退改与候补
- 退票：手续费计算、库存回补、退款进度可查询。
- 改签：新单锁票 + 旧单撤销（两阶段，防止用户“票丢”）。
- 候补：无票入队，有票自动尝试下单并通知。

#### E. 运营与风控
- 策略配置：限流阈值、验证码开关、黑名单、频控规则。
- 降级开关：关闭选座、降低余票精度、强制排队。
- 运营看板：库存、订单、支付、出票、队列堆积与告警。

---

## 3. 非功能需求（NFR）与 SLO

### 3.1 SLO 建议
| 维度 | 指标 | 目标 |
|---|---|---|
| 可用性 | 查询链路 | >= 99.99%（可降级） |
| 可用性 | 下单链路 | >= 99.95%（排队模式） |
| 延迟 | 余票查询 P95 | < 100ms（缓存命中） |
| 延迟 | 出队后下单处理 P95 | < 1s |
| 正确性 | 超卖 | 0 |
| 一致性 | 资金-订单-库存 | 最终一致（分钟级纠偏） |
| 可扩展 | 水平扩容 | 近线性提升 |

### 3.2 峰值容量假设（用于设计）
- 查询：1,000,000 QPS（热点车次集中）
- 下单入口：100,000 TPS（限流+排队后实际处理 20k~50k TPS）
- 支付回调：10,000 QPS（含重试）

---

## 4. 总体架构设计

### 4.1 架构原则
- 读写分离：查询强缓存；写入排队削峰。
- 热点隔离：按车次/日期分区队列与库存 key。
- 异步解耦：支付、出票、通知、对账全部通过 MQ。
- 全链路幂等：下单、回调、消费三层去重。
- 最终一致：Redis 预扣 + DB 账本 + 对账补偿。

### 4.2 服务划分（建议）
- API Gateway：鉴权、限流、灰度、WAF、路由
- Query Service：站点/车次/余票/票价查询
- Order Service：订单、状态机、幂等、排队
- Inventory Service：区间库存扣减/回滚（一致性核心）
- Seat Service：座位图、选座/自动分配
- Payment Service：支付单、回调、对账、退款
- Ticketing Service：出票、凭证
- User/Passenger Service：实名与乘客档案
- Risk Service：黑名单/频控/验证码策略
- Notification Service：短信/邮件/站内信
- Admin/OPS：运营后台、配置中心

基础设施：
- Redis Cluster：缓存、库存预扣、幂等、限流、队列进度
- Kafka/RocketMQ：削峰、异步、重试、DLQ
- MySQL 分库分表：订单、锁记录、支付账本（读写分离）
- 可选：ES（检索）、对象存储（凭证/对账文件）

---

## 5. 核心设计：区间库存与一致性

### 5.1 区间库存模型
将一趟车按站序拆成相邻 segments：
- segments = (S1->S2, S2->S3, ..., Sk-1->Sk)
- 对每个 segment、seat_type 维护 available_count

购买区间 [i, j) 需要：
- 检查 segments i..j-1 均可售
- 原子扣减全部涉及 segments（全成功或全失败）

### 5.2 扣库存方案（推荐）
方案：Redis Lua 原子预扣 + DB 账本（锁记录）+ 对账纠偏

关键点：
- 余票展示主要来自缓存（Redis 或短 TTL 缓存）
- 下单扣减使用 Lua 一次性完成“多 key 检查与扣减”的原子操作
- 扣减成功后写 DB 的 inventory_lock 作为可审计账本
- 支付超时/失败必须回滚库存（同样走 Lua，幂等）

Redis Key 示例：
- inv:{date}:{train}:{seatType}:{segment} -> int
- lock:{orderId} -> lock_payload (ttl=pay_timeout)
- idem:{idempotencyKey} -> orderId (ttl)

### 5.3 锁库存与过期释放
- 订单进入 LOCKED 状态时写入 expire_at（例如 10 分钟）
- 到期自动取消并释放库存
- 释放必须幂等：重复释放不产生负库存

### 5.4 对账与补偿
- 周期性对账任务核对 Redis 与 DB（分钟或小时级）
- 以 DB 锁记录与订单状态机为准进行纠偏重放
- 纠偏过程输出告警与审计日志，供运营与风控追踪

---

## 6. 排队削峰（百万级必选）

### 6.1 网关限流
- 维度：用户、IP、设备、车次热点、接口类型
- 策略：令牌桶/漏桶 + 动态阈值（配置中心热更新）
- 行为：返回 429（系统繁忙）或触发验证码（风控）

### 6.2 排队模型
- 创建订单后快速返回 queue_token
- 将任务写入 MQ，按 (train_date, train_no) 分区
- Worker 出队后执行：库存预扣 + 座位分配 + 写锁记录 + 更新订单状态
- 客户端通过轮询或 WebSocket 获取排队结果

### 6.3 峰值降级开关（可配置）
- 关闭选座/连座（仅自动分配）
- 余票精度降级（有/紧张/无）
- 强制排队（拒绝同步路径）

---

## 7. 订单：状态机与幂等

### 7.1 订单状态机
- INIT：已创建
- QUEUED：排队中
- LOCKED：已锁库存/座位，待支付（expire_at）
- PAID：已支付
- TICKETING：出票中
- SUCCESS：出票成功
- CANCELLED：取消或超时
- FAILED：失败（含出票失败已退款/待处理）

迁移约束：仅允许合法迁移；写入需校验当前状态与 version。

### 7.2 幂等设计（必须）
- 下单幂等：Idempotency-Key（客户端生成或服务端发放 token）
- 支付回调幂等：txn_id 去重
- MQ 消费幂等：message_id 或 order_id + step 去重（Redis set 或 DB 去重表）

---

## 8. 座位分配（Seat Service）

### 8.1 座位占用表示
- 每个座位用 bitmap 表示 segment 占用情况
- 区间可用检查：mask 与 occupied_bitmap 位运算判断
- 占用更新：bitmap OR 写回（带 version，避免并发覆盖）

### 8.2 策略与降级
- 正常：同乘客尽量同车厢、相邻座、窗/过道偏好
- 峰值：关闭精细策略，采用快速自动分配，提升吞吐与成功率

---

## 9. 数据设计（核心表与分片）

### 9.1 核心表（示例）
- train_schedule(train_no, date, station_idx, arr_time, dep_time, ...)
- inventory_segment(train_no, date, segment_idx, seat_type, available, version)
- inventory_lock(order_id, train_no, date, seat_type, segments_json, qty, expire_at, status)
- orders(order_id, user_id, status, amount, expire_at, idempotency_key, version, created_at)
- order_item(order_id, passenger_id, from_idx, to_idx, seat_type, seat_no, carriage)
- payment(payment_id, order_id, txn_id, status, amount, created_at)
- refund(refund_id, order_id, status, amount, created_at)
- queue_task(queue_token, order_id, status, result, created_at)

### 9.2 分库分表建议
- 库存域：按 date（月/日） + hash(train_no) 分片（热点隔离）
- 订单域：按 hash(user_id) 分片（便于用户查询）
- 支付/退款：按 order_id 分片，支持对账与审计

---

## 10. API 设计（节选）

### 10.1 查询类
- GET /stations?q= 站点联想
- GET /trains?from=&to=&date= 车次列表
- GET /availability?train=&date=&from=&to= 余票
- GET /price?train=&date=&from=&to=&seatType= 票价

### 10.2 下单与排队
- POST /orders 创建订单（返回 order_id + queue_token）
  - Header: Idempotency-Key: xxx
- GET /queue/result?token= 排队结果
- GET /orders/{id} 订单详情
- POST /orders/{id}/cancel 取消（幂等）

### 10.3 支付与出票
- POST /payments 发起支付
- POST /payments/callback 支付回调（幂等）
- GET /tickets/{orderId} 电子凭证

### 10.4 退改与候补
- POST /refunds 退票
- POST /reschedule 改签
- POST /waitlist 候补

错误码建议：
- 42901：限流/系统繁忙
- 40902：库存不足
- 40901：幂等冲突（已存在订单）
- 40903：规则不满足（限购/冲突）
- 50020：出票失败（进入补偿）

---

## 11. 缓存、消息与热点治理

### 11.1 缓存策略
- 站点/运行图：长 TTL + 版本号
- 余票：短 TTL（1-5 秒）+ 热点主动刷新
- 订单列表：短 TTL 或读从库（按业务取舍）

### 11.2 热点治理
- Redis Cluster 分片；热点 key 必要时打散与隔离资源
- 热点车次使用专用 MQ 分区或专用 Topic
- 余票展示支持降级为“有/紧张/无”（可选）

---

## 12. 安全与风控
- 鉴权：JWT/Session；关键操作二次校验
- 隐私：证件号/手机号字段级加密；全量审计
- 防刷：频控、验证码开关、黑名单、（可选）设备指纹/行为风控
- 运营后台：RBAC 权限与操作留痕

---

## 13. 可观测性（Observability）
- Metrics：QPS/TPS、排队长度、扣减成功率、回滚次数、支付成功率、出票耗时、MQ 堆积
- Tracing：order_id 贯穿 Query->Order->Inventory->Payment->Ticketing
- Logging：结构化日志（user_id、train、token、状态迁移、错误码）
- 告警：SLO 违约、队列堆积、Redis/DB P95 飙升、错误码激增

---

## 14. 容量规划与压测

### 14.1 容量规划要点
- 查询链路以缓存命中率为核心（建议 >95%）
- 下单链路以限流+排队为核心，Worker 按分区水平扩容
- Redis 是库存与幂等关键依赖，需要分片与热点隔离方案
- MQ 吞吐需保证消费能力 >= 出队处理能力，避免堆积

### 14.2 压测场景
1) 开售瞬间：余票查询 1,000,000 QPS  
2) 下单洪峰：入口 100,000 TPS（限流+排队后处理 30,000 TPS）  
3) 热点车次：单车次占 30% 流量（验证热点隔离）  
4) 外部抖动：支付延迟 5-10 秒、回调重复/乱序  
5) 故障注入：Redis/MQ/DB 抖动、服务重启、网络丢包

---

## 15. 风险与权衡（简表）
- Redis 预扣增加最终一致复杂度 -> 锁记录账本 + 对账纠偏 + 审计
- 精细选座耗时且冲突高 -> 峰值降级关闭
- 热点车次导致 key/分区热点 -> 专用分区/Topic + 资源隔离

---

## 附录：可选时序图（若平台 Mermaid 渲染不稳定可删除）
以下 Mermaid 在 GitHub 可用；若仍报错，请检查代码块围栏是否完整、且不要夹杂未闭合的反引号。

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
  W->>R: Lua multi-seg check and decr
  alt success
    W->>DB: write lock and order->LOCKED
  else fail
    W->>DB: order->FAILED
  end

