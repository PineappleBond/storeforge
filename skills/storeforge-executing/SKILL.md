---
name: storeforge-executing
description: 执行实施计划，子代理 + TodoWrite + Git
---

# 计划执行引擎

> Skill: `storeforge-executing` | 触发: 用户确认计划后自动调用

## 定位

按实施计划执行子任务，集成 Harness 约束检查、Git worktree 隔离工作流、TodoWrite 进度跟踪、三层测试验证和多代理 Review。

## 前置条件

- `storeforge-writing-plans` 已输出计划文档
- 用户已确认计划（审查并同意任务列表、优先级、验收标准）
- `storeforge-using` 的核心规则已加载（先计划再写代码、TodoWrite 跟踪、完成前验证）

## 执行流程（8 步循环）

### Step 1：获取下一个可执行任务

从计划文档中找出满足以下条件的任务：
- TodoWrite 状态为 `pending`（未开始）
- 所有依赖任务状态为 `completed`（已完成）
- 优先级最高（按 `storeforge-writing-plans` 排序）

**并行任务处理**：
- 如果多个任务依赖都已满足且无相互依赖，可并行分发到多个 Agent
- 标注为 `||` 的任务组可同时执行
- 每个并行任务使用独立的 git worktree 和 TodoWrite 条目

### Step 2：TodoWrite 标记为 in_progress

```
TodoWrite: 将任务状态从 pending 更新为 in_progress
```

**要求**：
- 每个子任务对应一个 TodoWrite 条目
- 条目描述包含任务 ID 和简要描述（如 `Task 1.3: JWT 实现`）
- 不得跳过 TodoWrite 直接开始编码

### Step 3：创建 Git Worktree

为每个任务创建隔离的工作分支：

```bash
git worktree add ../worktrees/task-{task-id} -b task-{task-id}
```

**分支命名规范**：`task-{task-id}`，例如 `task-1.3`、`task-2.1`

**原因**：
- 每个任务在独立 worktree 中开发，互不干扰
- 并行任务可以同时在不同 worktree 中执行
- 任务失败时可直接丢弃该 worktree，不影响主分支

### Step 4：在 Worktree 中执行开发

在 worktree 目录下完成任务的开发工作：

1. **加载领域约束**：根据任务所属技术端，调用对应的 domain skill
   - 后端任务 → `storeforge-domain-backend`
   - 管理后台任务 → `storeforge-domain-admin`
   - 官网任务 → `storeforge-domain-website`
   - Flutter 任务 → `storeforge-domain-flutter`

2. **加载电商模式**：根据任务需要，查阅 `knowledge/ecommerce-patterns/` 中的对应 pattern

3. **按 Harness 约束编码**：
   - 违反反模式 = 阻断，必须立即修复
   - 遵循 domain skill 中的技术栈约束和代码规范
   - 金额使用 `int64`（分），禁止 `float`/`double`
   - 所有列表接口必须分页（page/pageSize）
   - 所有写操作必须幂等（`X-Idempotency-Key`）

4. **生成代码**：编写业务逻辑、API 定义、数据模型等

5. **遵循 Git 规范**：
   - 每个任务一个 commit
   - Commit message 使用 Conventional Commits 格式：`<type>(<scope>): <description>`
   - type 取值：`feat`（新功能）、`fix`（修复）、`refactor`（重构）、`test`（测试）、`docs`（文档）、`chore`（杂项）
   - 示例：`feat(user): 实现 JWT RS256 认证和 Token 刷新`

### Step 5：Harness 约束检查

开发完成后，运行 Harness 约束检查（引用 `storeforge-harness`）：

```
加载 storeforge-harness，对当前 worktree 中的变更执行约束检查
```

**检查结果处理**：

| 级别 | 含义 | 处理 |
|------|------|------|
| BLOCK | 违反 domain 反模式 | **立即阻断**，必须修复后重新检查 |
| WARN | 建议性改进 | 记录到验证清单，可跳过（需记录原因） |

**BLOCK 示例**（必须修复）：
- 金额使用了 `float`/`double`
- 手写 handler（未通过 `.api` + `goctl` 生成）
- 列表接口未分页
- 订单状态随意跳转（未经状态机）
- 支付回调未验签

**WARN 示例**（可跳过，需记录）：
- 缺少注释
- 命名风格建议
- 可优化但未违反约束的代码结构

**操作**：
1. 加载当前任务对应的 domain skill 反模式清单
2. 逐项检查当前变更是否违反任何反模式
3. 有 BLOCK → 回到 Step 4 修复
4. 全部 BLOCK 通过 → 进入 Step 6

### Step 6：运行 L1 单元测试

先调用 `storeforge-testing` 加载测试规范，然后按该端要求执行测试：

```
加载 storeforge-testing，获取当前端的 L1 测试规范
按规范生成/执行测试
```

**各端测试要求**：

| 端 | 执行命令 | 覆盖率要求 | 特殊要求 |
|----|----------|-----------|----------|
| Backend (Go) | `go test -race -coverprofile=cover.out ./...` | ≥ 80% | race detector 必须通过 |
| Admin (Vue3) | `vitest run --coverage` | 组件覆盖率 ≥ 70% | vitest |
| Website (Next.js) | `vitest run --coverage` | 核心逻辑 ≥ 80% | vitest |
| Flutter | `flutter test --coverage` | ≥ 70% | unit + widget, desktop platform |

**测试数据规范**：
- 使用 UUID 生成测试 ID，禁止硬编码
- `TestMain` 统一 setup/teardown
- 禁止断言精确时间戳（用相对比较）
- 禁止依赖列表顺序（先排序再比较）
- 禁止无 `Eventually`/重试的异步断言

**测试结果处理**：
- 通过 → 进入 Step 7
- 失败 → 回到 Step 4 修复后重新从 Step 5 开始

### Step 7：Git Commit 并标记完成

1. **提交变更**：
   ```bash
   git add -A
   git commit -m "<type>(<scope>): <description>"
   git worktree remove ../worktrees/task-{task-id}
   ```

2. **TodoWrite 标记完成**：
   ```
   TodoWrite: 将任务状态从 in_progress 更新为 completed
   ```

3. **验证清单更新**：
   - 记录 Harness 检查结果（BLOCK 项必须为 0，WARN 项记录原因）
   - 记录测试覆盖率数据
   - 记录 commit hash

### Step 8：失败回环

如果 Step 5 或 Step 6 失败：

```
修复问题
→ 回到 Step 5（重新运行 Harness 约束检查）
→ 通过后进入 Step 6（重新运行 L1 测试）
→ 通过后进入 Step 7
```

**连续失败处理**：
- 同一任务连续 3 次测试失败 → 记录问题，升级到用户
- 附带：失败日志、当前 diff、已尝试的修复方案

## 全部任务完成后的 Review 流程

当计划中 **所有** 任务都标记为 `completed` 后：

### 自动触发 `storeforge-review`

详细的子代理调用协议、worktree 隔离、去重规则、结果格式见 `storeforge-review` SKILL.md。

1. 调用 `storeforge-review` 对整体 diff 进行多轮审查
2. 并行启动 5 个子代理（`isolation: "worktree"`）：
   - **storeforge-architect**：架构是否合理？状态机是否闭环？
   - **storeforge-security-auditor**：有无安全漏洞？支付验签是否完整？
   - **storeforge-performance-reviewer**：有无 N+1？热点是否走缓存？
   - **storeforge-api-contract-checker**：前后端字段是否对齐？
   - **storeforge-testing-validator**：测试覆盖是否达标？
3. 汇总结果，去重，按严重程度排序
4. Agent 逐一修复问题
5. 再次并行启动 5 个子代理
6. 如果没有新问题 → Review 通过（**修复保留在 worktree 中，不合并到主分支**）
7. 如果有新问题 → 修复 → 再 Review（**最多 5 轮**）
8. 5 轮后仍有新问题 → 升级至用户，附带未解决问题清单和影响评估

**审查范围**：仅审查本次所有 commit 变更的文件，不审查全量代码。

**Review 通过后**：
- 所有变更合并到主分支
- 调用 `storeforge-testing` 执行完整测试级联（L1 全量回归 → L2 集成场景 → L3 E2E）
- 全部测试通过后，调用 `storeforge-verification` 做最终验证
- 全部通过 → 标记项目完成

## Git 工作流规范

### 分支策略

```
main (主分支)
├── task-1.1 (用户 model)
├── task-1.2 (用户 API)
├── task-1.3 (JWT 实现)
├── task-2.1 (商品 model)
└── ...
```

每个任务完成后 worktree 被移除，分支保留用于 review。

### Commit 规范

```
<type>(<scope>): <description>

<body>（可选）

feat(user): 实现 JWT RS256 认证和 Token 刷新
- 使用 RSA-2048 密钥对
- refresh token 存储在 HttpOnly cookie
- 支持 Token 版本计数器撤销

fix(order): 修复订单超时取消的竞态条件
- 增加 Redis 分布式锁
- 超时检查使用 Lua 脚本保证原子性
```

### 并行执行

无依赖的任务组可通过并行 Agent 同时执行：

```
并行组: [task-2.1, task-3.1, task-4.1]
→ Agent A: worktree task-2.1 (商品 model)
→ Agent B: worktree task-3.1 (订单 model)
→ Agent C: worktree task-4.1 (支付 model)
→ 全部完成后 → 继续下一批并行组
```

每个并行 Agent：
- 使用独立的 worktree
- 独立追踪 TodoWrite 状态
- 完成后独立执行 Harness 检查和 L1 测试
- 全部通过后再合并

## 与 Harness 的集成点

`storeforge-harness` 在以下时刻被调用：

1. **每个子任务开发完成后（Step 5）**：约束检查
2. **BLOCK 级别违规**：必须修复，不允许跳过
3. **WARN 级别违规**：记录到验证清单，可在 `storeforge-verification` 中统一处理
4. **最终 Review 后**：`storeforge-verification` 汇总所有 Harness 检查结果

约束检查的触发由 `storeforge-executing` 控制，不是独立的命令行操作。

## 约束总结

- **必须先通过 Harness 检查才能进入测试阶段**
- **L1 测试不达标不能提交 commit**
- **每个任务一个 commit，不得合并多个任务到同一 commit**
- **并行任务必须使用独立 worktree**
- **全部任务完成后必须经过 `storeforge-review` 多轮审查（最多 5 轮）**
- **Review 通过后必须经过 `storeforge-verification` 最终验证**
