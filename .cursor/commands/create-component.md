# Create component

You are **adding a new React component** in this Excalidraw monorepo. Follow the instructions below exactly.

## Gather inputs (if missing)

From the user message, infer or ask briefly for:

1. **Component name** — PascalCase identifier (e.g. `UserBadge`, not `user-badge`).
2. **Location** — directory path under one of:
   - `packages/excalidraw/components/` (or a nested folder there, e.g. `components/MyFeature/`) — editor / embeddable package UI
   - `excalidraw-app/` — host app shell only (`AGENTS.md`: library behavior belongs in `packages/excalidraw`)
   - Another workspace package under `packages/*` only if the user specified it and it matches that package’s role

Default assumption: **`packages/excalidraw/components/`** at the top level **only** if the user gave a name but no path.

## Read first

- **`AGENTS.md`** — protected paths, no new npm packages without approval, verification commands.
- **`.cursor/rules/conventions.mdc`** — components, TypeScript, file naming.
- For the **target directory**: open **one existing sibling component** (or `index.tsx` in that folder) and **match** import style, CSS/SCSS usage, `classnames`/`clsx` patterns, and test helpers.

## Implementation rules

**Structure**

- **File name:** `ComponentName.tsx` (PascalCase) in the chosen directory.
- **Export:** **named export** only — `export const ComponentName = …` or `export function ComponentName`. **No default export.**
- **Props type:** `type ComponentNameProps = { … }` (or `{ComponentName}Props` naming — the type name must be `{ComponentName}Props`).
- **Implementation:** functional component; hooks allowed. Use **`import type`** for type-only imports.

**Styling**

- If sibling files in that folder import a co-located **`ComponentName.scss`**, do the same; otherwise do not invent new global styles unless the user asked. Prefer existing primitives/patterns in that folder.

**State and architecture**

- In **`packages/excalidraw/**`**: user-visible editor state changes belong in **ActionManager** / actions — do not add Redux/Zustand/MobX for editor state. Pure presentational components are fine without touching actions.
- **Do not edit** paths in **`do-not-touch.mdc`** / **`AGENTS.md`** without explicit user approval (`renderer`, `restore`, `actions/manager`, core `types`).

**Tests**

- Unless the user declines: add **`ComponentName.test.tsx`** next to the component (**colocated**), following naming and patterns in nearby tests (e.g. `packages/excalidraw/tests/test-utils` where used elsewhere).
- Keep tests meaningful but proportional; avoid empty placeholder tests.

**Barrel / public API**

- Add or re-export from **`packages/excalidraw/index.tsx`** (or similar package entry) **only** if this component must be part of the **published** surface; otherwise rely on normal file imports within the repo.

## After creating files

1. Fix imports and types until the editor shows no issues in the touched files.
2. Run the **narrowest** checks from **`AGENTS.md`**, typically:
   - `yarn test:typecheck`
   - `yarn test:code`
   - `yarn test:app --watch=false` (or a path filter matching the new test/component if the repo supports it)

## Output format

Summarize for the user:

- Paths created (and updated, if any).
- Any assumptions (location, props, tests, exports).
- Exact verification commands you ran or recommend.

Do not add unrelated refactors or new npm dependencies without explicit approval.
