---
name: coupling-aware-debug
category: debugging
description: >-
  Use when debugging bugs where a fix might break something else. Covers 6 coupling
  dimensions: spatial (containers/z-index), state (shared stores/Context), timing (async
  races/useEffect order), event chain (stopPropagation/preventDefault), side-effect cascade
  (useEffect chains), state boundary (loading/empty/error). Use when the user reports a bug,
  says "debug", "修bug", "有问题", "报错", "不正常", "修复", or describes unexpected behavior
  in a feature that coexists with other modes. Use when a fix might create regression.
  Use "深度debug" / "复杂bug" / "跨模块debug" / "团队debug" to trigger deep mode (Agent Team).
version: 1.4.2
---

# Coupling-Aware Debug (耦合感知调试)

Debug workflow that prevents regression in coupled features. Before proposing any fix,
map the coupling surface and verify impact across all dependent modes.

**Violating the letter of this workflow is violating the spirit of this workflow.**

## Mode Selection (Read FIRST)

Before starting any workflow, check what the user said:

| User said | Mode | What to do |
|-----------|------|------------|
| `debug` / `修bug` / `有问题` / `报错` / `不正常` / `修复` / `无法` | **Standard** | Go to → Standard Workflow (subagent parallel analysis) |
| `深度debug` / `深度调试` / `复杂bug` / `跨模块debug` / `多系统bug` / `团队debug` | **Deep** | Go to → Deep Mode (Agent Team, below) |

**If uncertain which mode to use, ask: "这个 bug 涉及几个独立模块？有多个假设需要验证吗？"**

---

## Standard Workflow (Subagent Parallel Analysis)

Default mode for most bugs. Use code-explorer subagents launched in parallel.

### When to Use

Trigger when user reports a bug AND the broken feature is part of a system with
multiple modes/features that share containers, state, or event routing.

## Workflow (4 Steps)

### Step 1: Understand the Bug

- Reproduce the user's description to confirm understanding.
- Identify the exact file(s) and line(s) where the symptom manifests.
- **Pre-check — eliminate two common blind spots:**
  - [ ] **Error swallowing?** Check `try-catch` / `.catch(() => null)` / conditional render returning `null` — a silent error may be hiding the real root cause.
  - [ ] **Environment-specific?** Try another browser / resize the viewport / switch locale / clear cache — does the bug disappear in a different environment?
- Then **immediately ask the coupling question** before going deeper:

> **"What other features/modes share the infrastructure that this feature depends on?"**

Common shared layers: container `div`, overlay wrapper, Fabric canvas, global state,
event routing (`pointer-events`), z-index stacking, conditional rendering gate.

### Step 2: Coupling Analysis (Parallel) — MANDATORY, DO NOT SKIP

**Launch parallel subagents** (Task tool, `subagent_name="code-explorer"`) to inspect
EVERY coupling surface. This means not just spatial coupling (containers), but ALL six
coupling dimensions below. Launch one subagent per dimension, all simultaneously.
Wait for ALL to complete before forming a conclusion.

**This step is NOT optional.** Proposing a fix without completing Step 2 is a violation.

#### Six Coupling Dimensions (Analyze ALL in parallel)

| # | Dimension | Key Question | Check For |
|---|-----------|-------------|-----------|
| 1 | 🏗️ **Spatial** (original) | Who else renders in this container? | Shared `div`, canvas, overlay; `pointer-events`, `z-index`, conditional mounting |
| 2 | 🔗 **State** | Who else reads/writes this state? | Global store, Context, localStorage, URL params, props drilling chain |
| 3 | ⏱️ **Timing** | Who depends on the execution order? | `useEffect` ordering, async callbacks, promise chains, `setTimeout` races |
| 4 | 📡 **Event Chain** | Whose events flow through this node? | `stopPropagation`, `preventDefault`, event bubbling, capture phase listeners |
| 5 | 🔄 **Side-Effect Chain** | What `useEffect`/watcher does this trigger? | State → effect → more state → more effects cascades |
| 6 | 🔲 **State Boundary** | Does this fix hold in all edge states? | Loading / empty / error / boundary-value / null / undefined |

**Each subagent must answer:**

1. **Spatial:** Does this module share a container? What `pointer-events`/z-index/handlers does it set? In its passive state, does it block events?
2. **State:** List all consumers of the modified state. If the state structure changes, does each consumer handle the new shape?
3. **Timing:** Does any code assume a specific async completion order? Check `Promise.all`, `await` chains, `useEffect` dependency arrays.
4. **Event Chain:** Trace event propagation through the modified node. List every downstream/upstream listener. Would `stopPropagation`/`preventDefault` break any of them?
5. **Side-Effect Chain:** Starting from this state change, trace forward through all `useEffect`/`useMemo`/`watch`/`computed` to map the full cascade.
6. **State Boundary:** Test the fix under all edge states. If the normal state works but loading crashes at `data.map()`, the fix is incomplete.

**Example parallel analysis:**

```
Parallel subagents (launched simultaneously):
├── subagent A → Spatial: WatermarkLayer, SignatureLayer, RedactionLayer
├── subagent B → State: Redux store consumers, Context providers
├── subagent C → Timing: useEffect order, async data fetching sequence
├── subagent D → Event Chain: bubbling path from click target to document root
├── subagent E → Side-Effect Chain: setState → useEffect → downstream effects
└── subagent F → State Boundary: loading / empty / error states of affected components
```

### Step 3: Impact Assessment

Aggregate subagent reports into a coupling matrix before proposing any fix:

| Mode / Component | Spatial (Container/Events) | State (Who consumes?) | Timing (Order dependent?) | Event Chain (Propagation?) | Side-Effect Chain (Cascade?) | State Boundary (Edge cases?) | Mitigation |
|------|:---:|:---:|:---:|:---:|:---:|:---:|------------|
| component-A | ✅/❌ | ✅/❌ | ✅/❌ | ✅/❌ | ✅/❌ | ✅/❌ | how to protect |

**If the fix would break ANY other mode, redesign it.** The fix must:
- Fix the reported bug
- Cause zero regression in all other modes
- If impossible with a local change, report the architectural constraint to the user.

**⛔ STOP: Do not proceed to Step 4 until the user approves the coupling impact matrix.**
**First align on "what's broken", then discuss "how to fix".**

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

### Mistake 5: Only Testing the Happy Path

**What it looks like:** Agent fixes a bug in the normal/happy-path state (e.g., list rendering)
and assumes loading, empty, and error states are fine. Empty list crashes on `.map()`,
loading skeleton breaks because a prop changed, error boundary swallows the new code path.

**Why it happens:** The bug report only describes the happy-path symptom, so the agent only
thinks about the happy-path fix.

**How this skill prevents it:** Step 2 state-boundary analysis forces inspection of all
edge states: normal / loading / empty / error / boundary-value.

### Mistake 6: Event Chain Breakage

**What it looks like:** Adding `stopPropagation()` or `preventDefault()` to fix one
component's event handling, but this silently breaks 3 other components that relied on
that event bubbling up or default behavior.

**Why it happens:** The agent sees the immediate chain (caller → this handler) but doesn't
trace the full event propagation path (ancestors, siblings, nested children).

**How this skill prevents it:** Step 2 event-chain analysis maps the full propagation path
and identifies every listener that depends on event flow through the modified node.

### Mistake 7: Side-Effect Cascade

**What it looks like:** Changing one piece of state triggers a `useEffect`, which sets
another state, which triggers another `useEffect`... The page re-renders 10+ times
and behavior becomes unpredictable.

**Why it happens:** The agent doesn't trace the dependency chain of reactive effects.
One `setState` can cascade through multiple hooks, watchers, and derived values.

**How this skill prevents it:** Step 2 side-effect chain analysis traces each state change
forward through all `useEffect`/`useMemo`/`watch` dependencies to find cascading triggers.

## Red Flags — STOP and Re-evaluate

- "This is just a simple CSS/z-index fix"
- "I found the root cause, it's clearly [X]"
- "I don't need to check other modes for this"
- "The fix is too small to affect anything else"
- "I already traced the problem to this line"
- "This state is mine — no other component uses it"
- "Empty state is fine, I don't need to test it"
- "It's just event handling, won't affect anything else"

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
| "This state is only used by A" | Did you search the entire codebase? Global store might have 10 unknown consumers. |
| "I just added a stopPropagation" | The event bubble chain may have 5 downstream listeners that now get nothing. |
| "Empty state won't have problems" | Empty lists crash `.map()`, empty objects fail destructuring, boundary values break math. |

## Key Principles

1. **Coupling check before root cause** — Always ask "what shares this?" before "what broke this?"
2. **Parallel analysis is mandatory** — Launch all coupling-inspection subagents in one batch.
3. **Zero regression is the bar** — A fix that breaks another mode is not a fix.
4. **Passive state is still present** — readOnly components are in the DOM and can interfere.

For detailed coupling patterns and checklists, see `references/coupling-patterns.md`.

---

## Deep Mode (Agent Team — Competitive Hypothesis Debugging)

Triggered when user says: `深度debug`, `深度调试`, `复杂bug`, `跨模块debug`, `多系统bug`, `团队debug`.

Use this mode when the bug:
- Spans multiple independent modules (e.g., frontend + backend + database)
- Has 3+ plausible hypotheses that need parallel verification
- Involves multiple technology layers (CSS events, React state, API calls, etc.)
- Previous subagent-level analysis failed to find the root cause

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

Use `team_create` to create the debug team:

```
team_create("debug-{feature-name}")
```

Then launch members with `Task` tool (one per hypothesis), ALL in the same tool call batch:

```
Task(name="分析员-A", prompt="验证假设：overlay container pointer-events...")
Task(name="分析员-B", prompt="验证假设：conditional unmount...")
Task(name="分析员-C", prompt="验证假设：Fabric.js coordinate...")
```

**Critical:** Each member gets `max_turns` limited to 20-30 turns. Based on real testing, medium-complexity modules need ~14 turns; this leaves ample headroom for larger modules. If a member hits the limit without converging, mark that hypothesis as "未收敛" rather than retrying indefinitely.

#### Step 3: Debate & Converge

Members communicate via `send_message` to:
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
5. `team_delete` to clean up

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
- Token budget is constrained → Use Standard Mode (subagents are cheaper)

### Deep Mode Red Flags

- "I can figure this out myself, don't need a team"
- "The bug is probably just [X], no need for multiple hypotheses"
- "Deep mode is overkill for this"

**If the bug has unknown root cause and multiple plausible explanations: Deep Mode is NOT overkill.**

---

## Exception & Fallback Handling

| Scenario | Detection | Action |
|----------|-----------|--------|
| Subagent (code-explorer) fails/errors | Task tool returns error or empty | Mark module as "unanalyzed", proceed with remaining subagents, note gap in impact matrix |
| Coupled modules > 8 | Main agent observes count during Step 2 | Ask user: "发现 N 个耦合模块，要全部分析还是优先关键模块？" |
| `team_create` fails | Tool returns error | Fall back to Standard Mode, inform user "Deep mode unavailable, using Standard instead" |
| Deep team member hits `max_turns` (30) | System auto-stops the agent | Mark hypothesis as "未收敛 — 分析范围过大，建议缩小" |
| Deep team member idle (3+ turns with no tool calls or output) | Main agent observes during monitoring | Shutdown that member, mark hypothesis as "卡死" |
| All hypotheses refuted | Debate produces no confirmed root cause | Re-decompose hypotheses from a different angle, or fall back to Standard Mode |
| `references/coupling-patterns.md` missing | File not found | Continue with built-in checklist (already inline in SKILL.md Common Mistakes section) |
| Single agent in Standard Mode produces ambiguous result | Report lacks clear yes/no on coupling | Mark "⚠️ needs manual review" in impact matrix, highlight to user |

