---
name: storeforge-brainstorm
description: 电商项目脑暴，分析需求并生成技术方案
---

# 电商项目脑暴

> Skill: `storeforge-brainstorm` | 触发: `/sf:brainstorm` 或手动调用

## 定位

分析电商需求，生成技术方案文档，作为 `storeforge-writing-plans` 的输入。
本阶段 **不编写任何代码**，只输出方案和决策。

## 输入

用户需求描述（自然语言），例如：
- "做一个面向 B2B 的跨境电商平台，支持微信/支付宝支付，管理后台需要订单监控和商品批量操作"
- "需要一个闪购功能，支持限时折扣、拼团、预售三种营销模式"

## 执行流程（6 步）

### Step 1：需求拆解

将需求拆分为以下核心模块（按需裁剪）：

| 模块 | 职责 | 典型子域 |
|------|------|----------|
| 用户 | 注册/登录/认证/个人信息 | JWT、OAuth、第三方登录、RBAC |
| 商品 | SPU/SKU/分类/搜索/详情 | 库存预占、SKU 属性矩阵、搜索索引 |
| 订单 | 创建/状态流转/超时取消 | 状态机、Saga 编排、超时任务 |
| 支付 | 支付发起/回调验签/退款 | 微信/支付宝/抖音、幂等、对账 |
| 营销 | 优惠券/满减/限时折扣/拼团 | 互斥规则、计算引擎、预算限制 |
| 管理 | 后台 CRUD/数据看板/监控 | 权限、审计、异步导出 |

**操作**：
1. 逐条读取需求，标注属于哪个模块
2. 识别跨模块依赖（如"下单"依赖用户+商品+订单+支付）
3. 标记产品层面的模糊点（需要 `storeforge-user-decision` 确认）

### Step 2：技术栈匹配

根据模块确定涉及的技术端（domain skill）：

| 端 | Skill | 技术栈 |
|----|-------|--------|
| 后端 | `storeforge-domain-backend` | Golang + go-zero + GORM + PostgreSQL + Redis |
| 管理后台 | `storeforge-domain-admin` | Vue 3 + Element Plus + TypeScript + Pinia |
| 官网 | `storeforge-domain-website` | Next.js (App Router) + shadcn/ui + next-intl |
| 移动/小程序 | `storeforge-domain-flutter` | Flutter + Riverpod + go_router + Dio |

**操作**：
- 需求涉及哪些端，就标记哪些端
- 每个端后续由对应的 domain skill 约束开发

### Step 3：模式匹配

从 `knowledge/ecommerce-patterns/` 中查找适用的模式：

| 模式文件 | 适用场景 |
|----------|----------|
| `auth-patterns.md` | 需要用户认证、第三方登录、Token 管理 |
| `payment-patterns.md` | 需要对接微信/支付宝/抖音支付 |
| `cart-patterns.md` | 需要购物车功能 |
| `inventory-patterns.md` | 需要库存扣减、防超卖 |
| `order-state-machine.md` | 需要订单状态流转 |
| `promotion-patterns.md` | 需要优惠券、满减、折扣等营销 |
| `search-patterns.md` | 需要商品搜索功能 |
| `shipping-patterns.md` | 需要物流、运费计算 |
| `flash-sale-pattern.md` | 需要秒杀/闪购功能 |
| `image-cdn-pattern.md` | 需要图片 CDN、响应式图片 |
| `error-codes-registry.md` | 所有模块统一错误码 |

**操作**：
- 根据模块拆解结果，列出需要加载的 pattern 文件
- 每个 pattern 的"触发条件"章节确认是否匹配当前需求

### Step 4：架构设计

输出以下内容：

#### 4.1 服务拆分

按模块划分微服务（参考 go-zero 架构）：
- `user-service`：用户认证、信息管理、权限
- `product-service`：商品 CRUD、SKU、库存
- `order-service`：订单创建、状态机、超时取消
- `payment-service`：支付网关、回调处理、退款
- `promotion-service`：优惠券、活动、计算引擎

服务间通信：
- 同步调用：gRPC (protobuf RPC)，**禁止 HTTP 调内部服务**
- 异步通信：消息队列（RabbitMQ/Kafka/NATS），采用 Outbox 模式保证事件不丢失

#### 4.2 数据流（关键链路）

```
用户 → API Gateway → User Service (认证) → Product Service (浏览)
  → Cart Service (Redis 购物车) → Order Service (创建订单)
  → Inventory Service (预扣库存, Lua 脚本) → Payment Service (支付)
  → Order Service (确认订单, 状态机) → Shipping Service (发货)
```

跨服务操作（如"下单"）采用 Saga 编排模式：
1. Order Service 发出 `OrderCreated` 事件
2. Inventory Service 预扣库存 → `InventoryReserved` 或 `InventoryFailed`
3. Promotion Service 计算优惠 → `PromotionApplied` 或 `PromotionFailed`
4. 任何步骤失败 → 触发补偿事务

#### 4.3 关键状态机

**订单状态**（参考 `order-state-machine.md`）：
```
正向: pending → paid → shipped → delivered → completed
逆向: pending → cancelled
逆向: paid → refunding → refunded
逆向: shipped → returning → returned
超时: pending → cancelled (30min)
超时: delivered → completed (7d)
```

**支付状态**：
```
pending → paying → paid / failed / refunded
```

### Step 5：风险识别

从以下维度识别风险点：

| 维度 | 典型风险 | 检查点 |
|------|----------|--------|
| 并发 | 库存超卖、重复下单 | Redis Lua 原子扣减、幂等键 |
| 安全 | 支付回调不验签、金额精度丢失 | RSA/SHA256 验签、int64 存储金额 |
| 性能 | N+1 查询、热点接口无缓存 | `Preload`、Redis 缓存 + singleflight |
| 一致性 | 跨服务数据不一致 | Saga + Outbox、补偿事务 |
| 可用性 | 单点故障、数据库连接耗尽 | 连接池配置、降级策略 |

**操作**：
- 根据具体需求，列出 TOP 5 风险点
- 每个风险点附带缓解措施

### Step 6：输出技术方案文档

将以上分析整理为结构化的技术方案文档，包含以下章节：

```
# [项目名称] 技术方案

## 一、模块清单
| 模块 | 所属端 | 关联 Pattern | 依赖模块 |
|------|--------|-------------|----------|

## 二、服务架构图
（文字描述服务拆分和通信方式）

## 三、关键技术决策
1. [决策点] → [选择] → [原因]

## 四、风险清单
| 风险 | 严重程度 | 缓解措施 |
|------|----------|----------|

## 五、技术端覆盖
- 后端: [涉及模块]
- Admin: [涉及模块]
- Website: [涉及模块]
- Flutter: [涉及模块]
```

## 输出格式要求

### 模块清单

每个模块必须明确：
- 名称和职责
- 所属技术端（backend / admin / website / flutter）
- 关联的电商 pattern（引用 `knowledge/ecommerce-patterns/` 中的文件）
- 上下游依赖关系

### 服务架构图（文字描述）

用缩进和箭头描述：
- 每个服务的名称和职责
- 服务间的同步/异步通信方式
- 外部依赖（数据库、Redis、消息队列、第三方支付）

### 关键技术决策列表

每个决策包含三要素：
- **决策点**：需要选择什么
- **选择**：最终选定的方案
- **原因**：为什么选这个（引用 pattern 或反模式）

示例：
```
1. 金额存储 → int64（分）→ 避免浮点精度丢失，参考电商金额规范
2. 库存扣减 → Redis Lua + PG 事务双写 → 防超卖 + 一致性补偿
3. 搜索方案 → PG full-text（商品 < 5000）→ 成本最优，无需引入 ES
```

### 风险清单

每个风险包含：
- **风险描述**：具体会发生什么
- **严重程度**：高 / 中 / 低
- **触发条件**：什么场景下会出现
- **缓解措施**：如何避免或应对

## 约束

- **禁止在本阶段编写代码**：只输出方案和决策
- 遇到产品层面的模糊问题（业务规则、UI/UX、运营策略），调用 `storeforge-user-decision` 确认，不自行猜测
- 技术问题（框架选型、数据库设计、缓存策略）由 domain skill 约束，无需询问用户
- 输出文档必须结构化，便于 `storeforge-writing-plans` 直接解析并生成实施计划

## 后续

脑暴完成后：
1. 将技术方案文档交给 `storeforge-writing-plans` 生成实施计划
2. 或直接调用 `/sf:plan` 触发计划生成流程
3. 计划确认后进入 `storeforge-executing` 执行阶段
