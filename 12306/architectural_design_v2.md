# 高并发火车票购票系统——概要设计文档
> 版本：v2.0（2025-12-26）  
> 适用范围：互联网/移动售票场景，读峰值≥100 万 QPS、写峰值（出队后下单）≥20 万 TPS（排队削峰后）  
> 文档类型：概要设计（高层架构、模型、关键策略）  
> 渲染说明：文档包含 mermaid 图表，推荐在支持 mermaid 的环境查看；不支持时可用 mermaid Live Editor 或文字版关系/时序/状态描述。

---

## 目录
1. 引言  
2. 任务概述  
3. 总体设计（处理流程、结构与功能分配）  
4. 接口设计（外部接口、内部接口）  
5. 数据结构设计（逻辑/物理、数据与程序关系）  
6. 运行设计（模块组合、运行控制、运行时间）  
7. 出错处理设计  
8. 安全保密设计  
9. 维护设计  
10. 模型与视图（对象、ER、时序、状态机）  
11. 演化与 ADR 摘要  
12. 附录  

---


## 一．引言
1. 编写目的：提供高并发火车票购票系统的高层架构、关键策略、模型与演化决策，指导后续详细设计与实现，读者包括架构/开发/测试/运维。强调与 SRS/测试计划的衔接，使设计可验证、可实施。  
2. 项目背景：春运/节假日高峰，对接铁路运行图/实名/支付/通知等外部系统；委托/开发/运维单位协同。突出对外依赖的 SLA 与容量约束，以及开售窗口的弹性与降级准备。  
3. 定义：

   | 术语/缩写 | 解释 |
   | --- | --- |
   | 区间段 Segment | 相邻站点的最小扣减单元。 |
   | 区间票 | 跨多个区间段的票，扣减只能全成全败。 |
   | 预扣库存 | 在缓存或分布式存储中的原子扣减，待支付后落库确认。 |
   | 锁座 | 占用具体席位，支付超时自动释放。 |
   | 幂等键 Idempotent Key | 请求唯一指纹，避免重复执行。 |
   | 排队 Token | 进入削峰排队后的查询凭据。 |
   | 订单状态机 | 订单、支付、出票状态的有限状态流转。 |
   | QPS/TPS | 每秒查询/事务数。 |
   | TTL | 生存时间。 |
   | AZ | 可用区。 |
   | DLQ | 死信队列。 |
   | SLO/SLA | 服务水平目标/协议。 |
 
4. 参考资料：
   | 标识 | 文档名称 | 编号 | 说明 |
   | --- | --- | --- | --- |
   | [GUIDE] | 《南京大学软件文档编写指南》 | - | 作为 SRS/概要设计的写作模板与评审检查清单依据。 |
   | [BOOK-01] | 《Site Reliability Engineering》 | - | 支撑 SLO/错误预算、过载保护与可观测性方法论。 |
   | [BOOK-02] | 《The Site Reliability Workbook》 | - | 支撑 SLO 落地、告警策略与演练/运行手册方法论。 |
   | [BOOK-03] | 《Designing Data-Intensive Applications》 | - | 支撑一致性、分片、事件流与补偿/幂等思路。 |
   | [DB] | MySQL 8.0 Reference Manual | - | 支撑事务/隔离级别、复制与分区等实现约束。 |
   | [SEARCH] | Elasticsearch 官方文档 | - | 支撑索引/查询与容量规划。 |
   | [CACHE] | Redis Cluster 官方文档 | - | 支撑集群一致性折衷、过期与原子操作。 |
   | [MQ-01] | Kafka 官方文档 | - | 支撑顺序、幂等生产与重试/DLQ 策略。 |
   | [MQ-02] | RabbitMQ 官方文档 | - | 支撑可靠投递、死信队列与消费语义。 |
   | [ISO-4217] | ISO 4217 | ISO 4217 | 货币编码与最小货币单位表达参考。 |
   | [ISO-8601] | ISO 8601 | ISO 8601:2019 | 时间格式与时区表示参考。 |
   | [RFC-3339] | RFC 3339 | RFC 3339 | 互联网时间戳格式参考（与 ISO 8601 协同）。 |
   | [RFC-9110] | HTTP Semantics | RFC 9110 | HTTP 语义与状态码参考。 |
   | [RFC-7231] | Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content | RFC 7231 | HTTP/1.1 语义与状态码参考（兼容性参考）。 |
   | [GB/T-35273] | 个人信息安全规范 | GB/T 35273—2020 | 个人信息处理、最小化、授权与保护要求参考。 |
   | [GB/T-31502] | 电子支付系统安全保护框架 | GB/T 31502—2015 | 支付安全域/鉴别/审计等框架性要求参考。 |
   | [GB/T-42015] | 网络支付服务数据安全要求 | GB/T 42015—2022 | 支付数据分类、保护与管理要求参考。 |
   | [GB/T-41460] | （按项目适用范围引用） | GB/T 41460—2022 | 支付相关安全/能力要求参考（按适用条款映射）。 |
   | [GB/T-41463] | （按项目适用范围引用） | GB/T 41463—2022 | 支付系统检测/验收相关要求参考（按适用条款映射）。 |
   | [GB/T-42582] | App 个人信息安全测评规范 | GB/T 42582—2023 | 若包含移动端 App：用于合规测评要求参考。 |
   | [GB/T-43578] | 通用密码服务接口规范 | GB/T 43578—2023 | 签名/验签、加解密接口与验证方法参考。 |
   | [GB/T-36964] | 软件工程成本度量规范 | GB/T 36964—2018 | 用于成本/工作量口径（如需要）。 |


---

## 二．任务概述
1. 目标：支撑高并发查询与下单，零超卖，满足可用性/一致性/性能目标（查询 P95<120ms/可用性 99.99%；写 P95<3s/可用性 99.95%），覆盖查询、下单/排队、支付/出票、退改/候补、通知、风控与运营。[^sre][^ddia] 目标拆分为链路级指标与错误预算，并在监控与告警中落地。  
2. 运行环境：云/数据中心 K8s，多 AZ；依赖 Redis/Kafka/MySQL/ES/对象存储等中间件。中间件具备高可用、扩缩容、监控告警与演练机制，部署拓扑支持同城多 AZ 容灾。  
3. 需求概述：参见 SRS 功能与非功能部分，概要设计侧重架构与策略实现路径。标注与 SRS 的差异或补充点，并明确接口契约、数据一致性与容量假设。  
4. 条件与限制：合规与外部依赖 SLA；容量/资源约束；发布与演练采用灰度与回滚预案；mermaid 渲染可退化为文字版。开售窗口保持变更冻结与容量保护，无法渲染 mermaid 时提供文字版步骤与依赖描述。  

---

## 三．总体设计
### 1．处理流程
- 核心流程：查询（缓存优先）→下单排队→库存预扣/锁座→支付→出票→通知→退改/候补补偿。详见时序图。各阶段暴露关键事件与指标（命中率、延迟、失败率），异常时提供重试或补偿路径与队列削峰策略。  

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
**图解**：\n- 客户端（Web/APP/运营）首先经 CDN/WAF 抗 DDoS 与静态分发，再进入 API Gateway 做鉴权、限流、灰度与路由。\n- 网关将请求分发到业务域：查询（缓存+ES/DB）、订单（排队/幂等/状态机）、库存（预扣）、座位（锁座）、支付（支付单/回调）、出票、退改候补、风控/配置、通知。\n- 中间件层：Redis 做缓存/分布式锁；Kafka/RabbitMQ 传递事件（订单、库存、通知等）；MySQL 分片存 OLTP；ES 提供搜索；OSS 存放报表/归档。\n- 外部集成：支付网关回调推动订单/出票；实名核验；运行图/座席同步供查询与预扣；短信/邮件推送通知用户。

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
**图解**：\n- 前端流量：CDN → LB → API Gateway，再进入 K8s 集群中的各微服务 Pod（查询、订单、库存、座位、支付、出票、退改候补、风控、通知）。\n- 数据/中间件：Redis Cluster（slot 分片+故障切换）、Kafka/RabbitMQ（多分区）、MySQL 分库分表（主从/Proxy）、ES 集群；对象存储/备份用于冷热归档。\n- 链路示意：各服务对 Redis/DB/ES 进行读写，对 Kafka 发送/消费事件（订单、库存、支付、通知）；备份/归档从数据库流向冷存储。\n- 可靠性：多 AZ 部署，中间件高可用，Pod 可横向扩展。

---

### 3．功能分配
- 接入层：鉴权、限流、灰度、路由、幂等键透传。负责统一安全与频控策略，支持按用户/地市/渠道灰度发布，并在回源失败时展示降级页面。  
- 查询域：站点/车次/余票/价目，缓存+ES+只读库。命中缓存优先，未命中读 ES/只读库，结合请求合并、防击穿与降级兜底。  
- 订单/库存/座位：排队、预扣、锁座、状态机、超时回收。排队限速避免写冲击；库存预扣与锁座保持幂等，超时自动回补，状态机单向推进。  
- 支付/出票：支付单、回调幂等、出票重试/补偿。支付回调落地签名与版本校验，出票失败触发退款或回补并入补偿队列。  
- 退改/候补：规则校验、双阶段改签、候补匹配与锁票。改签先锁新票成功再作废旧票；候补按优先级匹配区间库存并保障支付/锁票幂等。  
- 风控与运营：黑名单、频控、灰度策略；通知：短信/邮件/推送。风控提供策略下发与灰度开关，通知实现模板化、重试与渠道熔断。  
- 数据与中间件：Redis/Kafka/MySQL/ES/OSS 提供缓存、事件、存储、索引。明确分片、备份与容量阈值，监控缓存命中率、消息滞留与数据库延迟。  

---

## 四．接口设计
### 1．外部接口
- 运行图/座席：查询接口超时 500~1000ms，失败降级缓存；数据同步任务带重试与校验和。[^fault] 同步过程记录数据版本与来源，异常时回退上一版本，并在高峰前完成缓存预热。  
- 实名/证件：同步/异步校验，超时 3s，最多 3 次指数退避，必须验签。[^fault] 可用验证码或人工核验作为降级，并对失败次数实施频控与告警。  
- 支付平台：创建支付单、回调通知；超时 3s，回调验签+幂等，失败重试+DLQ。[^fault] 回调携带版本号与签名，落库失败进入重试队列并告警，退款与补偿路径保持闭环。  
- 短信/邮件/推送：模板化发送，超时 2s，失败重试+渠道熔断；备用通道切换。[^fault] 模板经过预审并落库，送达率与延迟纳入监控，渠道熔断后自动切换备用。  
- 风控服务：评分/校验接口，超时 1s，失败时走验证码/拒绝分支。[^fault] 风控策略支持灰度发布，命中风险时记录证据链并保留人工复核通道。  

### 2．内部接口
- 服务间通过 API Gateway（HTTP/REST）和 Kafka/RabbitMQ 事件通信；同订单/车次使用分区键保证局部有序；所有写接口携带 `idempotent_key`。[^mq] 接口声明契约版本与兼容策略，事件格式包含 `event_id/version/timestamp` 以便回放与幂等。  
- 调用超时与重试：核心写链路仅一次幂等重试或不重试；查询类可短重试；统一熔断策略。[^fault] 超时与重试参数按接口分级配置，失败进入告警与降级路径，并记录重试次数。  
- 接口安全：网关鉴权+签名校验；内部服务校验租户/渠道/幂等键。[^security] 权限模型细化为最小权限与租户隔离，敏感字段做脱敏并记录审计日志。  

---

## 五．数据结构设计
### 1．逻辑结构设计
- 缓存分层与热点治理：浏览器/APP 本地缓存→CDN/边缘→Gateway 本地缓存→Redis（5~20s TTL，二级本地缓存 200~500ms，弱一致）→只读库/ES；防击穿/雪崩：随机 TTL、请求合并、预热、兜底；版本戳/校验和纠偏。[^cache]
- 库存一致性/防超卖：区间库存 Redis Lua 原子预扣+版本号；写侧单线程/分片顺序消费 MQ 回写 MySQL 账本；定时对账与差值告警；支付超时/出票失败原路回补。[^dbmq]
- 幂等：入口 `idempotent_key`+`queue_token`（15 分钟），幂等记录在线 7 天、归档 30 天；状态只能前进。[^idemp]
- 支付回调：签名+幂等，指数退避重试（1/2/4/8... 最多 6~8 次），落库失败入 DLQ。[^fault]
- 排队与削峰：入口频控（账号/设备 3 QPS，突发桶 10），队列上限 5 万/车次/日，出队速率 `min(2%*remain, 2000)` 单/秒，超长转候补。[^queue]
- 分库分表：库按车次/日期哈希或区间，表按订单号雪花 ID 路由；跨分片读依赖 ES/OLAP；再平衡用双写/灰度切换和回放；历史归档策略。[^dbmq]
- MQ：按领域拆 topic（order/stock/payment/notify），同订单/车次分区键保证局部有序；消息含 `event_id/version`；延迟队列+DLQ 重试，支持 7 天回溯。[^mq]
- 候补：状态机（PENDING→MATCHING→LOCKED→AWAIT_PAY→SUCCESS/FAIL），分段队列与优先级（时间、会员、乘客数）；锁票/支付超时回补，库存/支付解耦。[^queue]
- 外部集成：实名/支付/短信等设定超时、最大重试（3 次指数退避）、熔断与降级返回码；SLA 明确，签名/验签算法契约化。[^fault]
- 安全与合规：实名、加密/脱敏、审计、风控联动；频控与灰度可配置（1%~20% 灰度）。[^security]
- 监控与告警：对账差值、补偿成功率、滞留订单、DLQ、支付回调重复率、排队 P95、缓存命中率、Redis/DB/消息 P95、外部超时/熔断。[^sre]
- 运行/批处理：运行图同步、库存对账、归档、告警巡检等批任务错峰调度，避免与开售高峰冲突。[^sre]

### 2．物理结构设计
- 分库分表：库按车次/日期哈希或区间，表按订单号雪花 ID 路由；跨分片读依赖 ES/OLAP；再平衡用双写/灰度切换和回放；历史归档策略。[^dbmq]
- 核心表举例：订单表（分库分表）、库存账本表（分库）、支付表、票号表、候补表、退款/改签表；索引围绕订单号、train_no+date、用户维度设计。
- 缓存/队列：Redis Cluster slot 分片+故障切换，单分片 <5 万 QPS；Kafka/RabbitMQ 多分区，关键链路局部有序；热点 key 拆分。[^dbmq][^mq]
- 存储：对象存储用于报表/归档；备份与冷热分层按附录容量建议；ES 用于搜索索引，冷热节点分层。[^dbmq]

### 3．数据结构与程序的关系
- 服务与数据域映射：Query→ES/Redis/只读库；Order/Inventory/Seat→Redis+MySQL 账本；Payment→支付单表+回调事件；Ticketing→票号表；After-sales→退款/改签/候补表；事件通过 MQ 串联状态机。各服务声明主写库/从库策略、缓存 TTL 与失效处理，事件消费记录偏移与幂等键，保障跨域一致性与可回溯。  

---

## 六．运行设计
### 1．运行模块的组合
- K8s 部署的无状态服务（查询/订单/库存/座位/支付/出票/退改候补/风控/通知），对应上文的 Pods 与中间件集群。无状态服务通过 ConfigMap/Secret 获取配置，支持滚动升级与自动扩缩容，中间件实例结合亲和与反亲和策略保障多 AZ。  

### 2．运行控制
- 排队与速率控制：`min(2%*remain, 2000)` 控制出队；队列上限 5 万/车次/日；频控与灰度可配置。速率按库存动态调整并支持人工降档，队列超限自动转候补，入口频控按用户/设备/IP 维度下发。  
- 超时控制：锁座/支付超时回收；外部接口超时+重试+熔断；幂等状态单向推进。各类超时与重试策略在配置中心统一管理，并在告警与看板中透出失败率与回收率。  
- 扩缩容：按 CPU/RT/队列长度触发弹性；开售前预热扩容；高峰后回缩。弹性策略与中间件容量联动，预热阶段加载热点数据与连接池。  
- 定时任务：运行图同步、库存对账、数据归档、告警巡检。定时任务错峰调度与限流，支持手动重跑与幂等，避免与开售高峰冲突。  

### 3．运行时间
- 性能目标：查询 P95<120ms；下单出队后 P95<3s；排队等待 P95<60s。运行窗口覆盖开售时段的弹性扩容与快速降级，在指标异常时自动启用保护策略。  
- 开售窗口：提前预热缓存与扩容；开售瞬时启用更严频控与队列限速；售罄后降级至候补。监控与客服联动预案同步推进，及时向用户提示候补或售罄状态，减少重复刷新。  

---

## 七．出错处理设计
1. 出错输出信息：对外错误码（如 400/401/403/409/423/504）与可诊断信息；内部告警含事件 ID、分区键、版本号。  
2. 出错处理对策：重试（指数退避）、幂等返回已有结果、降级（转候补/缓存兜底）、熔断/隔离外部依赖、DLQ+回溯重放；锁座/库存失败即时回补。  
3. 典型场景：  
   - 支付回调重复/乱序：以 `payment_id+version` 幂等处理，低版本丢弃。  
   - 出票失败：回滚库存+退款，记录补偿事件，通知用户。  
   - 热点查询击穿：启用本地缓存+请求合并，必要时返回降级数据。  
   - MQ 消费失败：多次重试后入 DLQ，人工/自动回溯重放。  

---

## 八．安全保密设计
- 身份与鉴权：依赖网关鉴权，服务内鉴权/签名校验。用户、渠道与租户身份通过 token 与签名双重校验，跨服务调用传递用户上下文并验证权限。  
- 数据安全：传输全链路 TLS；数据加密/脱敏（实名、证件、手机号、支付要素）；访问审计与留痕。存储侧采用列级加密与密钥轮换，日志/监控中的敏感字段掩码，备份数据加密并隔离权限。  
- 风控与频控：黑白名单、设备指纹（可选）、验证码、滑窗频控、灰度策略。风控规则支持配置与灰度下发，触发风险后提供验证码、人工复核与阻断路径。  
- 审计字段：记录用户、设备、IP、时间、操作、结果、幂等键、版本号；脱敏展示。审计日志集中存储与检索，满足合规留存周期并支持异常追踪。  

---

## 九．维护设计
- 可观测性：指标、日志、链路追踪；告警阈值参考关键策略。看板按域拆分（查询/下单/支付/库存），覆盖 P95/P99、错误率、队列长度、缓存命中率、DLQ、对账差值，并提供追踪样本。  
- 部署与变更：无状态服务滚动发布、灰度与回滚；健康检查与自动故障转移。发布具备自动化校验与回滚剧本，支持蓝绿/金丝雀，变更窗口配合审批与冻结策略。  
- 运行手册：压测与演练（排队/支付/回调/出票链路）、容量预测与再分片计划。手册包含日常巡检、常见故障处理、压测数据隔离、分片扩容步骤与演练清单。  


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
**图解**：\n- 车次域：TRAIN 对应多个 SEGMENTINVENTORY（按日期/区间/席别库存）和 SEATLOCK（锁座记录）。\n- 订单域：ORDER 关联 ORDERITEM（乘客/席别明细），一对一 PAYMENT（支付单）、一对多 TICKET（票号）、一对多 REFUNDCHANGE（退改）、一对多 STANDBYREQUEST（候补）。\n- 用户域：USER 与 PASSENGER 管理实名乘客，并与 ORDER 关联。\n- 关键字段：库存版本号（乐观锁）、金额（分）、状态/队列 token/幂等键，用于一致性与幂等控制。

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
**图解**：\n- 客户端提交下单，经网关透传幂等键/风控态到订单服务。\n- 订单服务调用库存预扣，成功后调用座位锁座（可选），返回 queue_token 给客户端。\n- 客户端轮询/推送查看排队进度；订单服务准备支付跳转。\n- 客户端完成支付，支付平台回调订单服务（幂等+验签），触发出票。\n- 出票服务出票成功/失败结果写回订单；通知服务发送短信/推送给用户。

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
**图解**：\n- 订单初始 WAIT_PAY，支付成功进入 PAID，出票中为 TICKETING。\n- 出票成功进入 SUCCESS，失败进入 FAIL 并触发退款回补；支付超时/取消进入 CLOSED。\n- SUCCESS 可走退票（REFUND）或改签（CHANGE），退票完成回到 CLOSED；改签成功返回 SUCCESS（新票）。\n- 所有状态单向推进，保障幂等与可重复执行。

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
