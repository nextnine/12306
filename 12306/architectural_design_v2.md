# 高并发火车票购票系统——概要设计说明书
> 版本：v2.0（2025-12-26）  
> 适用范围：互联网/移动售票场景，读峰值≥100 万 QPS、写峰值（出队后下单）≥20 万 TPS（排队削峰后）  
> 文档类型：概要设计（高层架构、模型、关键策略）  
> 渲染说明：包含 mermaid 图表，需在支持 mermaid 的环境查看；不支持时可用 mermaid Live Editor 或文字版关系/时序/状态描述。

---

## 章节摘要
- 引言：目的、背景、定义、参考。
- 任务概述：目标、运行环境、需求概述、条件与限制。
- 总体设计：处理流程，总体结构/模块外部设计，功能分配。
- 接口设计：外部接口、内部接口。
- 数据结构设计：逻辑、物理、数据与程序关系。
- 运行设计：运行模块组合、运行控制、运行时间。
- 出错处理设计：输出信息与对策。
- 安全保密设计、维护设计。
- ADR 与演化、附录。

---

## 一．引言
1. 编写目的：提供高并发火车票购票系统的高层架构、关键策略、模型与演化决策，指导后续详细设计与实现，读者包括架构/开发/测试/运维。  
2. 项目背景：春运/节假日高峰，对接铁路运行图/实名/支付/通知等外部系统；委托/开发/运维单位协同。  
3. 定义：专用术语与缩写见 SRS；架构决策详见 ADR 记录。  
4. 参考资料：项目任务书或合同（如有）、SRS、测试计划（初稿）、用户操作手册（初稿）、相关标准/规范及本文件引用的资料。  

---

## 二．任务概述
1. 目标：支撑高并发查询与下单，零超卖，满足可用性/一致性/性能目标，覆盖查询、下单/排队、支付/出票、退改/候补、通知、风控与运营。  
2. 运行环境：云/数据中心 K8s，多 AZ；依赖 Redis/Kafka/MySQL/ES/对象存储等中间件。  
3. 需求概述：参见 SRS 功能/非功能需求，概要设计侧重架构与策略实现路径。  
4. 条件与限制：合规与外部依赖 SLA；容量/资源约束；发布与演练需灰度、回滚预案；mermaid 渲染需兼容或用文字版替代。  

---

## 三．总体设计
### 1．处理流程
- 核心流程：查询（缓存优先）→下单排队→库存预扣/锁座→支付→出票→通知→退改/候补补偿。详见时序图。

### 2．总体结构和模块外部设计
#### 上下文/组件关系图
```mermaid
%%{init: {'theme':'neutral','flowchart':{'nodeSpacing':36,'rankSpacing':48}}}%%
flowchart LR
    subgraph Client[客户端]
      Web[Web/小程序]
      App[移动 App]
      Ops[运营/客服]
    end

    subgraph Access[接入层]
      CDN[CDN/WAF]
      GW[API Gateway\n鉴权/限流/灰度]
    end

    subgraph Services[业务域]
      Q[Query\n时刻表/余票]
      O[Order\n排队/幂等/状态机]
      I[Inventory\n区间库存预扣]
      S[Seat\n锁座/释放]
      P[Payment\n支付单/回调幂等]
      T[Ticketing\n出票/补偿]
      A[After-sales\n退改/候补]
      R[Risk & Config\n风控/策略]
      N[Notification\n短信/邮件/推送]
    end

    subgraph Data[中间件/数据]
      Redis[Redis Cluster]
      MQ[Kafka/RabbitMQ]
      DB[MySQL Shards]
      ES[Elasticsearch]
      OSS[对象存储]
    end

    subgraph External[外部系统]
      Pay[支付网关]
      Verify[实名/证件核验]
      Railway[铁路运行图/座席]
      Sms[短信/邮件/推送渠道]
    end

    Web --> CDN --> GW
    App --> CDN
    Ops --> GW
    CDN --> GW

    GW --> Q & O & P & A & R
    O --> I --> S
    O --> P
    P --> T
    T --> N
    A --> I
    Q --> ES
    Q --> DB
    O --> DB
    I --> Redis
    I --> MQ
    O --> MQ
    P --> MQ
    N --> MQ
    MQ --> T
    MQ --> A
    DB --> OSS

    P --> Pay
    R --> Verify
    Q --> Railway
    N --> Sms
```

#### 部署视图（高并发场景）
```mermaid
%%{init: {'theme':'neutral','flowchart':{'nodeSpacing':36,'rankSpacing':48}}}%%
flowchart TB
    CDN[CDN/WAF]
    LB[负载均衡]
    GW[API Gateway]
    subgraph K8s[容器编排集群]
      QPod[Query Pods]
      OPod[Order Pods]
      IPod[Inventory Pods]
      SPod[Seat Pods]
      PPod[Payment Pods]
      TPod[Ticketing Pods]
      APod[After-sales Pods]
      RPod[Risk Pods]
      NPod[Notify Pods]
    end
    subgraph Data[数据/中间件]
      Redis[Redis Cluster\nSlot 分片+故障切换]
      MQ[Kafka/RabbitMQ\n多分区]
      DB[MySQL 分库分表\n主从/Proxy]
      ES[ES 集群]
    end
    CDN --> LB --> GW --> K8s
    QPod --> Redis
    QPod --> DB
    QPod --> ES
    OPod --> Redis
    OPod --> DB
    OPod --> MQ
    IPod --> Redis
    IPod --> MQ
    SPod --> Redis
    PPod --> MQ
    TPod --> MQ
    APod --> MQ
    NPod --> MQ
    MQ --> TPod
    MQ --> APod
    MQ --> NPod
    DB -.-> Backup[冷热备份/归档]
```

---

### 3．功能分配
- 接入层：鉴权、限流、灰度、路由、幂等键透传。
- 查询域：站点/车次/余票/价目，缓存+ES+只读库。
- 订单/库存/座位：排队、预扣、锁座、状态机、超时回收。
- 支付/出票：支付单、回调幂等、出票重试/补偿。
- 退改/候补：规则校验、双阶段改签、候补匹配与锁票。
- 风控与运营：黑名单、频控、灰度策略；通知：短信/邮件/推送。
- 数据与中间件：Redis/Kafka/MySQL/ES/OSS 提供缓存、事件、存储、索引。

---

## 四．接口设计
### 1．外部接口
- 与铁路运行图/座席、实名/证件核验、支付平台、短信/邮件/推送、风控服务的接口：遵循各自 SLA/超时/重试/验签要求（详见 SRS/API 文档）。

### 2．内部接口
- 服务间通过 API Gateway（HTTP/REST）和 Kafka/RabbitMQ 事件通信；同订单/车次使用分区键保证局部有序；所有写接口携带 `idempotent_key`。

---

## 五．数据结构设计
### 1．逻辑结构设计
- 缓存分层与热点治理：浏览器/APP 本地缓存→CDN/边缘→Gateway 本地缓存→Redis（5~20s TTL，二级本地缓存 200~500ms，弱一致）→只读库/ES；防击穿/雪崩：随机 TTL、请求合并、预热、兜底；版本戳/校验和纠偏。
- 库存一致性/防超卖：区间库存 Redis Lua 原子预扣+版本号；写侧单线程/分片顺序消费 MQ 回写 MySQL 账本；定时对账与差值告警；支付超时/出票失败原路回补。
- 幂等：入口 `idempotent_key`+`queue_token`（15 分钟），幂等记录在线 7 天、归档 30 天；状态只能前进。
- 支付回调：签名+幂等，指数退避重试（1/2/4/8... 最多 6~8 次），落库失败入 DLQ。
- 排队与削峰：入口频控（账号/设备 3 QPS，突发桶 10），队列上限 5 万/车次/日，出队速率 `min(2%*remain, 2000)` 单/秒，超长转候补。
- 分库分表：库按车次/日期哈希或区间，表按订单号雪花 ID 路由；跨分片读依赖 ES/OLAP；再平衡用双写/灰度切换和回放；历史归档策略。
- MQ：按领域拆 topic（order/stock/payment/notify），同订单/车次分区键保证局部有序；消息含 `event_id/version`；延迟队列+DLQ 重试，支持 7 天回溯。
- 候补：状态机（PENDING→MATCHING→LOCKED→AWAIT_PAY→SUCCESS/FAIL），分段队列与优先级（时间、会员、乘客数）；锁票/支付超时回补，库存/支付解耦。
- 外部集成：实名/支付/短信等设定超时、最大重试（3 次指数退避）、熔断与降级返回码；SLA 明确，签名/验签算法契约化。
- 安全与合规：实名、加密/脱敏、审计、风控联动；频控与灰度可配置（1%~20% 灰度）。
- 监控与告警：对账差值、补偿成功率、滞留订单、DLQ、支付回调重复率、排队 P95、缓存命中率、Redis/DB/消息 P95、外部超时/熔断。

### 2．物理结构设计
- 分库分表：库按车次/日期哈希或区间，表按订单号雪花 ID 路由；跨分片读依赖 ES/OLAP；再平衡用双写/灰度切换和回放；历史归档策略。
- 缓存/队列：Redis Cluster slot 分片+故障切换，单分片 <5 万 QPS；Kafka/RabbitMQ 多分区，关键链路局部有序。
- 存储：对象存储用于报表/归档；备份与冷热分层按附录容量建议。

### 3．数据结构与程序的关系
- 服务与数据域映射：Query→ES/Redis/只读库；Order/Inventory/Seat→Redis+MySQL 账本；Payment→支付单表+回调事件；Ticketing→票号表；After-sales→退款/改签/候补表；事件通过 MQ 串联状态机。

---

## 六．运行设计
### 1．运行模块的组合
- K8s 部署的无状态服务（查询/订单/库存/座位/支付/出票/退改候补/风控/通知），对应上文的 Pods 与中间件集群。

### 2．运行控制
- 排队与速率控制：`min(2%*remain, 2000)` 控制出队；队列上限 5 万/车次/日；频控与灰度可配置。
- 超时控制：锁座/支付超时回收；外部接口超时+重试+熔断；幂等状态单向推进。

### 3．运行时间
- 性能目标：查询 P95<120ms；下单出队后 P95<3s；排队等待 P95<60s。运行窗口需满足开售时段的弹性扩容与快速降级。

---

## 七．出错处理设计
1. 出错输出信息：对外错误码（如 400/401/403/409/423/504）与可诊断信息；内部告警含事件 ID、分区键、版本号。  
2. 出错处理对策：重试（指数退避）、幂等返回已有结果、降级（转候补/缓存兜底）、熔断/隔离外部依赖、DLQ+回溯重放；锁座/库存失败即时回补。  

---

## 八．安全保密设计
- 身份与鉴权：依赖网关鉴权，服务内鉴权/签名校验。
- 数据安全：传输全链路 TLS；数据加密/脱敏（实名、证件、手机号、支付要素）；访问审计与留痕。
- 风控与频控：黑白名单、设备指纹（可选）、验证码、滑窗频控、灰度策略。

---

## 九．维护设计
- 可观测性：指标、日志、链路追踪；告警阈值参考关键策略。
- 部署与变更：无状态服务滚动发布、灰度与回滚；健康检查与自动故障转移。
- 运行手册：压测与演练（排队/支付/回调/出票链路）、容量预测与再分片计划。

---

## 模型与视图

### 对象模型
- Train、Station、SegmentInventory、SeatLock、Order、OrderItem、Payment、Ticket、Refund/Change、StandbyRequest、RiskProfile、ConfigToggle。

### ER 示意
```mermaid
erDiagram
  TRAIN ||--o{ SEGMENTINVENTORY : has
  TRAIN ||--o{ SEATLOCK : alloc
  ORDER ||--|{ ORDERITEM : contains
  ORDER ||--|| PAYMENT : maps
  ORDER ||--o{ TICKET : issues
  ORDER ||--o{ REFUNDCHANGE : aftersales
  ORDER ||--o{ STANDBYREQUEST : fallback
  USER ||--o{ ORDER : creates
  USER ||--o{ PASSENGER : manages
  SEGMENTINVENTORY {
    string train_no
    date run_date
    string segment_id
    string seat_class
    int remain
    int version
  }
  ORDER {
    string id
    string user_id
    string status
    string queue_token
    string idempotent_key
    datetime expire_at
    int amount
  }
  PAYMENT {
    string id
    string order_id
    string channel
    string status
    int amount
    int notify_version
  }
  TICKET {
    string id
    string order_id
    string ticket_no
    string passenger_id
  }
```

### 时序（下单→支付→出票）
```mermaid
%%{init: {'theme':'neutral','sequence':{'mirrorActors':false,'rightAngles':true}}}%%
sequenceDiagram
    participant C as Client
    participant GW as API Gateway
    participant O as Order Service
    participant I as Inventory
    participant S as Seat
    participant P as Payment
    participant T as Ticketing
    participant N as Notification
    C->>GW: 提交下单请求
    GW->>O: 透传幂等键/风控态
    O->>I: 预扣库存
    I-->>O: 预扣结果
    O->>S: 锁座（可选）
    S-->>O: 锁座结果
    O-->>C: 返回 queue_token
    C->>GW: 轮询/推送查询 queue_token
    GW->>O: 读取排队状态
    O-->>C: 返回支付跳转
    C->>P: 调起支付
    P-->>O: 回调（幂等）
    O->>T: 触发出票
    T-->>O: 出票结果
    O->>N: 通知
    N-->>C: 送达短信/推送
```

### 状态机（订单/支付/出票）
```mermaid
%%{init: {'theme':'neutral','flowchart':{'nodeSpacing':32,'rankSpacing':40}}}%%
flowchart LR
    A[WAIT_PAY] -->|支付成功| B[PAID]
    A -->|超时/取消| X[CLOSED]
    B -->|出票中| C[TICKETING]
    C -->|出票成功| D[SUCCESS]
    C -->|出票失败/回滚| E[FAIL]
    E -->|退款成功| X
    D -->|退票| R[REFUND]
    D -->|改签| CH[CHANGE]
    R -->|退款完成| X
    CH -->|新票成功| D
```

---

## 演化与 ADR 摘要
- 阶段演进：核心闭环→削峰与弹性→一致性与成本优化→多活与全链路治理。
- 排队削峰：`min(2%*remain, 2000)` 控速，超长队列转候补。
- 库存一致性：Redis 预扣+账本化+对账，放弃跨库分布式事务。
- Redis 选型：Cluster（slot 分片+自动故障转移），单分片 <5 万 QPS，热点拆 key。
- 数据分片迁移：双写灰度+回放，不停服迁移，历史归档。
- MQ 顺序/幂等：分区键保证局部有序，`event_id/version` 幂等，DLQ+回溯。
- 候补策略：分段队列+优先级，锁票/支付超时回补。
- 容灾：同城双活优先，RPO≈0，RTO<30 分钟；异地容灾按冷备演练推进。

---

## 附录
- 容量建议：峰值 100 万 QPS 查询 / 20 万 TPS 写；Redis 分片 8~16（单分片 <5 万 QPS）；Kafka 分区≥核数（32~64 示例）；MySQL 分片 TPS 5000 假设。
- 最小部署（非生产）：单 Region、多 AZ；双节点 Redis 主从、双节点 MySQL 主从、三节点 Kafka、3 节点 ES、2 节点 Gateway/业务服务；开启限流与压测隔离，带健康检查与自动故障转移，定期演练切换。
