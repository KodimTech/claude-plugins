---
description: "Execute a boost-dev implementation plan (.md file): write specs, implement, run tests, fix rubocop. Human reviews the output before creating a PR."
---

# Implementation Executor

You are executing a pre-approved implementation plan. Your job is to write correct, tested, rubocop-clean code. You do not commit. A human reviews everything you produce.

**Input:** `$ARGUMENTS` — path to the plan `.md` file (e.g., `plan-sc-46134-add-invoice.md`)

---

## Step 1 — Read the plan

Read the file at `$ARGUMENTS` in full. Extract and hold in memory:

- Branch name
- Files to Read Before Implementing
- Files to Create
- Files to Modify
- Database Migration (if any)
- Business Rules (all of them)
- Failure Conditions (condition → exact error message)
- Side Effects (jobs, emails, activity)
- Spec Skeleton (exact specs to write)
- Implementation Steps (ordered)

If any of these sections is missing or too vague to implement without guessing, **stop immediately** and tell the user which section needs more detail before you can proceed.

---

## Step 2 — Load context

Read:
1. `${CLAUDE_PLUGIN_ROOT}/context/architecture.md`
2. `${CLAUDE_PLUGIN_ROOT}/context/graphql.md`
3. `${CLAUDE_PLUGIN_ROOT}/context/testing.md`
4. `${CLAUDE_PLUGIN_ROOT}/context/database.md`
5. `${CLAUDE_PLUGIN_ROOT}/context/rubocop.md`

---

## Step 3 — Read existing files

Read every file listed in **Files to Read Before Implementing**.
Read every file listed in **Files to Modify** (full content — you will edit these).

Do not skip this step. Modifying a file without reading it first causes broken code.

---

## Step 4 — Show execution plan

Output this before touching any file:

```
### Execution Plan

Files to create:
  - [list]

Files to modify:
  - [list with exact change]

Migration: [yes/no]

Spec files to write first:
  - [list]

Test command:
  bundle exec rspec [paths] --format documentation --no-color

Rubocop command:
  bundle exec rubocop [paths] --format simple
```

Proceed immediately — no waiting for approval.

---

## Step 5 — Run migration (if needed)

If the plan includes a migration, create and run it first:

```bash
bundle exec rails generate migration MigrationName
# Edit the generated file with the exact columns from the plan
bundle exec rails db:migrate RAILS_ENV=test
bundle exec annotaterb annotate_models
```

Verify the migration ran cleanly. If it fails, fix before proceeding.

---

## Step 6 — Write specs first (RED phase)

Write every spec file from the **Spec Skeleton** section of the plan.

Rules:
- Use `let_it_be` for data shared across all examples (`SpecContext.firm`, users)
- Use `let!` for eager data needed within a context block
- Use `let` for lazy data
- Use exact values in assertions — not `be_present` where the value is knowable
- Match error messages EXACTLY as specified in Failure Conditions
- Test side effects with `have_enqueued_job(JobClass)` and `have_enqueued_mail`

After writing specs, run them:

```bash
bundle exec rspec spec/path/to/spec_file.rb --format documentation --no-color
```

**Expected result: ALL FAIL.** If any pass before implementation, flag it — either the spec is wrong or something already exists.

---

## Step 7 — Implement

Follow the **Implementation Steps** from the plan in order.

For each step:
1. Create/edit the files specified
2. Apply ALL business rules from the plan — every single one
3. Use the EXACT error messages from Failure Conditions for every `context.fail!`
4. Implement ALL side effects listed — do not skip jobs, emails, or activity tracking
5. Run specs after each step

```bash
bundle exec rspec spec/path/to/spec_file.rb --format documentation --no-color
```

**Failure handling:**
- Specs fail → read the full error output → attempt fix → re-run (max 3 attempts)
- After 3 attempts on the same failure: **STOP**. Report:
  - The exact failing spec
  - The error output
  - What you tried
  - What you believe is wrong
  - What you need from the human to continue

Never loop blindly. Three attempts and stop.

---

## Step 8 — Run full spec suite for touched files

Once all individual specs pass, run:

```bash
bundle exec rspec [all spec files created or related to this feature] --format documentation --no-color
```

If new failures appear that weren't there before, fix them. Same 3-attempt rule applies.

---

## Step 9 — Rubocop

Run rubocop on every file you created or modified:

```bash
bundle exec rubocop [app files] [spec files] --format simple
```

1. First attempt: `bundle exec rubocop -a [files]` (safe autocorrect)
2. If offenses remain: fix manually
3. Re-run to verify clean
4. If a cop cannot be fixed: add inline disable with a comment explaining why

Do not modify `.rubocop_todo.yml`.

---

## Step 10 — GraphQL schema dump (if GraphQL changed)

If any GraphQL file was created or modified:

```bash
bundle exec rake graphql:dump
```

Verify the command succeeds. The updated schema file must be part of the changeset.

---

## Step 11 — Final report

Output a summary for the human reviewer:

```
### Execution Report

Branch: [branch name from plan]

Files created:
  - path/to/file.rb

Files modified:
  - path/to/file.rb (what changed)

Specs: X passing, 0 failing
Rubocop: clean
Migration: [ran / not needed]
GraphQL schema: [updated / not needed]

Manual review recommended:
  - [anything the executor is not 100% confident about]
  - [edge cases not covered by specs]
  - [any deviation from the plan and why]

To run specs locally:
  bundle exec rspec [paths] --format documentation

To verify rubocop:
  bundle exec rubocop [paths]
```

---

## Hard rules

- **Never commit.** Human creates the PR.
- **Never modify `.rubocop_todo.yml`.**
- **Never use `safety_assured` unless the plan explicitly says to and explains why.**
- **Never skip a Business Rule** — if it's in the plan, it must be in the code AND in a spec.
- **Never assume a side effect** — only implement what's in the Side Effects section.
- **Stop at 3 spec failures** on the same error — ask the human, don't loop.
