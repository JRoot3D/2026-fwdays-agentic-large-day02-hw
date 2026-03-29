# AGENTS.md — guidance for AI coding agents

This file orients automated assistants working in the **Excalidraw** monorepo (`excalidraw-monorepo`). It does not replace upstream docs; it summarizes constraints and entry points so changes stay consistent with this fork.

## Project overview

Excalidraw is an open-source, hand-drawn whiteboard: users sketch diagrams on an infinite canvas with shapes, text, arrows, and collaboration-friendly export. This repository ships a **Vite**-powered web app and an embeddable **`@excalidraw/excalidraw`** React library so consumers can embed the editor in their own products. The editor renders the scene with **Canvas 2D**, not a DOM-based drawing stack; application state is centralized and updated through a structured action layer rather than ad-hoc global stores. Deeper product and technical narrative lives in `docs/memory/projectbrief.md` and `docs/technical/architecture.md`.

## Tech stack

- **UI:** React (functional components and hooks).
- **Language:** TypeScript (strict; avoid `any` and `@ts-ignore`).
- **App bundling / dev:** Vite.
- **Monorepo:** Yarn 1.x workspaces (`packageManager` in root `package.json`); **Node ≥ 18**.
- **Tests:** Vitest; colocate tests as `ComponentName.test.tsx` next to components.
- **Quality:** ESLint, Prettier.

## Project structure

The repo is a **Yarn workspace** monorepo:

- **`excalidraw-app/`** — Host application that runs the full editor experience in development and production builds.
- **`packages/excalidraw/`** — Core editor: actions, canvas scene, UI shell, and the publishable `@excalidraw/excalidraw` package surface.
- **`packages/element/`**, **`packages/math/`**, **`packages/common/`**, **`packages/utils/`** — Shared geometry, element model, utilities, and cross-cutting helpers consumed by the editor.
- **`examples/`** — Example integrations and demos.

Prefer extending existing packages over introducing new workspace packages without discussion.

## Key commands

Run these from the **repository root**:

| Command | Purpose |
|---------|---------|
| `yarn start` | Dev server for `excalidraw-app` |
| `yarn build` | Production build (app) |
| `yarn build:packages` | Build workspace packages in order |
| `yarn test` / `yarn test:app` | Vitest |
| `yarn test:all` | Typecheck + lint + Prettier check + tests |
| `yarn test:typecheck` | `tsc` |
| `yarn test:code` | ESLint |

After substantive edits, run the narrowest check that covers your change (for example `yarn test` or `yarn test:all`).

## Architecture

- **State:** UI and editor updates flow through **`ActionManager`** (`dispatch` / registered actions), not Redux, Zustand, MobX, or other ad-hoc global state libraries. **`AppState`** and related types live around `packages/excalidraw/types.ts`.
- **Rendering:** The scene is drawn with **Canvas 2D**. Do not add react-konva, Fabric.js, PixiJS, or similar for the core editor canvas.
- **Main pieces:** The editor package combines the action system, scene/rendering pipeline, persistence/restore paths, and React UI; supporting packages own math, element definitions, and shared utilities.
- **Dependencies:** Do not add new **npm** dependencies without explicit approval; prefer `packages/utils` and existing helpers first.

## Conventions

- **Components:** Functional components and hooks only; **named exports**; props type named `{ComponentName}Props`.
- **Files:** Kebab-case for non-component files; **PascalCase** for component files.
- **TypeScript:** Use `import type { … }` for type-only imports.
- **Tests:** Colocate `ComponentName.test.tsx` with the component.
- **PR / contribution workflow:** Follow `CONTRIBUTING.md` for branching, commit expectations, and review norms; run `yarn test:all` (or the narrowest relevant scripts) before pushing when your change touches shared behavior.
- **Rule files:** When editing, consult `.cursor/rules/` — especially `conventions.mdc` (TS/React patterns), `architecture.mdc` (Excalidraw-specific constraints), and `do-not-touch.mdc` (protected paths).

## Do-not-touch and constraints

**Protected files** — do not modify without explicit user approval and a clear verification plan (see `.cursor/rules/do-not-touch.mdc`):

- `packages/excalidraw/scene/renderer.ts` — render pipeline (`Renderer.ts` on disk).
- `packages/excalidraw/data/restore.ts` — file format compatibility.
- `packages/excalidraw/actions/manager.ts` — action system (`manager.tsx` on disk).
- `packages/excalidraw/types.ts` — core `AppState` and shared types.

If the user explicitly asks to change one of these, confirm scope and verification steps first. When a change is approved, expect to run **`yarn test:all`**, **`yarn build:packages`** (and **`yarn build`** if packaging interactions apply), plus manual QA on load/save `.excalidraw` flows, undo/redo, export, and canvas visuals as appropriate.

**Other constraints:** No new npm packages without approval; no client-side scene rendering via DOM-heavy drawing libraries for the core editor; treat user-controlled inputs per `.cursor/rules/security.mdc`.

## Priority of instructions

1. **User’s explicit request** in the chat (and any project-specific files they ask you to follow).
2. **This file** and other repo instruction files (e.g. `.github/copilot-instructions.md`).
3. **Cursor rules** under `.cursor/rules/`.
4. General best practices.

## Cursor rules (quick map)

| Rule | Scope |
|------|--------|
| `conventions.mdc` | TS/React: functional components, hooks, naming, tests, strict TS — `packages/**/*.ts(x)` |
| `architecture.mdc` | Excalidraw: `actionManager` + `AppState`, canvas 2D, no ad-hoc state libs — `packages/excalidraw/**` |
| `do-not-touch.mdc` | Protected files — do not edit without explicit approval |

## Math and geometry

For math-related code, use types from `packages/math` (e.g. **`Point`**) rather than ad-hoc `{ x, y }` objects. See `packages/math/src/types.ts` and `.github/copilot-instructions.md`.

## Human-oriented docs

- Contributing: `CONTRIBUTING.md`
- Dev setup: `docs/technical/dev-setup.md`
- Product/architecture memory: `docs/memory/`, `docs/technical/architecture.md`

When in doubt, read the surrounding code and match existing patterns before introducing new abstractions.
