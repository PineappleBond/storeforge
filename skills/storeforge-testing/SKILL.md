---
name: storeforge-testing
description: 三层测试体系：单元测试 + 集成测试 + E2E
---

# 三层测试体系

> L1 单元测试 + L2 集成测试 + L3 E2E 测试。全端覆盖，覆盖率不达标阻断 merge。`/sf:test` 或每个子任务完成后自动触发。

## 定位

本 skill 定义了 StoreForge 的三层测试体系。每一层都有明确的触发时机、执行方式、覆盖率要求和阻断条件。测试按 **L1 → L2 → L3** 顺序执行，下层通过后才可进入上层。

**触发方式**：
- 用户手动调用 `/sf:test`
- `storeforge-executing` 在每个子任务完成后自动触发 L1
- L1 全部通过后自动触发 L2
- L2 全部通过后自动触发 L3

---

## L1 单元测试

### 触发时机

每个子任务开发完成后，`storeforge-harness` 约束检查通过之后。

### 执行方式

子代理在 worktree 中生成并运行测试。按技术栈分别执行：

#### Backend（Golang）

```bash
go test -race -coverprofile=coverage.out ./...
go tool cover -func=coverage.out
```

- 测试框架：`testify`
- **覆盖率要求**：≥ 80%
- **Race detector 必须通过**：`-race` 标志无报错
- `TestMain` 统一管理 setup/teardown
- 测试数据：使用 UUID 生成 ID，禁止硬编码

#### Admin（Vue 3 + Element Plus）

```bash
npx vitest run --coverage
```

- 测试框架：`vitest`
- **覆盖率要求**：组件覆盖率 ≥ 70%

#### Website（Next.js）

```bash
npx vitest run --coverage
```

- 测试框架：`vitest`
- **覆盖率要求**：核心逻辑覆盖率 ≥ 80%

#### Flutter（多端）

```bash
flutter test --coverage --platform=linux
```

- 测试框架：`flutter test`（unit + widget）
- 平台：desktop platform（linux 或 macos）
- **覆盖率要求**：≥ 70%
- **Flutter 5 端平台特定集成测试**（Android、iOS、微信小程序、支付宝小程序、抖音小程序）标记为 **「手动 QA 必需」**，不在 CI 中自动化

### 测试数据规范

- 使用 UUID 生成测试 ID：`uuid.New()` / `crypto.randomUUID()`
- **禁止硬编码**任何时间戳、ID、Token、手机号
- `TestMain`（Go）/ `beforeAll`（Vitest）统一管理测试环境 setup/teardown
- 每个测试用例独立，不依赖执行顺序

### 测试反模式（违反即阻断）

| 反模式 | 说明 | 正确做法 |
|--------|------|---------|
| 精确时间戳断言 | `assert.Equal(t, time.Now(), result.CreatedAt)` | 用相对比较：`assert.True(t, result.CreatedAt.After(before))` |
| 依赖列表顺序 | `assert.Equal(t, expected, result.List)` | 先排序再比较，或用 `ElementsMatch` |
| 无 Eventually/重试的异步断言 | 直接断言异步操作结果 | 使用 `Eventually` / retry loop / 轮询等待 |

---

## L2 集成测试

### 触发时机

L1 单元测试全部通过且覆盖率达标后。

### 执行方式

1. 通过 Docker Compose 启动后端服务 + PostgreSQL + Redis（隔离网络）
2. `testdata/` 目录提供确定性 seed 脚本，每轮测试前重置数据库
3. 使用 `curl` 或集成测试脚本验证 API 场景
4. 第三方服务通过 mock 模拟

### 测试环境

```yaml
# docker-compose.test.yml
services:
  backend:
    build: .
    ports: ["8080:8080"]
    depends_on: [postgres, redis]
    environment:
      DB_HOST: postgres
      REDIS_HOST: redis
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: storeforge_test
  redis:
    image: redis:7
```

- 隔离网络：测试服务在独立的 Docker 网络中运行
- `testdata/` 目录：提供确定性 seed 脚本，确保每次测试初始状态一致
- **每轮测试前重置数据库**：DROP + CREATE + seed，保证可重复性

### 第三方服务模拟

| 服务 | Mock 方式 |
|------|---------|
| 微信支付/支付宝/抖音支付 | `httptest` mock server，回放捕获的 webhook golden fixture |
| 物流 API（顺丰/三通一达） | mock 响应，基于预定义的 JSON fixture |

### 覆盖率测量

- Go：`go test -cover`
- Frontend：vitest coverage reporter
- **覆盖率不达标阻断 merge**

### 13 个必测场景

| # | 场景 | 关键验证点 |
|---|------|-----------|
| 1 | **用户认证** | 注册/登录/Token 刷新/Token 撤销 |
| 2 | **商品浏览** | 商品浏览/搜索/详情/SKU 选择 |
| 3 | **购物车** | 购物车增/删/改/失效标记/跨端同步 |
| 4 | **订单** | 订单创建/状态流转/超时取消 |
| 5 | **支付** | 支付发起/回调验签（mock）/退款 |
| 6 | **优惠券** | 优惠券领取/使用/互斥计算 |
| 7 | **管理后台** | 商品 CRUD / 订单管理 / 数据看板 |
| 8 | **官网** | SEO 渲染 / SSR 性能（TTFB < 800ms）/ i18n |
| 9 | **并发场景** | 同一 SKU 同时创建 2 笔订单 → 验证库存不超卖 |
| 10 | **幂等场景** | 同一支付回调 payload 发送 3 次 → 验证只处理一次 |
| 11 | **竞争场景** | 互斥优惠券同时使用 → 验证只生效一张 |
| 12 | **异常场景** | 支付超时后回调到达 → 验证订单状态不被错误更新 |
| 13 | **负向场景** | 库存不足下单 / 无效 Token 访问 / 参数校验失败 |

> 场景 1-8 为**功能场景**，场景 9-13 为**特殊场景**（并发/幂等/竞争/异常/负向）。

---

## L3 E2E 测试

### 触发时机

L2 集成测试全部通过后。

### 执行范围

**仅限 Next.js 官网**（chrome-mcp 能力范围内）。

- 单浏览器 headless 模式
- **Flutter App E2E 不在范围内**，标记为 **「手动 QA 必需」**（需要移动模拟器/真机）
- **多端购物车同步不在范围内**，通过 L2 API 场景覆盖

### 测试场景

| # | 场景 | 验证内容 |
|---|------|---------|
| 1 | **黄金路径** | 官网浏览 → 商品搜索 → 详情页 → SSR 渲染验证 |
| 2 | **多语言/SEO** | 语言切换 / SEO metadata 验证 / OG 标签 |
| 3 | **移动端响应式** | 移动端布局 / 触摸交互 / 响应式断点 |

### 黄金路径（场景 1）详细步骤

```
1. 打开首页 → 验证页面加载，LCP < 2.5s
2. 搜索商品关键词 → 验证搜索结果列表展示
3. 点击商品进入详情 → 验证详情页 SSR 渲染
   - 检查 HTML 中包含商品名称、价格、SKU 信息
   - 验证 structured data（JSON-LD）
4. 验证 SSR 性能：TTFB < 800ms
```

---

## 覆盖率要求汇总

| 层级 | 指标 | 要求 |
|------|------|------|
| **L1** | Go `go test -coverprofile` | ≥ 80% |
| **L1** | Vue3 Admin `vitest coverage` | ≥ 70% |
| **L1** | Next.js Website `vitest coverage` | ≥ 80% |
| **L1** | Flutter `flutter test --coverage` | ≥ 70% |
| **L2** | API endpoint 覆盖率 | ≥ 95% |
| **L2** | 核心代码路径覆盖率 | ≥ 80% |
| **L3** | 黄金路径（登录→浏览→加购→下单→支付） | 100% 通过 |
| **L3** | 异常路径（库存不足、支付超时、无效参数） | ≥ 80% 覆盖 |

### 阻断规则

- L1 覆盖率不达标 → **阻断**，不得进入 L2
- L2 endpoint 覆盖率 < 95% → **阻断 merge**
- 黄金路径非 100% 通过 → **阻断 merge**
- L1 race detector 失败 → **立即阻断**，不得提交

---

## 与其他 Skill 的协作

- **`storeforge-executing`**：每个子任务完成后自动触发 L1 测试
- **`storeforge-verification`**：验证清单中的「测试检查」引用本 skill 的覆盖率标准
- **`storeforge-review`**：testing-validator agent 的检查清单引用本 skill 的 13 个必测场景和覆盖率要求
- **`storeforge-harness`**：L1 测试在 harness 约束检查通过后执行
