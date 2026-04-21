---
description: 启动5个审查代理进行自循环review
---

# /sf:review — 多代理代码审查

启动 5 个审查代理进行自循环 review。

## 执行步骤

1. 调用 `storeforge-review` skill
2. 按 review skill 定义的多轮流程执行：
   - 并行启动 5 个子代理（architect / security-auditor / performance-reviewer / api-contract-checker / testing-validator）
   - 汇总结果，去重并按严重程度排序
   - Agent 逐一修复问题
   - 再次并行启动 5 个子代理验证修复
3. 审查范围：仅本次 commit 变更的文件，不审查全量代码
4. 最多 5 轮，若 5 轮后仍有未解决问题，升级至用户

## 输出

- Review 结果汇总（按轮次列出问题清单）
- 如触发升级：Review 升级报告（未解决问题 + 影响评估 + 建议）

## 前置条件

- 需要已有代码变更（commit）

## 注意事项

- 每个子代理使用 worktree 隔离审查
- 结果按 [严重] / [中等] / [建议] 分级展示
