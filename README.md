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

### 2. 安装触发规则

将仓库中的规则文件拷到用户规则目录：

```bash
cp rules/05-debug-coupling.md ~/.codebuddy/rules/
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
├── SKILL.md                          # 主工作流定义 (346行, v1.4.2)
├── references/
│   └── coupling-patterns.md          # 10种耦合模式 + 检查清单
├── test-prompts.json                 # 测试提示词
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
