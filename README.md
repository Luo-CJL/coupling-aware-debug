# coupling-aware-debug

Platform-agnostic debug skill with coupling impact analysis. Before proposing any fix, use parallel sub-agents to inspect all modules that share infrastructure with the broken feature, then verify the fix won't break coupled modes.

## Platform Compatibility

This is the **platform-agnostic** edition — works with any AI coding platform that supports agent dispatch.

| Platform | Standard Mode | Deep Mode |
|----------|:---:|:---:|
| CodeBuddy | ✅ | ✅ Full team + debate |
| Claude Code | ✅ | ↪ Falls back to Standard |
| Cursor | ✅ | ↪ Falls back to Standard |
| Other | ✅ (if has agent dispatch) | ↪ Falls back to Standard |

> Deep Mode requires agent team + inter-agent messaging. On platforms without these, the skill automatically falls back to Standard Mode.

## Two Modes

| Trigger | Mode | Method |
|---------|------|--------|
| `debug` / `修bug` | **Standard** | Parallel sub-agents analyze coupling |
| `深度debug` / `复杂bug` | **Deep** | Agent team competitive hypothesis debugging |

## Install

Skill files alone don't auto-trigger — you also need a platform-level rule.

### 1. Install Skill

```bash
# Depends on your platform. Example for CodeBuddy:
cd ~/.codebuddy/skills
git clone https://github.com/Luo-CJL/coupling-aware-debug.git
```

### 2. Configure Auto-Trigger Rule

The skill needs a companion rule that tells your AI platform "load this skill when user says debug".

**CodeBuddy** — create `~/.codebuddy/rules/debug-coupling.md`:

```markdown
---
description: Debug 时自动加载耦合感知调试 skill
alwaysApply: true
---

当用户提到 debug/修bug/有问题/报错/无法 时，加载 coupling-aware-debug skill。
先说"深度debug/复杂bug"时，走 Deep Mode (Agent Team)。
```

**Claude Code** — add to `CLAUDE.md`:

```markdown
## Debug Protocol
When user reports a bug, load skill: coupling-aware-debug (check ~/.claude/skills/)
```

**Cursor** — add to `.cursorrules`:

```
When user says "debug" or reports a bug, reference coupling-aware-debug workflow:
1. Identify shared infrastructure
2. Launch parallel sub-agents to check coupling
3. Present impact matrix before proposing fix
```

> ⚠️ Skill + trigger rule are a pair. One without the other won't work.

## Structure

```
coupling-aware-debug/
├── SKILL.md                          # Main workflow (platform-agnostic)
├── references/
│   └── coupling-patterns.md          # 5 coupling patterns + checklist
└── README.md
```

## Design

- **TDD verified** — RED/GREEN baseline testing confirms behavior change
- **Tunnel vision protection** — forces inspection of ALL coupled modules before proposing fixes
- **Rationalization table** — counters extracted from real baseline test failures
- **Exception handling** — 8 fallback scenarios with detection + action

## License

MIT
