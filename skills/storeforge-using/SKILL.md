---
name: storeforge-using
description: StoreForge 工作流入门，所有任务的起点
---

# StoreForge 工作流

> StoreForge 所有任务的入口点。每次会话开始时，session-start hook 会注入此 skill 的核心规则。

## 定位

本 skill 是所有 StoreForge 任务的起点。当用户发起任何与电商平台开发相关的任务时，首先加载此工作流规则，然后根据任务类型路由到对应的 domain skill、工作流 skill 或知识模式。

## 核心规则（不可跳过）

以下规则是 P0 强制级别，任何 StoreForge 会话都必须遵守，**不得跳过或绕过**。

### 1. 必须先计划再写代码

任何开发任务都**不得直接写代码**。第一步永远是：

1. 如果是新需求或复杂变更，先调用 `storeforge-brainstorm` 进行脑暴分析
2. 脑暴完成后（或需求已足够清晰），调用 `storeforge-writing-plans` 生成实施计划
3. 计划经用户确认后，才调用 `storeforge-executing` 开始执行

### 2. 必须用 TodoWrite 跟踪进度

每个子任务都必须对应一个 `TodoWrite` 条目：

- 任务开始前标记为 `in_progress`
- 任务完成后标记为 `completed`
- 一个时间点只能有一个 `in_progress` 的条目
- 并行任务通过 dispatch parallel agents 实现，每个 agent 管理自己的 TodoWrite

### 3. 完成前必须验证

标记任何任务完成之前，**必须**调用 `storeforge-verification` 运行完整检查清单：

- 代码规范检查（domain 反模式、命名规范、错误处理）
- 测试检查（L1 覆盖率、L2 场景覆盖、测试数据无硬编码）
- Git 检查（commit 格式、变更完整性、无敏感信息）
- 文档检查（OpenAPI spec 更新、枚举注册）

连续 3 次验证失败 → 升级至用户。

### 4. 金额规范（硬性约束）

- 金额一律使用 `int64`（单位：分），**禁止**使用 `float` / `double` 存储或计算金额
- 金额计算必须指定舍入规则：`ROUND_HALF_UP`
- 多币种场景必须携带 `currency_code` 字段
- 前端展示时从 int64（分）转换为元，**转换逻辑必须经过严格测试**
- 违反此规则 → `storeforge-harness` BLOCK 级别阻断

### 5. 订单状态机原则

- 订单状态转换必须经过合法路径，**禁止**任意状态间直接跳转
- 正向路径：`pending → paid → shipped → delivered → completed`
- 逆向路径：`pending → cancelled`，`paid → refunding → refunded`，`shipped → returning → returned`
- 超时自动取消（30min），超时自动确认收货（7d）
- 每个状态变更必须记录原因和操作人
- 详见 `knowledge/ecommerce-patterns/order-state-machine.md`

### 6. 认证授权原则

- JWT 必须使用 `RS256` 或 `ES256` 非对称算法，**禁止** `HS256` 对称加密
- Admin 角色与 User 角色严格分离，权限不交叉
- Token 必须支持撤销（版本计数器 + Redis 撤销列表）
- Refresh Token 存储于 `HttpOnly` cookie，**禁止** localStorage
- OAuth 流程必须携带 `state` 参数防 CSRF

## 工作流步骤

收到任务后，按以下步骤执行：

### Step 1：理解需求

- 仔细阅读用户需求描述
- 判断这是**技术问题**还是**产品问题**
  - 技术问题（框架选型、代码风格、数据库设计、缓存策略）→ 自行决策，参考 domain skill
  - 产品问题（业务规则、UI/UX、运营策略、功能优先级）→ 按 `storeforge-user-decision` 流程处理

### Step 2：分类路由

| 问题类型 | 处理方式 |
|---------|---------|
| 技术架构问题 | 调用对应 `storeforge-domain-*` skill |
| 产品/业务问题 | 调用 `storeforge-user-decision` |
| 新需求需要分析 | 进入 Step 3 |

### Step 3：分析与计划

- 复杂需求 → 调用 `storeforge-brainstorm` 进行技术方案分析
- 脑暴完成后 → 调用 `storeforge-writing-plans` 生成实施计划
- 需求已足够清晰 → 直接调用 `storeforge-writing-plans`

### Step 4：执行

- 调用 `storeforge-executing` 按计划执行子任务
- 每个子任务完成后 → `storeforge-harness` 约束检查 → L1 测试 → 提交

### Step 5：测试 → Review → 验证

1. 调用 `storeforge-testing` 按 L1 → L2 → L3 顺序执行测试
2. 调用 `storeforge-review` 启动 5 代理并行审查（最多 5 轮）
3. 调用 `storeforge-verification` 运行完成前检查清单
4. 全部通过 → 标记任务完成

## 能力索引（P1 按需加载）

### Commands

| 命令 | 描述 |
|------|------|
| `/sf:brainstorm` | 电商项目脑暴，分析需求并生成技术方案 |
| `/sf:review` | 多代理代码审查，5 个角度自循环 review |
| `/sf:test` | 三层测试执行（L1/L2/L3） |
| `/sf:plan` | 生成实施计划 |
| `/sf:verify` | 完成前验证 |

### Domain Skills

| Skill 名称 | 技术栈 | 描述 |
|-----------|--------|------|
| `storeforge-domain-backend` | Golang + go-zero | 后端微服务硬约束（17 条反模式） |
| `storeforge-domain-admin` | Vue 3 + Element Plus | 管理后台硬约束（8 条反模式） |
| `storeforge-domain-website` | Next.js + shadcn/ui | 官网硬约束（9 条反模式） |
| `storeforge-domain-flutter` | Flutter（5 端） | 多端 App 硬约束（12 条反模式） |

### 电商知识模式（knowledge/ecommerce-patterns/）

共 11 个模式文件，按需查阅：

| 模式文件 | 关键内容 |
|---------|---------|
| `auth-patterns.md` | JWT RS256 + refresh token + 微信/支付宝 OAuth |
| `payment-patterns.md` | 微信/支付宝/抖音支付对接、回调验签、退款 |
| `cart-patterns.md` | Redis + PG 双写购物车、跨端同步、失效清理 |
| `inventory-patterns.md` | Redis Lua 库存扣减、PG 事务、防超卖、一致性补偿 |
| `order-state-machine.md` | 完整状态转换图、超时自动处理、审计 log |
| `promotion-patterns.md` | 优惠券/满减/限时折扣/拼团互斥规则、优先级 |
| `search-patterns.md` | PG full-text / Elasticsearch 选型、缓存、分词 |
| `shipping-patterns.md` | 物流接入、运费计算、面单生成、轨迹查询 |
| `flash-sale-pattern.md` | 秒杀架构：预售 Token、Redis 预热、MQ 异步、限流降级 |
| `image-cdn-pattern.md` | CDN 策略、WebP/AVIF 转换、响应式图片、防 SSRF |
| `error-codes-registry.md` | 错误码命名空间、分类、BaseResponse 信封 |

### 核心机制

- `storeforge-harness` — 约束引擎机制（BLOCK/WARN 级别，违反即阻断）
- `storeforge-user-decision` — 技术/产品问题分类与决策流程

## 命名规范声明

所有 skill 在 `plugin.json` 中注册时，`name` 字段使用 `storeforge-*`（横杠）格式，如 `"storeforge-brainstorm"`。本 skill 及所有 SKILL.md 内部引用其他 skill 时，统一使用此格式，**不使用冒号格式**（`storeforge:`）。
