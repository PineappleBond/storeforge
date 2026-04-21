---
description: 启动电商项目脑暴流程，分析需求并生成技术方案
---

# /sf:brainstorm — 电商项目脑暴

启动电商项目脑暴流程，分析需求并生成技术方案。

## 执行步骤

1. 确保工作流上下文已加载：如 `storeforge-using` 未加载，先加载该 skill 获取核心规则
2. 调用 `storeforge-brainstorm` skill
3. 按 brainstorm skill 定义的 6 步流程执行：
   - Step 1：需求拆解（模块划分）
   - Step 2：技术栈匹配（匹配 domain skill）
   - Step 3：Pattern 匹配（查找电商模式库）
   - Step 4：架构设计（服务拆分 + 数据流）
   - Step 5：风险识别（并发/安全/性能）
   - Step 6：输出技术方案文档
4. 脑暴阶段仅输出方案，不编写代码

## 输出

技术方案文档，包含：
- 模块清单（每个模块对应的 domain skill 和 pattern）
- 服务架构图（文字描述）
- 关键技术决策列表
- 风险清单

## 后续

脑暴完成后，建议调用 `/sf:plan` 将方案转化为实施计划。

## 前置条件

- 用户提供需求描述（自然语言）
