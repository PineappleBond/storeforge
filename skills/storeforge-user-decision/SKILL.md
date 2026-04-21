---
name: storeforge-user-decision
description: 用户决策流程，区分技术问题和产品问题
---

# 用户决策流程

> 区分技术问题和产品问题，避免用技术细节骚扰用户。

## 定位

在电商开发过程中，Agent 会不断遇到需要做决策的场景。本 skill 定义了哪些决策 Agent 可以自行决定（**不骚扰用户**），哪些决策必须通过 `AskUserQuestion` 询问用户。

**核心原则**：技术问题自行解决，产品问题必须询问。

## 技术问题（自行决策，不询问用户）

以下技术问题由 Agent 根据 domain skills 和 harness 约束自行决策，**不需要**也不应该询问用户：

### 框架选型

框架和技术栈已经由 domain skills 固定：

| 领域 | 技术栈 | 决策来源 |
|------|--------|---------|
| 后端 | Golang + go-zero + GORM + PostgreSQL + Redis | `storeforge-domain-backend` |
| 管理后台 | Vue 3 + Element Plus + TypeScript + Pinia | `storeforge-domain-admin` |
| 官网 | Next.js App Router + shadcn/ui + next-intl | `storeforge-domain-website` |
| 多端 App | Flutter + Riverpod + go_router + Dio | `storeforge-domain-flutter` |

当遇到"用什么框架/库"的问题时，直接查阅对应 domain skill，不要问用户。

### 代码风格与规范

代码风格已由 domain skill 约束：

- 后端：handler 必须 `goctl` 生成，返回 `BaseResponse<T>`，列表必须分页
- Admin：必须 Composition API（`<script setup>`），TypeScript 严格模式
- Website：必须 App Router，禁止 `any`，每页配置 metadata
- Flutter：必须 freezed + json_serializable，Dio + interceptor

### 数据库设计

数据库设计已由 knowledge patterns 定义：

- 用户表：密码 bcrypt 加密、手机号 AES-256-GCM、身份证号独立密钥
- 订单表：状态机字段 + 审计字段（原因、操作人、时间戳）
- 金额字段：统一 `int64`（分），带 `currency_code`
- 索引策略：查询字段建索引，外键关联字段建索引

### 缓存策略

缓存策略已由 harness 指定：

- 热点商品：Redis 缓存 + singleflight + stale-while-revalidate + jitter
- 热门搜索：Redis 缓存，5min TTL
- 库存：Redis Lua 脚本原子扣减
- 购物车：Redis + PG 双写，跨端同步

### 其他技术决策

| 场景 | 决策 |
|------|------|
| 消息队列选型 | RabbitMQ / Kafka / NATS（domain skill 允许任一） |
| 搜索方案选型 | 商品 < 5000 用 PG full-text，>= 5000 用 Elasticsearch |
| 错误码设计 | `{MODULE}_{SEQ}` 命名空间，protobuf enum 统一定义 |
| 分布式事务 | Saga 编排 + Outbox 模式 |
| 支付回调处理 | 幂等键 + Redis SET NX + 验签（RSA/SHA256） |

## 产品问题（必须 AskUserQuestion）

以下产品问题**必须**使用 `AskUserQuestion` 询问用户，Agent 不得自行猜测：

### 业务规则

- **运费规则**：包邮门槛？按重量/件数/地区计费？偏远地区加价？
- **优惠券策略**：优惠券怎么发放（自动/手动/领取）？面额？使用条件？有效期？
- **会员等级**：等级划分标准？每个等级的权益？升级/降级规则？
- **积分规则**：积分获取方式（消费/签到/邀请）？积分兑换比例？有效期？
- **退款政策**：支持哪些退款场景？退款时效？手续费？
- **税务规则**：是否需要发票？税率如何计算？

### UI/UX 细节

- **品牌色系**：主色、辅色、背景色、文字色
- **布局偏好**：首页布局（Banner + 分类 + 推荐？瀑布流？网格？）
- **文案风格**：正式/亲切/简洁/详细
- **图片规格**：商品图尺寸比例、背景要求、水印
- **交互细节**：动画效果、加载方式（分页/无限滚动）、空状态设计

### 运营策略

- **促销活动**：秒杀频率？拼团人数要求？满减门槛？
- **推送策略**：推送频次？推送时段？推送内容类型？
- **客服接入**：人工客服工作时间？自动回复规则？
- **内容审核**：用户评论是否审核？审核标准？
- **数据看板指标**：管理层关注哪些核心指标？

### 功能优先级

- **开发顺序**：先做用户模块还是商品模块？先做支付还是先做促销？
- **MVP 范围**：第一版必须包含哪些功能？哪些可以后续迭代？
- **平台优先级**：先做 Android/iOS 还是先做小程序？
- **国际化**：第一版是否支持多语言？支持哪些语言？

## AskUserQuestion 使用规范

当必须询问用户时，遵循以下规范：

### 问题必须具体

- **不好**："你觉得会员体系怎么设计？"
- **好**："会员等级分为 3 级还是 5 级？3 级（普通/银卡/金卡） vs 5 级（普通/铜/银/金/钻石）。3 级简单易懂运营成本低，5 级用户成长感更强但运营复杂。"

### 给选项而非开放式

- **不好**："运费怎么算？"
- **好**："运费方案：A) 满 99 元包邮，否则固定 10 元；B) 按重量计费，首重 1kg 10 元，续重 5 元/kg；C) 按地区差异化定价（华东 8 元，其他地区 12 元）"

### 附带影响评估

每个选项都要说明对用户的影响：

```
问题：[具体问题描述]

选项 A: [描述]
  影响: [对用户体验/成本/开发量的影响]

选项 B: [描述]
  影响: [对用户体验/成本/开发量的影响]

建议: [Agent 基于行业经验给出的推荐，说明理由]
```

## 决策记录机制

### 记录到 Memory

每个用户决策都必须写入 memory，避免重复提问同一问题。决策记录格式：

```
[决策]
问题: [原始问题]
选择: [用户的回答]
日期: [ISO 日期时间]
上下文: [相关模块/功能/场景]
影响范围: [此决策影响的功能模块列表]
```

### 避免重复提问

在提出新问题前，先检查 memory 中是否已有相关决策：

1. 搜索 memory 中相同或类似的问题
2. 如果已存在 → 使用已有决策，不重复提问
3. 如果上下文已变化（如业务调整）→ 确认用户是否需要更新之前的决策
4. 全新问题 → 按 AskUserQuestion 规范提问

## 反模式（禁止行为）

以下行为是明确禁止的：

### 不要问技术选择

- 不要在 "用 go-zero 还是 gin" 上问用户 → 去查 `storeforge-domain-backend`
- 不要在 "金额用 float 还是 int64" 上问用户 → harness 已指定 int64
- 不要在 "用 Provider 还是 Riverpod" 上问用户 → 去查 `storeforge-domain-flutter`
- 不要在 "订单状态怎么设计" 上问用户 → 去查 `knowledge/ecommerce-patterns/order-state-machine.md`

### 不要用开放式问题问产品

- 不要问 "你想做什么样的促销？" → 给出具体的促销方案选项
- 不要问 "你想要什么功能？" → 给出 MVP 范围建议
- 不要问 "你觉得用户体验怎么设计？" → 给出 UI/UX 方案选项

### 不要频繁打断

- 积累多个产品问题后，**批量**用 AskUserQuestion 询问，而不是每遇到一个问题就打断用户
- 建议在脑暴阶段（`storeforge-brainstorm`）或计划阶段（`storeforge-writing-plans`）集中收集所有产品问题
- 一次性列出所有需要用户决策的事项，附带选项和影响评估

## 与 storeforge-brainstorm 的协同

`storeforge-brainstorm` 在技术方案分析阶段，会识别出所有需要用户决策的产品问题。最佳实践：

1. 调用 `storeforge-brainstorm` 分析需求
2. 脑暴输出中包含"需用户决策事项清单"
3. 集中向用户展示清单，批量获取决策
4. 所有决策记录到 memory
5. 调用 `storeforge-writing-plans` 时，将已确认的决策写入计划文档
