# Code review

You are performing a **code review** for this repository. Follow the instructions below exactly.

## Scope

1. If the user referenced specific files, commits, or a selection, **limit the review to that scope**.
2. Otherwise, review **the current working change** (staged and unstaged diffs, or the most recently edited files the user likely cares about). Say what scope you assumed if it is not obvious.

## Read first (when relevant)

- `AGENTS.md` — architecture constraints, protected paths, verification commands.
- `.cursor/rules/*.mdc` — especially `security.mdc`, `testing.mdc`, `architecture.mdc`, `do-not-touch.mdc`, `conventions.mdc` for paths that match the change.

## Review dimensions

Work through these in order. Skip a bullet only if it clearly does not apply.

**Correctness and design**

- Does the change solve the stated problem without obvious regressions?
- For `packages/excalidraw/**`: state should go through **ActionManager** / actions; avoid ad-hoc global state libraries for editor state.
- Math/geometry: prefer `packages/math` types (e.g. `Point`) over ad-hoc objects where the codebase already does so.

**Conventions**

- Functional React components and hooks; **named exports**; props type `{ComponentName}Props`.
- File naming: kebab-case for non-components, PascalCase for component files.
- Strict TypeScript: flag `any`, `@ts-ignore`, and avoidable type escapes.

**Protected and high-risk areas**

- If the diff touches paths listed in `do-not-touch.mdc` / `AGENTS.md` (`renderer`, `restore`, `manager`, core `types`), call that out explicitly and require explicit approval + full verification.

**Security** (any untrusted input, URLs, HTML, files, network, secrets)

- No secrets in code or logs; safe URL handling; no unsafe HTML/DOM patterns; no `eval` / dynamic import from user content; shell/build-from-user-input risks in scripts.

**Dependencies**

- Flag **new npm packages** unless the change clearly has explicit approval per project rules.

**Tests and verification**

- Note missing tests for new behavior or tricky edge cases.
- Suggest the **narrowest** appropriate checks from `AGENTS.md` (e.g. `yarn test:typecheck`, `yarn test:code`, `yarn test:app --watch=false`, `yarn test:all` for wide or protected changes).

## Output format

Use this structure:

### Summary

2–4 sentences: what changed and whether it is on track.

### Findings

For each issue (ordered by severity: **Blocker** → **Major** → **Minor** → **Nit**):

- **Severity** — Blocker | Major | Minor | Nit  
- **Location** — `path:line` (or best available)  
- **Issue** — what is wrong and why it matters  
- **Suggestion** — concrete fix or alternative (code snippet only when it clarifies)

If something is done well and non-obvious, add a short **Strengths** subsection.

### Verification

Bulleted list of commands the author should run before merge, tailored to this diff.

Be direct and specific; avoid generic praise. Do not rewrite the whole change unless the user asked.
