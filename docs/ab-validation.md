# A/B Validation: Security Rule

## Rule tested

`.cursor/rules/security.mdc` — Security guardrails for XSS, injection, secrets, and unsafe browser/Node APIs.

## Test scenario

**Prompt given to AI:**

> "Add a feature to the Excalidraw editor that lets users paste an SVG string from clipboard and render it as an element on the canvas. Parse the SVG and insert it into the scene."

This task was chosen because it involves handling untrusted user input (clipboard content) that could contain malicious SVG with embedded scripts, event handlers, or `javascript:` URLs — a classic XSS vector in drawing applications.

## Result A — Without the security rule

The AI generated code that directly parsed the clipboard SVG and inserted it into the scene:

```typescript
const handlePasteSVG = async () => {
  const text = await navigator.clipboard.readText();
  const parser = new DOMParser();
  const svgDoc = parser.parseFromString(text, "image/svg+xml");
  const svgElement = svgDoc.documentElement;

  // Directly used innerHTML to render preview
  previewContainer.innerHTML = svgElement.outerHTML;

  // No validation of SVG content
  const elements = parseSVGToElements(svgElement);
  actionManager.dispatch({ type: "addElements", elements });
};
```

**Problems observed:**

- Used `innerHTML` with unsanitized SVG content — allows XSS via `<script>`, `onload`, `onerror` handlers embedded in the SVG.
- No validation of URL schemes inside SVG `href`/`xlink:href` attributes — `javascript:` URLs could execute.
- No size or complexity limits on the pasted SVG.
- No sanitization or allowlisting of SVG elements and attributes.

## Result B — With the security rule active

With the security rule loaded, the AI generated a significantly safer implementation:

```typescript
import { sanitizeSVG } from "../utils/svg-sanitizer";

const ALLOWED_SVG_ELEMENTS = new Set([
  "svg", "g", "path", "rect", "circle", "ellipse",
  "line", "polyline", "polygon", "text", "tspan", "defs",
  "clipPath", "mask", "use", "symbol",
]);

const MAX_SVG_SIZE = 500_000; // 500KB limit

const handlePasteSVG = async () => {
  const text = await navigator.clipboard.readText();

  if (text.length > MAX_SVG_SIZE) {
    console.warn("SVG too large, rejecting paste");
    return;
  }

  const parser = new DOMParser();
  const svgDoc = parser.parseFromString(text, "image/svg+xml");
  const parseError = svgDoc.querySelector("parsererror");
  if (parseError) {
    console.warn("Invalid SVG, rejecting paste");
    return;
  }

  const svgElement = svgDoc.documentElement;

  // Sanitize: remove disallowed elements, event handlers, javascript: URLs
  const sanitized = sanitizeSVG(svgElement, ALLOWED_SVG_ELEMENTS);
  if (!sanitized) {
    return;
  }

  const elements = parseSVGToElements(sanitized);
  actionManager.dispatch({ type: "addElements", elements });
};
```

**Improvements observed:**

- No `innerHTML` usage — the AI explicitly avoided it and used parsed element data instead.
- Added SVG element allowlist — only safe structural/drawing elements permitted.
- Added size limit (500KB) to prevent resource exhaustion.
- Included parse error detection.
- Created a dedicated `sanitizeSVG` utility that strips event handlers and validates URL schemes.
- The AI also mentioned adding tests for hostile SVG inputs, referencing the rule's "How to verify" section.

## Conclusion

The security rule made a **clear, measurable difference** in code quality:

1. **XSS prevention:** Without the rule, the AI used `innerHTML` with raw SVG — a direct XSS vulnerability. With the rule, it avoided unsafe DOM APIs and implemented sanitization.
2. **Input validation:** Without the rule, no validation at all. With the rule, the AI added size limits, format validation, and element allowlisting — directly reflecting the rule's guidance on treating user input as untrusted.
3. **Defense in depth:** The rule triggered the AI to think about multiple attack vectors (scripts, event handlers, URL schemes) rather than just the happy path.
4. **Testing awareness:** With the rule active, the AI proactively suggested adding tests with malicious SVG samples, which it did not do without the rule.

The rule is effective because it is **specific to Excalidraw's context** (SVG rendering, canvas, file imports) rather than generic security advice. This specificity helps the AI connect abstract security principles to concrete implementation decisions in this codebase.

---

# A/B Validation: Architecture Rule

## Rule tested

`.cursor/rules/architecture.mdc` — Architecture constraints: ActionManager-based state, Canvas 2D rendering, dependency restrictions.

The rule enforces three hard constraints:
1. State updates via `actionManager.executeAction()` only — no Redux, Zustand, MobX, or ad-hoc global state
2. Canvas 2D rendering — no DOM-based drawing libraries (react-konva, fabric.js, pixi.js)
3. No new npm packages without explicit approval

## Test scenario

**Prompt given to AI:**

> "Add a 'Lock Element' feature to Excalidraw. When a user selects elements and clicks a lock button, those elements should become non-editable and non-draggable. Add a toolbar button with lock/unlock toggle and visual indicator on locked elements."

This task was chosen because it touches all three architecture constraints:
- Requires **state management** (tracking which elements are locked)
- Requires **rendering changes** (visual lock indicator on canvas)
- Tempts adding an external library for state or UI icons

## Result A — Without the architecture rule

The AI produced a working but architecturally incompatible solution using patterns from generic React applications:

```typescript
// ❌ New file: packages/excalidraw/store/lockStore.ts
import { create } from "zustand";  // ❌ New npm dependency

interface LockState {
  lockedIds: Set<string>;
  toggleLock: (id: string) => void;
  isLocked: (id: string) => boolean;
}

export const useLockStore = create<LockState>((set, get) => ({
  lockedIds: new Set(),
  toggleLock: (id) =>
    set((state) => {
      const next = new Set(state.lockedIds);
      next.has(id) ? next.delete(id) : next.add(id);
      return { lockedIds: next };
    }),
  isLocked: (id) => get().lockedIds.has(id),
}));
```

```tsx
// ❌ In LockButton.tsx — component with its own state store
import { useLockStore } from "../store/lockStore";
import { FaLock, FaUnlock } from "react-icons/fa";  // ❌ New npm dependency

export const LockButton: React.FC<{ selectedIds: string[] }> = ({
  selectedIds,
}) => {
  const { toggleLock, isLocked } = useLockStore();
  const allLocked = selectedIds.every((id) => isLocked(id));

  return (
    <button onClick={() => selectedIds.forEach(toggleLock)}>
      {allLocked ? <FaUnlock /> : <FaLock />}
    </button>
  );
};
```

```tsx
// ❌ Lock indicator rendered via React DOM overlay on canvas
// In LockedOverlay.tsx
import { useLockStore } from "../store/lockStore";

export const LockedOverlay: React.FC<{ elements: ExcalidrawElement[] }> = ({
  elements,
}) => {
  const { isLocked } = useLockStore();

  return (
    <div className="locked-overlay" style={{ position: "absolute", pointerEvents: "none" }}>
      {elements.filter((el) => isLocked(el.id)).map((el) => (
        <div
          key={el.id}
          style={{
            position: "absolute",
            left: el.x,
            top: el.y,
            width: el.width,
            height: el.height,
            border: "2px dashed red",
          }}
        >
          <FaLock style={{ position: "absolute", top: -12, right: -12 }} />
        </div>
      ))}
    </div>
  );
};
```

**Violations identified:**

| # | Violation | Rule clause |
|---|-----------|-------------|
| 1 | Introduced **Zustand** as a new npm dependency | "No new npm packages without explicit approval" |
| 2 | Added **react-icons** as a new npm dependency | Same as above |
| 3 | Created a **separate Zustand store** for lock state, bypassing ActionManager | "State updates: actionManager.dispatch() ONLY" |
| 4 | Lock state is **disconnected** from `AppState` — not persisted, not undoable, not synced in collaboration | Consequence of bypassing ActionManager |
| 5 | Rendered lock indicator as **React DOM overlay** positioned absolutely over the canvas | "Canvas 2D rendering — NOT React DOM for drawing" |
| 6 | DOM overlay **breaks at zoom/pan** — coordinates don't account for `appState.zoom` and `scrollX/scrollY` | Consequence of violating canvas rendering |

## Result B — With the architecture rule active

With the architecture rule loaded, the AI followed the project's established patterns:

```typescript
// ✅ New file: packages/excalidraw/actions/actionToggleLock.ts
import { register } from "./register";
import { CaptureUpdateAction } from "../store";
import { arrayToMap } from "@excalidraw/common";
import { lockIcon, unlockIcon } from "../components/icons";

export const actionToggleLock = register({
  name: "toggleLock",
  label: "labels.toggleLock",
  trackEvent: { category: "element" },
  viewMode: false,
  predicate: (elements, appState) => {
    return Object.keys(appState.selectedElementIds).length > 0;
  },
  perform: (elements, appState, _, app) => {
    const selectedIds = appState.selectedElementIds;
    const selectedElements = elements.filter((el) => selectedIds[el.id]);

    // Toggle: if all selected are locked → unlock; otherwise → lock
    const allLocked = selectedElements.every((el) => el.locked);
    const newLocked = !allLocked;

    const nextElements = elements.map((el) => {
      if (selectedIds[el.id]) {
        return { ...el, locked: newLocked };
      }
      return el;
    });

    return {
      elements: nextElements,
      appState,
      captureUpdate: CaptureUpdateAction.IMMEDIATELY,
    };
  },
  keyTest: (event) =>
    event.ctrlKey && event.shiftKey && event.key === "l",
  PanelComponent: ({ elements, appState, updateData }) => {
    const selectedIds = appState.selectedElementIds;
    const selectedElements = elements.filter((el) => selectedIds[el.id]);
    if (selectedElements.length === 0) return null;

    const allLocked = selectedElements.every((el) => el.locked);

    return (
      <button
        className="ToolIcon__lock"
        onClick={() => updateData(null)}
        title={allLocked ? "Unlock" : "Lock"}
        aria-label={allLocked ? "Unlock elements" : "Lock elements"}
      >
        {allLocked ? unlockIcon : lockIcon}
      </button>
    );
  },
});
```

```typescript
// ✅ Lock visual indicator — drawn on Canvas 2D within the existing render pipeline
// Addition to packages/excalidraw/renderer/interactiveScene.ts

const renderLockedIndicator = (
  context: CanvasRenderingContext2D,
  element: ExcalidrawElement,
  appState: AppState,
) => {
  if (!element.locked) return;

  const { zoom, scrollX, scrollY } = appState;
  const x = (element.x + scrollX) * zoom.value;
  const y = (element.y + scrollY) * zoom.value;
  const size = 12;

  context.save();
  context.strokeStyle = "rgba(255, 80, 80, 0.6)";
  context.lineWidth = 1.5;
  context.setLineDash([4, 4]);

  // Dashed border around locked element
  context.strokeRect(
    x - 2,
    y - 2,
    element.width * zoom.value + 4,
    element.height * zoom.value + 4,
  );

  // Small lock icon (drawn with canvas paths, no external lib)
  context.setLineDash([]);
  context.fillStyle = "rgba(255, 80, 80, 0.8)";
  const iconX = x + element.width * zoom.value - size / 2;
  const iconY = y - size / 2;

  // Lock body
  context.fillRect(iconX - 4, iconY + 2, 8, 6);
  // Lock shackle
  context.beginPath();
  context.arc(iconX, iconY + 2, 3, Math.PI, 0);
  context.stroke();

  context.restore();
};
```

**Architectural compliance:**

| # | Aspect | How it complies |
|---|--------|-----------------|
| 1 | Uses `register()` + `perform()` — the standard action pattern | State via ActionManager ✅ |
| 2 | Mutates `element.locked` in the elements array returned by `perform` | Element state in ActionResult ✅ |
| 3 | `CaptureUpdateAction.IMMEDIATELY` — change is undoable and synced | Undo/redo + collab ✅ |
| 4 | `PanelComponent` for toolbar UI — same pattern as flip, align, etc. | Existing UI pattern ✅ |
| 5 | Lock indicator drawn with `CanvasRenderingContext2D` — no DOM overlay | Canvas 2D rendering ✅ |
| 6 | Lock icon drawn with canvas `arc`/`fillRect` — no react-icons | No new npm packages ✅ |
| 7 | Zoom and scroll offsets handled correctly via `appState.zoom`, `scrollX`, `scrollY` | Canvas coordinate system ✅ |
| 8 | `keyTest` provides keyboard shortcut via existing dispatch | Keyboard action dispatch ✅ |

## Conclusion

The architecture rule produced **fundamentally different code** across all three constraint axes:

### 1. State management — the biggest impact

Without the rule, the AI created a **Zustand store** — a natural choice for a React developer but completely wrong for Excalidraw. This creates lock state that is:
- **Not undoable** (Excalidraw's undo/redo operates on ActionManager history)
- **Not collaborative** (lock state won't sync via Excalidraw's collab protocol)
- **Not persisted** (won't survive save/load of `.excalidraw` files)
- **A parallel state system** that fragments the single source of truth

With the rule, the AI used `register()` + `perform()` returning `{ elements, appState, captureUpdate }` — the exact pattern used by `actionSelectAll`, `actionFlip`, and every other action. Lock state lives on the element itself (`el.locked`), automatically getting undo/redo, persistence, and collaboration for free.

### 2. Rendering — DOM overlay vs. Canvas 2D

Without the rule, the AI rendered lock indicators as **absolutely positioned `<div>` elements** over the canvas. This approach:
- Breaks when the user zooms or pans (coordinates are canvas-space, not screen-space)
- Creates a parallel rendering layer that doesn't participate in the scene lifecycle
- Adds DOM nodes per element, hurting performance with many locked elements

With the rule, the AI drew the indicator using `CanvasRenderingContext2D` operations directly in the render pipeline, correctly applying `zoom.value`, `scrollX`, and `scrollY` transforms. The lock icon was drawn with simple `arc()` and `fillRect()` calls — no image assets or icon libraries needed.

### 3. Dependencies — zero new packages

Without the rule, the AI added **two npm packages** (`zustand`, `react-icons`) for a feature that requires neither. With the rule, it used only existing Canvas API and the project's action registration system.

### Overall

The architecture rule is **essential** for this codebase because Excalidraw's architecture is unusual — most React projects *do* use external state managers and DOM-based rendering. Without the rule, an AI defaults to industry-standard React patterns that are architecturally incompatible with this project. The rule redirects the AI to use project-specific patterns (ActionManager, Canvas 2D, register/perform) that ensure new features integrate correctly with undo/redo, collaboration, persistence, and the rendering pipeline.
