# Rails Planner Plugin

Plugin de Claude Code para el equipo de **boost-api**. Actúa como un **Senior Rails Developer**: toma una historia de Shortcut, analiza los requisitos y genera un plan de implementación detallado en formato `.md`, siguiendo los patrones y convenciones del proyecto.

---

## Instalación

### Opción A — Uso por sesión (recomendado para empezar)

```bash
claude --plugin-dir /ruta/local/al/rails-planner-plugin
```

### Opción B — Instalación permanente (disponible en todas las sesiones)

1. Clona el plugin en tu máquina:

```bash
git clone https://github.com/boost-legal/rails-planner-plugin ~/rails-planner-plugin
```

2. Instala el plugin desde Claude Code:

```
/plugin install ~/rails-planner-plugin
```

A partir de ese momento el skill estará disponible en cualquier sesión.

---

## Uso

Dentro de Claude Code, ejecuta:

```
/rails-planner:plan https://app.shortcut.com/boost/story/46134/nombre-historia
```

O con el ID directamente:

```
/rails-planner:plan 46134
```

El plugin:
1. Obtiene los detalles de la historia desde Shortcut
2. Analiza el dominio, los patrones a aplicar y los archivos afectados
3. Genera un archivo `plan-sc-<ID>-<slug>.md` en tu directorio actual

---

## Requisitos

- Claude Code instalado (`npm install -g @anthropic-ai/claude-code`)
- MCP de Shortcut configurado (el workspace ya lo tiene configurado como `Lawmatics-Shortcut`)

---

## Estructura del plugin

```
rails-planner-plugin/
├── .claude-plugin/
│   └── plugin.json         # Manifiesto del plugin
├── skills/
│   └── plan/
│       └── SKILL.md        # Prompt del Senior Rails Developer
└── README.md
```

---

## Actualizar el plugin

```bash
cd ~/rails-planner-plugin && git pull
```

Luego en Claude Code:

```
/reload-plugins
```
