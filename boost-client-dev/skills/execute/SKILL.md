---
description: "Execute a boost-client implementation plan (.md file): write component tests (RED), implement, lint. Human reviews the output before creating a PR."
---

# Frontend Implementation Executor

You are executing a pre-approved implementation plan. Your job is to write correct, tested, lint-clean React/TypeScript code. You do not commit. A human reviews everything you produce.

**Input:** `$ARGUMENTS` — path to the plan `.md` file (e.g., `plan-sc-46134-add-invoices.md`)

---

## Step 1 — Read the plan

Read the file at `$ARGUMENTS` in full. Extract and hold in memory:

- Branch name
- Backend Required (yes/no)
- Files to Read Before Implementing
- Files to Create
- Files to Modify
- Component Spec (props, behavior, layout)
- Failure Conditions (condition → what the user sees)
- Test Scenarios (exact test descriptions)
- Implementation Steps (ordered)

If any of these sections is missing or too vague to implement without guessing, **stop immediately** and tell the user which section needs more detail before proceeding.

If `Backend Required: Yes`, **stop immediately** and output:

```
⛔ This plan requires backend changes in boost-api that must be merged first.

Verify:
1. The backend PR is merged
2. npm run codegen has been run
3. The required types appear in src/graphql/types.ts:
   - [list types/mutations from the Backend Changes Required section]

Once verified, re-run this command.
```

---

## Step 2 — Load context

Read:
1. `${CLAUDE_PLUGIN_ROOT}/context/architecture.md`
2. `${CLAUDE_PLUGIN_ROOT}/context/graphql.md`
3. `${CLAUDE_PLUGIN_ROOT}/context/testing.md`
4. `${CLAUDE_PLUGIN_ROOT}/context/styling.md`

---

## Step 3 — Read existing files

Read every file listed in **Files to Read Before Implementing**.
Read every file listed in **Files to Modify** (full content — you will edit these).

Also grep for similar test files to understand the project's test patterns:
```bash
# Find a similar test file for reference
find src -name "*.test.tsx" -path "*<domain>*" | head -3
```

Do not skip this step.

---

## Step 4 — Show execution plan

Output this before touching any file:

```
### Execution Plan

Files to create:
  - [list]

Files to modify:
  - [list with exact change]

New GraphQL files: [yes/no — list .graphql files to create]
Codegen needed: [yes/no]

Test files to write first:
  - [list]

Test command:
  npm test -- [test file path] --watchAll=false

Lint command:
  npx biome check [file paths]
  # or: npx eslint [file paths]
```

Proceed immediately — no waiting for approval.

---

## Step 5 — Create GraphQL files (if needed)

If the plan includes new `.graphql` operation files, create them first:

```bash
# Create the mutation/query file
# Example: src/graphql/mutations/invoices/create_invoice.graphql
```

Then run codegen:
```bash
npm run codegen
```

Verify `src/graphql/types.ts` now contains the generated hook (e.g., `useCreateInvoiceMutation`). If codegen fails, fix the GraphQL syntax before proceeding.

---

## Step 6 — Write tests first (RED phase)

Write every test file from the **Test Scenarios** section of the plan.

Rules:
- Use `MockedProvider` for Apollo — never mock individual hooks
- Use `userEvent.setup()` for interactions — not `fireEvent`
- Use `findBy*` queries for async content — not `waitFor` + `getBy*`
- Test user-visible behavior — not implementation details
- Exact text assertions: use the exact strings from Failure Conditions in the plan
- Every data-fetching component must test: loading state, loaded state, empty state, error state

After writing tests, run them:

```bash
npm test -- <test-file-path> --watchAll=false
```

**Expected result: ALL FAIL (or "cannot find module" for files not yet created).** If any pass before implementation, flag it — either the test is wrong or the feature already exists.

---

## Step 7 — Implement

Follow the **Implementation Steps** from the plan in order.

For each step:
1. Create/edit the files specified
2. Apply ALL behavior described in the Component Spec
3. Use the EXACT user-visible messages from Failure Conditions for every error state
4. Apply ALL styling with Tachyons + boost-* classes — never inline styles
5. Use ONLY icon names from the styling context — verify before using
6. Use ONLY existing component library components (BoostButton, LMIcon, etc.)
7. Run tests after each step

```bash
npm test -- <test-file-path> --watchAll=false
```

**Failure handling:**
- Tests fail → read the full error output → attempt fix → re-run (max 3 attempts)
- After 3 attempts on the same failure: **STOP**. Report:
  - The exact failing test
  - The error output
  - What you tried
  - What you believe is wrong
  - What you need from the human to continue

Never loop blindly. Three attempts and stop.

---

## Step 8 — Run full test suite for touched files

Once all individual tests pass:

```bash
npm test -- --testPathPattern="<domain>" --watchAll=false
```

If new failures appear that weren't there before, fix them. Same 3-attempt rule applies.

---

## Step 9 — Lint

Run lint on every file you created or modified:

```bash
npx biome check --write src/<paths>
# If Biome doesn't cover it:
npx eslint src/<paths> --fix
```

Then verify no remaining errors:
```bash
npx biome check src/<paths>
npx eslint src/<paths>
```

If a lint rule cannot be auto-fixed:
- Fix manually if the fix is clear
- Add an inline disable comment with an explanation if it cannot be fixed without breaking behavior
- Never disable rules wholesale

---

## Step 10 — TypeScript check

Run TypeScript on the modified files to catch type errors not caught by tests:

```bash
npx tsc --noEmit --project tsconfig.json 2>&1 | grep -E "error TS" | head -20
```

Fix all type errors before proceeding. Never use `as any` to suppress errors — fix the underlying type issue.

---

## Step 11 — Final report

Output a summary for the human reviewer:

```
### Execution Report

Branch: [branch name from plan]

Files created:
  - path/to/component/index.tsx
  - path/to/component/__tests__/component.test.tsx

Files modified:
  - path/to/parent/index.tsx (what changed)

New GraphQL files:
  - src/graphql/mutations/invoices/create_invoice.graphql

Codegen: [run / not needed]
Tests: X passing, 0 failing
Lint (Biome): clean
TypeScript: no errors

Manual review recommended:
  - [anything the executor is not 100% confident about]
  - [edge cases not covered by tests]
  - [any deviation from the plan and why]
  - [styling decisions that need visual verification]

To run tests locally:
  npm test -- <paths> --watchAll=false

To verify lint:
  npx biome check src/<paths>
```

---

## Hard rules

- **Never commit.** Human creates the PR.
- **Never use inline styles** for properties that have Tachyons equivalents.
- **Never use Tailwind classes.** This is a Tachyons project.
- **Never invent icon names** — only use names verified in `icons.global.less`.
- **Never create a custom Button, Icon, or Input component** — use BoostButton, LMIcon, BoostInput.
- **Never use `as any`** — fix the underlying type issue.
- **Never skip a test scenario** from the Test Scenarios section.
- **Stop at 3 test failures** on the same error — ask the human, don't loop.
- **Never proceed if Backend Required: Yes** — backend must be merged first.
