# Repo Layout

Both repos live as siblings in the same parent directory:

```
<parent>/
├── boost-api/      # Rails backend — GraphQL API, PostgreSQL, Sidekiq
└── boost-client/   # React frontend — Apollo Client, Redux, Tachyons CSS
```

## Discovering paths at runtime

```bash
PARENT=$(dirname $(pwd))
BACKEND_PATH="$PARENT/boost-api"
FRONTEND_PATH="$PARENT/boost-client"

# Verify both exist
ls "$BACKEND_PATH" && ls "$FRONTEND_PATH"
```

## Accessing a sibling repo from a subagent

Subagents run in the current working directory but can access any absolute path.
When prompting a subagent to work on a specific repo, always pass the absolute path
and instruct it to use absolute paths for all file operations:

```
Your target repository is: /absolute/path/to/boost-api
Use absolute paths for ALL file operations (Read, Glob, Grep, Bash cd).
```

## Plan file locations

Each repo owns its own plan files:
- Backend plans: `$BACKEND_PATH/plan-sc-<ID>-<slug>.md`
- Frontend plans: `$FRONTEND_PATH/plan-sc-<ID>-<slug>.md`

## GraphQL interface boundary

The contract between repos is defined by:
- `$BACKEND_PATH/app/graphql/` — mutations, types, queries (source of truth)
- `$BACKEND_PATH/schema.graphql` (or wherever `rake graphql:dump` writes) — generated schema
- `$FRONTEND_PATH/src/graphql/types.ts` — generated TypeScript types (derived from schema via codegen)

The GraphQL schema is the **handshake**. Backend implements it; frontend consumes it.
