---
description: "Full-stack coordinator: discovers both repos, defines the GraphQL contract, then delegates to the existing boost-dev:plan and boost-client-dev:plan skills in parallel. No planning logic is duplicated here."
---

# Full-Stack Plan Coordinator

You are the tech lead. Your job is coordination — not planning.
The actual planning logic lives in `boost-dev` and `boost-client-dev`.
This skill finds those skills at runtime, adds the cross-repo contract layer,
and launches them in parallel.

**Input:** `$ARGUMENTS` — Shortcut story URL or ID

---

## Step 1 — Verify dependencies

Find the installed plan skills:

```bash
BACKEND_SKILL=$(find ~/.claude -path "*/boost-dev/skills/plan/SKILL.md" 2>/dev/null | head -1)
FRONTEND_SKILL=$(find ~/.claude -path "*/boost-client-dev/skills/plan/SKILL.md" 2>/dev/null | head -1)

echo "backend skill: $BACKEND_SKILL"
echo "frontend skill: $FRONTEND_SKILL"
```

If either is missing:
```
⛔ Missing required plugins:
  [boost-dev not found]      → claude plugin install boost-dev@kodim
  [boost-client-dev not found] → claude plugin install boost-client-dev@kodim
```

Read both skill files in full — their instructions will be passed to the subagents.

---

## Step 2 — Discover repos

```bash
PARENT=$(dirname $(pwd))
BACKEND_PATH="$PARENT/boost-api"
FRONTEND_PATH="$PARENT/boost-client"

ls "$BACKEND_PATH" > /dev/null 2>&1 && echo "backend: OK" || echo "backend: MISSING"
ls "$FRONTEND_PATH" > /dev/null 2>&1 && echo "frontend: OK" || echo "frontend: MISSING"
```

If either repo is missing, stop and tell the user.

---

## Step 3 — Fetch the story and determine scope

- If URL: extract numeric ID from segment after `/story/`
- Call `stories-get-by-id` with `story_public_id: <ID>`
- Read: `name`, `description`, `story_type`, `labels`, `tasks`, `comments`

Determine which layers are affected:
- **backend-only** → only `boost-dev:plan`
- **frontend-only** → only `boost-client-dev:plan`
- **full-stack** → both in parallel

---

## Step 4 — Define the GraphQL contract (full-stack only)

This is the only unique contribution `boost-team` makes.
Before the agents start, establish the names that both sides will use.

Explore boost-api to anchor naming conventions:

```bash
# Find existing mutations in the same domain for naming reference
grep -r "<feature-keyword>" "$BACKEND_PATH/app/graphql/" --include="*.rb" -l 2>/dev/null | head -5
```

Read 1 existing mutation file to match the naming style.

Then define:

```
### GraphQL Contract

Mutations:
  <camelCase name>(<argName>: <Type>!): <ReturnType>!

Types:
  type <Entity> {
    id: ID!
    <field>: <Type>!
    createdAt: ISO8601DateTime!
  }

Enums (if any):
  enum <Entity>Status { ... }
```

This contract will be injected into both agent prompts so they agree on names
before writing a single line of code.

---

## Step 5 — Launch agents in parallel

Spawn both agents simultaneously (two Agent tool calls in a single message).

### Backend agent prompt

```
You are boost-dev:senior-rails-dev.

Your target repository is: $BACKEND_PATH
Use absolute paths for ALL file operations (Read, Glob, Grep, Bash).

Story to plan:
  ID: [story ID]
  Title: [title]
  Description: [full description]

GraphQL contract agreed with the frontend team:
[paste contract from Step 4]

Instructions: Follow the plan skill below exactly, using the story and contract above
as your input. Save the plan to: $BACKEND_PATH/plan-sc-[ID]-[slug].md

--- BEGIN PLAN SKILL ---
[full content of $BACKEND_SKILL]
--- END PLAN SKILL ---

Return when done:
  - Plan file absolute path
  - Final GraphQL surface (may differ from contract — flag deviations)
  - Key risks
```

### Frontend agent prompt

```
You are boost-client-dev:senior-react-dev.

Your target repository is: $FRONTEND_PATH
Use absolute paths for ALL file operations (Read, Glob, Grep, Bash).

Story to plan:
  ID: [story ID]
  Title: [title]
  Description: [full description]

GraphQL contract (to be implemented by backend — PR not available yet):
[paste contract from Step 4]

The backend PR does not exist yet. Mark the plan: Backend Required: Yes — no PR yet.
The executor will require the backend PR URL before running.

Instructions: Follow the plan skill below exactly, using the story and contract above
as your input. Save the plan to: $FRONTEND_PATH/plan-sc-[ID]-[slug].md

--- BEGIN PLAN SKILL ---
[full content of $FRONTEND_SKILL]
--- END PLAN SKILL ---

Return when done:
  - Plan file absolute path
  - GraphQL operations expected from backend
  - Any contract questions (field names, error messages, etc.)
  - Key risks
```

---

## Step 6 — Resolve contract conflicts

After both agents complete, compare:
- Did the backend agent use different names than the contract?
- Did the frontend agent expect fields the backend didn't include?

If there are mismatches, update the affected plan file(s) to align them.
Document any resolved conflicts in the final summary.

---

## Step 7 — Output coordination summary

```
### Full-Stack Plan — SC-[ID]: [Title]

─────────────────────────────────────────────
BACKEND  →  $BACKEND_PATH/plan-sc-[ID]-[slug].md
─────────────────────────────────────────────
Branch: [branch]   Complexity: [Low/Med/High]
GraphQL surface: [list of mutations/types]
Risks: [from backend agent]

─────────────────────────────────────────────
FRONTEND  →  $FRONTEND_PATH/plan-sc-[ID]-[slug].md
─────────────────────────────────────────────
Branch: [branch]   Complexity: [Low/Med/High]
Consumes: [list of hooks/operations]
Risks: [from frontend agent]

─────────────────────────────────────────────
CONTRACT  (agreed — [N deviations resolved / clean])
─────────────────────────────────────────────
[final mutations/types both sides aligned on]

─────────────────────────────────────────────
EXECUTION ORDER
─────────────────────────────────────────────
1. cd $BACKEND_PATH
   /boost-dev:execute plan-sc-[ID]-[slug].md
   → tests green → rubocop clean → open PR

2. cd $FRONTEND_PATH
   /boost-client-dev:execute plan-sc-[ID]-[slug].md <backend-pr-url>
   → verifies PR merged → codegen → tests → lint

─────────────────────────────────────────────
OPEN QUESTIONS
─────────────────────────────────────────────
[Blockers neither agent could resolve — need human input]
```

---

## Scope shortcuts

**Backend-only:** Launch only the backend agent (Step 5), skip contract and frontend sections.

**Frontend-only:** Launch only the frontend agent (Step 5), skip contract. Pass `Backend Required: No` in the prompt.
