# boost-team

Plugin de Claude Code para planificación full-stack. Coordina `boost-dev` (backend) y `boost-client-dev` (frontend) como un equipo de agentes.

**Un solo comando genera ambos planes:**
```
/boost-team:plan-full 46134
```

---

## Qué hace

1. **Descubre ambos repos** — busca `boost-api` y `boost-client` como hermanos en el directorio padre
2. **Determina el scope** — backend-only, frontend-only, o full-stack
3. **Define el contrato GraphQL** — mutations, types, campos acordados antes de que los agentes empiecen (evita nombres incompatibles entre repos)
4. **Lanza ambos agentes en paralelo** — backend y frontend planean simultáneamente
5. **Resuelve conflictos** — si los agentes proponen names diferentes, el coordinator los alinea
6. **Produce un summary** con ambos planes y el orden exacto de ejecución

---

## Flujo completo

```
/boost-team:plan-full 46134
  ↓
  ├── backend_agent → $PARENT/boost-api/plan-sc-46134-<slug>.md
  └── frontend_agent → $PARENT/boost-client/plan-sc-46134-<slug>.md
  ↓
  Coordination summary + execution order

Ejecución (en orden):

[boost-api]
  /boost-dev:execute plan-sc-46134-<slug>.md
  → tests pasan → abrir PR

[boost-client]
  /boost-client-dev:execute plan-sc-46134-<slug>.md <backend-pr-url>
  → verifica PR mergeado → codegen → implementa
```

---

## Requisitos

- `boost-dev@kodim` instalado
- `boost-client-dev@kodim` instalado
- Ambos repos como hermanos en el mismo directorio padre:
  ```
  <parent>/
  ├── boost-api/
  └── boost-client/
  ```
- MCP de Shortcut configurado

---

## Instalación

```
claude plugin install boost-team@kodim
claude plugin enable boost-team@kodim
/reload-plugins
```

---

## Estructura

```
boost-team/
├── agents/tech-lead.md        # Coordinador: define contratos, lanza agentes, resuelve conflictos
├── context/repos.md           # Cómo descubrir y acceder a ambos repos
├── skills/plan-full/SKILL.md  # Skill de planificación full-stack
└── settings.json
```
