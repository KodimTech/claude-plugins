# KodimTech Claude Plugins

Plugins de Claude Code para el equipo de Boost. Cada plugin actúa como un agente especializado que lee historias de Shortcut, explora el codebase real, y genera planes de implementación detallados listos para ejecutar en modo TDD.

---

## Plugins disponibles

| Plugin | Repo | Comando |
|--------|------|---------|
| `boost-dev` | boost-api (Rails) | `/boost-dev:plan` · `/boost-dev:execute` |
| `boost-client-dev` | boost-client (React/TS) | `/boost-client-dev:plan` · `/boost-client-dev:execute` |
| `boost-team` | ambos | `/boost-team:plan-full` |

---

## Instalación

### Paso 1 — Registrar el marketplace (una sola vez)

Dentro de Claude Code, ejecuta:

```
/plugin marketplace add KodimTech/claude-plugins
```

### Paso 2 — Instalar los plugins

```
claude plugin install boost-dev@kodim
claude plugin install boost-client-dev@kodim
claude plugin install boost-team@kodim
```

### Paso 3 — Habilitar los plugins

```
claude plugin enable boost-dev@kodim
claude plugin enable boost-client-dev@kodim
claude plugin enable boost-team@kodim
```

### Paso 4 — Recargar

```
/reload-plugins
```

---

## Requisitos

- [Claude Code](https://claude.ai/code) instalado
- MCP de Shortcut configurado (`stories-get-by-id` disponible)
- Para `boost-team`: ambos repos como hermanos en el mismo directorio padre:
  ```
  <parent>/
  ├── boost-api/
  └── boost-client/
  ```

---

## Uso rápido

### Historia solo de backend (en boost-api)

```
/boost-dev:plan 46134
/boost-dev:execute plan-sc-46134-<slug>.md
```

### Historia solo de frontend (en boost-client)

```
/boost-client-dev:plan 46134 https://github.com/boost-legal/boost-api/pull/N
/boost-client-dev:execute plan-sc-46134-<slug>.md <pr-url>
```

### Historia full-stack (desde cualquier repo)

```
/boost-team:plan-full 46134
```

Genera ambos planes en paralelo y devuelve el orden de ejecución:

```
1. cd boost-api  →  /boost-dev:execute plan-sc-46134-<slug>.md  →  abrir PR
2. cd boost-client  →  /boost-client-dev:execute plan-sc-46134-<slug>.md <pr-url>
```

---

## Actualizar

```
claude plugin update boost-dev@kodim --scope local
claude plugin update boost-client-dev@kodim --scope local
claude plugin update boost-team@kodim --scope local
```

Si da error de scope, reinstalar:

```
claude plugin uninstall <plugin>@kodim
claude plugin install <plugin>@kodim
```

---

## Estructura del repositorio

```
KodimTech/claude-plugins
├── .claude-plugin/
│   └── marketplace.json        # Registro del marketplace
├── boost-dev/                  # Plugin para boost-api (Rails)
│   ├── agents/senior-rails-dev.md
│   ├── context/                # architecture, graphql, testing, database, rubocop
│   ├── skills/plan/            # Genera plan desde historia de Shortcut
│   └── skills/execute/         # Ejecuta plan en modo TDD
├── boost-client-dev/           # Plugin para boost-client (React/TS)
│   ├── agents/senior-react-dev.md
│   ├── context/                # architecture, graphql, testing, styling
│   ├── skills/plan/            # Genera plan (requiere PR de backend si aplica)
│   └── skills/execute/         # Ejecuta plan, verifica PR, corre codegen
└── boost-team/                 # Plugin coordinador full-stack
    ├── agents/tech-lead.md
    ├── context/repos.md        # Cómo descubrir y acceder a ambos repos
    └── skills/plan-full/       # Lanza boost-dev:plan y boost-client-dev:plan en paralelo
```
