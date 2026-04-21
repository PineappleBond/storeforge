---
name: storeforge-writing-plans
description: 将电商需求转化为实施计划
---

# 实施计划生成

> Skill: `storeforge-writing-plans` | 触发: `/sf:plan` 或 `storeforge-brainstorm` 完成后自动调用

## 定位

将脑暴输出的技术方案转化为可执行的实施计划，输出 TodoWrite 兼容格式的任务列表，供 `storeforge-executing` 按序执行。

## 输入

`storeforge-brainstorm` 输出的技术方案文档，包含：
- 模块清单
- 服务架构图
- 关键技术决策
- 风险清单

如果没有脑暴输出，直接基于用户需求进行理解后生成计划。

## 执行流程（5 步）

### Step 1：任务分解

将技术方案拆分为独立子任务。拆分原则：

- **按模块拆分**：用户模块、商品模块、订单模块、支付模块、营销模块各自独立
- **按技术端拆分**：后端、Admin、Website、Flutter 分别建任务
- **按层次拆分**：每个模块内部分为 Model → RPC/API → Handler/Logic → 测试

#### 标准拆分模板

```
[模块名] - [技术端]
├── Task 1: 数据模型定义（GORM model / TypeScript interface / Dart model）
├── Task 2: 服务接口定义（.api 文件 / protobuf / OpenAPI）
├── Task 3: 业务逻辑实现（logic 层）
├── Task 4: API 实现（handler 层，goctl 自动生成）
├── Task 5: 单元测试（L1）
└── Task 6: 集成测试场景（L2，如适用）
```

**约束**：每个子任务必须可在 **1-2 个 Agent 会话内完成**。如果一个任务太大，继续拆分。

### Step 2：依赖分析

标记任务间的关系：

| 关系类型 | 标记 | 含义 |
|----------|------|------|
| 严格前置 | `→ Task X` | 必须等 Task X 完成后才能开始 |
| 并行 | `|| Task Y` | 可与 Task Y 同时执行，无依赖 |
| 弱依赖 | `~→ Task Z` | 依赖接口定义，不依赖实现 |

#### 典型依赖链

```
用户认证 (backend) → 商品管理 (backend) → 订单创建 (backend) → 支付 (backend)
       ↓                     ↓                    ↓                    ↓
用户认证 (admin)      商品管理 (admin)      订单管理 (admin)      支付看板 (admin)
       ↓                     ↓                    ↓
官网用户 (website)    商品展示 (website)    购物车/下单 (website)
       ↓                     ↓                    ↓                    ↓
用户端 (flutter)      商品端 (flutter)      购物车/下单 (flutter)  支付 (flutter)
```

**操作**：
1. 为每个任务标注依赖列表
2. 识别可并行执行的任务组（同一行 `||` 的任务）
3. 标记核心路径（最长依赖链）

### Step 3：优先级排序

按以下顺序排列任务：

1. **核心路径优先**：用户 → 商品 → 订单 → 支付 → 促销
2. **后端优先**：后端 API 就绪后，前端才能对接
3. **基础模块优先**：认证、基础 CRUD 先于复杂业务逻辑
4. **测试伴随**：每个模块的开发任务后面紧跟对应测试任务

#### 排序规则

```
优先级 P0: 用户认证 + 基础用户 CRUD
优先级 P1: 商品 SPU/SKU + 库存基础
优先级 P2: 订单创建 + 状态机
优先级 P3: 支付接入 + 回调验签
优先级 P4: 营销活动 + 优惠券
优先级 P5: 管理后台 + 数据看板
优先级 P6: 官网前端 + SEO
优先级 P7: Flutter 移动端
```

**注意**：每个优先级内的任务如无依赖关系，可并行执行。

### Step 4：验收标准

为每个子任务定义可验证的验收标准：

#### 验收标准模板

```
验收标准：
- [ ] 功能验收: [具体功能点是否实现]
- [ ] L1 测试: [覆盖率达标，race detector 通过]
- [ ] Harness 检查: [无反模式违规]
- [ ] API 契约: [OpenAPI spec 已更新]
- [ ] Git 规范: [Commit message 符合 Conventional Commits]
```

#### 各端 L1 测试要求

| 端 | 工具 | 覆盖率要求 |
|----|------|-----------|
| Backend (Go) | `go test -race -coverprofile` + testify | ≥ 80%，race detector 必须通过 |
| Admin (Vue3) | vitest | 组件覆盖率 ≥ 70% |
| Website (Next.js) | vitest | 核心逻辑覆盖率 ≥ 80% |
| Flutter | `flutter test`（unit + widget, desktop） | 覆盖率 ≥ 70% |

#### 验收标准示例

```
Task: 用户注册/登录 API
验收:
- [ ] 功能: 支持手机号+验证码注册、JWT 登录、Token 刷新
- [ ] 安全: JWT RS256 签名、密码 bcrypt、refresh token HttpOnly cookie
- [ ] L1 测试: go test -race 通过率 100%，覆盖率 ≥ 80%
- [ ] Harness: 无 BLOCK 级别违规
- [ ] API: OpenAPI spec 包含注册/登录/刷新/撤销 4 个 endpoint
```

### Step 5：输出计划文档

将所有信息整理为标准格式的计划文档。

## 计划输出格式

```markdown
# [项目名称] 实施计划

> 基于: [技术方案文档标题]
> 生成时间: YYYY-MM-DD

## Phase 1: 用户模块
- [ ] Task 1.1: 定义 User GORM model 和枚举 | 验收: model 字段完整，枚举定义在 event/ protobuf | 依赖: 无
- [ ] Task 1.2: 用户认证 .api 定义 + goctl 生成 | 验收: .api 文件覆盖注册/登录/刷新/撤销，goctl 生成无错误 | 依赖: Task 1.1
- [ ] Task 1.3: JWT 实现（RS256 + refresh token + 撤销）| 验收: Token 签发/验证/刷新/撤销完整，bcrypt 密码 | 依赖: Task 1.2
- [ ] Task 1.4: 用户认证 L1 单元测试 | 验收: go test -race 通过，覆盖率 ≥ 80% | 依赖: Task 1.3
- [ ] Task 1.5: Admin 用户管理页面 | 验收: el-table 支持排序/筛选/分页，axios interceptor 统一处理 | 依赖: Task 1.4

## Phase 2: 商品模块
- [ ] Task 2.1: 商品 SPU/SKU GORM model | 验收: SPU/SKU 关联完整，SKU 属性矩阵 | 依赖: 无 || Task 1.1
- [ ] Task 2.2: 商品 .api 定义 + goctl 生成 | 验收: CRUD + 搜索 + SKU 选择接口 | 依赖: Task 2.1
...
```

### 格式说明

- 每个 Phase 代表一个逻辑分组（按模块或优先级）
- `## Phase N: [name]` 为阶段标题
- `- [ ] Task N.M: [描述] | 验收: [标准] | 依赖: [前置任务ID]` 为任务行
- 并行任务用 `||` 标记
- 任务 ID 格式：`{Phase}.{序号}`

## 约束

1. **任务粒度**：每个子任务必须在 1-2 个 Agent 会话内完成。超出则拆分
2. **测试伴随**：每个开发任务后必须紧跟 L1 单元测试任务
3. **必须包含 Review 任务**：所有模块开发完成后安排 `storeforge-review` 多轮审查
4. **依赖不可省略**：每个任务必须标注依赖，无依赖则写"无"
5. **验收标准必须可验证**：禁止"功能正常"等模糊描述，必须具体到接口、覆盖率数字、检查项

## 并行执行标注

在计划中标注哪些任务组可并行执行：

```markdown
### 并行组 A（可并行）
- Task 1.1 (User Model)
- Task 2.1 (Product Model)
- Task 3.1 (Order Model)

### 并行组 B（等待组 A 完成）
- Task 1.2 (User API)
- Task 2.2 (Product API)
- Task 3.2 (Order API)
```

## 后续

1. 用户确认计划后，`storeforge-executing` 自动接手
2. 执行引擎按依赖顺序逐一取出任务执行
3. 无依赖的任务组可并行分发到多个 Agent
4. 全部任务完成后自动触发 `storeforge-review` 对整体 diff 进行多轮审查（最多 5 轮）
