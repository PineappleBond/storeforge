---
description: 将需求转化为可执行的实施计划
---

# /sf:plan — 生成实施计划

将需求转化为可执行的实施计划。

## 执行步骤

1. 调用 `storeforge-writing-plans` skill
2. 确定输入来源：
   - 如果有 `/sf:brainstorm` 输出的技术方案，使用该方案作为输入
   - 如果没有，先进行需求理解，再进入计划生成
3. 按 writing-plans skill 定义的 5 步流程执行：
   - Step 1：任务分解（拆分为独立子任务）
   - Step 2：依赖分析（标记前置/并行关系）
   - Step 3：优先级排序（核心路径优先）
   - Step 4：验收标准（为每个子任务定义可验证标准）
   - Step 5：输出 TodoWrite 兼容格式计划

## 输出

实施计划文档，格式：

```
## Phase N: [阶段名]
- [ ] Task N.1: [描述] | 验收: [标准] | 依赖: [前置任务]
```

## 后续

计划确认后，调用 `storeforge-executing` 执行实施计划。

## 约束

- 每个子任务必须可在 1-2 个 agent 会话内完成
- 必须包含测试任务和 review 任务
