---
name: storeforge-harness
description: Harness Engineering 核心机制，技术约束引擎
---

# Harness Engineering 核心机制

> 技术约束引擎，不是建议。违反规则 = 阻断当前工作。

## 定位

Harness Engineering 是 StoreForge 的强制约束系统。它定义了一套不可跳过的技术规则，每次代码生成或修改后自动触发检查。约束是**强制执行的**，不是建议或参考。

**核心语义**：违反约束 ≠ 需要 review，违反约束 = **立即阻断当前工作**，要求修复后重新检查，通过后才允许继续。

## 约束执行机制

### 执行流程

```
代码生成/修改
  ↓
Harness 检查当前 domain 的反模式清单
  ↓
是否违反任何反模式？
  ├─ 是 → BLOCK → 立即停止 → 要求修复 → 重新检查
  ├─ 是 → WARN → 建议修改 → 记录到验证清单 → 用户确认后可跳过
  └─ 否 → 继续下一个检查项
```

### 触发时机

Harness 约束检查在以下时机自动触发：

1. **`storeforge-executing` 每个子任务完成后**：开发完成 → harness 检查 → 通过才进入 L1 测试
2. **手动调用 `/sf:verify` 时**：作为完整验证清单的一部分
3. **`storeforge-review` 的子代理审查中**：security-auditor 和 architect 会引用 harness 约束

### 约束级别

| 级别 | 语义 | 处理方式 |
|------|------|---------|
| **BLOCK** | 违反即停止，不允许跳过 | 立即修复，修复后重新检查 |
| **WARN** | 建议修改但允许跳过 | 记录到 `storeforge-verification` 清单，需用户确认后方可跳过 |

> **注**：BLOCK/WARN 命名与 SessionStart hook 的 P0/P1 注入优先级区分，避免术语碰撞。

## 约束加载方式

### 自动加载

当调用 domain skill 时，Harness 自动加载对应 domain 的硬约束：

| Domain Skill | 反模式数量 | 约束文件 |
|-------------|-----------|---------|
| `storeforge-domain-backend` | 17 条 | 后端微服务硬约束 |
| `storeforge-domain-admin` | 8 条 | 管理后台硬约束 |
| `storeforge-domain-website` | 9 条 | 官网硬约束 |
| `storeforge-domain-flutter` | 12 条 | Flutter 多端硬约束 |

### 加载规则

1. 调用 `storeforge-domain-backend` → 自动加载 backend 17 条反模式到 harness 检查清单
2. 调用 `storeforge-domain-admin` → 自动加载 admin 8 条反模式
3. 调用 `storeforge-domain-website` → 自动加载 website 9 条反模式
4. 调用 `storeforge-domain-flutter` → 自动加载 flutter 12 条反模式
5. 多 domain 任务（如同时开发后端和前端）→ 加载所有涉及 domain 的反模式清单，合并检查

### 约束检查结果

检查结果反馈到 `storeforge-verification` 检查清单中：

- BLOCK 违规 → 验证清单中标记为「失败 - 阻断项」
- WARN 违规 → 验证清单中标记为「警告 - 需用户确认」
- 全部通过 → 验证清单中对应项标记为「通过」

## 与各 Domain 的集成

### Backend（storeforge-domain-backend）— 17 条 BLOCK 反模式

| # | 反模式 | 原因 |
|---|--------|------|
| 1 | 手写 handler | handler 必须 `.api` → `goctl` 生成，手写会导致与 .api 定义不一致 |
| 2 | 使用 raw SQL（非 migration/明确 JOIN 优化场景）或循环内数据库查询 | 必须 GORM model；N+1 查询严重性能问题 |
| 3 | HTTP 调用内部服务 | 必须 protobuf RPC；服务间异步通信必须消息队列 |
| 4 | float/double 存储金额 | 必须 int64（分）；浮点数精度丢失导致金额错误 |
| 5 | 接口不分页 | 所有列表必须 page/pageSize，默认 page=1, pageSize=20, max=100 |
| 6 | 写操作不幂等 | 必须携带 `X-Idempotency-Key` 或业务幂等键，Redis TTL=24h |
| 7 | 敏感操作不记录 audit log | 价格变更、订单状态修改、退款审批、用户角色变更必须记录 |
| 8 | 直接 SQL 扣减库存 | 必须 Redis Lua + PG 事务 + 一致性补偿；直接 SQL 无法防超卖 |
| 9 | 订单状态随意跳转 | 必须状态机，每个变更记录原因和操作人 |
| 10 | 接口返回不统一 | 必须 `BaseResponse<T>` 信封 |
| 11 | 跨服务直接 DB 访问 | 必须通过 RPC 或事件（Saga/Outbox） |
| 12 | GORM 字符串拼接构造条件 | 必须参数化查询，防止 SQL 注入 |
| 13 | 接口不限流 | 所有接口必须全局 rate limit，登录/SMS/秒杀/领券更严格 |
| 14 | PG 连接池使用默认配置 | 必须显式设置 MaxOpenConns=CPU*2-4, MaxIdleConns=CPU, ConnMaxLifetime |
| 15 | PII 明文存储 | 手机号 AES-256-GCM、地址加密、身份证独立密钥，展示时脱敏 |
| 16 | API 直接返回 GORM model | 必须使用显式 Response DTO 过滤敏感字段 |
| 17 | 热点缓存无防护过期 | 必须 singleflight/互斥锁/概率提前过期，防止缓存击穿 |

### Admin（storeforge-domain-admin）— 8 条 BLOCK 反模式

| # | 反模式 | 原因 |
|---|--------|------|
| 1 | 引入其他 UI 组件库 | 必须 Element Plus，保证统一性和维护性 |
| 2 | 表格不支持排序/筛选/分页/列宽拖拽 | 管理后台核心交互能力 |
| 3 | 表单不支持动态校验/草稿保存/重置 | 后台操作容错机制 |
| 4 | 订单管理页不嵌入监控 iframe | 运维监控集成 |
| 5 | API 调用不经过 axios interceptor | 必须统一错误处理和 token 刷新 |
| 6 | 商品列表不支持 SKU 维度/批量操作/快速搜索 | 管理效率 |
| 7 | 同步导出大文件 | 必须异步任务，避免阻塞和超时 |
| 8 | 直接操作 DOM | 必须 Vue 响应式，违反组件化原则 |

### Website（storeforge-domain-website）— 9 条 BLOCK 反模式

| # | 反模式 | 原因 |
|---|--------|------|
| 1 | 使用 Pages Router | 必须 App Router，Pages Router 已被弃用 |
| 2 | 手动替换文案 | 必须 next-intl，否则无法维护多语言 |
| 3 | 引入其他 UI 库 | 必须 shadcn/ui，保证统一性 |
| 4 | 页面不配置 metadata 和 OG 标签 | SEO 基础要求 |
| 5 | 商品列表 CSR-only | 必须 SSR + ISR（revalidate < 30s，热点 < 5s） |
| 6 | 客户端直连后端 API | 必须 route handler 代理（含 SSRF 防护） |
| 7 | Lighthouse 性能不达标 | LCP < 2.5s, CLS < 0.1, TTFB < 800ms, FCP < 1.2s |
| 8 | 使用 `any` 类型 | 必须 TypeScript 严格模式 |
| 9 | 安全头缺失 | 必须 CSP, X-Frame-Options, X-Content-Type-Options, HSTS |

### Flutter（storeforge-domain-flutter）— 12 条 BLOCK 反模式

| # | 反模式 | 原因 |
|---|--------|------|
| 1 | 使用 Provider/Bloc/GetX | 必须 Riverpod |
| 2 | `Navigator.push` 手动跳转 | 必须 go_router |
| 3 | 手动写 JSON 解析 | 必须 freezed + json_serializable |
| 4 | `http` 包直接请求 | 必须 Dio + interceptor（token 刷新、重试、日志） |
| 5 | 商品详情页不支持 SKU 选择/多图轮播/分享 | 电商核心功能 |
| 6 | 购物车不支持离线缓存/跨端同步/失效标记 | 购物车基本能力 |
| 7 | 支付不接入条件编译 | 微信/支付宝/抖音支付需要平台差异处理 |
| 8 | 使用 `Image.network` | 必须 CachedNetworkImage（含 maxSizeBytes 限制） |
| 9 | 埋点分散 | 必须统一 AnalyticsProvider |
| 10 | 平台特定代码不用条件编译 | 5 端差异必须条件编译处理 |
| 11 | ListView 全量渲染 | 必须 ListView.builder / SliverList 懒加载 |
| 12 | 小程序端调用非条件编译的原生 API | 会导致其他平台崩溃 |

## 与 storeforge-executing 的集成

`storeforge-executing` 的执行流程中，Harness 约束检查是关键节点：

```
子任务开发完成
  ↓
触发 Harness 约束检查（自动加载对应 domain 反模式清单）
  ↓
检查结果：
  ├─ BLOCK 违规 → 停止，修复代码，回到开发阶段
  ├─ WARN 违规 → 记录到验证清单，请求用户确认是否跳过
  └─ 全部通过 → 继续执行 L1 单元测试
```

每个子任务都必须通过这个检查点，不存在"先提交后修复"的情况。

## 与 storeforge-verification 的集成

Harness 约束检查的结果直接写入 `storeforge-verification` 的检查清单：

- BLOCK 违规项 → 验证清单中标记为「失败」，必须修复才能通过验证
- WARN 违规项 → 验证清单中标记为「警告」，用户可以确认跳过但会留痕
- 所有违规项都有明确的原因说明和修复建议

验证清单中的"代码规范检查"部分直接依赖 Harness 的输出。

## 约束的不可绕过性

Harness 约束**不是建议，不是参考，不是 best practice**。它们是 StoreForge 插件的硬性规则：

- BLOCK 级别违规 → 代码不得提交，任务不得标记完成
- WARN 级别违规 → 必须用户明确确认才可跳过，且跳过记录留痕
- 不存在"特殊情况下可以绕过"的例外

这是 StoreForge 区别于普通代码助手的核心理念：**约束引擎确保 Agent 不走错误路径，即使 Agent "想"走。**
