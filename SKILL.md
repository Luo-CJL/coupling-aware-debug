---
name: coupling-aware-debug
category: debugging
description: >-
  Use when debugging bugs in multi-mode features where several modes share infrastructure
  (containers, state, event routing). Use when the user reports a bug, says "debug", "修bug",
  "有问题", "报错", "不正常", "修复", or describes unexpected behavior in a feature that coexists
  with other modes in the same UI. Use when a fix might create regression in coupled modes.
  Use "深度debug" / "复杂bug" / "跨模块debug" / "团队debug" to trigger deep mode (Agent Team).
version: 1.3
platform: agnostic
---

# Coupling-Aware Debug (耦合感知调试)

Debug workflow that prevents regression in coupled features. Before proposing any fix,
map the coupling surface and verify impact across all dependent modes.

**Violating the letter of this workflow is violating the spirit of this workflow.**

## Platform Compatibility

This skill is platform-agnostic. Tool references use generic terms; adapt to your platform:

| Concept | CodeBuddy | Claude Code | Cursor | Other |
|---------|-----------|-------------|--------|-------|
| Parallel sub-agents | `Task(code-explorer)` | `Task` subagent | Agent mode | Your platform's agent dispatch |
| Team creation | `team_create` | Not available → skip Deep Mode | Not available → skip Deep Mode | Platform-specific |
| Inter-agent messaging | `send_message` | Not available → skip Step 3 | Not available → skip Step 3 | Platform-specific |

**If your platform lacks team creation, use Standard Mode only.**

## Mode Selection (Read FIRST)

Before starting any workflow, check what the user said:

| User said | Mode | What to do |
|-----------|------|------------|
| `debug` / `修bug` / `有问题` / `报错` / `不正常` / `修复` / `无法` | **Standard** | Go to → Standard Workflow (subagent parallel analysis) |
| `深度debug` / `深度调试` / `复杂bug` / `跨模块debug` / `多系统bug` / `团队debug` | **Deep** | Go to → Deep Mode (Agent Team, below) |

**If uncertain which mode to use, ask: "这个 bug 涉及几个独立模块？有多个假设需要验证吗？"**

---

## Standard Workflow (Subagent Parallel Analysis)

Default mode for most bugs. Use parallel sub-agents to read and analyze code simultaneously.

### When to Use

Trigger when user reports a bug AND the broken feature is part of a system with
multiple modes/features that share containers, state, or event routing.

## Workflow (4 Steps)

### Step 1: Understand the Bug

- Reproduce the user's description to confirm understanding.
- Identify the exact file(s) and line(s) where the symptom manifests.
- Then **immediately ask the coupling question** before going deeper:

> **"What other features/modes share the infrastructure that this feature depends on?"**

Common shared layers: container `div`, overlay wrapper, canvas, global state,
event routing (`pointer-events`), z-index stacking, conditional rendering gate.

### Step 2: Coupling Analysis (Parallel) — MANDATORY, DO NOT SKIP

**Launch parallel sub-agents** (use your platform's agent dispatch tool) to inspect
EVERY module that shares the broken feature's infrastructure. One per module, all launched
simultaneously. Wait for ALL to complete before forming a conclusion.

**This step is NOT optional.** Proposing a fix without completing Step 2 is a violation.

Each sub-agent must answer:
1. Does this module render in the shared container?
2. What `pointer-events` / z-index / event handlers does it set?
3. In its passive state (e.g., `readOnly=true`, hidden, inactive), does it block events?
4. Would the proposed fix interfere with this module's normal behavior?

Example for a PDF editor with 8 modes:

```
Parallel sub-agents (launched simultaneously):
├── agent A → WatermarkLayer (passive mode behavior)
├── agent B → SignatureLayer (passive mode behavior)
├── agent C → RedactionLayer (passive mode behavior)
└── agent D → ImageLayer (passive mode behavior)
```

### Step 3: Impact Assessment

Aggregate sub-agent reports into a coupling matrix before proposing any fix:

| Mode | Shares What | Affected? | Mitigation |
|------|-------------|:---:|------------|
| mode-A | shared container | ✅/❌ | how to protect |

**If the fix would break ANY other mode, redesign it.** The fix must:
- Fix the reported bug
- Cause zero regression in all other modes
- If impossible with a local change, report the architectural constraint to the user.

### Step 4: Propose & Execute

Present to the user:
1. Root cause (one sentence)
2. Fix approach (one paragraph)
3. **Coupling impact matrix** (proof all modes are safe)
4. Ask for approval before making changes.

After approval, apply the fix and verify with linter.

## Common Mistakes (Observed in Baseline Testing)

### Mistake 1: Tunnel Vision on a Single Technical Detail

**What it looks like:** Agent identifies a plausible-sounding root cause (e.g., z-index conflict)
and proposes fixes without checking whether the shared container itself is the real problem.

**Why it happens:** The first plausible explanation feels satisfying, so the agent stops
investigating. It never asks "what else depends on this?"

**How this skill prevents it:** Step 2 forces parallel inspection of ALL coupled modules,
which reveals whether the suspected root cause is actually correct or merely a symptom.

### Mistake 2: Fix That Creates New Coupling

**What it looks like:** Fixing z-index 5 → 7 "solves" the watermark visibility issue,
but now the watermark canvas (z=7) blocks the signature layer's interactive elements
that need z=6→10.

**Why it happens:** The agent only tested the fix against the reported symptom,
not against ALL modes.

**How this skill prevents it:** Step 3's coupling matrix catches this. If raising
watermark z-index breaks signature mode, the matrix shows "❌" and forces a redesign.

### Mistake 3: Skipping Impact Analysis Because "It's Simple"

**What it looks like:** "This is just a one-line CSS fix, I don't need to check
other modes."

**Why it happens:** The agent confuses simplicity of implementation with simplicity
of impact. One CSS property change can cascade through an entire stacked layout.

**Counter:** One-line fixes are the MOST dangerous for coupled systems because
they're easy to make and hard to notice breaking.

### Mistake 4: Not Checking Passive/readOnly State

**What it looks like:** Agent assumes a component in readOnly mode is "inactive"
and will not interfere. But in CSS, an invisible parent with `pointer-events: auto`
still intercepts events regardless of its children.

**Counter:** Always trace the DOM tree from the event target up to the first
ancestor with `pointer-events: auto`. That ancestor is the true blocker.

## Red Flags — STOP and Re-evaluate

- "This is just a simple CSS/z-index fix"
- "I found the root cause, it's clearly [X]"
- "I don't need to check other modes for this"
- "The fix is too small to affect anything else"
- "I already traced the problem to this line"

**ALL of these mean: You are about to propose a fix without coupling analysis.
Stop. Run Step 2. Then propose.**

## Rationalization Table (from Baseline Testing)

| Excuse | Reality |
|--------|---------|
| "The bug is clearly about z-index" | z-index is the symptom; the shared container's conditional rendering is the cause. |
| "I just need to change one number" | One number change in a stacked layout affects every layer above and below it. |
| "I checked all the relevant files" | Reading files ≠ checking coupling. Did you ask "what else shares this?" |
| "It's a visual bug, not a logic bug" | Visual bugs in shared containers affect ALL modes that render there. |
| "The fix works for the reported mode" | Working for ONE mode is not enough. Must work for ALL modes. |

## Key Principles

1. **Coupling check before root cause** — Always ask "what shares this?" before "what broke this?"
2. **Parallel analysis is mandatory** — Launch all coupling-inspection sub-agents in one batch.
3. **Zero regression is the bar** — A fix that breaks another mode is not a fix.
4. **Passive state is still present** — readOnly components are in the DOM and can interfere.

For detailed coupling patterns and checklists, see `references/coupling-patterns.md`.

---

## Deep Mode (Agent Team — Competitive Hypothesis Debugging)

Triggered when user says: `深度debug`, `深度调试`, `复杂bug`, `跨模块debug`, `多系统bug`, `团队debug`.

**⚠️ Requires platform support for agent teams/inter-agent messaging. If unavailable, fall back to Standard Mode.**

Use this mode when the bug:
- Spans multiple independent modules (e.g., frontend + backend + database)
- Has 3+ plausible hypotheses that need parallel verification
- Involves multiple technology layers (CSS events, React state, API calls, etc.)
- Previous sub-agent-level analysis failed to find the root cause

### Deep Mode Workflow

#### Step 1: Hypothesis Decomposition

Break the bug into 3-5 independent, testable hypotheses. Each hypothesis must be:
- **Self-contained** — one team member can verify it independently
- **Falsifiable** — there's a clear way to prove it wrong
- **Non-overlapping** — no two hypotheses depend on the same verification

Example for a complex bug:
```
Hypothesis A: overlay container pointer-events blocks Fabric canvas events
Hypothesis B: conditional component unmount destroys state on mode switch
Hypothesis C: Fabric.js coordinate transform breaks after zoom change
```

#### Step 2: Create Team

Use your platform's team/group creation tool to form a debug team named `debug-{feature-name}`.

Then dispatch members (one per hypothesis), ALL in the same tool call batch:

```
Dispatch agent: name="分析员-A", prompt="验证假设：overlay container pointer-events..."
Dispatch agent: name="分析员-B", prompt="验证假设：conditional unmount..."
Dispatch agent: name="分析员-C", prompt="验证假设：Fabric.js coordinate..."
```

**Critical:** Each member gets a turn limit (recommended 20-30 turns). Based on real testing,
medium-complexity modules need ~14 turns; this leaves ample headroom for larger modules.
If a member hits the limit without converging, mark that hypothesis as "未收敛".

#### Step 3: Debate & Converge

Members communicate via your platform's inter-agent messaging to:
- Share findings with the lead (you)
- Challenge each other's conclusions
- Provide evidence that confirms or refutes hypotheses

Wait for ALL members to complete before forming a conclusion.

#### Step 4: Synthesize

When all members finish:
1. The hypothesis with strongest evidence is the root cause
2. If multiple hypotheses confirmed → the bug has multiple contributing factors
3. Propose a fix that addresses ALL confirmed factors
4. Present the coupling impact matrix (same as Standard Mode Step 3)
5. Clean up the team when done

### Team Composition Guidelines

| Role | Count | Purpose |
|------|:---:|---------|
| Hypothesis validators | 2-4 | Verify independent hypotheses in parallel |
| Verifier (optional) | 1 | Cross-check the winning hypothesis after convergence |

Lead (you) does NOT write code during analysis — only coordinates and approves.

### When NOT to Use Deep Mode

- Bug is in a single file/component → Use Standard Mode
- Only one hypothesis makes sense → Use Standard Mode
- Bug is already well-understood → Just fix it directly
- Token budget is constrained → Use Standard Mode (sub-agents are cheaper)
- Platform lacks team/group support → Use Standard Mode only

### Deep Mode Red Flags

- "I can figure this out myself, don't need a team"
- "The bug is probably just [X], no need for multiple hypotheses"
- "Deep mode is overkill for this"

**If the bug has unknown root cause and multiple plausible explanations: Deep Mode is NOT overkill.**

---

## Exception & Fallback Handling

| Scenario | Detection | Action |
|----------|-----------|--------|
| Sub-agent fails/errors | Agent dispatch returns error or empty | Mark module as "unanalyzed", proceed with remaining agents, note gap in impact matrix |
| Coupled modules > 8 | Observed during Step 2 | Ask user: "发现 N 个耦合模块，要全部分析还是优先关键模块？" |
| Team creation fails | Tool returns error | Fall back to Standard Mode, inform user |
| Deep team member hits turn limit (~30) | System auto-stops the agent | Mark hypothesis as "未收敛 — 分析范围过大，建议缩小" |
| Deep team member idle (3+ turns with no tool calls or output) | Observed during monitoring | Shut down that member, mark hypothesis as "卡死" |
| All hypotheses refuted | Debate produces no confirmed root cause | Re-decompose hypotheses from a different angle, or fall back to Standard Mode |
| `references/coupling-patterns.md` missing | File not found | Continue with built-in checklist (already inline in Common Mistakes section) |
| Single agent produces ambiguous result | Report lacks clear yes/no on coupling | Mark "⚠️ needs manual review" in impact matrix, highlight to user |
