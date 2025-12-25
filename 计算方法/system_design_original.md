# 高并发火车票购票系统架构设计文档

## 版本历史

| 版本号 | 修订日期 | 修订人 | 修订原因 | 修订内容概述 |
|--------|----------|--------|----------|--------------|
| 1.0 | 2025-12-23 | Emma | 初始版本 | 创建系统架构设计文档 |
| 1.1 | 2025-12-24 | Emma | 规范优化 | 统一性能指标、规范UML图表、删除业务代码、添加组件图 |
| 1.2 | 2025-12-24 | Emma | 逻辑修正 | 统一订单锁定时间为15分钟、修正削峰链路数据、补充候补服务完整设计 |

---

## 1. 系统整体架构

### 1.1 架构设计原则

- **高可用性**: 系统可用性达到99.99%，年停机时间不超过52.56分钟
- **高并发**: 支持三种流量场景（日常10万QPS、高峰50万QPS、极限100万QPS）
- **高性能**: 查询响应时间<200ms，下单响应时间<3s
- **可扩展性**: 支持水平扩展，弹性伸缩
- **数据一致性**: 保证订单和库存的最终一致性,核心业务场景(下单、支付)保证事务一致性,一致性时间窗口<3秒

### 1.2 性能指标说明

系统需要支持三种不同的流量场景：

| 流量场景 | QPS | TPS | 日订单量 | 瞬时并发 | 说明 |
|---------|-----|-----|---------|---------|------|
| 日常流量 | 10万/秒 | 2万/秒 | 500万单 | 50万 | 工作日正常业务量 |
| 高峰流量 | 50万/秒 | 10万/秒 | 1000万单 | 100万 | 节假日、春运期间 |
| 极限流量 | 100万/秒 | 20万/秒 | 2000万单 | 200万 | 热门车次开售瞬间 |

**性能目标**：
- 查询服务：支持100万QPS
- 订单服务：支持20万TPS
- 平均响应时间：查询<200ms，下单<3s
- 系统可用性：99.99%

### 1.3 微服务架构划分

#### 1.3.1 组件图

```mermaid
graph TB
    subgraph "客户端层"
        Client1[Web浏览器]
        Client2[移动App]
        Client3[小程序]
    end
    
    subgraph "接入层"
        CDN[CDN内容分发]
        LB[负载均衡器]
        Gateway[API网关]
    end
    
    subgraph "应用服务层"
        UserSvc[用户服务]
        SearchSvc[查询服务]
        OrderSvc[订单服务]
        PaymentSvc[支付服务]
        NotifySvc[通知服务]
        QueueSvc[排队服务]
        WaitlistSvc[候补服务]
    end
    
    subgraph "中间件层"
        Redis[Redis集群]
        Kafka[Kafka消息队列]
        ES[Elasticsearch]
    end
    
    subgraph "数据层"
        MySQL[MySQL集群]
        MongoDB[MongoDB]
        OSS[对象存储]
    end
    
    subgraph "外部系统"
        PayPlatform[支付平台]
        IDVerify[实名认证]
        Railway[铁路运营系统]
    end
    
    Client1 --> CDN
    Client2 --> CDN
    Client3 --> CDN
    CDN --> LB
    LB --> Gateway
    
    Gateway --> UserSvc
    Gateway --> SearchSvc
    Gateway --> OrderSvc
    Gateway --> PaymentSvc
    Gateway --> NotifySvc
    Gateway --> QueueSvc
    Gateway --> WaitlistSvc
    
    UserSvc --> Redis
    SearchSvc --> Redis
    OrderSvc --> Redis
    WaitlistSvc --> Redis
    
    UserSvc --> MySQL
    SearchSvc --> MySQL
    OrderSvc --> MySQL
    WaitlistSvc --> MySQL
    
    OrderSvc --> Kafka
    NotifySvc --> Kafka
    WaitlistSvc --> Kafka
    
    SearchSvc --> ES
    
    PaymentSvc --> PayPlatform
    UserSvc --> IDVerify
    SearchSvc --> Railway
```

#### 1.3.2 核心服务模块

**1. 用户服务 (User Service)**
- 用户注册、登录、认证
- 用户信息管理
- 常用联系人管理
- 用户权限管理

**2. 车票查询服务 (Search Service)**
- 车次查询
- 余票查询
- 站点查询
- 价格查询
- 热点数据缓存

**3. 订单服务 (Order Service)**
- 订单创建
- 订单查询
- 订单状态管理
- 订单超时处理

**4. 库存服务 (Inventory Service)**
- 库存管理
- 库存扣减
- 库存回退
- 座位分配

**5. 支付服务 (Payment Service)**
- 支付接口集成
- 支付状态查询
- 退款处理
- 对账管理

**6. 排队服务 (Queue Service)**
- 虚拟排队
- 流量控制
- 令牌分发
- 排队状态查询

**7. 通知服务 (Notification Service)**
- 短信通知
- 邮件通知
- 站内消息
- 推送通知

**8. 风控服务 (Risk Service)**
- 防刷票检测
- 异常行为识别
- 黑名单管理
- 验证码服务

**9. 车票服务 (Ticket Service)**
- 车票信息管理
- 车次管理
- 站点管理
- 票价管理

**10. 候补购票服务 (Waitlist Service)** - 独立微服务
- 候补订单管理
- 候补队列处理
- 库存回补监听
- 自动购票处理
- 候补订单过期处理

---

## 2. 技术选型

### 2.1 编程语言与框架

| 组件类型 | 技术选型 | 说明 |
|---------|---------|------|
| 后端语言 | Java 17 | 成熟稳定，生态丰富 |
| 微服务框架 | Spring Cloud Alibaba | 国内最佳实践，集成Nacos、Sentinel等 |
| RPC框架 | Dubbo 3.x | 高性能RPC，支持多协议 |
| Web框架 | Spring Boot 3.x | 快速开发，约定优于配置 |
| 前端框架 | React 18 + TypeScript | 组件化开发，类型安全 |

### 2.2 中间件选型

| 中间件类型 | 技术选型 | 说明 |
|-----------|---------|------|
| 注册中心 | Nacos 2.x | 服务发现、配置管理 |
| 配置中心 | Nacos Config | 动态配置管理 |
| 网关 | Spring Cloud Gateway | 统一入口，路由转发 |
| 负载均衡 | Nginx + LVS | 四层+七层负载均衡 |
| 缓存 | Redis Cluster | 分布式缓存 |
| 消息队列 | Kafka + RabbitMQ | Kafka削峰填谷，RabbitMQ业务解耦 |
| 搜索引擎 | Elasticsearch | 车次搜索、日志分析 |
| 限流熔断 | Sentinel | 流量控制、熔断降级 |
| 分布式事务 | Seata | TCC、AT模式 |
| 任务调度 | XXL-Job | 分布式任务调度 |
| 链路追踪 | SkyWalking | 全链路监控 |
| 监控告警 | Prometheus + Grafana | 指标监控、可视化 |

### 2.3 数据存储选型

| 存储类型 | 技术选型 | 说明 |
|---------|---------|------|
| 关系数据库 | MySQL 8.0 | 主数据存储 |
| 缓存数据库 | Redis 7.x | 热点数据缓存 |
| 文档数据库 | MongoDB | 日志、会话存储 |
| 时序数据库 | InfluxDB | 监控指标存储 |
| 对象存储 | MinIO/OSS | 文件存储 |

---

## 3. 数据库设计

### 3.1 分库分表策略

#### 3.1.1 分库策略

**用户库分库**: 按用户ID取模分16库
```
database_index = user_id % 16
database_name = user_db_{database_index}
```

**订单库分库**: 按订单ID取模分32库
```
database_index = order_id % 32
database_name = order_db_{database_index}
```

**车票库**: 按车次号哈希分8库
```
database_index = hash(train_number) % 8
database_name = ticket_db_{database_index}
```

#### 3.1.2 分表策略

**订单表分表**: 每库按订单ID取模分64表

```python
# 分库分表路由算法
database_index = order_id % 32
table_index = order_id % 64
database_name = f"order_db_{database_index}"
table_name = f"t_order_{table_index}"

# 示例
order_id = 123456789
database_index = 123456789 % 32  # = 21
table_index = 123456789 % 64     # = 53
# 最终路由到: order_db_21.t_order_53
```

**说明**:
- 总表数: 32库 × 64表 = 2048张订单表
- 分表数量从256降低到64,减少运维复杂度
- 优点: 数据分布均匀,查询性能稳定,支持按订单ID快速定位
- 缺点: 无法按时间范围查询,无法直接归档历史数据

**历史数据归档方案**:
- 使用定时任务(每月1日凌晨)将1年前的订单迁移到历史库
- 历史库不分库,按月分表: `t_order_history_202501`, `t_order_history_202502`
- 归档流程: 扫描所有分库分表 → 筛选1年前订单 → 迁移到历史库 → 删除原数据
- 用户查询历史订单时,先查当前库,未找到则查历史库

**用户表分表**: 每库按用户ID取模分64表
```python
database_index = user_id % 16
table_index = user_id % 64
database_name = f"user_db_{database_index}"
table_name = f"t_user_{table_index}"
```

#### 3.1.3 核心表结构

**用户表 (t_user)**
```sql
CREATE TABLE t_user_{N} (
    id BIGINT PRIMARY KEY COMMENT '用户ID',
    username VARCHAR(50) UNIQUE NOT NULL COMMENT '用户名',
    password VARCHAR(128) NOT NULL COMMENT '密码(加密)',
    real_name VARCHAR(50) COMMENT '真实姓名',
    id_card VARCHAR(18) COMMENT '身份证号',
    phone VARCHAR(11) UNIQUE COMMENT '手机号',
    email VARCHAR(100) COMMENT '邮箱',
    user_type TINYINT DEFAULT 0 COMMENT '用户类型:0普通,1VIP',
    status TINYINT DEFAULT 1 COMMENT '状态:0禁用,1正常',
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    version INT DEFAULT 0 COMMENT '乐观锁版本号',
    INDEX idx_phone (phone),
    INDEX idx_id_card (id_card)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户表';
```

**说明**:
- **身份证号字段**: 移除UNIQUE约束,允许一个身份证号关联多个账号(最多5个)
- **业务规则**: 在应用层控制同一身份证号最多关联5个账号的限制,与需求分析文档FR-UM-003保持一致
- **查询优化**: 保留idx_id_card索引,支持按身份证号快速查询
- **购票限制**: 
  - 同一用户同一车次同一天最多购买5张票(与需求分析UC-004保持一致)
  - 同一身份证号一天最多购买10张票(与需求分析UC-004保持一致)

**订单主表 (t_order)**
```sql
CREATE TABLE t_order_{N} (
    id BIGINT PRIMARY KEY COMMENT '订单ID',
    order_no VARCHAR(32) UNIQUE NOT NULL COMMENT '订单号',
    user_id BIGINT NOT NULL COMMENT '用户ID',
    train_number VARCHAR(20) NOT NULL COMMENT '车次号',
    departure_station VARCHAR(50) NOT NULL COMMENT '出发站',
    arrival_station VARCHAR(50) NOT NULL COMMENT '到达站',
    departure_time DATETIME NOT NULL COMMENT '出发时间',
    arrival_time DATETIME NOT NULL COMMENT '到达时间',
    seat_type TINYINT NOT NULL COMMENT '座位类型',
    passenger_count TINYINT NOT NULL COMMENT '乘客数量',
    total_price DECIMAL(10,2) NOT NULL COMMENT '订单总金额',
    order_type TINYINT DEFAULT 0 COMMENT '订单类型:0正常订单,1候补订单',
    original_order_id BIGINT COMMENT '原订单ID(改签时使用)',
    is_rescheduled TINYINT DEFAULT 0 COMMENT '是否改签:0否,1是',
    rescheduled_from VARCHAR(32) COMMENT '改签前的订单号',
    rescheduled_fee DECIMAL(10,2) COMMENT '改签手续费',
    order_status TINYINT DEFAULT 0 COMMENT '订单状态:0待支付,1已支付,2已出票,3已完成,4已取消,5已退票',
    pay_time DATETIME COMMENT '支付时间',
    expire_time DATETIME COMMENT '过期时间(订单创建后15分钟)',
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    version INT DEFAULT 0 COMMENT '乐观锁版本号',
    INDEX idx_user_id (user_id),
    INDEX idx_order_no (order_no),
    INDEX idx_train_number (train_number),
    INDEX idx_create_time (create_time),
    INDEX idx_original_order (original_order_id),
    INDEX idx_order_type (order_type, order_status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='订单主表';
```

**说明**:
- **订单锁定时间**: 统一为15分钟,与需求分析文档保持一致
- **多人购票支持**: 通过passenger_count字段记录乘客数量,详细信息存储在订单明细表中
- **一对多关系**: 一个订单对应多个乘客(通过订单明细表关联)

**订单明细表 (t_order_detail)**
```sql
CREATE TABLE t_order_detail_{N} (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_id BIGINT NOT NULL COMMENT '订单ID',
    order_no VARCHAR(32) NOT NULL COMMENT '订单号',
    passenger_name VARCHAR(50) NOT NULL COMMENT '乘客姓名',
    passenger_id_card VARCHAR(18) NOT NULL COMMENT '乘客身份证',
    passenger_mobile VARCHAR(11) COMMENT '乘客手机号',
    passenger_type TINYINT DEFAULT 0 COMMENT '乘客类型:0成人,1儿童,2学生',
    seat_number VARCHAR(10) COMMENT '座位号',
    ticket_price DECIMAL(10,2) NOT NULL COMMENT '票价',
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_order_id (order_id),
    INDEX idx_order_no (order_no),
    INDEX idx_passenger_id_card (passenger_id_card)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='订单明细表';
```

**说明**:
- **订单主表**: 存储订单的基本信息,一个订单对应一次购票行为
- **订单明细表**: 存储每个乘客的详细信息,一个订单可以包含多个乘客(最多5人)
- **关系**: 订单主表与订单明细表是一对多关系
- **分表策略**: 订单明细表与订单主表使用相同的分库分表规则,根据order_id路由

**库存表 (t_inventory)**
```sql
CREATE TABLE t_inventory (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    train_number VARCHAR(20) NOT NULL COMMENT '车次号',
    train_date DATE NOT NULL COMMENT '发车日期',
    departure_station VARCHAR(50) NOT NULL COMMENT '出发站',
    arrival_station VARCHAR(50) NOT NULL COMMENT '到达站',
    seat_type TINYINT NOT NULL COMMENT '座位类型',
    total_seats INT NOT NULL COMMENT '总座位数',
    available_seats INT NOT NULL COMMENT '可用座位数',
    locked_seats INT DEFAULT 0 COMMENT '锁定座位数',
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    version INT DEFAULT 0 COMMENT '乐观锁版本号',
    UNIQUE KEY uk_train_route_seat (train_number, train_date, departure_station, arrival_station, seat_type),
    INDEX idx_train_date (train_number, train_date)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='库存表';
```

**车次表 (t_train)**
```sql
CREATE TABLE t_train (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    train_number VARCHAR(20) UNIQUE NOT NULL COMMENT '车次号',
    train_type TINYINT NOT NULL COMMENT '列车类型:1高铁,2动车,3普快',
    departure_station VARCHAR(50) NOT NULL COMMENT '始发站',
    arrival_station VARCHAR(50) NOT NULL COMMENT '终点站',
    departure_time TIME NOT NULL COMMENT '发车时间',
    arrival_time TIME NOT NULL COMMENT '到达时间',
    duration INT NOT NULL COMMENT '运行时长(分钟)',
    status TINYINT DEFAULT 1 COMMENT '状态:0停运,1正常',
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_train_number (train_number),
    INDEX idx_departure_station (departure_station),
    INDEX idx_arrival_station (arrival_station)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='车次表';
```

### 3.2 读写分离方案

#### 3.2.1 主从架构
- **主库**: 1主，负责写操作
- **从库**: 2-4从，负责读操作
- **同步方式**: 半同步复制，保证数据一致性

#### 3.2.2 读写分离中间件
使用 **ShardingSphere-JDBC** 实现读写分离和分库分表

```yaml
spring:
  shardingsphere:
    datasource:
      names: master,slave0,slave1
      master:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://master:3306/ticket_db
      slave0:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://slave0:3306/ticket_db
      slave1:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://slave1:3306/ticket_db
    rules:
      readwrite-splitting:
        data-sources:
          ticket_ds:
            type: Static
            props:
              write-data-source-name: master
              read-data-source-names: slave0,slave1
              load-balancer-name: round_robin
```

---

## 4. 缓存架构

### 4.1 Redis集群设计

#### 4.1.1 集群模式
采用 **Redis Cluster** 模式，16384个槽位分布在6个节点(3主3从)

```
Master1 (0-5460)  ←→  Slave1
Master2 (5461-10922) ←→  Slave2
Master3 (10923-16383) ←→  Slave3
```

#### 4.1.2 缓存分层策略

**L1缓存 - 本地缓存 (Caffeine)**
- 缓存车次基础信息
- TTL: 5分钟
- 容量: 10000条

**L2缓存 - Redis缓存**
- 缓存余票信息、订单信息
- TTL: 根据业务场景设置

**L3缓存 - 数据库**
- 持久化存储

### 4.2 缓存策略

#### 4.2.1 热点数据缓存

**余票信息缓存**
```
Key: ticket:inventory:{train_number}:{train_date}:{departure}:{arrival}:{seat_type}
Value: {
    "total_seats": 1000,
    "available_seats": 856,
    "locked_seats": 144
}
TTL: 60s (1分钟)
说明: 余票信息实时性要求高,缓存时间短,保证数据准确性
```

**车次信息缓存**
```
Key: train:info:{train_number}
Value: {JSON格式车次详情}
TTL: 3600s (1小时)
说明: 车次基础信息变化频率低,可以缓存较长时间
```

**用户会话缓存**
```
Key: session:{session_id}
Value: {用户信息}
TTL: 1800s (30分钟)
说明: 会话过期时间与需求分析文档保持一致
```

**排队令牌缓存**
```
Key: queue:token:{user_id}
Value: {token_id, position, expire_time}
TTL: 900s (15分钟)
说明: 排队令牌有效期与订单锁定时间保持一致
```

#### 4.2.2 缓存更新策略

**Cache Aside Pattern (旁路缓存)**
- 读操作: 先查缓存，缓存未命中则查数据库并写入缓存
- 写操作: 先更新数据库，再删除缓存

**预扣库存场景的缓存更新流程**：
1. Redis原子扣减库存
2. 检查扣减结果，如果小于0则回滚
3. 发送订单消息到MQ
4. 异步更新数据库（由消费者处理）

### 4.3 缓存一致性保证

#### 4.3.1 延迟双删策略
1. 删除缓存
2. 更新数据库
3. 延迟删除缓存(500ms后)

#### 4.3.2 基于Canal的数据同步
- 监听MySQL binlog
- 实时同步数据到Redis
- 保证最终一致性

---

## 5. 消息队列设计

### 5.1 Kafka架构

#### 5.1.1 集群配置
- **Broker**: 3个节点
- **Partition**: 每个Topic 16个分区
- **Replication**: 副本因子为2

#### 5.1.2 Topic设计

| Topic名称 | 分区数 | 用途 | 消费者组 |
|----------|-------|------|---------| 
| order-create-topic | 16 | 订单创建事件 | order-consumer-group |
| order-pay-topic | 16 | 订单支付事件 | payment-consumer-group |
| order-cancel-topic | 8 | 订单取消事件 | cancel-consumer-group |
| inventory-update-topic | 16 | 库存更新事件 | inventory-consumer-group |
| notification-topic | 8 | 通知事件 | notification-consumer-group |

### 5.2 RabbitMQ架构

#### 5.2.1 交换机设计

**订单延迟队列**
```
Exchange: order.delay.exchange (x-delayed-message)
Queue: order.delay.queue
Routing Key: order.delay
用途: 订单超时自动取消
```

**死信队列**
```
Exchange: order.dlx.exchange
Queue: order.dlx.queue
用途: 处理失败消息
```

### 5.3 削峰填谷方案

#### 5.3.1 流量削峰
```
用户请求 → API网关限流 → 排队系统 → 消息队列 → 订单服务集群
 (100万/s)    (50万/s)      (10万/s)     (缓冲)    (5万/s)
                                                   ↓
                                            数据库TPS: 2万/s
```

**说明**：
- 订单服务单实例处理能力: 1000 TPS
- 需要部署50个实例才能达到5万TPS
- 通过Kubernetes HPA自动扩缩容
- 消息队列作为缓冲层，削峰填谷
- 最终写入数据库的TPS控制在2万/秒，保护数据库

#### 5.3.2 异步处理订单流程

**流程说明**：
1. 接收订单请求，快速返回
   - 参数校验
   - 预扣库存
   - 发送订单消息到Kafka
   - 返回"订单提交成功"

2. 异步消费订单消息
   - 创建订单
   - 设置订单过期时间(15分钟)
   - 发送延迟消息到RabbitMQ

3. 处理订单超时
   - 监听延迟队列
   - 检查订单状态
   - 如果未支付则取消订单并回退库存

---

## 6. 高并发解决方案

### 6.1 负载均衡架构

#### 6.1.1 四层负载均衡 (LVS)
```
                    Internet
                       │
                   ┌───▼───┐
                   │  DNS  │
                   └───┬───┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
    ┌───▼───┐      ┌───▼───┐      ┌───▼───┐
    │ LVS-1 │      │ LVS-2 │      │ LVS-3 │
    └───┬───┘      └───┬───┘      └───┬───┘
        │              │              │
        └──────────────┼──────────────┘
                       │
                  (Keepalived)
```

**LVS配置**
- 模式: DR模式(Direct Routing)
- 调度算法: 加权最小连接(WLC)
- 会话保持: 源地址哈希

#### 6.1.2 七层负载均衡 (Nginx)
```nginx
upstream api_servers {
    least_conn;  # 最小连接数算法
    server 192.168.1.10:8080 weight=3 max_fails=3 fail_timeout=30s;
    server 192.168.1.11:8080 weight=3 max_fails=3 fail_timeout=30s;
    server 192.168.1.12:8080 weight=2 max_fails=3 fail_timeout=30s;
    keepalive 32;  # 保持连接
}

server {
    listen 80;
    server_name api.ticket.com;
    
    location /api/ {
        proxy_pass http://api_servers;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # 超时设置
        proxy_connect_timeout 5s;
        proxy_send_timeout 10s;
        proxy_read_timeout 10s;
    }
}
```

### 6.2 限流策略

#### 6.2.1 网关层限流配置示例

**Sentinel流控规则配置**：
- 查询接口限流: 100,000 QPS
- 下单接口限流: 50,000 QPS，预热模式，预热时长10秒
- 支付接口限流: 30,000 QPS

#### 6.2.2 用户维度限流算法

**令牌桶算法流程**：
1. 获取当前令牌数和上次更新时间
2. 计算时间差，按速率生成新令牌
3. 令牌数不超过桶容量
4. 判断是否有足够令牌
5. 有则扣减令牌并返回成功，否则返回失败

### 6.3 降级方案

#### 6.3.1 服务降级策略

**降级处理方式**：
- 限流降级：返回"系统繁忙，请稍后重试"
- 异常降级：记录错误日志，返回"系统异常，请稍后重试"

#### 6.3.2 降级优先级
1. **非核心功能降级**: 推荐车次、历史订单查询
2. **读服务降级**: 返回缓存数据或默认数据
3. **写服务降级**: 记录日志，异步补偿
4. **核心服务保护**: 保证查询和下单功能

### 6.4 熔断机制

#### 6.4.1 熔断配置说明

**慢调用比例熔断**：
- 资源: paymentService
- 响应时间阈值: 1000ms
- 熔断时长: 10秒
- 最小请求数: 5
- 慢调用比例: 50%

**异常比例熔断**：
- 资源: inventoryService
- 异常比例: 50%
- 熔断时长: 10秒
- 最小请求数: 5

#### 6.4.2 熔断状态机
```
关闭状态 (Closed) → 打开状态 (Open) → 半开状态 (Half-Open) → 关闭状态
    │                      │                    │
    │ 异常率>阈值           │ 熔断时长到期        │ 探测成功
    └──────────────────────┴────────────────────┘
```

---

## 7. 分布式事务处理

### 7.1 TCC模式 (订单+库存)

#### 7.1.1 Try阶段 - 资源预留

**流程说明**：
1. 检查库存是否充足
2. 预留库存（锁定座位）
   - 可用座位数减少
   - 锁定座位数增加
3. 记录事务日志，状态为TRY

#### 7.1.2 Confirm阶段 - 确认提交

**流程说明**：
1. 确认扣减库存
   - 锁定座位数减少
2. 更新事务状态为COMMIT

#### 7.1.3 Cancel阶段 - 回滚

**流程说明**：
1. 释放锁定库存
   - 可用座位数增加
   - 锁定座位数减少
2. 更新事务状态为ROLLBACK

### 7.2 SAGA模式 (订单+支付+通知)

#### 7.2.1 正向流程
```json
{
  "sagaDefinition": {
    "name": "orderSaga",
    "steps": [
      {
        "name": "createOrder",
        "service": "orderService",
        "method": "create",
        "compensate": "cancelOrder"
      },
      {
        "name": "deductInventory",
        "service": "inventoryService",
        "method": "deduct",
        "compensate": "addInventory"
      },
      {
        "name": "processPayment",
        "service": "paymentService",
        "method": "pay",
        "compensate": "refund"
      },
      {
        "name": "sendNotification",
        "service": "notificationService",
        "method": "send",
        "compensate": "none"
      }
    ]
  }
}
```

#### 7.2.2 补偿流程

**SAGA执行流程**：
1. 按顺序执行各个步骤
2. 如果某步骤失败，自动执行补偿流程
3. 补偿流程按相反顺序执行各步骤的补偿方法

### 7.3 本地消息表 (最终一致性)

#### 7.3.1 本地消息表设计
```sql
CREATE TABLE t_local_message (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    message_id VARCHAR(64) UNIQUE NOT NULL COMMENT '消息ID',
    topic VARCHAR(100) NOT NULL COMMENT '消息主题',
    content TEXT NOT NULL COMMENT '消息内容',
    status TINYINT DEFAULT 0 COMMENT '状态:0待发送,1已发送,2发送失败',
    retry_count INT DEFAULT 0 COMMENT '重试次数',
    max_retry INT DEFAULT 3 COMMENT '最大重试次数',
    next_retry_time DATETIME COMMENT '下次重试时间',
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_status_retry (status, next_retry_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='本地消息表';
```

#### 7.3.2 实现流程

**事务处理流程**：
1. 在同一数据库事务中：
   - 创建订单
   - 插入本地消息表
2. 定时任务扫描本地消息表
3. 发送消息到Kafka
4. 更新消息状态
5. 失败时更新重试信息

---

## 8. 候补购票服务详细设计

### 8.1 架构决策 (ADR)

**为什么候补购票是独立服务?**

我们评估了三种方案：
- **方案A: 独立候补服务** ✅ (采用)
- 方案B: 挂在订单服务
- 方案C: 挂在排队服务

**选择方案A的原因：**

1. **业务逻辑复杂度**: 候补涉及长期监控、定时任务、事件处理，与订单服务的实时性要求不同
2. **性能隔离**: 候补处理不应影响核心订单服务的性能
3. **独立扩缩容**: 春运期间候补量激增，需要独立扩容
4. **职责单一**: 符合微服务的单一职责原则

### 8.2 核心数据结构

#### 8.2.1 候补订单表

**候补订单表 (t_waitlist_order)**
```sql
CREATE TABLE t_waitlist_order (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    waitlist_no VARCHAR(32) UNIQUE NOT NULL COMMENT '候补订单号',
    user_id BIGINT NOT NULL COMMENT '用户ID',
    train_number VARCHAR(20) NOT NULL COMMENT '车次号',
    train_date DATE NOT NULL COMMENT '乘车日期',
    departure_station VARCHAR(50) NOT NULL COMMENT '出发站',
    arrival_station VARCHAR(50) NOT NULL COMMENT '到达站',
    seat_type TINYINT NOT NULL COMMENT '座位类型',
    passenger_name VARCHAR(50) NOT NULL COMMENT '乘客姓名',
    passenger_id_card VARCHAR(18) NOT NULL COMMENT '乘客身份证',
    ticket_price DECIMAL(10,2) NOT NULL COMMENT '票价',
    status TINYINT DEFAULT 0 COMMENT '状态:0待候补,1已成功,2已失败,3已取消',
    submit_time DATETIME NOT NULL COMMENT '提交时间',
    expire_time DATETIME NOT NULL COMMENT '过期时间(7天后)',
    success_time DATETIME COMMENT '候补成功时间',
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_user_status (user_id, status),
    INDEX idx_train_date (train_number, train_date, status),
    INDEX idx_expire_time (expire_time, status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='候补订单表';
```

#### 8.2.2 候补队列 (Redis)

**候补队列 (Redis Sorted Set)**
```
Key: waitlist:queue:{train_number}:{train_date}:{seat_type}
Score: 提交时间戳(毫秒)
Value: 候补订单号
示例: waitlist:queue:G123:2025-01-20:2
```

**说明**：
- 使用Sorted Set保证按提交时间排序
- Score为提交时间戳，确保先到先得
- 每个车次+日期+座位类型维护一个独立队列

### 8.3 业务流程

#### 8.3.1 提交候补流程

```
用户提交候补请求
  ↓
验证用户候补资格(最多2个待候补订单)
  ↓
创建候补订单(状态:待候补)
  ↓
加入Redis候补队列(Sorted Set,按提交时间排序)
  ↓
调用支付服务预付款
  ↓
返回候补订单号
```

#### 8.3.2 库存回补触发流程

```
用户取消订单/退票
  ↓
订单服务更新订单状态
  ↓
库存服务释放座位
  ↓
发送库存回补事件到Kafka(topic: inventory-replenish)
  ↓
候补服务监听Kafka事件
```

#### 8.3.3 自动购票流程

```
候补服务接收到库存回补事件
  ↓
从Redis队列取出第一个候补订单(ZRANGE score最小)
  ↓
检查候补订单是否有效(未过期、未取消)
  ↓
调用订单服务创建正式订单
  ↓
订单创建成功 → 更新候补订单状态为"已成功"
  ↓
调用支付服务自动扣款
  ↓
发送候补成功通知(短信+推送)
  ↓
从Redis队列移除该候补订单
```

#### 8.3.4 候补失败处理

```
定时任务扫描过期候补订单(expire_time < now)
  ↓
更新候补订单状态为"已失败"
  ↓
调用支付服务退款
  ↓
发送候补失败通知
  ↓
从Redis队列移除该候补订单
```

### 8.4 与其他服务的交互

#### 8.4.1 服务交互时序图

```mermaid
sequenceDiagram
    participant U as 用户
    participant W as 候补服务
    participant O as 订单服务
    participant I as 库存服务
    participant P as 支付服务
    participant K as Kafka
    participant N as 通知服务
    
    U->>W: 提交候补请求
    W->>W: 验证资格
    W->>W: 创建候补订单
    W->>P: 预付款
    P-->>W: 预付成功
    W-->>U: 返回候补订单号
    
    Note over O,I: 其他用户退票
    O->>I: 释放库存
    I->>K: 发送库存回补事件
    K->>W: 候补服务监听事件
    W->>W: 从队列取出候补订单
    W->>O: 创建正式订单
    O-->>W: 订单创建成功
    W->>P: 自动扣款
    P-->>W: 扣款成功
    W->>N: 发送成功通知
    N->>U: 短信+推送通知
```

### 8.5 关键技术点

#### 8.5.1 公平性保证
- 使用Redis Sorted Set，按提交时间戳排序
- 确保先到先得的公平原则
- 防止插队和作弊行为

#### 8.5.2 实时性保证
- 使用Kafka事件驱动架构
- 库存释放后立即触发候补处理
- 异步处理不阻塞主流程

#### 8.5.3 并发控制
- 使用Redis分布式锁
- 防止同一座位被多个候补订单抢占
- 保证库存扣减的原子性

#### 8.5.4 高可用设计
- 候补服务部署多个实例
- Kafka消费者组保证消息不丢失
- 失败重试机制

#### 8.5.5 过期处理
- 定时任务(每小时)扫描过期候补订单
- 自动退款并通知用户
- 清理Redis队列中的过期数据

### 8.6 Kafka Topic设计

**库存回补事件Topic**
```
Topic名称: inventory-replenish-topic
分区数: 16
副本因子: 2
消息格式: {
    "train_number": "G123",
    "train_date": "2025-01-20",
    "seat_type": 2,
    "replenish_count": 1,
    "timestamp": 1705737600000
}
```

---

## 9. 安全设计

### 9.1 防刷票机制

#### 9.1.1 滑动验证码

**验证码生成流程**：
1. 生成背景图和滑块图
2. 生成随机偏移量
3. 存储验证信息到Redis（TTL 2分钟）
4. 返回验证码图片

**验证码校验流程**：
1. 获取存储的验证信息
2. 比较用户滑动位置与实际偏移量
3. 允许5像素误差
4. 验证后删除缓存

#### 9.1.2 行为分析

**异常行为检测**：
1. 统计1分钟内的请求次数
2. 超过30次/分钟判定为异常
3. 检查操作时间间隔
4. 间隔小于100ms可能是机器人
5. 异常用户加入黑名单

### 9.2 防黄牛策略

#### 9.2.1 实名制验证

**验证流程**：
1. 校验身份证格式
2. 调用公安部接口验证
3. 缓存验证结果（24小时）

#### 9.2.2 购票限制

**限制规则**：
1. 同一用户同一车次同一天最多购买5张票
2. 同一用户同一身份证号一天最多购买10张票
3. 使用Redis计数器实现限制

### 9.3 排队系统

#### 9.3.1 虚拟排队设计

**排队流程**：
1. 生成排队令牌
2. 加入Redis有序集合（按时间戳排序）
3. 存储令牌信息（TTL 15分钟）
4. 返回排队位置和预计等待时间

**排队权限检查**：
- 前1000名可以进入购票
- 其他用户继续排队等待

#### 9.3.2 令牌桶限流

**限流参数**：
- 桶容量: 10000
- 令牌生成速率: 1000个/秒

**限流算法**：
使用Lua脚本保证原子性，实现令牌桶算法

---

## 10. 部署架构

### 10.1 容器化部署

#### 10.1.1 Docker镜像构建
```dockerfile
# Dockerfile
FROM openjdk:17-jdk-alpine
LABEL maintainer="ticket-system"

# 设置工作目录
WORKDIR /app

# 复制jar包
COPY target/ticket-service.jar app.jar

# 设置JVM参数
ENV JAVA_OPTS="-Xms2g -Xmx4g -XX:+UseG1GC -XX:MaxGCPauseMillis=200"

# 暴露端口
EXPOSE 8080

# 健康检查
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
    CMD wget --quiet --tries=1 --spider http://localhost:8080/actuator/health || exit 1

# 启动应用
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

#### 10.1.2 Docker Compose编排
```yaml
version: '3.8'

services:
  # MySQL主库
  mysql-master:
    image: mysql:8.0
    container_name: mysql-master
    environment:
      MYSQL_ROOT_PASSWORD: root123
      MYSQL_DATABASE: ticket_db
    volumes:
      - mysql-master-data:/var/lib/mysql
      - ./mysql/master.cnf:/etc/mysql/conf.d/master.cnf
    ports:
      - "3306:3306"
    networks:
      - ticket-network

  # MySQL从库
  mysql-slave:
    image: mysql:8.0
    container_name: mysql-slave
    environment:
      MYSQL_ROOT_PASSWORD: root123
    volumes:
      - mysql-slave-data:/var/lib/mysql
      - ./mysql/slave.cnf:/etc/mysql/conf.d/slave.cnf
    ports:
      - "3307:3306"
    depends_on:
      - mysql-master
    networks:
      - ticket-network

  # Redis集群
  redis-node1:
    image: redis:7-alpine
    container_name: redis-node1
    command: redis-server /usr/local/etc/redis/redis.conf
    volumes:
      - ./redis/node1.conf:/usr/local/etc/redis/redis.conf
      - redis-node1-data:/data
    ports:
      - "6379:6379"
    networks:
      - ticket-network

  # Kafka
  kafka:
    image: confluentinc/cp-kafka:7.5.0
    container_name: kafka
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    ports:
      - "9092:9092"
    networks:
      - ticket-network

  # Zookeeper
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    container_name: zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"
    networks:
      - ticket-network

  # Nacos
  nacos:
    image: nacos/nacos-server:v2.2.3
    container_name: nacos
    environment:
      MODE: standalone
      SPRING_DATASOURCE_PLATFORM: mysql
      MYSQL_SERVICE_HOST: mysql-master
      MYSQL_SERVICE_DB_NAME: nacos
      MYSQL_SERVICE_USER: root
      MYSQL_SERVICE_PASSWORD: root123
    ports:
      - "8848:8848"
    depends_on:
      - mysql-master
    networks:
      - ticket-network

  # 用户服务
  user-service:
    build: ./user-service
    container_name: user-service
    environment:
      SPRING_PROFILES_ACTIVE: prod
      NACOS_SERVER_ADDR: nacos:8848
    ports:
      - "8081:8080"
    depends_on:
      - nacos
      - mysql-master
      - redis-node1
    networks:
      - ticket-network

  # 订单服务
  order-service:
    build: ./order-service
    container_name: order-service
    environment:
      SPRING_PROFILES_ACTIVE: prod
      NACOS_SERVER_ADDR: nacos:8848
    ports:
      - "8082:8080"
    depends_on:
      - nacos
      - mysql-master
      - redis-node1
      - kafka
    networks:
      - ticket-network

networks:
  ticket-network:
    driver: bridge

volumes:
  mysql-master-data:
  mysql-slave-data:
  redis-node1-data:
```

### 10.2 Kubernetes部署

#### 10.2.1 Deployment配置
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: ticket-system
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order-service
        image: ticket-system/order-service:v1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "prod"
        - name: NACOS_SERVER_ADDR
          value: "nacos-service:8848"
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: ticket-system
spec:
  selector:
    app: order-service
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 8080
  type: ClusterIP
```

#### 10.2.2 HPA自动扩缩容
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
  namespace: ticket-system
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
      - type: Pods
        value: 2
        periodSeconds: 60
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
      selectPolicy: Min
```

#### 10.2.3 Ingress配置
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ticket-system-ingress
  namespace: ticket-system
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.ticket.com
    secretName: ticket-tls-secret
  rules:
  - host: api.ticket.com
    http:
      paths:
      - path: /api/user
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 8080
      - path: /api/order
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 8080
      - path: /api/search
        pathType: Prefix
        backend:
          service:
            name: search-service
            port:
              number: 8080
```

### 10.3 自动扩缩容策略

#### 10.3.1 基于指标的扩缩容
```yaml
# 基于CPU和内存的扩缩容
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: search-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: search-service
  minReplicas: 5
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
```

#### 10.3.2 定时扩缩容
```yaml
# CronHPA - 春运期间定时扩容
apiVersion: autoscaling.alibabacloud.com/v1beta1
kind: CronHorizontalPodAutoscaler
metadata:
  name: order-service-cron-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  jobs:
  - name: scale-up-spring-festival
    schedule: "0 0 15 1 *"  # 每年1月15日0点扩容
    targetSize: 50
  - name: scale-down-after-festival
    schedule: "0 0 1 3 *"   # 每年3月1日0点缩容
    targetSize: 10
  - name: scale-up-peak-hours
    schedule: "0 8 * * *"   # 每天8点扩容
    targetSize: 20
  - name: scale-down-off-peak
    schedule: "0 22 * * *"  # 每天22点缩容
    targetSize: 5
```

### 10.4 异地双活架构

#### 10.4.1 多地域部署
```
┌─────────────────────────────────────────────────────────────┐
│                      Global Load Balancer                    │
│                         (DNS + GSLB)                         │
└──────────────────┬──────────────────┬───────────────────────┘
                   │                  │
        ┌──────────▼────────┐  ┌──────▼──────────┐
        │  Region A (北京)   │  │  Region B (上海) │
        │  ┌──────────────┐ │  │  ┌──────────────┐│
        │  │   K8s Cluster│ │  │  │   K8s Cluster││
        │  │   (Active)   │ │  │  │   (Active)   ││
        │  └──────────────┘ │  │  └──────────────┘│
        │  ┌──────────────┐ │  │  ┌──────────────┐│
        │  │  MySQL主从   │ │  │  │  MySQL主从   ││
        │  └──────────────┘ │  │  └──────────────┘│
        │  ┌──────────────┐ │  │  ┌──────────────┐│
        │  │ Redis Cluster│ │  │  │ Redis Cluster││
        │  └──────────────┘ │  │  └──────────────┘│
        └───────────────────┘  └───────────────────┘
                   │                  │
                   └────────┬─────────┘
                            │
                    ┌───────▼────────┐
                    │  数据同步中心   │
                    │  (Canal/Otter) │
                    └────────────────┘
```

#### 10.4.2 数据同步方案
```yaml
# Canal配置 - MySQL binlog同步
canal.instance.master.address=mysql-master-beijing:3306
canal.instance.dbUsername=canal
canal.instance.dbPassword=canal123
canal.instance.filter.regex=ticket_db\\..*

# 同步目标
canal.destinations=beijing-to-shanghai
canal.destination.beijing-to-shanghai.target=mysql-master-shanghai:3306
```

#### 10.4.3 流量调度策略

**地域感知负载均衡流程**：
1. 获取用户所在地域
2. 优先选择同地域服务实例
3. 同地域无可用实例时，选择其他地域

### 10.5 容灾设计

#### 10.5.1 故障切换流程
```
1. 健康检查失败 → 2. 从负载均衡摘除 → 3. 流量切换到备用节点
                                           ↓
4. 告警通知运维 ← 5. 自动重启容器 ← 6. 持续监控恢复
```

#### 10.5.2 数据备份策略
```bash
#!/bin/bash
# MySQL全量备份脚本

BACKUP_DIR="/data/backup/mysql"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/ticket_db_$DATE.sql.gz"

# 1. 全量备份
mysqldump -h mysql-master \
  -u backup_user \
  -p'backup_pass' \
  --single-transaction \
  --master-data=2 \
  --all-databases \
  | gzip > $BACKUP_FILE

# 2. 上传到对象存储
aws s3 cp $BACKUP_FILE s3://ticket-backup/mysql/

# 3. 保留最近7天的备份
find $BACKUP_DIR -name "*.sql.gz" -mtime +7 -delete

# 4. 验证备份文件
gunzip -t $BACKUP_FILE
if [ $? -eq 0 ]; then
    echo "Backup successful: $BACKUP_FILE"
else
    echo "Backup failed: $BACKUP_FILE"
    # 发送告警
    curl -X POST "https://alert.ticket.com/api/alert" \
      -d "message=MySQL backup failed"
fi
```

#### 10.5.3 灾难恢复预案
```yaml
# 灾难恢复流程
disaster_recovery_plan:
  level_1_partial_failure:
    - 单个Pod故障: K8s自动重启
    - 单个Node故障: K8s自动迁移Pod
    - 恢复时间: < 1分钟
    
  level_2_service_failure:
    - 整个服务不可用: 触发HPA扩容
    - 数据库主库故障: 自动切换到从库
    - 恢复时间: < 5分钟
    
  level_3_region_failure:
    - 整个地域不可用: DNS切换到备用地域
    - 数据同步延迟: 使用最近一致性数据
    - 恢复时间: < 10分钟
    
  level_4_total_disaster:
    - 所有地域不可用: 启用灾备中心
    - 从备份恢复数据: 使用最近的全量+增量备份
    - 恢复时间: < 2小时
```

---

## 11. 监控与运维

### 11.1 监控体系

#### 11.1.1 Prometheus监控配置
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'order-service'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
            - ticket-system
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        regex: order-service
        action: keep

  - job_name: 'mysql'
    static_configs:
      - targets: ['mysql-exporter:9104']

  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']

  - job_name: 'kafka'
    static_configs:
      - targets: ['kafka-exporter:9308']
```

#### 11.1.2 Grafana仪表盘
```json
{
  "dashboard": {
    "title": "火车票购票系统监控",
    "panels": [
      {
        "title": "QPS",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[1m])) by (service)"
          }
        ]
      },
      {
        "title": "响应时间",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))"
          }
        ]
      },
      {
        "title": "错误率",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status=~\"5..\"}[1m])) / sum(rate(http_requests_total[1m]))"
          }
        ]
      },
      {
        "title": "库存剩余",
        "targets": [
          {
            "expr": "ticket_inventory_available_seats"
          }
        ]
      }
    ]
  }
}
```

#### 11.1.3 告警规则
```yaml
groups:
  - name: ticket-system-alerts
    rules:
      - alert: HighErrorRate
        expr: sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "错误率过高"
          description: "服务 {{ $labels.service }} 错误率超过5%"

      - alert: HighResponseTime
        expr: histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)) > 3
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "响应时间过长"
          description: "服务 {{ $labels.service }} P95响应时间超过3秒"

      - alert: LowInventory
        expr: ticket_inventory_available_seats < 10
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "库存不足"
          description: "车次 {{ $labels.train_number }} 余票不足10张"

      - alert: HighCPUUsage
        expr: rate(container_cpu_usage_seconds_total[5m]) > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "CPU使用率过高"
          description: "Pod {{ $labels.pod }} CPU使用率超过80%"
```

### 11.2 日志管理

#### 11.2.1 ELK架构
```
应用日志 → Filebeat → Logstash → Elasticsearch → Kibana
```

#### 11.2.2 日志格式规范
```json
{
  "timestamp": "2024-01-15T10:30:45.123Z",
  "level": "INFO",
  "service": "order-service",
  "traceId": "1a2b3c4d5e6f",
  "spanId": "7g8h9i0j",
  "userId": 123456,
  "action": "createOrder",
  "message": "订单创建成功",
  "orderId": "ORDER20240115001",
  "duration": 245,
  "status": "success"
}
```

### 11.3 链路追踪

#### 11.3.1 SkyWalking配置
```yaml
agent:
  service_name: ${SW_AGENT_NAME:order-service}
  sample_n_per_3_secs: 1000
  trace_segment_ref_limit_per_span: 300

collector:
  backend_service: skywalking-oap:11800

logging:
  level: INFO
```

---

## 12. 性能优化总结

### 12.1 关键性能指标

| 指标 | 目标值 | 优化措施 |
|-----|-------|---------|
| 查询QPS | 100,000+ | Redis缓存 + CDN加速 |
| 下单QPS | 50,000+ | 排队系统 + 消息队列削峰 |
| 查询响应时间 | < 200ms | 多级缓存 + 数据库索引优化 |
| 下单响应时间 | < 3s | 异步处理 + 预扣库存 |
| 系统可用性 | 99.99% | 异地双活 + 自动故障切换 |
| 数据一致性 | 最终一致(秒级) | TCC分布式事务 + 本地消息表 + 补偿机制 |

### 12.2 架构优势

1. **高并发处理能力**: 通过负载均衡、缓存、消息队列等技术，支持100万+QPS
2. **高可用性**: 异地双活、自动故障切换，系统可用性达99.99%
3. **数据一致性**: TCC、SAGA等分布式事务方案，保证订单和库存一致性
4. **弹性伸缩**: 基于K8s的HPA，根据负载自动扩缩容
5. **安全防护**: 多层防刷票机制，保证公平性
6. **可观测性**: 完善的监控、日志、链路追踪体系

---

## 13. 未来演进方向

1. **AI智能推荐**: 基于用户行为的车次推荐
2. **动态定价**: 根据需求实时调整票价
3. **区块链溯源**: 防伪票、黄牛票
4. **边缘计算**: CDN边缘节点处理查询请求
5. **Serverless**: 非核心服务使用Serverless降低成本

---

## 文档结束

本文档详细描述了高并发火车票购票系统的架构设计，涵盖了从技术选型到部署运维的各个方面。文档采用了标准UML图表和规范化描述，包括：

- **组件图**：展示系统组件及其依赖关系
- **架构图**：展示系统的整体架构
- **状态机图**：展示熔断等机制的状态转换
- **配置示例**：Docker、K8s等部署配置
- **流程说明**：各种算法和机制的流程描述

文档遵循了系统设计的标准结构，删除了具体业务代码实现，保留了架构设计说明和配置示例，确保了文档的规范性、实用性和可读性。
#### 3.1.4 跨库查询解决方案

##### 3.1.4.1 问题分析

**分库分表后受影响的查询场景**:

| 查询场景 | 原查询方式 | 分库后的问题 | 影响程度 |
|---------|-----------|-------------|---------|
| 按用户ID查询订单 | `SELECT * FROM t_order WHERE user_id = ?` | 需要扫描所有分库分表 | 高 |
| 按订单号查询订单 | `SELECT * FROM t_order WHERE order_no = ?` | 订单号无法定位到具体分库分表 | 高 |
| 按车次查询订单 | `SELECT * FROM t_order WHERE train_number = ?` | 需要扫描所有分库分表 | 中 |
| 按时间范围查询 | `SELECT * FROM t_order WHERE create_time BETWEEN ? AND ?` | 无法按时间定位分表 | 中 |

##### 3.1.4.2 解决方案1: 全局索引表 (推荐)

**方案概述**: 创建一个独立的全局索引表,存储订单的关键信息和路由信息,用于快速定位订单所在的分库分表。

**表结构设计**:

```sql
CREATE TABLE t_user_order_index (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL COMMENT '用户ID',
    order_id BIGINT NOT NULL COMMENT '订单ID',
    order_no VARCHAR(32) NOT NULL COMMENT '订单号',
    train_number VARCHAR(20) NOT NULL COMMENT '车次号',
    train_date DATE NOT NULL COMMENT '乘车日期',
    order_status TINYINT NOT NULL COMMENT '订单状态',
    total_price DECIMAL(10,2) NOT NULL COMMENT '订单金额',
    create_time DATETIME NOT NULL COMMENT '创建时间',
    update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    -- 路由信息
    db_index TINYINT NOT NULL COMMENT '数据库索引',
    table_index TINYINT NOT NULL COMMENT '表索引',
    -- 索引
    UNIQUE KEY uk_order_id (order_id),
    UNIQUE KEY uk_order_no (order_no),
    INDEX idx_user_time (user_id, create_time DESC),
    INDEX idx_train_date (train_number, train_date),
    INDEX idx_status (order_status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户订单全局索引表';
```

**查询流程**:

```
用户查询订单(按user_id)
  ↓
查询全局索引表,获取订单列表和路由信息
  ↓
根据order_id计算分库分表位置
  ↓
批量查询各分库分表,获取订单详情
  ↓
合并结果返回
```

**一致性保证方案**:

**方案A: 本地消息表 + 定时任务**

```sql
-- 本地消息表
CREATE TABLE t_local_message (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    message_id VARCHAR(64) UNIQUE NOT NULL COMMENT '消息ID',
    business_type VARCHAR(50) NOT NULL COMMENT '业务类型',
    business_id BIGINT NOT NULL COMMENT '业务ID',
    content TEXT NOT NULL COMMENT '消息内容',
    status TINYINT DEFAULT 0 COMMENT '状态:0待发送,1已发送,2发送失败',
    retry_count INT DEFAULT 0 COMMENT '重试次数',
    max_retry INT DEFAULT 3 COMMENT '最大重试次数',
    next_retry_time DATETIME COMMENT '下次重试时间',
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_status_retry (status, next_retry_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='本地消息表';
```

**实现流程**:
1. 创建订单时,在同一数据库事务中:
   - 插入订单表 `t_order_{N}`
   - 插入本地消息表 `t_local_message`
2. 定时任务(每秒执行)扫描本地消息表
3. 将待发送消息同步到全局索引表
4. 更新消息状态为"已发送"
5. 失败时更新重试次数和下次重试时间

**方案B: Canal实时同步 (双重保障)**

```yaml
# Canal配置
canal.instance.master.address=mysql-master:3306
canal.instance.dbUsername=canal
canal.instance.dbPassword=canal123
canal.instance.filter.regex=order_db_.*\\.t_order_.*

# 监听订单表binlog
# 实时同步到全局索引表
# 作为本地消息表的补充和校验
```

**对账机制**:
- 每日凌晨执行对账任务
- 对比订单表和索引表的数据一致性
- 发现不一致数据自动修复或告警

**优缺点分析**:

| 维度 | 优点 | 缺点 |
|------|------|------|
| 性能 | 查询性能高,只需查询索引表+目标分库 | 写入时需要维护索引表 |
| 一致性 | 最终一致性,延迟<3秒 | 存在短暂的不一致窗口 |
| 复杂度 | 实现相对简单 | 需要维护本地消息表和对账任务 |
| 成本 | 存储成本低 | 需要额外的索引表存储 |
| 适用场景 | 适合按用户ID、订单号查询 | 不适合复杂的多条件查询 |

##### 3.1.4.3 解决方案2: Elasticsearch (适合复杂查询)

**方案概述**: 将订单数据同步到Elasticsearch,利用ES的强大查询能力支持复杂的多条件查询。

**索引设计**:

```json
{
  "mappings": {
    "properties": {
      "order_id": { "type": "long" },
      "order_no": { "type": "keyword" },
      "user_id": { "type": "long" },
      "train_number": { "type": "keyword" },
      "train_date": { "type": "date" },
      "departure_station": { "type": "keyword" },
      "arrival_station": { "type": "keyword" },
      "seat_type": { "type": "byte" },
      "passenger_name": { "type": "text", "analyzer": "ik_max_word" },
      "order_status": { "type": "byte" },
      "total_price": { "type": "scaled_float", "scaling_factor": 100 },
      "create_time": { "type": "date" },
      "update_time": { "type": "date" }
    }
  },
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 1,
    "refresh_interval": "1s"
  }
}
```

**数据同步方案**:

```
订单创建/更新
  ↓
发送消息到Kafka (topic: order-sync-topic)
  ↓
ES同步服务消费Kafka消息
  ↓
写入Elasticsearch
  ↓
同步完成
```

**查询流程**:

```
用户发起复杂查询(多条件)
  ↓
查询Elasticsearch,获取订单ID列表
  ↓
根据订单ID批量查询分库分表
  ↓
返回订单详情
```

**优缺点分析**:

| 维度 | 优点 | 缺点 |
|------|------|------|
| 性能 | 支持复杂查询,性能优秀 | 写入延迟较高(1-2秒) |
| 一致性 | 最终一致性,延迟1-2秒 | 不保证强一致性 |
| 复杂度 | 查询功能强大 | 运维复杂度高 |
| 成本 | 存储和计算成本较高 | 需要ES集群资源 |
| 适用场景 | 适合复杂的多条件查询、全文搜索 | 不适合强一致性要求的场景 |

##### 3.1.4.4 解决方案3: 数据冗余 (适合特定场景)

**方案概述**: 在用户库中冗余存储用户的订单列表,用于快速查询用户订单。

**冗余表设计**:

```sql
-- 用户订单列表表(存储在用户库中)
CREATE TABLE t_user_order_list (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL COMMENT '用户ID',
    order_id BIGINT NOT NULL COMMENT '订单ID',
    order_no VARCHAR(32) NOT NULL COMMENT '订单号',
    train_number VARCHAR(20) NOT NULL COMMENT '车次号',
    order_status TINYINT NOT NULL COMMENT '订单状态',
    create_time DATETIME NOT NULL COMMENT '创建时间',
    INDEX idx_user_time (user_id, create_time DESC)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户订单列表';
```

**查询流程**:

```
查询用户订单列表
  ↓
根据user_id定位用户库
  ↓
查询t_user_order_list表
  ↓
返回订单列表(简要信息)
  ↓
如需详情,再根据order_id查询订单表
```

**一致性保证**: 与全局索引表方案相同,使用本地消息表 + Canal双重保障。

**优缺点分析**:

| 维度 | 优点 | 缺点 |
|------|------|------|
| 性能 | 查询性能极高,单库查询 | 数据冗余,存储成本高 |
| 一致性 | 最终一致性,延迟<3秒 | 存在短暂的不一致窗口 |
| 复杂度 | 实现简单 | 需要维护多份数据 |
| 成本 | 存储成本较高 | 数据冗余导致存储翻倍 |
| 适用场景 | 适合高频的用户订单列表查询 | 只适合特定查询场景 |

##### 3.1.4.5 方案对比与选择

**三种方案对比**:

| 对比维度 | 全局索引表 | Elasticsearch | 数据冗余 |
|---------|-----------|--------------|---------|
| 查询性能 | ★★★★☆ | ★★★★★ | ★★★★★ |
| 写入性能 | ★★★★☆ | ★★★☆☆ | ★★★☆☆ |
| 一致性 | 最终一致(3秒) | 最终一致(1-2秒) | 最终一致(3秒) |
| 查询灵活性 | ★★★☆☆ | ★★★★★ | ★★☆☆☆ |
| 运维复杂度 | ★★★☆☆ | ★★★★☆ | ★★☆☆☆ |
| 存储成本 | ★★★★☆ | ★★★☆☆ | ★★☆☆☆ |
| 实现难度 | ★★★☆☆ | ★★★★☆ | ★★☆☆☆ |

**推荐组合方案**:

```
┌─────────────────────────────────────────────┐
│           查询场景分流策略                    │
└─────────────────────────────────────────────┘
                    │
        ┌───────────┼───────────┐
        │           │           │
        ▼           ▼           ▼
  ┌──────────┐ ┌──────────┐ ┌──────────┐
  │核心查询  │ │复杂查询  │ │高频查询  │
  │(80%)    │ │(15%)    │ │(5%)     │
  └──────────┘ └──────────┘ └──────────┘
        │           │           │
        ▼           ▼           ▼
  ┌──────────┐ ┌──────────┐ ┌──────────┐
  │全局索引表│ │Elasticsearch│ │数据冗余  │
  └──────────┘ └──────────┘ └──────────┘
```

**具体应用**:

1. **核心查询场景** (使用全局索引表)
   - 按用户ID查询订单列表
   - 按订单号查询订单详情
   - 按车次+日期查询订单

2. **复杂查询场景** (使用Elasticsearch)
   - 多条件组合查询
   - 模糊搜索(乘客姓名、车次号)
   - 统计分析查询

3. **高频查询场景** (使用数据冗余)
   - 用户最近订单列表(首页展示)
   - 待支付订单提醒

**实施建议**:

1. **第一阶段**: 实现全局索引表方案
   - 满足80%的核心查询需求
   - 实现简单,风险可控

2. **第二阶段**: 引入Elasticsearch
   - 支持复杂查询和统计分析
   - 提升用户体验

3. **第三阶段**: 针对性数据冗余
   - 优化高频查询场景
   - 降低核心接口响应时间
### 7.4 一致性分级与CAP权衡

#### 7.4.1 CAP定理权衡

**CAP定理解释**:

在分布式系统中,以下三个特性最多只能同时满足两个:
- **C (Consistency)**: 一致性 - 所有节点在同一时间看到相同的数据
- **A (Availability)**: 可用性 - 每个请求都能得到响应(成功或失败)
- **P (Partition Tolerance)**: 分区容错性 - 系统在网络分区的情况下仍能继续运行

**火车票系统的选择: AP > C**

```
┌─────────────────────────────────────────────┐
│            CAP权衡决策                        │
└─────────────────────────────────────────────┘
                    │
        ┌───────────┼───────────┐
        │           │           │
        ▼           ▼           ▼
    ┌──────┐    ┌──────┐    ┌──────┐
    │  CP  │    │  AP  │    │  CA  │
    │      │    │  ✅  │    │      │
    └──────┘    └──────┘    └──────┘
                    │
            ┌───────┴───────┐
            │               │
            ▼               ▼
    ┌──────────────┐ ┌──────────────┐
    │ 高可用性      │ │ 最终一致性    │
    │ 99.99%       │ │ 秒级同步      │
    └──────────────┘ └──────────────┘
```

**选择AP的原因**:

1. **业务特性**: 火车票购票是高并发场景,系统可用性优先于强一致性
2. **用户体验**: 用户更关心能否成功购票,而不是数据的实时一致性
3. **技术可行性**: 通过最终一致性方案,可以在保证高可用的前提下,实现秒级数据一致性
4. **成本考虑**: 强一致性需要分布式锁、两阶段提交等重量级方案,性能开销大

**权衡说明**:

| 场景 | 一致性要求 | 可用性要求 | 选择 | 说明 |
|------|-----------|-----------|------|------|
| 订单创建 | 高 | 高 | AP + 事务 | 使用TCC保证订单和库存的事务一致性 |
| 订单支付 | 高 | 高 | AP + 补偿 | 支付失败自动回滚,保证最终一致性 |
| 库存扣减 | 中 | 高 | AP | 先扣Redis,异步更新DB,3秒内一致 |
| 缓存同步 | 低 | 高 | AP | Canal监听binlog,1秒内同步 |
| 数据统计 | 低 | 中 | AP | 离线计算,T+1统计 |

#### 7.4.2 一致性分级表

**系统各场景的一致性级别**:

| 场景 | 一致性级别 | 实现方案 | 一致性时间 | 说明 |
|------|-----------|---------|-----------|------|
| 订单创建 | 事务一致 | TCC分布式事务 | 实时 | 订单和库存在同一事务中,要么都成功,要么都失败 |
| 订单支付 | 事务一致 | SAGA + 补偿 | 实时 | 支付失败自动回滚订单和库存 |
| 库存扣减 | 最终一致 | Redis预扣 + 异步写DB | < 3秒 | 先扣Redis保证性能,异步更新DB保证持久化 |
| 缓存同步 | 最终一致 | Canal binlog同步 | < 1秒 | 监听DB变更,实时同步到Redis缓存 |
| 跨地域同步 | 最终一致 | MySQL主从复制 | < 5秒 | 异地机房数据同步,保证灾备 |
| 订单索引表 | 最终一致 | 本地消息表 + 定时任务 | < 3秒 | 异步同步到全局索引表,支持跨库查询 |
| 候补订单处理 | 最终一致 | Kafka事件驱动 | < 2秒 | 库存释放后触发候补处理 |
| 数据统计 | 最终一致 | 离线计算 | 分钟级 | T+1统计,不要求实时 |
| 用户会话 | 最终一致 | Redis + 定期刷新 | < 30秒 | 会话过期自动刷新 |

**一致性级别定义**:

- **事务一致**: 使用分布式事务(TCC/SAGA)保证,实时一致,适用于核心业务
- **最终一致**: 使用异步消息、定时任务等方案,秒级或分钟级一致,适用于非核心业务

#### 7.4.3 一致性保证机制

##### 7.4.3.1 核心业务场景 - TCC流程

**订单创建的TCC流程**:

```
┌─────────────────────────────────────────────┐
│              订单创建TCC流程                  │
└─────────────────────────────────────────────┘

Try阶段:
┌──────────────┐    ┌──────────────┐
│ 订单服务      │    │ 库存服务      │
│ 预创建订单    │───▶│ 预扣库存      │
│ (状态:TRY)   │    │ (锁定座位)    │
└──────────────┘    └──────────────┘
        │                   │
        └─────────┬─────────┘
                  ▼
            ┌──────────┐
            │ 记录事务  │
            │ 日志      │
            └──────────┘

Confirm阶段(Try成功):
┌──────────────┐    ┌──────────────┐
│ 订单服务      │    │ 库存服务      │
│ 确认订单      │───▶│ 确认扣减      │
│ (状态:PAID)  │    │ (释放锁定)    │
└──────────────┘    └──────────────┘

Cancel阶段(Try失败):
┌──────────────┐    ┌──────────────┐
│ 订单服务      │    │ 库存服务      │
│ 取消订单      │───▶│ 回滚库存      │
│ (状态:CANCEL)│    │ (释放锁定)    │
└──────────────┘    └──────────────┘
```

**TCC实现要点**:

1. **幂等性**: 每个阶段都要保证幂等,防止重复执行
2. **超时处理**: Try阶段超时自动Cancel
3. **补偿机制**: Confirm/Cancel失败时自动重试
4. **事务日志**: 记录每个阶段的执行状态,用于故障恢复

##### 7.4.3.2 非核心场景 - 最终一致性方案

**库存扣减的最终一致性流程**:

```
用户下单
  ↓
Redis原子扣减库存 (DECR命令)
  ↓
检查扣减结果
  ├─ 成功(>= 0) ─────────┐
  │                      ▼
  │              发送订单消息到Kafka
  │                      ↓
  │              订单服务消费消息
  │                      ↓
  │              异步更新MySQL库存
  │                      ↓
  │              更新成功 ──────────┐
  │                                ▼
  └──────────────────────────> 订单创建完成
  
  └─ 失败(< 0) ───> 回滚Redis库存 ───> 返回"余票不足"
```

**最终一致性保证措施**:

1. **本地消息表**: 在同一事务中写入业务数据和消息
2. **定时任务**: 扫描本地消息表,重试失败消息
3. **Canal同步**: 监听binlog,实时同步到缓存
4. **对账任务**: 定期对比Redis和MySQL数据,发现不一致自动修复

##### 7.4.3.3 异常场景处理表

| 异常场景 | 检测方式 | 处理方式 | 恢复时间 |
|---------|---------|---------|---------|
| Redis扣减成功,DB更新失败 | 定时任务扫描本地消息表 | 重试DB更新,最多3次 | < 10秒 |
| 订单创建成功,索引表未同步 | 对账任务发现不一致 | 补偿写入索引表 | < 1分钟 |
| 支付成功,订单状态未更新 | 支付回调重试机制 | 重新处理支付回调 | < 30秒 |
| 库存回滚失败 | 监控告警 | 人工介入修复 | < 5分钟 |
| 跨地域数据不一致 | Canal监控延迟 | 触发全量同步 | < 10分钟 |
| 缓存与DB不一致 | 定期对账任务 | 删除缓存,重新加载 | < 1分钟 |

#### 7.4.4 一致性监控

##### 7.4.4.1 监控指标

**关键监控指标**:

```yaml
# 一致性监控指标
consistency_metrics:
  # 本地消息表积压
  - name: local_message_pending_count
    type: gauge
    description: 待发送消息数量
    threshold: 1000
    alert_level: warning
    
  # 消息处理延迟
  - name: message_process_delay_seconds
    type: histogram
    description: 消息处理延迟
    threshold: 5
    alert_level: critical
    
  # Canal同步延迟
  - name: canal_sync_delay_seconds
    type: gauge
    description: Canal同步延迟
    threshold: 2
    alert_level: warning
    
  # 对账不一致数量
  - name: reconciliation_inconsistent_count
    type: counter
    description: 对账发现的不一致数据量
    threshold: 100
    alert_level: critical
    
  # TCC事务成功率
  - name: tcc_transaction_success_rate
    type: gauge
    description: TCC事务成功率
    threshold: 0.99
    alert_level: warning
```

##### 7.4.4.2 告警规则

**Prometheus告警规则**:

```yaml
groups:
  - name: consistency-alerts
    rules:
      # 本地消息积压告警
      - alert: LocalMessageBacklog
        expr: local_message_pending_count > 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "本地消息表积压严重"
          description: "待发送消息数量: {{ $value }},超过阈值1000"
      
      # 消息处理延迟告警
      - alert: MessageProcessDelayHigh
        expr: histogram_quantile(0.95, message_process_delay_seconds) > 5
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "消息处理延迟过高"
          description: "P95延迟: {{ $value }}秒,超过阈值5秒"
      
      # Canal同步延迟告警
      - alert: CanalSyncDelayHigh
        expr: canal_sync_delay_seconds > 2
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Canal同步延迟过高"
          description: "同步延迟: {{ $value }}秒,超过阈值2秒"
      
      # 对账不一致告警
      - alert: ReconciliationInconsistent
        expr: increase(reconciliation_inconsistent_count[1h]) > 100
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "对账发现大量不一致数据"
          description: "1小时内不一致数据量: {{ $value }},超过阈值100"
```

##### 7.4.4.3 对账机制

**对账任务设计**:

```sql
-- 对账任务: 检查Redis库存与MySQL库存的一致性
-- 每小时执行一次

-- 1. 查询Redis库存
SELECT train_number, train_date, seat_type, available_seats AS redis_seats
FROM redis_inventory;

-- 2. 查询MySQL库存
SELECT train_number, train_date, seat_type, available_seats AS mysql_seats
FROM t_inventory;

-- 3. 对比差异
SELECT 
    r.train_number,
    r.train_date,
    r.seat_type,
    r.redis_seats,
    m.mysql_seats,
    (r.redis_seats - m.mysql_seats) AS diff
FROM redis_inventory r
JOIN t_inventory m
    ON r.train_number = m.train_number
    AND r.train_date = m.train_date
    AND r.seat_type = m.seat_type
WHERE r.redis_seats != m.mysql_seats;

-- 4. 修复不一致数据
-- 如果差异在合理范围内(< 10),以MySQL为准更新Redis
-- 如果差异过大,触发告警,人工介入
```

**对账流程**:

```
定时任务启动(每小时)
  ↓
查询Redis和MySQL库存数据
  ↓
对比差异
  ├─ 无差异 ───────────────────> 记录日志,任务结束
  │
  ├─ 差异 < 10 ─────────────────┐
  │                             ▼
  │                      以MySQL为准更新Redis
  │                             ↓
  │                      记录修复日志
  │                             ↓
  │                      任务结束
  │
  └─ 差异 >= 10 ────────────────┐
                                ▼
                         触发告警
                                ↓
                         人工介入分析
                                ↓
                         手动修复或回滚
```

**对账报告**:

```json
{
  "task_id": "reconciliation_20250124_10",
  "start_time": "2025-01-24T10:00:00Z",
  "end_time": "2025-01-24T10:05:23Z",
  "total_checked": 50000,
  "inconsistent_count": 5,
  "auto_fixed_count": 5,
  "manual_fix_count": 0,
  "inconsistent_details": [
    {
      "train_number": "G123",
      "train_date": "2025-01-25",
      "seat_type": 2,
      "redis_seats": 100,
      "mysql_seats": 98,
      "diff": 2,
      "action": "auto_fixed",
      "fix_time": "2025-01-24T10:02:15Z"
    }
  ]
}
```

---
