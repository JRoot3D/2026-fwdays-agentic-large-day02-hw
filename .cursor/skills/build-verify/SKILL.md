---
name: build-verify
description: Runs yarn build at the project root after code changes, fixes compilation errors without ts-ignore or test hacks, and reports status with fixes and full output. Use when the user asks to build, verify, or check compilation, or after edits that might affect compilation.
---

# Skill: Build & Verify

## When to use

After making code changes that might affect compilation.
Triggered by: "build", "verify", "check compilation".

## Inputs

- Changed files (from git diff or conversation context)

## Steps

1. Run `yarn build` in the project root
2. If build succeeds → report success, list changed files
3. If build fails:
   a. Read error output — identify file, line, error type
   b. Open the file at the error line
   c. Fix the issue (type error, missing import, syntax)
   d. Re-run `yarn build`
   e. Repeat until build passes (max 3 attempts)

## Outputs

- Build status: PASS / FAIL
- List of fixes applied (if any)
- Full build output

## Safety

- Do NOT fix errors by adding `@ts-ignore` or `any`
- Do NOT modify test files to fix build errors
- If 3 attempts fail — stop and report to user