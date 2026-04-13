# boost-dev

Plugin de Claude Code para el equipo de **boost-api**. Actúa como un **Senior Rails Developer**: toma una historia de Shortcut, genera un plan de implementación detallado en `.md`, y luego puede ejecutarlo escribiendo specs, código y rubocop-clean en modo TDD.

**Flujo:**
```
/boost-dev:plan <shortcut-url>      → genera plan-sc-XXXXX.md
/boost-dev:execute plan-sc-XXXXX.md → escribe specs (RED) → implementa → rubocop → reporte
```

El humano revisa el output antes de crear el PR. El plugin nunca hace commit.

---

## Requisitos

- [Claude Code](https://claude.ai/code) instalado
- MCP de Shortcut configurado (el plugin lo usa para leer historias)

---

## Instalación

### 1 — Registrar el marketplace

Dentro de Claude Code, ejecuta:

```
/plugin marketplace add KodimTech/claude-plugins
```

Esto registra el marketplace `kodim` en tu configuración local. Solo se hace una vez.

### 2 — Instalar el plugin

```
claude plugin install boost-dev@kodim
```

### 3 — Habilitar el plugin

```
claude plugin enable boost-dev@kodim
```

### 4 — Recargar

```
/reload-plugins
```

Deberías ver `2 plugins · 2 skills · 6 agents` en el output.

---

## Uso

### Generar un plan

```
/boost-dev:plan https://app.shortcut.com/boost/story/46134/nombre-historia
```

O con el ID directamente:

```
/boost-dev:plan 46134
```

El planner:
1. Obtiene los detalles de la historia desde Shortcut
2. Lee el contexto del proyecto (architecture, graphql, testing, database, rubocop)
3. Explora el codebase real para anclar el plan en código existente
4. Muestra un análisis visible antes de escribir
5. Genera `plan-sc-<ID>-<slug>.md` en tu directorio actual

### Ejecutar un plan

```
/boost-dev:execute plan-sc-46134-nombre.md
```

El executor:
1. Lee el plan (si alguna sección es ambigua, se detiene y pregunta)
2. Carga los archivos de contexto
3. Lee todos los archivos que el plan indica
4. Muestra el plan de ejecución antes de tocar nada
5. Corre la migración si aplica
6. Escribe los specs primero (fase RED — todos deben fallar)
7. Implementa siguiendo el plan al pie de la letra
8. Corre specs después de cada paso (máx. 3 intentos por fallo, luego se detiene)
9. Corre rubocop en todos los archivos modificados
10. Hace `rake graphql:dump` si hubo cambios en GraphQL
11. Genera un reporte final para el reviewer

---

## Actualizar

```
claude plugin update boost-dev@kodim --scope local
```

Si da error de scope, desinstala y reinstala:

```
claude plugin uninstall boost-dev@kodim
claude plugin install boost-dev@kodim
```

---

## Estructura

```
boost-dev/
├── .claude-plugin/
│   └── plugin.json
├── agents/
│   └── senior-rails-dev.md    # Persona + reglas del Senior Dev (Opus)
├── context/
│   ├── architecture.md         # Interactors, services, concerns
│   ├── graphql.md              # Mutations, types, CI schema check
│   ├── testing.md              # RSpec + FactoryBot + TestProf
│   ├── database.md             # Migrations, strong_migrations, CI checks
│   └── rubocop.md              # Cops activos/desactivados en boost-api
├── skills/
│   ├── plan/
│   │   └── SKILL.md            # Flujo del planner
│   └── execute/
│       └── SKILL.md            # Flujo del executor (TDD)
└── settings.json               # Activa el agente senior-rails-dev
```
