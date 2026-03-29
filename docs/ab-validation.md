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
