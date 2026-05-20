# Common Coupling Patterns in Multi-Mode UIs

This reference catalogues recurring coupling patterns observed in real projects.
Use these as a checklist when performing Step 2 (Coupling Analysis).

---

## Pattern 1: Shared Overlay Container

**Symptom:** Multiple feature modes render into a single `<div>` wrapper that controls `pointer-events` and `z-index`.

**Example:** A PDF editor with 8 modes (pages, editText, addShape, addText, redact, watermark, signature, image)
all render their overlays into one container in `PagePreview`.

**Coupling surface:**
- The container's `pointer-events` affects ALL child modes simultaneously.
- Making the container `pointer-events: none` (to fix shape drawing) kills text editing.
- Making it `pointer-events: auto` (to fix text editing) blocks shape drawing.

**Correct fix pattern:** Add a prop (`overlayInteractive`) that lets each mode control the container's
`pointer-events` independently, while keeping all children always rendered for preview.

---

## Pattern 2: Z-Index Stacking Conflict

**Symptom:** Two layers overlap visually, and the upper layer (even with `pointer-events: none`)
intercepts events because its parent has `pointer-events: auto`.

**Key insight:** CSS `pointer-events: none` on a child does NOT make events fall through to layers
behind the parent. The parent still sits in the stacking context and catches events.

**Checklist for z-index coupling:**
- [ ] List all layers in the stacking context and their z-index values.
- [ ] For each layer, check if it or its parent has `pointer-events: auto`.
- [ ] For modes that need event passthrough, trace up the DOM tree to find the first ancestor
      with `pointer-events: auto` — that ancestor is the true blocker.

---

## Pattern 3: Conditional Mount/Unmount

**Symptom:** Components are conditionally mounted based on mode, so switching modes
destroys state (annotations disappear from preview).

**Example:** A component checks `{isWatermarkMode ? <WatermarkLayer /> : null}`,
so switching to pages mode unmounts the watermark preview.

**Correct fix pattern:** Always mount the preview layer; use `readOnly` / `hidePanel` props
to control interactivity without unmounting. This decouples "visibility of annotations"
from "current editing mode".

---

## Pattern 4: Global State Shared Across Modes

**Symptom:** Multiple modes share the same React state (useState/useReducer/Context).
Resetting state for one mode clears data for another.

**Checklist:**
- [ ] Identify all state variables shared across modes.
- [ ] For each variable, check if a mode change triggers a reset.
- [ ] If a mode change should NOT reset data, ensure the reset logic is gated.

---

## Pattern 5: Fabric.js / Canvas Layer Sharing

**Symptom:** A Fabric.js canvas is reused across addText and addShape modes.
Enabling interaction for one mode affects the other.

**Checklist:**
- [ ] Check `fabricOverlayInteractive` or equivalent props.
- [ ] Verify that `readOnly` mode on the Fabric layer doesn't prevent rendering
      of previously drawn annotations.
- [ ] Verify that switching modes preserves Fabric annotations (they should be cached per page).

---

## Debug Trigger Checklist

When the user reports a bug, before proposing a fix, check:

- [ ] Which component is the symptom in?
- [ ] What shared container/parent does it render into?
- [ ] How many other modes/features render into the same container?
- [ ] What controls `pointer-events`, `z-index`, and conditional mounting for that container?
- [ ] For each coupled mode, what is its readOnly/passive state behavior?
- [ ] Would the proposed fix change any of the above?
- [ ] If yes, does it break any coupled mode?

If the answer to the last question is "yes", redesign the fix.
