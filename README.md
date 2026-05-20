# coupling-aware-debug

[![CodeBuddy Skill](https://img.shields.io/badge/CodeBuddy-Skill-blue)](https://www.codebuddy.ai)

Debug skill with coupling impact analysis. Before proposing any fix, use parallel subagents to inspect all modules that share infrastructure with the broken feature, then verify the fix won't break coupled modes.

Darwin 评分: **88.3/100**

---

## 安装

```bash
# 克隆到 CodeBuddy skills 目录
cd ~/.codebuddy/skills
git clone https://github.com/Luo-CJL/coupling-aware-debug.git
```

## 双模式

| 触发词 | 模式 | 做法 |
|--------|------|------|
| `debug` / `修bug` | **Standard** | Subagent 并行耦合分析 |
| `深度debug` / `复杂bug` | **Deep** | Agent Team 竞争性假设调试 |

## 目录结构

```
coupling-aware-debug/
├── SKILL.md                          # 主工作流定义 (280行)
├── references/
│   └── coupling-patterns.md          # 5种耦合模式 + 检查清单
└── README.md
```

## 设计理念

- **TDD verified** — 通过 RED/GREEN 基线测试验证
- **Tunnel vision protection** — 强制检查所有耦合模块
- **Dual mode** — 日常 bug 用 subagent，复杂 bug 用 Agent Team
- **Rationalization table** — 从真实基线测试提取的防绕行表

## License

MIT
