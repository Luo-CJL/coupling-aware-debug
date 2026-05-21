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

## Pattern 6: Global State Shared Across Components

**Symptom:** Multiple components consume or mutate the same global state (Redux, Zustand, Context).
Changing the state shape, key, or update logic in one component silently breaks consumers
that expect the old shape.

**Example:** A `filters` object is shared by SearchBar (writes) and ProductList, PriceRange,
SortDropdown (all read). Changing `filters.price` from `number` to `{min, max}` breaks
PriceRange and SortDropdown because they read `filters.price` as a number.

**Coupling surface:**
- Any component that imports the same store/context/hook.
- Derived state (`useMemo`, `useSelector` with transforms) that assumes a specific shape.
- State reset/clear actions that don't account for all consumers' expectations.

**Checklist:**
- [ ] Search for ALL consumers of the modified state variable (grep for the state key).
- [ ] For each consumer, verify it handles the new type/shape/value range.
- [ ] Check if any `useEffect` has this state in its dependency array.
- [ ] If changing state structure, add migration/fallback for existing data.

---

## Pattern 7: Async Timing / Race Condition

**Symptom:** Two or more async operations (API calls, `setTimeout`, event handlers) complete
in an unpredictable order, and code assumes one always finishes first.

**Example:** Component A fetches user data and sets `user` state. Component B reads `user.avatar`
in its `useEffect([])`. On first render, `user` is `null` → crash. The fix (add `user` to deps)
now triggers double-fetch because another effect also depends on `user`.

**Coupling surface:**
- All `useEffect` / `useLayoutEffect` with shared dependencies.
- Promise chains that might resolve while a component is unmounting.
- `setTimeout` / `setInterval` that execute after state has changed.

**Checklist:**
- [ ] For each async operation, list every piece of state it writes to.
- [ ] List every `useEffect` that reads that state → what happens if state is stale?
- [ ] Check for missing cleanup (`useEffect` return / `AbortController`).
- [ ] Check for "if (data) render" guards that skip the race instead of handling it.

---

## Pattern 8: Event Propagation Chain

**Symptom:** Adding `stopPropagation()` or `preventDefault()` to fix one event handler
silently breaks all other handlers that depend on the event flowing through that DOM node.

**Example:** A modal adds `e.stopPropagation()` to prevent clicks inside from closing the modal.
But a dropdown menu nested inside the modal depends on `document.addEventListener('click')`
(via event bubbling) to close itself. The modal's stopPropagation traps the event, and
the dropdown stays open forever.

**Coupling surface:**
- All ancestor elements with event listeners on the same event type.
- All descendant elements whose events bubble up through this node.
- Global/document-level listeners that rely on event propagation.

**Checklist:**
- [ ] Trace the full propagation path: target → ... → document.
- [ ] List every event listener on every node in the path (same event type).
- [ ] If using `stopPropagation`, does any parent node have a listener that should still fire?
- [ ] If using `preventDefault`, does any code rely on the default behavior (form submit, link navigation)?

---

## Pattern 9: Side-Effect Reactivity Chain

**Symptom:** A state change triggers a `useEffect`, which sets another state, which triggers
another `useEffect`... creating an unintended cascade of re-renders and state mutations.

**Example:** `setFilters(newFilter)` → `useEffect([filters])` fetches data → `setData(res)`
→ `useEffect([data])` re-calculates aggregates → `setAggregates(val)` → `useEffect([aggregates])`
updates UI. 4 renders for one action. Worse: changing `filters` during any intermediate
render creates stale-closure bugs.

**Coupling surface:**
- Any `useEffect`/`useMemo`/`useCallback` whose dependency array includes the modified state.
- State that is both written by an effect and read by another effect.
- Derived state (computed, selectors) that fans out to multiple components.

**Checklist:**
- [ ] Starting from the modified state, follow every `useEffect` dependency chain forward.
- [ ] Count the minimum renders: 1 action should cause at most 2 renders (state set + effect).
- [ ] Check for "ping-pong": effect A sets X, effect B reads X and sets Y, effect A also reads Y...
- [ ] Consider: can this be a single derived value instead of state + effect?

---

## Pattern 10: State Boundary / Edge Case

**Symptom:** A fix is tested and works for normal data, but crashes, renders blank, or
behaves incorrectly in loading, empty, error, or boundary-value states.

**Example:** A list component's sort function is fixed for populated data. But when the list
is empty, `data.sort()` returns `undefined` (API changed response format on empty), and
the component white-screens. Loading state shows old cached data with the new sort key
that doesn't exist in cached entries.

**Coupling surface:**
- Four mandatory states: **loading** (data is null/undefined), **empty** (data is `[]`/`{}`),
  **error** (error object exists), **normal** (data populated).
- Boundary values: `0`, `""`, `Infinity`, negative numbers, very long strings.
- Authentication states: logged out, expired token, insufficient permissions.

**Checklist:**
- [ ] Test the fix in ALL four states: loading, empty, error, normal.
- [ ] What happens when the value is `0`, `""`, `null`, `undefined`, `NaN`?
- [ ] Does the fix introduce a new field that doesn't exist in cached/old data?
- [ ] If the component has a skeleton/placeholder, does the fix break its rendering?

---

## Debug Trigger Checklist

When the user reports a bug, before proposing a fix, check:

**Spatial coupling:**
- [ ] Which component is the symptom in?
- [ ] What shared container/parent does it render into?
- [ ] How many other modes/features render into the same container?
- [ ] What controls `pointer-events`, `z-index`, and conditional mounting for that container?
- [ ] For each coupled mode, what is its readOnly/passive state behavior?

**State coupling:**
- [ ] Which state variable is being modified?
- [ ] List ALL consumers of that state (grep the key name).
- [ ] If changing state structure, can all consumers handle the new shape?

**Timing coupling:**
- [ ] What async operations are involved?
- [ ] In what order do `useEffect` hooks execute?
- [ ] Could a race condition produce stale or missing data?

**Event chain coupling:**
- [ ] Trace event propagation from target to document root.
- [ ] Would `stopPropagation`/`preventDefault` break any listener on the path?

**Side-effect chain coupling:**
- [ ] Trace each state change forward through all `useEffect` dependencies.
- [ ] Is there a "ping-pong" cascade where effects trigger each other?

**State boundary:**
- [ ] Test the fix in loading / empty / error / normal states.
- [ ] What happens with boundary values (`0`, `""`, `null`, `undefined`)?

If the answer to any coupling question is "yes, it breaks something", redesign the fix.
