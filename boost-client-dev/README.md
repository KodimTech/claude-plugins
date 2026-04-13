# boost-client-dev

Plugin de Claude Code para el equipo de **boost-client**. Actúa como un **Senior React/TypeScript Developer**: toma una historia de Shortcut, genera un plan de implementación detallado en `.md`, y luego puede ejecutarlo escribiendo tests, componentes y código lint-clean en modo TDD.

**Flujo:**
```
/boost-client-dev:plan <shortcut-url>      → genera plan-sc-XXXXX.md
/boost-client-dev:execute plan-sc-XXXXX.md → escribe tests (RED) → implementa → lint → reporte
```

El humano revisa el output antes de crear el PR. El plugin nunca hace commit.

---

## Flujo con backend (boost-api)

Cuando una historia requiere cambios en **ambos** repos, el planner detecta las dependencias automáticamente:

```
[boost-client] /boost-client-dev:plan <story>
  → detecta que necesita nuevas mutaciones/tipos en boost-api
  → genera plan-sc-XXXXX.md con sección "Backend Changes Required"
  → indica qué ejecutar en boost-api:

[boost-api] /boost-dev:plan <story>
  → genera plan del backend

[boost-api] /boost-dev:execute plan-sc-XXXXX.md
  → implementa backend + GraphQL schema dump

[boost-client] npm run codegen
  → actualiza src/graphql/types.ts

[boost-client] /boost-client-dev:execute plan-sc-XXXXX.md
  → implementa frontend usando los tipos generados
```

---

## Requisitos

- [Claude Code](https://claude.ai/code) instalado
- MCP de Shortcut configurado (el plugin lo usa para leer historias)
- Plugin `boost-dev` instalado (para el flujo con backend)

---

## Instalación

### 1 — Registrar el marketplace (solo una vez)

```
/plugin marketplace add KodimTech/claude-plugins
```

### 2 — Instalar el plugin

```
claude plugin install boost-client-dev@kodim
```

### 3 — Habilitar el plugin

```
claude plugin enable boost-client-dev@kodim
```

### 4 — Recargar

```
/reload-plugins
```

---

## Uso

### Generar un plan

```
/boost-client-dev:plan https://app.shortcut.com/boost/story/46134/nombre-historia
```

O con el ID directamente:

```
/boost-client-dev:plan 46134
```

El planner:
1. Obtiene los detalles de la historia desde Shortcut
2. Lee el contexto del proyecto (architecture, graphql, testing, styling)
3. Detecta si se necesitan cambios en el backend (boost-api)
4. Explora el codebase real para anclar el plan en código existente
5. Muestra un análisis visible antes de escribir
6. Genera `plan-sc-<ID>-<slug>.md` en tu directorio actual
7. Si hay cambios de backend: indica exactamente qué mutations/types crear en boost-api

### Ejecutar un plan

```
/boost-client-dev:execute plan-sc-46134-nombre.md
```

El executor:
1. Lee el plan — si `Backend Required: Yes`, se detiene y verifica que esté completo
2. Carga los archivos de contexto
3. Lee todos los archivos que el plan indica
4. Muestra el plan de ejecución antes de tocar nada
5. Crea archivos `.graphql` y corre codegen si aplica
6. Escribe los tests primero (fase RED — todos deben fallar)
7. Implementa siguiendo el plan al pie de la letra
8. Corre tests después de cada paso (máx. 3 intentos por fallo, luego se detiene)
9. Corre Biome/ESLint en todos los archivos modificados
10. Corre TypeScript check (`tsc --noEmit`)
11. Genera un reporte final para el reviewer

---

## Actualizar

```
claude plugin update boost-client-dev@kodim --scope local
```

---

## Estructura

```
boost-client-dev/
├── .claude-plugin/
│   └── plugin.json
├── agents/
│   └── senior-react-dev.md    # Persona + reglas del Senior Dev (Opus)
├── context/
│   ├── architecture.md         # Componentes, hooks, Redux, Apollo Client
│   ├── graphql.md              # Apollo operations, codegen, fragment conventions
│   ├── testing.md              # Jest + Testing Library + Cypress
│   └── styling.md              # Tachyons CSS, boost color system, componentes existentes
├── skills/
│   ├── plan/
│   │   └── SKILL.md            # Flujo del planner (detecta dependencias de backend)
│   └── execute/
│       └── SKILL.md            # Flujo del executor (TDD)
└── settings.json               # Activa el agente senior-react-dev
```
