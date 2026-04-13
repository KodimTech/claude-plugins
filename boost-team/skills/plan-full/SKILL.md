---
description: "Full-stack planner: fetches a Shortcut story, discovers both repos, defines the GraphQL contract, and launches backend and frontend planners in parallel — each generating its own plan file."
---

# Full-Stack Implementation Planner

You are the tech lead coordinating backend and frontend planning for a single story.
Two specialized agents will work in parallel. Your job is to set up the contract between
them before they start, collect their output, and produce a clear execution order.

**Input:** `$ARGUMENTS` — Shortcut story URL or ID

---

## Step 1 — Fetch the story

- If URL: extract numeric ID from segment after `/story/`
- If number: use directly
- Call `stories-get-by-id` with `story_public_id: <ID>`
- Read: `name`, `description`, `story_type`, `labels`, `tasks`, `comments`

---

## Step 2 — Discover repos

```bash
PARENT=$(dirname $(pwd))
BACKEND_PATH="$PARENT/boost-api"
FRONTEND_PATH="$PARENT/boost-client"
ls "$BACKEND_PATH" > /dev/null 2>&1 && echo "backend: OK" || echo "backend: MISSING"
ls "$FRONTEND_PATH" > /dev/null 2>&1 && echo "frontend: OK" || echo "frontend: MISSING"
```

If either repo is missing, stop:
```
⛔ Cannot find [boost-api / boost-client] at $PARENT.

Expected layout:
  <parent>/
  ├── boost-api/
  └── boost-client/

Run this skill from either repo directory.
```

---

## Step 3 — Determine story scope

Based on the story, classify what layers are touched:

| Layer | Backend (boost-api) | Frontend (boost-client) |
|-------|--------------------|-----------------------|
| Data model / DB | ✓ | — |
| Business logic (interactors) | ✓ | — |
| GraphQL mutations / queries | ✓ | ✓ (codegen consumer) |
| UI components / hooks | — | ✓ |
| Redux state | — | ✓ |

Determine: **backend-only / frontend-only / full-stack**

- If **backend-only**: skip to Step 4a (backend agent only)
- If **frontend-only**: skip to Step 4b (frontend agent only)
- If **full-stack**: proceed to Step 4 (both agents)

---

## Step 4 — Define the GraphQL contract (full-stack only)

Before launching the agents, define the interface between them.
This prevents the backend and frontend from independently inventing incompatible names.

Explore the existing schema in boost-api to anchor the contract:

```bash
# Find existing related mutations/types
grep -r "<feature-keyword>" "$BACKEND_PATH/app/graphql/" --include="*.rb" -l
grep -r "<feature-keyword>" "$BACKEND_PATH/app/graphql/" --include="*.graphql" -l 2>/dev/null
```

Read 1–2 existing mutation files to understand naming conventions.

Then define the proposed contract:

```
### Proposed GraphQL Contract

Mutations:
  create<Entity>(arg1: Type!, arg2: Type): EntityType!
  update<Entity>(id: ID!, arg1: Type): EntityType
  delete<Entity>(id: ID!): EntityType

Types:
  type <Entity> {
    id: ID!
    field1: Type!
    field2: Type
    createdAt: ISO8601DateTime!
  }

Enums (if any):
  enum <Entity>Status { pending active archived }
```

This contract will be passed to both agents so they align on the same names.
Agents may propose adjustments — those must be flagged in the final summary.

---

## Step 5 — Launch agents in parallel

Spawn both agents simultaneously using the Agent tool with two calls in a single message.

### Backend Agent prompt

```
You are boost-dev:senior-rails-dev, a Senior Rails Developer working on boost-api.

Your target repository is: $BACKEND_PATH
Use absolute paths for ALL file operations.

Story: [story title]
Story description: [full description]
SC story ID: [ID]
Git username (for branch name): [git config user.name or ask]

Proposed GraphQL contract to implement:
[paste the contract from Step 4]

Your task:
1. Read $BACKEND_PATH/${CLAUDE_PLUGIN_ROOT relative to boost-api}/context/architecture.md
   Actually read these context files using absolute paths:
   - Look for context files in the boost-dev plugin: find ~/.claude -name "architecture.md" -path "*boost-dev*"
2. Explore the boost-api codebase at $BACKEND_PATH using absolute paths
3. Generate a complete implementation plan following the boost-dev patterns:
   - Interactors, GraphQL mutations, RSpec specs, migrations
   - Use the proposed GraphQL contract above — flag any deviations
4. Save the plan as: $BACKEND_PATH/plan-sc-[ID]-[slug].md
5. Return:
   - Plan file path
   - Final GraphQL surface (mutations/types as you plan to implement them — may differ from proposal)
   - Any deviations from the proposed contract and why
   - Complexity estimate and risk flags
```

### Frontend Agent prompt

```
You are boost-client-dev:senior-react-dev, a Senior React/TypeScript Developer working on boost-client.

Your target repository is: $FRONTEND_PATH
Use absolute paths for ALL file operations.

Story: [story title]
Story description: [full description]
SC story ID: [ID]
Git username (for branch name): [git config user.name or ask]

Expected GraphQL contract (to be implemented by backend — backend PR not available yet):
[paste the contract from Step 4]

IMPORTANT: The backend PR does not exist yet. The frontend plan will be based on the
proposed contract. The executor MUST verify against the real backend PR before running.
Mark plan as: Backend Required: Yes — no PR yet

Your task:
1. Read context files from the boost-client-dev plugin (find them with: find ~/.claude -name "architecture.md" -path "*boost-client-dev*")
2. Explore the boost-client codebase at $FRONTEND_PATH using absolute paths
3. Generate a complete frontend plan:
   - React components, Apollo hooks, tests (Jest + Testing Library), Tachyons styling
   - Reference the proposed contract for hook names and types
   - Note clearly: "types are proposed — confirm against backend PR before executing"
4. Save the plan as: $FRONTEND_PATH/plan-sc-[ID]-[slug].md
5. Return:
   - Plan file path
   - Which GraphQL operations it expects to consume
   - Any frontend-side questions about the contract (field names, error messages, etc.)
   - Complexity estimate
```

---

## Step 6 — Collect agent results and resolve conflicts

After both agents complete, read their outputs.

Check for contract mismatches:
- Did the backend agent use different mutation names than proposed?
- Did the frontend agent expect fields that the backend didn't include?
- Any incompatible argument names?

If there are mismatches, update the relevant plan file(s) to align them.

---

## Step 7 — Output coordination summary

```
### Full-Stack Plan Summary

Story: [SC-ID] — [Title]

─────────────────────────────────────────────
BACKEND PLAN
─────────────────────────────────────────────
File: $BACKEND_PATH/plan-sc-[ID]-[slug].md
Branch: [backend branch name]
Complexity: [Low / Medium / High]

GraphQL surface (final):
  [list of mutations and types the backend will implement]

Key risks:
  - [anything flagged by backend agent]

─────────────────────────────────────────────
FRONTEND PLAN
─────────────────────────────────────────────
File: $FRONTEND_PATH/plan-sc-[ID]-[slug].md
Branch: [frontend branch name]
Complexity: [Low / Medium / High]

Consumes:
  [list of mutations/hooks the frontend expects]

Key risks:
  - [anything flagged by frontend agent]

─────────────────────────────────────────────
CONTRACT
─────────────────────────────────────────────
[Final agreed GraphQL mutations/types — both agents aligned]

Deviations from proposal:
  [any changes the agents made and why — "none" if clean]

─────────────────────────────────────────────
EXECUTION ORDER
─────────────────────────────────────────────

1. Backend:
   cd $BACKEND_PATH
   git checkout -b [backend branch]
   /boost-dev:execute plan-sc-[ID]-[slug].md
   → open PR when done

2. Frontend (after backend PR is merged):
   cd $FRONTEND_PATH
   git checkout -b [frontend branch]
   /boost-client-dev:execute plan-sc-[ID]-[slug].md <backend-pr-url>
   → executor will verify PR state, run codegen, implement

─────────────────────────────────────────────
OPEN QUESTIONS
─────────────────────────────────────────────
[Anything neither agent could resolve — needs human input before executing]
```

---

## Step 4a — Backend-only story

If the story only touches the backend, launch only the backend agent (Step 5 backend prompt, without contract section) and skip Step 4/frontend agent/contract.

---

## Step 4b — Frontend-only story

If the story only touches the frontend, launch only the frontend agent (Step 5 frontend prompt) without the contract concern. Mark `Backend Required: No` in the prompt.
