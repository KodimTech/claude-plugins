# boost-dev

Plugin de Claude Code para el equipo de **boost-api**. Actúa como un **Senior Rails Developer**: toma una historia de Shortcut, analiza los requisitos y genera un plan de implementación detallado en formato `.md`, siguiendo los patrones y convenciones del proyecto.

---

## Instalación

### 1 — Agregar el marketplace a `~/.claude/settings.json`

```json
"extraKnownMarketplaces": {
  "kodim": {
    "source": {
      "source": "github",
      "repo": "KodimTech/claude-plugins"
    }
  }
}
```

### 2 — Instalar el plugin

```bash
claude plugin install boost-dev@kodim
```

---

## Uso

```
/boost-dev:plan https://app.shortcut.com/boost/story/46134/nombre-historia
```

O con el ID directamente:

```
/boost-dev:plan 46134
```

El plugin:
1. Obtiene los detalles de la historia desde Shortcut
2. Lee el contexto del proyecto (architecture, graphql, testing, database)
3. Explora el codebase real para anclar el plan en código existente
4. Muestra un análisis visible antes de escribir
5. Genera `plan-sc-<ID>-<slug>.md` en tu directorio actual

---

## Requisitos

- Claude Code instalado
- MCP de Shortcut configurado (`Lawmatics-Shortcut`)

---

## Estructura

```
boost-dev/
├── .claude-plugin/
│   └── plugin.json
├── agents/
│   └── senior-rails-dev.md   # Persona + reglas del Senior Dev
├── context/
│   ├── architecture.md        # Interactors, services, concerns
│   ├── graphql.md             # Mutations, types, CI schema check
│   ├── testing.md             # RSpec + FactoryBot
│   └── database.md            # Migrations, strong_migrations, CI checks
├── skills/
│   └── plan/
│       └── SKILL.md           # Flujo del planner
└── settings.json              # Activa el agente senior-rails-dev
```

---

## Actualizar

```bash
claude plugin update boost-dev@kodim
```
