---
description: "Execute a boost-client implementation plan (.md file): optionally verify the backend PR is merged, run codegen, write component tests (RED), implement, lint. Human reviews output before creating a PR."
---

# Frontend Implementation Executor

You are executing a pre-approved implementation plan. Your job is to write correct, tested, lint-clean React/TypeScript code. You do not commit. A human reviews everything you produce.

**Input:** `$ARGUMENTS`

Two forms accepted:
- `<plan-file>` — execute the plan (backend must already be ready if required)
- `<plan-file> <backend-pr-url>` — execute after verifying PR state and running codegen

---

## Step 1 — Parse arguments and read the plan

Split `$ARGUMENTS` into plan file path and optional PR URL (starts with `https://github.com/`).

Read the plan file in full. Extract and hold in memory:

- Branch name
- `Backend Required` field value
- Backend PR URL from the plan (if any — in the `Backend GraphQL Surface` section)
- Files to Read Before Implementing
- Files to Create / Modify
- Component Spec (props, behavior, layout)
- Failure Conditions (exact user-visible text)
- Test Scenarios
- Implementation Steps
- Checklist

If any section is too vague to implement without guessing, **stop** and tell the user which section needs more detail.

---

## Step 2 — Resolve backend status

Determine the PR URL to check. Priority:
1. PR URL from `$ARGUMENTS` (user-provided at runtime)
2. PR URL extracted from the plan's `Backend Required` field (e.g., `PR #123`)

### Case A — Backend Required: No

Skip to Step 3.

### Case B — Backend Required: Yes + PR URL available

```bash
gh pr view <PR-URL> --json number,title,state,mergedAt
```

**If `state` is `MERGED`:**

Output:
```
✅ Backend PR #[N] is merged. Running codegen to sync types...
```

Run codegen (Step 3a below).

**If `state` is `OPEN` or `CLOSED` without merge:**

Output:
```
⛔ Backend PR #[N] is not merged yet.

  PR: [title]
  State: [OPEN / CLOSED]
  URL: [url]

The frontend executor cannot run until this PR is merged and codegen is run.
Come back and re-run once it's merged:

  /boost-client-dev:execute [plan-file] [pr-url]
```

**Stop. Do not proceed.**

### Case C — Backend Required: Yes + NO PR URL available

Output:
```
⛔ This plan requires backend changes in boost-api.

Provide the backend PR URL to verify it's merged before executing:

  /boost-client-dev:execute [plan-file] https://github.com/boost-legal/boost-api/pull/N

Or, if you've already run codegen manually, confirm the required types
exist in src/graphql/types.ts:
```

```bash
grep -n "use.*Invoice.*Mutation\|use.*Invoice.*Query" src/graphql/types.ts
```

```
If the types are present, you can proceed without the PR URL — just re-run
without it and confirm when prompted.
```

**Stop. Do not proceed until resolved.**

---

## Step 3a — Run codegen (only when backend PR is confirmed merged)

```bash
npm run codegen
```

Verify the types from the plan's `Backend GraphQL Surface` section now exist:

```bash
grep -n "useCreateInvoiceMutation\|InvoiceStatus\|InvoiceFieldsFragment" src/graphql/types.ts
```

For each expected hook/type from the plan:
- If present: confirm with `✅ useCreateInvoiceMutation — found`
- If missing: **STOP** and report:

```
⛔ Codegen ran but expected types are missing from src/graphql/types.ts:
  - useCreateInvoiceMutation — NOT FOUND
  - InvoiceStatus — NOT FOUND

This means either:
  1. The backend PR introduced different names than the plan expected
  2. The .graphql operation files in boost-client need updating to match

Compare the plan's Backend GraphQL Surface section with the actual
merged PR diff:
  gh pr diff <PR-URL> -- "*.graphql" "*schema*"

Update the .graphql files to match the exact names in the backend,
then re-run codegen.
```

---

## Step 4 — Load context

Read:
1. `${CLAUDE_PLUGIN_ROOT}/context/architecture.md`
2. `${CLAUDE_PLUGIN_ROOT}/context/graphql.md`
3. `${CLAUDE_PLUGIN_ROOT}/context/testing.md`
4. `${CLAUDE_PLUGIN_ROOT}/context/styling.md`

---

## Step 5 — Read existing files

Read every file listed in **Files to Read Before Implementing**.
Read every file listed in **Files to Modify** (full content).

Also find a similar test file for pattern reference:

```bash
find src -name "*.test.tsx" -path "*<domain>*" | head -3
```

---

## Step 6 — Show execution plan

Output before touching any file:

```
### Execution Plan

Backend: [not required / PR #N verified merged — codegen ran]

Files to create:
  - [list]

Files to modify:
  - [list with exact change]

New .graphql client files: [list or "none"]

Test files to write first:
  - [list]

Test command:
  npm test -- [test file path] --watchAll=false

Lint command:
  npx biome check [file paths]
```

Proceed immediately.

---

## Step 7 — Create client-side .graphql files (if any)

If the plan lists new `.graphql` operation files to create (client-side mutations/queries/fragments), create them now using the exact names from the **Backend GraphQL Surface** section.

If codegen wasn't already run (Step 3a didn't run), run it now:

```bash
npm run codegen
```

Verify the generated hooks are present in `src/graphql/types.ts`.

---

## Step 8 — Write tests first (RED phase)

Write every test file from the **Test Scenarios** section.

Rules:
- `MockedProvider` for Apollo — mock shapes must match the Backend GraphQL Surface types exactly
- `userEvent.setup()` for interactions — never `fireEvent`
- `findBy*` for async — never `waitFor` + `getBy*`
- Test user-visible behavior only — not implementation details
- Exact text in assertions — use strings from Failure Conditions verbatim
- Every data-fetching component must cover: loading, loaded, empty, error

Run:
```bash
npm test -- <test-file-path> --watchAll=false
```

**Expected: ALL FAIL.** If any pass before implementation, flag it.

---

## Step 9 — Implement

Follow **Implementation Steps** in order. For each:
1. Create/edit the specified files
2. Apply ALL behavior from Component Spec
3. Use EXACT user-visible text from Failure Conditions for error/empty states
4. Use hook names exactly as they appear in `src/graphql/types.ts`
5. Tachyons + boost-* classes only — never inline styles, never Tailwind
6. Icon names only from `icons.global.less`
7. Only existing component library (BoostButton, LMIcon, etc.)
8. Run tests after each step

```bash
npm test -- <test-file-path> --watchAll=false
```

**Failure handling:** max 3 attempts per failing test, then **STOP** and report:
- Exact failing test
- Full error output
- What you tried
- What you believe is wrong
- What the human needs to provide to continue

---

## Step 10 — Run domain-wide tests

```bash
npm test -- --testPathPattern="<domain>" --watchAll=false
```

Fix any regressions (same 3-attempt rule).

---

## Step 11 — Lint

```bash
npx biome check --write src/<paths>
npx eslint src/<paths> --fix
```

Verify clean:
```bash
npx biome check src/<paths>
```

---

## Step 12 — TypeScript check

```bash
npx tsc --noEmit --project tsconfig.json 2>&1 | grep -E "error TS" | head -20
```

Fix all errors. Never use `as any`.

---

## Step 13 — Final report

```
### Execution Report

Branch: [branch name]

Backend: [not required / PR #N merged — codegen ran — N new types]

Files created:
  - path/to/component/index.tsx
  - path/to/component/__tests__/component.test.tsx
  - src/graphql/mutations/<domain>/create_invoice.graphql

Files modified:
  - path/to/parent/index.tsx (added route for new layer)

Codegen: [ran — N new hooks generated / not needed]
Tests: X passing, 0 failing
Lint (Biome): clean
TypeScript: no errors

Manual review recommended:
  - [styling decisions needing visual verification]
  - [any deviation from the plan and why]
  - [edge cases not covered by tests]

To run tests locally:
  npm test -- <paths> --watchAll=false

To verify lint:
  npx biome check src/<paths>
```

---

## Hard rules

- **Never commit.**
- **Never proceed with Backend Required: Yes** until PR is confirmed merged (or types manually verified).
- **Never use inline styles** for Tachyons-equivalent properties.
- **Never use Tailwind classes.**
- **Never invent icon names.**
- **Never create custom Button/Icon/Input** — use BoostButton, LMIcon, BoostInput.
- **Never use `as any`.**
- **Never skip a test scenario.**
- **Stop at 3 failures** on the same test — don't loop.
