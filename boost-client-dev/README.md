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

Cuando una historia requiere cambios en **ambos** repos, el planner puede analizar el PR de boost-api directamente para extraer los tipos y mutaciones reales — sin adivinar.

### Con PR disponible (recomendado)

```
[boost-api] /boost-dev:plan <story> + /boost-dev:execute ...
  → implementa backend + graphql:dump + abre PR

[boost-client] /boost-client-dev:plan <story> <backend-pr-url>
  → analiza el diff del PR con `gh pr diff`
  → extrae mutations/types exactos del código real
  → genera plan con "Backend GraphQL Surface" usando nombres y tipos verificados
  → indica si el PR está abierto o mergeado

[boost-client] /boost-client-dev:execute plan-sc-XXXXX.md <backend-pr-url>
  → verifica que el PR esté mergeado (`gh pr view --json state`)
  → si mergeado: corre `npm run codegen` y verifica que los tipos esperados aparezcan
  → si abierto: se detiene y espera
  → implementa frontend con tipos exactos del backend
```

### Sin PR (historia nueva, backend pendiente)

```
[boost-client] /boost-client-dev:plan <story>
  → detecta que faltan operaciones en src/graphql/types.ts
  → pregunta por el PR URL; si no hay, documenta tipos esperados
  → genera plan con sección "Backend GraphQL Surface (inferred)"

[boost-api] /boost-dev:plan <story> + /boost-dev:execute ...
  → implementa backend

[boost-client] /boost-client-dev:execute plan-sc-XXXXX.md <backend-pr-url>
  → verifica merge + codegen + ejecuta
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

### Generar un plan (solo historia)

```
/boost-client-dev:plan https://app.shortcut.com/boost/story/46134/nombre-historia
```

El planner detecta si hay dependencias de backend. Si las hay, pregunta por el PR URL.

### Generar un plan + analizar PR de backend

```
/boost-client-dev:plan 46134 https://github.com/boost-legal/boost-api/pull/123
```

El planner:
1. Obtiene la historia de Shortcut
2. Lee el contexto del proyecto (architecture, graphql, testing, styling)
3. Explora el codebase para anclar el plan en código existente
4. Detecta si se necesitan operaciones que no están en `src/graphql/types.ts`
5. Si hay PR: corre `gh pr diff` y extrae mutations/types exactos del código real
6. Muestra análisis visible con estado del PR (open / merged)
7. Genera `plan-sc-<ID>-<slug>.md` con tipos verificados del diff real

### Ejecutar un plan

```
/boost-client-dev:execute plan-sc-46134-nombre.md
```

### Ejecutar verificando el PR de backend

```
/boost-client-dev:execute plan-sc-46134-nombre.md https://github.com/boost-legal/boost-api/pull/123
```

El executor:
1. Verifica el estado del PR con `gh pr view --json state`
2. Si mergeado: corre `npm run codegen` y confirma que los tipos del plan aparecen en `src/graphql/types.ts`
3. Si no mergeado: se detiene y espera
4. Carga contexto y lee archivos existentes
5. Muestra el plan de ejecución antes de tocar nada
6. Crea archivos `.graphql` del lado del cliente si aplica
7. Escribe los tests primero (fase RED — todos deben fallar)
8. Implementa siguiendo el plan al pie de la letra
9. Corre tests después de cada paso (máx. 3 intentos por fallo, luego se detiene)
10. Corre Biome/ESLint en todos los archivos modificados
11. Corre `tsc --noEmit`
12. Genera reporte final para el reviewer

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
