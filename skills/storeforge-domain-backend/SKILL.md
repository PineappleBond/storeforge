---
name: storeforge-domain-backend
description: Golang + go-zero 硬约束和最佳实践
---

# Backend 硬约束 — Golang + go-zero

> 本文档定义 StoreForge 后端开发的不可变硬约束。违反反模式清单中任何一条 = BLOCK（阻断），必须修复后方可继续。
> 约束引擎由 `storeforge-harness` 驱动，违反规则时立即停止当前工作。

---

## 项目结构（固定，不可变）

目录树固定如下，每层职责明确，不得随意增减顶层目录：

```
api/
├── user.api          # 用户模块（注册/登录/资料/OAuth）
├── product.api       # 商品模块（CRUD/SKU/分类/搜索）
├── order.api         # 订单模块（创建/状态/退款）
├── payment.api       # 支付模块（微信/支付宝/抖音/退款）
└── promotion.api     # 营销模块（优惠券/满减/限时折扣/拼团）
rpc/
├── user/             # 用户 RPC 服务
├── product/          # 商品 RPC 服务
├── order/            # 订单 RPC 服务
├── payment/          # 支付 RPC 服务
└── promotion/        # 营销 RPC 服务
event/                # 事件定义（protobuf），所有领域枚举的唯一来源
model/                # GORM models，每模块独立目录
internal/
├── config/           # 配置加载（YAML/环境变量）
├── handler/          # .api 自动生成，禁止手写
├── logic/            # 业务逻辑实现
├── svc/              # 服务上下文（DB/Redis/MQ 连接初始化）
└── middleware/       # 中间件（auth、rate-limit、trace）
```

**各目录职责**：
- `api/`：仅存放 `.api` 描述文件，定义路由、请求/响应结构，通过 `goctl` 生成 handler 骨架
- `rpc/`：各微服务的 protobuf 定义 + gRPC 服务端实现，服务间通信唯一通道
- `event/`：protobuf 事件消息定义，Go 常量 + TypeScript enum + Dart enum 的唯一来源
- `model/`：GORM 模型定义，包含表结构映射、钩子、关联关系
- `internal/config/`：配置结构体 + 加载逻辑，支持环境变量覆盖
- `internal/handler/`：由 `goctl api gen` 自动生成，**禁止手动编辑**
- `internal/logic/`：业务逻辑，handler 仅做参数校验后委托给 logic
- `internal/svc/`：ServiceContext，统一管理 DB/Redis/MQ/gRPC 客户端
- `internal/middleware/`：可复用中间件（JWT 鉴权、限流、链路追踪）

---

## 技术栈约束

### 必须使用

| 类别 | 技术 | 说明 |
|------|------|------|
| 语言 | Golang 1.21+ | 统一使用泛型特性 |
| 框架 | go-zero | API 网关 + RPC 框架 |
| ORM | GORM | 禁止 raw SQL（见反模式 #2） |
| 数据库 | PostgreSQL 15+ | 主存储 |
| 缓存 | Redis 7+ | 缓存/分布式锁/幂等键/库存预扣 |
| 消息队列 | RabbitMQ / Kafka / NATS | 服务间异步通信 |
| RPC | protobuf + gRPC | 服务间同步通信 |
| 代码生成 | goctl + protoc | handler 和 protobuf 代码生成 |

### 禁止使用

| 禁止项 | 原因 |
|--------|------|
| gin / echo / fiber | 非 go-zero 体系，破坏统一约束 |
| raw SQL（除 migration 和明确标注的 JOIN 优化） | 绕过 GORM 安全机制，SQL 注入风险 |
| HTTP 调用内部服务 | 必须走 protobuf RPC，HTTP 仅用于外部接口 |
| float / double 存储金额 | 精度丢失，电商场景致命 |

---

## 推荐第三方库

**原则：优先使用 go-zero 内置能力（见下方 "go-zero 内置能力" 章节），内置无解决方案时才使用第三方库。**

### 必选库（不可缺失）

| 用途 | 推荐库 | 导入路径 |
|------|--------|----------|
| JWT 签发与验证 | golang-jwt | `github.com/golang-jwt/jwt/v5` |
| OAuth 2.0（微信/支付宝登录） | golang.org/x/oauth2 | `golang.org/x/oauth2` |
| 微信支付 | wechatpay-go（官方） | `github.com/wechatpay-apiv3/wechatpay-go` |
| 支付宝/抖音支付 | gopay | `github.com/go-pay/gopay` |
| RBAC 权限引擎 | casbin | `github.com/casbin/casbin/v2` |
| Casbin GORM 适配器 | casbin-gorm-adapter | `github.com/casbin/gorm-adapter/v3` |
| 密码哈希 | golang.org/x/crypto | `golang.org/x/crypto/bcrypt` |
| 参数校验 | go-playground/validator | `github.com/go-playground/validator/v10` |
| 状态机 | fsm | `github.com/looplab/fsm`（go-zero 无内置状态机，此为指定例外） |
| 单飞（缓存击穿防护） | singleflight | `golang.org/x/sync/singleflight` |

### 推荐库（按需使用）

| 用途 | 推荐库 | 导入路径 |
|------|--------|----------|
| HTTP 客户端（外部 API） | go-resty | `github.com/go-resty/resty/v3` |
| 重试与退避 | backoff | `github.com/cenkalti/backoff/v4` |
| 限流（Redis 分布式） | redis_rate | `github.com/go-redis/redis_rate/v10` |
| 分布式锁 | redsync | `github.com/go-redsync/redsync/v4` |
| 定时任务 | cron | `github.com/robfig/cron/v3` |
| 异步任务 | asynq | `github.com/hibiken/asynq` |
| 精确十进制计算 | decimal | `github.com/shopspring/decimal` |
| AI Agent 框架 | eino | `github.com/cloudwego/eino` |
| OpenAI 接入 | eino-ext model | `github.com/cloudwego/eino-ext/components/model/openai` |
| 全文搜索（本地） | bleve | `github.com/blevesearch/bleve/v2` |
| 搜索（Elasticsearch） | elasticsearch-go | `github.com/elastic/go-elasticsearch/v8` |
| OLAP 查询 | clickhouse-go | `github.com/ClickHouse/clickhouse-go/v2` |
| UUID 生成 | uuid | `github.com/google/uuid` |
| Excel 导入导出 | excelize | `github.com/xuri/excelize/v2` |
| PDF 生成 | gofpdf | `github.com/jung-kurt/gofpdf` |
| 图片处理 | imaging | `github.com/disintegration/imaging` |
| HTML 过滤 | bluemonday | `github.com/microcosm-cc/bluemonday` |
| 手机号校验 | phonenumbers | `github.com/nyaruka/phonenumbers` |
| 规则引擎 | expr | `github.com/expr-lang/expr` |
| IP 地理位置 | geoip2 | `github.com/oschwald/geoip2-golang` |
| 布隆过滤器 | bloom | `github.com/bits-and-blooms/bloom/v3` |
| 国际化 | go-i18n | `github.com/nicksnyder/go-i18n/v2` |
| 本地限流 | time/rate | `golang.org/x/time/rate` |
| 邮件发送 | gomail | `gopkg.in/gomail.v2` |
| WebSocket | gorilla/websocket | `github.com/gorilla/websocket` |

**原则：能复用库的绝不手写。特别是支付、OAuth、限流、重试、加密等安全和精度敏感场景。**

---

## 代码规范

### Handler 生成规则
- Handler 必须由 `.api` 文件通过 `goctl api gen` 自动生成
- 禁止手写 handler 代码
- Handler 职责：解析请求参数 -> 调用 logic -> 返回响应

### 分页规范
- 所有列表接口必须支持 `page` / `pageSize` 参数
- 默认值：`page=1`, `pageSize=20`
- 最大值：`pageSize` 不得超过 100
- 响应体使用 `PaginatedData<T>` 封装

### 统一响应格式
- 所有接口（包括列表、详情、写操作）必须返回 `BaseResponse<T>`
- 结构定义见下方代码示例
- 列表接口中 `Data` 字段为 `PaginatedData<T>` 类型

```go
type BaseResponse[T any] struct {
    Code      int    `json:"code"`      // 0=成功, 正数=业务错误
    Message   string `json:"msg"`       // 中文错误信息
    Data      T      `json:"data"`      // 数据体
    RequestID string `json:"requestId"` // 分布式追踪 ID
}

type PaginatedData[T any] struct {
    List     []T   `json:"list"`
    Total    int64 `json:"total"`
    Page     int   `json:"page"`
    PageSize int   `json:"pageSize"`
}
```

### 幂等性规范
- 所有写操作（POST/PUT/DELETE）必须支持幂等
- 客户端通过 `X-Idempotency-Key` header 传入幂等键
- 如无业务幂等键（订单号、支付流水号），必须使用 `X-Idempotency-Key`
- 幂等键存储使用 Redis，TTL=24h
- 幂等键冲突时返回之前的结果（不重复执行）

### API 版本化
- URL 前缀固定为 `/api/v1/...`
- 破坏性变更必须新增版本号（`/api/v2/...`）
- 旧版本必须保留并在响应头中附加 `Deprecation` header
- 禁止原地修改接口签名（删除字段、改变类型、修改必填属性）

### PostgreSQL 连接池
- 禁止使用 GORM 默认连接池配置
- 必须显式配置：
  - `MaxOpenConns = CPU 核心数 * 2 - 4`（最小 4，最大 64）
  - `MaxIdleConns = CPU 核心数`
  - `ConnMaxLifetime = 30 * time.Minute`
  - `ConnMaxIdleTime = 10 * time.Minute`

---

## 反模式清单（违反即阻断）

以下 17 条反模式任何一条被违反，harness 将 BLOCK 当前工作，必须修复后方可继续。

### 反模式 #1：手写 Handler

**禁止**：手动编写 handler 代码。
**必须**：所有 handler 由 `.api` 文件通过 `goctl api gen` 自动生成。
**原因**：手写 handler 无法与 API 契约保持同步，破坏 `goctl` 代码生成链条，导致 API spec 与实际实现不一致。

### 反模式 #2：Raw SQL 与 N+1 查询

**禁止**：
- 使用 raw SQL（除 migration 脚本和性能优化场景中明确标注的 `JOIN`）
- 在循环内执行数据库查询（N+1 问题）
**必须**：使用 GORM model 进行所有数据库操作；批量查询使用 `Preload` 或 `JOIN`。
**原因**：raw SQL 绕过 GORM 参数化机制，引入 SQL 注入风险；N+1 查询导致数据库连接数暴增，高并发下服务雪崩。

### 反模式 #3：HTTP 调用内部服务

**禁止**：使用 HTTP 客户端调用其他内部服务。
**必须**：服务间同步通信使用 protobuf RPC（gRPC）；服务间异步通信必须通过消息队列（RabbitMQ/Kafka/NATS）。
**原因**：HTTP 调用缺乏强类型契约、序列化效率低、无法利用 gRPC 的流式传输和连接复用，服务间耦合度无法保障。

### 反模式 #4：Float 存储金额

**禁止**：使用 `float32`/`float64` 存储或计算金额。
**必须**：金额使用 `int64`（单位：分）；金额计算必须指定舍入规则（`ROUND_HALF_UP`）；多币种场景必须携带 `currency_code`。
**原因**：IEEE 754 浮点数无法精确表示十进制小数，累加操作会累积精度误差，在电商场景导致金额对账失败。

### 反模式 #5：接口不分页

**禁止**：列表接口不提供 `page` / `pageSize` 参数。
**必须**：所有列表接口强制分页，默认 `page=1, pageSize=20`，`pageSize` 上限 100。
**原因**：不分页导致单次查询返回全量数据，数据库内存耗尽、网络带宽打满、前端渲染卡顿。

### 反模式 #6：写操作不幂等

**禁止**：写操作（POST/PUT/DELETE）不携带幂等键。
**必须**：所有写操作通过 `X-Idempotency-Key` header 或业务幂等键（订单号、支付流水号）保证幂等，幂等键存储 Redis TTL=24h。
**原因**：网络超时重试、客户端重复提交、支付回调重复推送都会导致非幂等写操作产生脏数据（重复订单、重复扣款）。

### 反模式 #7：敏感操作无 Audit Log

**禁止**：以下敏感操作不记录 audit log：
- 价格变更
- 订单状态修改
- 退款审批
- 用户角色变更
**必须**：audit log 写入只读追加表（append-only），包含操作人、操作时间、旧值、新值、操作原因。
**原因**：敏感操作无法追溯，合规审计失败，故障排查时缺少操作历史。

### 反模式 #8：库存直接扣减

**禁止**：直接使用 SQL `UPDATE inventory SET stock = stock - N` 扣减库存。
**必须**：
1. Redis Lua 脚本原子扣减（防超卖）
2. PostgreSQL 事务双写（持久化）
3. 一致性补偿（Redis 成功但 PG 失败时回滚 Redis）
4. 库存回滚（订单取消/支付超时）必须幂等
**原因**：直接 SQL 扣减在高并发下产生超卖；Redis + PG 双写保证高性能与数据一致性。

### 反模式 #9：订单状态随意跳转

**禁止**：订单状态从任意状态直接跳转到任意状态。
**必须**：使用状态机管理订单状态转换，每个状态变更必须记录原因和操作人。合法转换路径见 `knowledge/ecommerce-patterns/order-state-machine.md`。
**原因**：随意跳转导致订单生命周期混乱，退款/发货/售后逻辑无法正确执行。

### 反模式 #10：返回格式不统一

**禁止**：接口返回自定义格式或不包裹 `BaseResponse<T>`。
**必须**：所有接口返回 `BaseResponse<T>`，列表接口返回 `BaseResponse<PaginatedData<T>>`。
**原因**：前端无法统一处理响应，错误处理逻辑散落在各处，API 契约无法自动化生成。

### 反模式 #11：跨服务直接 DB 访问

**禁止**：服务 A 直接连接服务 B 的数据库读写数据。
**必须**：服务间数据依赖通过 RPC（同步）或事件（异步，Saga/Outbox 模式）获取。
**原因**：跨服务 DB 访问破坏服务边界，Schema 变更产生连锁反应，数据库成为单点瓶颈。

### 反模式 #12：GORM 字符串拼接构造条件

**禁止**：使用 `fmt.Sprintf` 或字符串拼接构造 GORM 查询条件。
**必须**：所有查询条件使用参数化查询（GORM 的 `Where("field = ?", value)` 形式）。
**原因**：字符串拼接是 SQL 注入的经典路径，参数化查询由数据库驱动处理转义，从根本上杜绝注入。

### 反模式 #13：接口不限流

**禁止**：接口不配置 rate limit。
**必须**：所有接口默认配置全局 rate limit；登录/SMS/秒杀/领券接口必须配置更严格的限制。
**原因**：无限流导致服务被恶意请求打满，正常用户无法访问，系统失去可用性保障。

### 反模式 #14：PG 连接池使用默认配置

**禁止**：使用 GORM 默认的数据库连接池参数。
**必须**：显式配置 `MaxOpenConns=CPU*2-4`、`MaxIdleConns=CPU`、`ConnMaxLifetime`、`ConnMaxIdleTime`。
**原因**：默认配置在高并发下连接数无限增长导致 OOM，或连接数过少导致请求排队超时。

### 反模式 #15：PII 明文存储

**禁止**：以下 PII（个人身份信息）明文存储：
- 手机号（必须 AES-256-GCM 加密）
- 地址（必须加密存储）
- 身份证号（必须使用独立密钥加密）
**必须**：PII 加密存储，展示时脱敏（手机号：`138****1234`）。
**原因**：数据库泄露时明文 PII 直接暴露用户隐私，违反数据安全法规（个人信息保护法），面临巨额罚款。

### 反模式 #16：API 直接返回 GORM Model

**禁止**：将 GORM model 直接作为 API 响应返回。
**必须**：使用显式 Response DTO（Data Transfer Object），在 DTO 层过滤敏感字段（密码、加密密钥、内部状态）。
**原因**：GORM model 包含内部字段（密码哈希、软删除标记、关联 ID），直接返回导致敏感信息泄露和 API 契约不稳定。

### 反模式 #17：热点缓存无防护过期

**禁止**：热点缓存（商品详情、首页配置）设置固定 TTL 后无防护地过期。
**必须**：使用以下至少一种防护机制：
- `singleflight`（Go 标准库 `golang.org/x/sync/singleflight`）合并并发请求
- 互斥锁防止缓存击穿
- 概率提前过期（stale-while-revalidate + jitter）防止同一时刻大量缓存同时失效
**原因**：热点缓存同时过期导致缓存击穿，大量请求直接打到数据库，引发雪崩。

---

## 特殊 API 处理

### 文件上传
- 使用 `multipart/form-data` 接收文件
- 上传后异步处理（压缩、格式转换、CDN 上传）
- 立即返回 `202 Accepted`，异步完成后返回 CDN URL
- 图片 URL 必须经过 SSRF 防护（白名单/公网 IP 校验）

### 实时通知
- 管理后台监控：WebSocket 长连接
- 订单状态推送（客户端）：SSE（Server-Sent Events）
- 连接断开自动重连，消息支持去重

### 外部 Webhook
- ERP 对接、物流回调等外部系统调用
- 必须使用 HMAC 签名验证请求来源
- 携带 `X-Timestamp` 和 `X-Nonce` 防重放攻击
- Webhook 处理必须幂等（同反模式 #6）

---

## 分布式事务

### Saga 编排模式

跨服务操作（如"下单"涉及订单 + 库存 + 促销 + 支付）采用 Saga 编排模式：

```
Order Service（编排器）
  └─> 发出 OrderCreated 事件
       ├─> Inventory Service: 预扣库存 → InventoryReserved / InventoryFailed
       ├─> Promotion Service: 计算优惠 → PromotionApplied / PromotionFailed
       └─> 任何步骤失败 → 触发补偿事务
            ├─> ReleaseInventory（释放预扣库存）
            └─> RestorePromotion（恢复优惠券/优惠额度）
```

**关键原则**：
- 每个 Saga 步骤必须有对应的补偿操作
- 补偿操作必须幂等
- 编排器记录 Saga 执行状态，支持断点恢复

### Outbox + Relay 模式

保证事件不丢失的可靠投递机制：

1. 业务操作与事件写入本地 `outbox` 表（同一数据库事务）
2. Relay 进程扫描 `outbox` 表，投递到消息队列
3. 投递成功后标记 `outbox` 记录为已发送
4. Relay 支持重试，保证至少一次投递

```sql
-- outbox 表结构
CREATE TABLE outbox (
    id          BIGSERIAL PRIMARY KEY,
    event_type  VARCHAR(64) NOT NULL,
    payload     JSONB NOT NULL,
    status      VARCHAR(16) DEFAULT 'pending',
    retries     INT DEFAULT 0,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    sent_at     TIMESTAMPTZ
);
```

### 延迟队列与死信队列

- **延迟队列**：订单超时取消（30 分钟未支付自动取消）
- **死信队列**：消费失败超过重试次数的事件转入死信队列，人工介入处理
- **消费者幂等**：每个消费者通过 `X-Idempotency-Key` 保证重复消费不产生副作用

### 事件定义

- 所有事件定义在 `event/` 目录的 protobuf 文件中
- 事件类型使用 protobuf `enum` 定义，保证唯一性
- 通过 `protoc` 生成 Go 常量 + TypeScript enum + Dart enum，保证跨端一致

---

## 模式推荐

| 场景 | 推荐方案 | 详细说明 |
|------|----------|----------|
| 金额计算 | `int64`（分）+ `ROUND_HALF_UP` | 禁止 float；多币种带 `currency_code` |
| 库存扣减 | Redis Lua + PG 事务 + 一致性补偿 | Lua 脚本原子扣减防超卖，PG 持久化，失败时补偿 |
| 跨服务操作 | Saga 编排 + Outbox | 每个步骤有补偿，outbox 保证事件不丢失 |
| 缓存热点 | `singleflight` + stale-while-revalidate + jitter | 防止缓存击穿和同时过期 |
| 支付回调 | 幂等键 + Redis SET NX | 同一回调重复推送只处理一次 |
| 订单状态 | 严格状态机 | 参见 `knowledge/ecommerce-patterns/order-state-machine.md` |

### go-zero 内置能力（优先使用）

**必须优先使用 go-zero 框架内置能力，禁止手写等价的通用方案：**

#### 存储层

| go-zero 能力 | 用途 | 说明 |
|-------------|------|------|
| `core/stores/redis` | Redis 客户端 | 内置连接池、命令封装、Sentinel/Cluster 支持 |
| `core/stores/cache` | 缓存管理 | Cache 结构体 + 统计 + 防穿透 + `singleflight` |
| `core/stores/sqlc` | 带缓存 SQL 层 | 查询自动缓存、写入自动失效、key 自动生成 |
| `core/stores/sqlx` | 数据库连接 | 连接池、事务、慢查询日志、PostgreSQL/MySQL 驱动 |
| `core/stores/kv` | KV 存储 | 抽象 KV 操作，支持 Redis 切换底层存储 |
| `core/stores/mon` | MongoDB | 内置 MongoDB 连接 + 连接池管理 |

#### 计算层

| go-zero 能力 | 用途 | 说明 |
|-------------|------|------|
| `core/mr` (MapReduce) | 并发数据处理 | `MapReduce` / `MapFinish` / `Finish` 替代手写 goroutine + channel |
| `core/executors` | 延迟/批量执行器 | `ChunkExecutor` / `DelayExecutor` / `PeriodicalExecutor` |
| `core/search` (权重) | 推荐排序 | `TreeBasedWeight` 等权重搜索，非全文搜索 |

#### 可靠性

| go-zero 能力 | 用途 | 说明 |
|-------------|------|------|
| `core/breaker` | 熔断器 | Google SRE 算法，自动 tripping + half-open 恢复 |
| `core/limit` | 分布式限流 | `TokenLimit` / `PeriodLimit`，支持 Redis 分布式 |
| `core/load` | 负载保护 | adaptive shedding，CPU 过载时自动拒绝 |
| `core/retry/retry` | 重试策略 | 内置重试机制 |
| `core/proc` | 优雅关闭 | 信号监听 + 超时关闭 + 并发安全 |

#### 可观测性

| go-zero 能力 | 用途 | 说明 |
|-------------|------|------|
| `core/logx` | 结构化日志 | JSON 日志 + 级别控制 + trace ID 注入 + 日志轮转 |
| `core/trace` | 链路追踪 | OpenTelemetry 原生支持 + Jaeger/Zipkin |
| `core/stat` | 指标统计 | Prometheus metrics + 内置计数器/直方图 |
| `core/prof` | 性能分析 | CPU/Memory profiling，自动触发 |

#### 网络层

| go-zero 能力 | 用途 | 说明 |
|-------------|------|------|
| `rest.Server` | HTTP 服务 | 路由注册 + 中间件 + CORS + TLS + graceful shutdown |
| `zrpc` | gRPC 服务 | 内置负载均衡 + 熔断 + 链路追踪 + 服务发现 |
| `rest/httpc` | HTTP 客户端 | 内置服务发现 + 重试 + 超时 + 链路追踪 |
| `core/discov` | 服务发现 | etcd/consul 服务注册 + 发现 + 健康检查 |

#### 工具层

| go-zero 能力 | 用途 | 说明 |
|-------------|------|------|
| `core/mapping` | 配置映射 | 从 JSON/YAML 映射到 Go struct，支持 default/validate |
| `core/conf` | 配置加载 | 文件加载 + 环境变量覆盖 + 热更新 |
| `core/collection` | 数据结构 | RingBuffer / Cache / TimingWheel / 线程安全集合 |
| `core/stringx` | 字符串工具 | 字符串过滤 + 随机生成 + 截取 |
| `core/lang` | 函数式工具 | Map / Filter / Reduce 等函数式编程 |
| `core/timex` | 时间工具 | 超时控制 + 时间轮 + 倒计时 |
| `core/fs` | 文件工具 | 文件操作 + 目录遍历 + 临时文件 |
| `core/hash` | 哈希工具 | 一致性哈希 + 分布式哈希环 |
| `core/naming` | 命名服务 | 服务命名 + 路由策略 |
| `goctl` | 代码生成 | API/RPC/Model 代码生成 + Docker/K8s 模板 |

**示例 — 使用 go-zero MapReduce 处理批量任务：**
```go
import "github.com/zeromicro/go-zero/core/mr"

// 替代手写 goroutine + channel + WaitGroup
func BatchProcess(ctx context.Context, skuIDs []int64) ([]Result, error) {
	return mr.MapReduce(func(source chan<- int64) {
		for _, id := range skuIDs {
			source <- id
		}
	}, func(ctx context.Context, id int64) (Result, error) {
		// 并发处理每个 SKU
		return processSKU(ctx, id)
	}, func(ctx context.Context, pipe <-chan Result, writer chan<- Result) {
		// 合并结果
		for r := range pipe {
			writer <- r
		}
	})
}
```

**示例 — 使用 go-zero Cache 替代手写 Redis 缓存：**
```go
import "github.com/zeromicro/go-zero/core/stores/sqlc"

// 替代手写: if redis.Get == nil { db.Query; redis.Set }
func GetProduct(db *sqlc.CachedConn, id int64) (*Product, error) {
	var p Product
	err := db.QueryRow(&p, id, func(v interface{}) error {
		return db.QueryRowCtx(context.Background(), &p, "SELECT id, name FROM products WHERE id = ?", id)
	})
	return &p, err
}
```

### 决策树

```
遇到金额计算？
  └─> int64（分）+ ROUND_HALF_UP
遇到库存操作？
  └─> Redis Lua 原子扣减 → PG 事务 → 一致性补偿
遇到跨服务调用？
  └─> 同步：protobuf RPC
  └─> 异步：MQ + Outbox + Saga 编排
遇到热点缓存？
  └─> singleflight + stale-while-revalidate + jitter 过期
遇到列表接口？
  └─> 必须分页 page/pageSize + BaseResponse<PaginatedData<T>>
遇到写操作？
  └─> 必须幂等 X-Idempotency-Key + Redis TTL=24h
```

---

## 常见坑

### GORM N+1 查询
- **现象**：查询列表后循环查询关联数据（如查商品列表后循环查 SKU），导致 N+1 次数据库查询
- **解决**：使用 `Preload` 预加载关联数据，或在 SQL 层使用 `JOIN` 一次性查询
- **代码示例**：
  ```go
  // 错误：N+1
  products := []Product{}
  db.Find(&products)
  for i := range products {
      db.Where("product_id = ?", products[i].ID).Find(&products[i].SKUs)
  }

  // 正确：Preload
  db.Preload("SKUs").Find(&products)
  ```

### 支付回调重复
- **现象**：支付网关在网络抖动时重复推送同一笔支付回调
- **解决**：使用幂等键 + `Redis SET NX` 保证同一回调只处理一次；处理前检查订单状态是否已经是已支付
- **关键**：先检查幂等性，再处理业务逻辑

### 库存超卖
- **现象**：并发扣减库存时，多个请求同时读取到相同库存值，导致实际扣减后库存为负
- **解决**：Redis Lua 脚本原子扣减，Lua 脚本内判断库存是否充足，不足直接返回错误
- **关键**：Lua 脚本在 Redis 中是原子执行的，不存在并发竞争

### 订单状态混乱
- **现象**：订单从 `pending` 直接跳到 `delivered`，跳过 `paid` 和 `shipped`
- **解决**：使用严格状态机，每个状态转换必须经过合法路径；状态变更前验证当前状态 + 目标状态的合法性
- **参考**：`knowledge/ecommerce-patterns/order-state-machine.md`

---

## 引用

本 skill 依赖以下知识模式文件，在实现相关功能时必须查阅：

| 文件 | 路径 | 用途 |
|------|------|------|
| 订单状态机 | `knowledge/ecommerce-patterns/order-state-machine.md` | 订单状态转换合法路径、超时处理、状态变更审计 |
| 支付模式 | `knowledge/ecommerce-patterns/payment-patterns.md` | 微信/支付宝/抖音支付对接、回调验签、退款流程 |
| 库存模式 | `knowledge/ecommerce-patterns/inventory-patterns.md` | Redis Lua 库存扣减、PG 双写、防超卖、秒杀预扣 |
| 认证模式 | `knowledge/ecommerce-patterns/auth-patterns.md` | JWT RS256 + refresh token、OAuth 流程、token 撤销 |
| 促销模式 | `knowledge/ecommerce-patterns/promotion-patterns.md` | 优惠券/满减/限时折扣/拼团、优惠叠加规则、预算控制 |
| 物流模式 | `knowledge/ecommerce-patterns/shipping-patterns.md` | 物流对接、运单创建、物流追踪、电子面单 |
| 购物车 | `knowledge/ecommerce-patterns/cart-patterns.md` | 购物车增删改、离线同步、价格变动检测 |
| 秒杀 | `knowledge/ecommerce-patterns/flash-sale-pattern.md` | 高并发秒杀、Redis 预分配、MQ 异步下单、降级策略 |
| 图片 CDN | `knowledge/ecommerce-patterns/image-cdn-pattern.md` | 图片上传、CDN 分发、缩略图生成、WebP 转换 |
| 文件存储 | `knowledge/ecommerce-patterns/file-system.md` | MinIO/OSS 对象存储、分片上传、预签名 URL、文件类型检测 |
| 错误码 | `knowledge/ecommerce-patterns/error-codes-registry.md` | 统一错误码定义、前后端共享、错误映射 |
| 搜索 | `knowledge/ecommerce-patterns/search-patterns.md` | 全文搜索、分词、拼写纠错、搜索建议 |
| 后台权限 | `knowledge/ecommerce-patterns/admin-permission.md` | RBAC 权限、数据权限隔离、操作审计、多租户 |
| 客服系统 | `knowledge/ecommerce-patterns/customer-service.md` | WebSocket 会话、AI 客服分流、工单流转、满意度 |
| AI 能力 | `knowledge/ecommerce-patterns/ai-eino.md` | LLM Wiki 知识库、Tool Calling、智能客服、内容生成 |
| 发票系统 | `knowledge/ecommerce-patterns/invoice-patterns.md` | 电子发票开具、PDF 生成、红冲/作废、税务计算 |
| 数据分析 | `knowledge/ecommerce-patterns/analytics-patterns.md` | 销售报表、漏斗分析、经营看板、Excel 导出 |
| 内容管理 | `knowledge/ecommerce-patterns/cms-i18n.md` | Banner/富文本、多语言、货币转换、图片处理 |
| 风控系统 | `knowledge/ecommerce-patterns/risk-control.md` | 防刷单、设备指纹、黑名单、异常交易检测 |

---

## 与 Harness 的集成

- 本 skill 的 17 条反模式由 `storeforge-harness` 的 BLOCK 级别约束管理
- 在 `storeforge-executing` 的每个子任务完成后自动触发约束检查
- 违反任何反模式 → 阻断 → 要求修复 → 重新检查
- 修复后通过 `storeforge-verification` 完成最终验证
