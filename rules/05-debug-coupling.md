---
description: 
alwaysApply: true
enabled: true
updatedAt: 2026-05-24T08:33:39.608Z
provider: 
---

# 5. 耦合感知调试规则

## 触发条件

当用户提到以下关键词时，必须加载 `coupling-aware-debug` skill 进行耦合分析：

- **标准模式**：`debug`、`修bug`、`修一下`、`有问题`、`报错`、`不正常`、`修复`、`不能用`、`没反应`、`不显示`、`无法`
- **深度模式**：`深度debug`、`深度调试`、`复杂bug`、`跨模块debug`、`多系统bug`、`团队debug`

> 深度模式启动 Agent Team（多个独立 agent 并行验证不同假设），标准模式使用 subagent 并行分析。

## 执行要求

1. **必须先加载 skill**：使用 `use_skill` 工具加载 `coupling-aware-debug`
2. **根据关键词选择模式**：标准模式用 subagent 并行分析，深度模式用 Agent Team（`team_create` + 多个 `Task`）
3. **必须并行分析**：在定位根因后，并行检查所有耦合模块
4. **必须验证影响面**：修复方案必须确认不影响其他功能
5. **必须先提方案**：耦合分析完成后，提出方案等待用户批准再执行