# coupling-aware-debug

[![CodeBuddy Skill](https://img.shields.io/badge/CodeBuddy-Skill-blue)](https://www.codebuddy.ai)

Debug skill with **6-dimension coupling impact analysis**. Before proposing any fix, use parallel subagents to inspect all modules that share infrastructure, state, events, timing, side-effects, and edge states with the broken feature — then verify the fix won't break anything.

**v1.4.2** — Spatial · State · Timing · Event Chain · Side-Effect Chain · State Boundary

Darwin 评分: **89.5/100**（full_test verified）

---

## 安装

### 1. 克隆 Skill

```bash
cd ~/.codebuddy/skills
git clone https://github.com/Luo-CJL/coupling-aware-debug.git
```

### 2. 添加用户规则（自动触发）

在 `~/.codebuddy/rules/` 下创建文件，内容：

```markdown
---
description: Debug 时自动加载耦合感知调试 skill
alwaysApply: true
enabled: true
---

# 耦合感知调试规则

当用户提到以下关键词时，必须加载 `coupling-aware-debug` skill：

- **标准模式**：`debug`、`修bug`、`有问题`、`报错`、`无法`
- **深度模式**：`深度debug`、`复杂bug`、`跨模块debug`、`团队debug`

执行要求：先并行分析耦合影响面，再提出修复方案，等用户批准后执行。
```

> ⚠️ skill + 规则缺一不可：规则负责触发，skill 负责执行。

## 双模式

| 触发词 | 模式 | 做法 |
|--------|------|------|
| `debug` / `修bug` | **Standard** | Subagent 并行耦合分析 |
| `深度debug` / `复杂bug` | **Deep** | Agent Team 竞争性假设调试 |

## 目录结构

```
coupling-aware-debug/
├── SKILL.md                          # 主工作流定义 (342行, v1.4)
├── references/
│   └── coupling-patterns.md          # 10种耦合模式 + 检查清单
└── README.md
```

## 设计理念

- **TDD verified** — 通过 RED/GREEN 基线测试验证
- **6 coupling dimensions** — 空间·状态·时序·事件链·副作用链·状态边界
- **Tunnel vision protection** — 强制检查所有耦合维度
- **Dual mode** — 日常 bug 用 subagent，复杂 bug 用 Agent Team
- **Rationalization table** — 从真实基线测试提取的防绕行表

## License

MIT
