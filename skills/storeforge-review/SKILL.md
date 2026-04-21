---
name: storeforge-review
description: 多代理代码审查，5 个角度自循环 review
---

# 多代理代码审查

> 并行启动 5 个子代理进行自循环 review，最多 5 轮，超限升级至用户。`/sf:review` 或 `storeforge-executing` 计划执行完成后自动触发。

## 定位

本 skill 定义了 StoreForge 的多代理代码审查机制。通过并行调用 5 个专门的审查子代理（架构、安全、性能、API 契约、测试验证），对当前 commit 变更进行多角度深度审查，并在发现问题后自动进入修复 → 再审查的自循环流程。

**触发方式**：
- 用户手动调用 `/sf:review`
- `storeforge-executing` 在所有计划任务执行完成后自动调用

**审查范围**：仅审查**本次 commit 变更的文件**，不审查全量代码。通过 `git diff` 获取变更文件列表，传递给每个子代理。

---

## 审查流程

```
工作成果（代码/文档）
  ↓
获取本次 commit 的 git diff 和变更文件列表
  ↓
并行启动 5 个子代理（isolation: worktree）：
  - storeforge-architect        → 架构是否合理？状态机是否闭环？
  - storeforge-security-auditor → 有无安全漏洞？支付验签是否完整？
  - storeforge-performance-reviewer → 有无 N+1？热点是否走缓存？
  - storeforge-api-contract-checker → 前后端字段是否对齐？
  - storeforge-testing-validator   → 测试覆盖是否达标？
  ↓
汇总结果，去重，按严重程度排序
  ↓
Agent 逐一修复问题
  ↓
再次并行启动 5 个子代理（新一轮）
  ↓
没有新问题 → review 通过
有新问题 → 修复 → 再 review（最多 5 轮）
5 轮后仍有新问题 → 升级至用户
```

---

## 子代理调用协议

### 调用方式

使用 Claude Code 的 `Agent` 工具，每个 agent 独立并行调用：

- `isolation: "worktree"` — 每个 agent 在隔离的工作区中审查
- `subagent_type: "general-purpose"`
- 传入 prompt 包含：
  1. 当前 commit 的 git diff（变更内容）
  2. 该 agent 专用的检查清单
  3. 输出格式要求

**并行调用**：5 个 agent 在**同一消息中**并行调用（多次 `Agent` 工具调用），不等待前一个完成。

### 修复后流程

1. Agent 根据 review 结果逐一修复问题（**修复保留在 worktree 中，不合并到主分支**）
2. 再次并行调用全部 5 个 agent（新一轮）
3. 对比新旧结果，只关注**新增问题**
4. 所有轮次通过后，一次性合并 worktree 变更到主分支

---

## 5 个 Agent 的详细检查清单

### Agent 1：storeforge-architect（架构审查）

**角色**：资深电商架构师，审查代码的架构合理性和分布式设计正确性。

| # | 检查项 | 详细说明 |
|---|--------|---------|
| A1 | 状态机闭环 | 订单状态机是否有完整的入口和出口？每个状态都有合法的转换路径？无悬空状态？ |
| A2 | 幂等设计 | 支付回调、库存扣减、订单创建等写操作是否有幂等键（`X-Idempotency-Key` 或业务幂等键）？同一请求重复调用不产生副作用？ |
| A3 | 防超卖机制 | 库存扣减是否使用 Redis Lua 脚本 + PostgreSQL 事务双写 + 一致性补偿？ |
| A4 | 模块边界 | 内部服务是否跨层调用？各模块（user/product/order/payment/promotion）边界是否清晰？ |
| A5 | Saga 编排 | 跨服务操作（如"下单"涉及订单+库存+促销）是否使用 Saga 编排模式？补偿事务是否完整？ |
| A6 | Outbox 同事务 | 事件发出是否通过 Outbox 表与业务操作同事务写入？relay 是否正确投递？ |
| A7 | RPC 非 HTTP | 服务间通信是否走 protobuf RPC，而非 HTTP 直接调用？ |

**阻断条件**：状态机不闭环、无幂等设计、无防超卖机制。

### Agent 2：storeforge-security-auditor（安全审查）

**角色**：安全审计专家，识别代码中的安全漏洞和合规风险。

| # | 检查项 | 详细说明 |
|---|--------|---------|
| S1 | 支付回调验签 | 微信 RSA/SHA256、支付宝 RSA2、抖音签名是否完整？nonce + timestamp 防重放是否实现？HTTPS-only？ |
| S2 | 敏感信息加密 | 密码 bcrypt、手机号 AES-256-GCM、地址加密、身份证号独立密钥？展示时脱敏？ |
| S3 | 金额精度 | 金额是否 `int64`（分）+ `ROUND_HALF_UP`，**禁止** float？ |
| S4 | SQL 注入 | GORM 是否参数化查询？无 `fmt.Sprintf` 拼接条件？ |
| S5 | XSS/CSRF | CSP headers、SameSite cookies 是否配置？ |
| S6 | SSRF | 图片 URL 是否有白名单/公网 IP 校验？ |
| S7 | JWT 安全 | RS256/ES256、有过期、有撤销机制？refresh token 存 HttpOnly cookie？ |
| S8 | Rate Limit | 登录/SMS/秒杀/领券接口是否配置 rate limit？防爆破？ |
| S9 | Response DTO | 是否使用显式 Response DTO 过滤敏感字段？禁止直接返回 GORM model？ |
| S10 | OAuth state | OAuth 流程是否携带 `state` 参数防 CSRF？ |

**阻断条件**：支付不验签、金额用 float、JWT 无撤销、敏感信息明文存储。

### Agent 3：storeforge-performance-reviewer（性能审查）

**角色**：性能优化专家，识别代码中的性能瓶颈和扩展性隐患。

| # | 检查项 | 详细说明 |
|---|--------|---------|
| P1 | N+1 查询 | 商品列表等批量查询是否有 N+1 问题？是否使用 `Preload` 或手动 JOIN？ |
| P2 | Redis 缓存 | 热点接口（商品详情、首页）是否走 Redis 缓存？ |
| P3 | CDN | 图片等静态资源是否 CDN 加速？ |
| P4 | 分页限制 | 所有列表接口是否有分页限制（page/pageSize, max=100）？ |
| P5 | 批量操作 | 管理后台是否有批量操作接口（批量上架/下架等）？ |
| P6 | Redis 缓存防护 | 缓存是否有 singleflight / 互斥锁 / 概率提前过期（stale-while-revalidate + jitter）防止缓存击穿？ |
| P7 | PG 连接池 | 是否显式配置 `MaxOpenConns=CPU*2-4`、`MaxIdleConns=CPU`、`ConnMaxLifetime`？ |

**阻断条件**：N+1 查询、列表接口无分页、热点接口无缓存。

### Agent 4：storeforge-api-contract-checker（API 契约检查）

**角色**：API 契约审查员，确保前后端接口一致性和类型安全。

| # | 检查项 | 详细说明 |
|---|--------|---------|
| C1 | 字段对齐 | 前后端 API 字段名称和类型是否完全一致？（基于 OpenAPI 生成的 TypeScript/Dart 类型验证） |
| C2 | 枚举对齐 | 枚举值是否从 `event/` protobuf 统一来源生成？跨端一致？ |
| C3 | 错误码一致 | 错误码是否统一使用 `{MODULE}_{SEQ}` 命名空间，无冲突？ |
| C4 | 分页一致 | 分页参数是否统一使用 page/pageSize？response 是否包含 total？ |
| C5 | 必填/可选对齐 | 前后端必填/可选字段是否完全对齐？ |
| C6 | 日期格式 | 日期时间格式是否统一使用 RFC 3339？ |
| C7 | 金额转换 | 前端是否正确处理 int64（分）到元的转换？ |
| C8 | BaseResponse 信封 | 所有接口是否正确使用 `BaseResponse<T>` 信封？ |

**阻断条件**：字段名/类型不一致、错误码冲突、分页参数缺失。

### Agent 5：storeforge-testing-validator（测试验证）

**角色**：测试质量审查员，确保测试覆盖达标且无测试反模式。

| # | 检查项 | 详细说明 |
|---|--------|---------|
| T1 | 单元测试覆盖 | 核心逻辑是否有单元测试？覆盖率是否达标（见 `storeforge-testing`）？ |
| T2 | 集成测试路径 | 关键路径（登录 → 下单 → 支付）是否有集成测试？ |
| T3 | 异常场景 | 异常场景是否覆盖（库存不足、支付超时、无效 Token、参数校验失败）？ |
| T4 | 测试数据无硬编码 | 测试数据是否使用 UUID 生成？无硬编码时间/ID/Token？ |
| T5 | Race condition 测试 | 是否使用 `-race` 标志通过？并发场景测试覆盖？ |
| T6 | 幂等性测试 | 同一请求重复调用是否只产生一次副作用？ |
| T7 | 测试反模式 | 无精确时间戳断言？无列表顺序依赖（先排序再比较）？无无重试的异步断言（使用 Eventually/retry）？ |

**覆盖率标准**（来自 `storeforge-testing`）：
- Go：`go test -race -coverprofile` ≥ 80%
- Vue3 Admin：`vitest` ≥ 70%
- Next.js Website：`vitest` ≥ 80%
- Flutter：`flutter test` ≥ 70%

**阻断条件**：核心逻辑无测试、覆盖率不达标、测试数据硬编码。

---

## 结果汇总格式

每轮审查完成后，输出以下格式的汇总报告：

```
## Review Round N

### 汇总统计
- 严重：X 项
- 中等：Y 项
- 建议：Z 项
- 本轮新增（相比上轮）：N 项

### 问题清单

- [严重] storeforge-architect: 订单状态机缺少 paid → refunding 的转换路径 @ internal/logic/order_fsm.go:142
- [严重] storeforge-security-auditor: 支付回调未验证签名，存在伪造风险 @ internal/handler/payment_callback.go:67
- [中等] storeforge-performance-reviewer: 商品列表查询存在 N+1，未使用 Preload 加载 SKU @ internal/logic/product_list.go:38
- [中等] storeforge-api-contract-checker: 响应字段 productName 与前端期望 name 不一致 @ api/product.api:23
- [建议] storeforge-testing-validator: OrderCreate 逻辑缺少单元测试 @ internal/logic/order_create_test.go（不存在）
- [建议] storeforge-architect: Saga 补偿事务缺少库存回滚幂等检查 @ internal/saga/order_saga.go:89
```

严重程度定义：
- **[严重]**：BLOCK 级别，必须修复后才能通过 review（安全漏洞、架构缺陷、数据一致性风险）
- **[中等]**：WARN 级别，应当修复但不会阻断核心功能（性能问题、契约不一致）
- **[建议]**：优化建议，不修复也可通过 review 但会降低代码质量

---

## 多轮审查自循环

### Round 流程

```
Round 1：5 个 agent 并行审查 → 汇总 → 去重 → 排序 → 展示
  ↓
Agent 逐一修复所有「严重」和「中等」问题（修复保留在 worktree，不合并）
  ↓
Round 2：5 个 agent 再次并行审查（相同流程）
  ↓
对比 Round 1 结果 → 仅关注新增问题（Round 1 已修复的不再报告）
  ↓
有新增问题 → 修复 → Round 3
无新增问题 → 一次性合并 worktree 变更到主分支 → Review 通过
```

### 去重规则

- 同一问题被多个 agent 同时发现 → 只保留一次，标注所有发现的 agent
- 同一问题的不同表现形式 → 合并为一条
- 已修复的问题 → 后续轮次不再报告

### 轮次上限

**最多 5 轮**。5 轮后仍有未解决的新问题 → 升级至用户。

---

## 升级机制

5 轮审查后仍有未解决问题 → 生成升级报告：

```
## Review 升级报告

### 未解决问题

- [严重] storeforge-architect: [问题描述] @ [file:line]
  - Round 2 首次发现
  - 已尝试修复方案：[描述]
  - 仍存在问题：[描述]
- [中等] storeforge-performance-reviewer: [问题描述] @ [file:line]
  - Round 3 首次发现
  - ...

### 影响评估

- 未解决的严重问题：[说明对系统的影响，如"可能导致库存超卖"]
- 未解决的中等问题：[说明对系统的影响]
- 整体风险评估：[高/中/低]

### 建议

- 建议 1：[具体修复建议]
- 建议 2：[替代方案]
- 是否可跳过某些问题继续交付：[说明]
```

---

## 与其他 Skill 的协作

- **`storeforge-executing`**：所有计划任务执行完成后自动触发本 review
- **`storeforge-testing`**：testing-validator agent 的检查标准引用 `storeforge-testing` 的覆盖率要求
- **`storeforge-harness`**：security-auditor 和 architect agent 的检查清单引用 `storeforge-harness` 的 BLOCK 反模式
- **`storeforge-verification`**：review 通过后，交付前再次触发 `storeforge-verification` 进行最终验证
