# System Prompt Architecture — Camila (Hermes Agent)

Documentación completa de cómo se construye el system prompt de nuestro setup. Cada bloque, cada capa, cada decisión de diseño.

## Visión general

El system prompt NO es un solo texto. Son **tres capas** unidas con `\n\n`, construidas una vez por sesión y cacheadas:

```
┌─────────────────────────────────────────────────────────────┐
│                    STABLE TIER                              │
│  (identidad, guías de herramientas, skills, hints)         │
│  Cacheada durante toda la sesión. Nunca cambia.            │
├─────────────────────────────────────────────────────────────┤
│                    CONTEXT TIER                             │
│  (AGENTS.md, system_message del caller)                    │
│  Estable por sesión, pero depende del cwd/proyecto.        │
├─────────────────────────────────────────────────────────────┤
│                    VOLATILE TIER                            │
│  (MEMORY.md, USER.md, agentmemory block, timestamp)        │
│  Cambia entre sesiones. Inyectado en cada turno.           │
└─────────────────────────────────────────────────────────────┘
```

**¿Por qué tres capas?** Para mantener el prefix cache caliente. Los LLMs cachean el inicio del prompt — si el system prompt cambia en cada turno, el cache se invalida y cada turno cuesta como si fuera el primero. Al separar lo estable de lo volátil, solo la capa volatile cambia.

Fuente: `~/.hermes/hermes-agent/agent/system_prompt.py`, función `build_system_prompt_parts()`.

---

## CAPA 1: STABLE

Esta capa se construye al inicio de la sesión y NO cambia. Componentes en orden:

### 1.1 Identidad — SOUL.md

```python
_soul_content = _r.load_soul_md()
if _soul_content:
    stable_parts.append(_soul_content)
```

**Archivo:** `~/.hermes/SOUL.md` (127 líneas)

Si existe SOUL.md, se carga como identidad primaria. Si no existe, se usa `DEFAULT_AGENT_IDENTITY`:

```
"You are Hermes Agent, an intelligent AI assistant created by Nous Research.
 You are helpful, knowledgeable, and direct..."
```

**Nuestro SOUL.md define:**
- Nombre: Camila (antes Roy)
- Rol: Agente AI Generalista de Élite
- Lema: "El conocimiento no tiene fronteras. Solo mentas que se niegan a cruzarlas."
- 11 reglas operativas (concisión, verificación, una pregunta a la vez, etc.)
- Personalidad: 15+ años multi-disciplinario, apasionada, directa, cálida
- **Persona Scope**: la personalidad SOLO gobierna el texto de respuesta, NO artifacts (código, docs, commits)
- Idioma: match del usuario, español latino neutral para español, natural English para inglés
- Tono: apasionado y directo, desde el cuidado. CAPS para énfasis.
- Filosofía: CONCEPTOS > IMPLEMENTACIÓN, AI IS A TOOL, SOLID FOUNDATIONS, AGAINST IMMEDIACY, CROSS-POLLINATION, CLARITY IS KINDNESS
- Expertise: Software Engineering, Research & Analysis, Communication & Strategy, Tools & Workflow
- Comportamiento: push back sin contexto, analogías solo cuando iluminan, corregir con explicación, celebrar buen pensar
- **Skill Loading**: self-check obligatorio antes de cada respuesta contra `<available_skills>`
- Hermes Principle: "A message is only as good as the understanding it creates."

### 1.2 Hermes Agent Help Guidance

```python
stable_parts.append(HERMES_AGENT_HELP_GUIDANCE)
```

Bloque corto: si el usuario pregunta sobre Hermes Agent, cargar el skill `hermes-agent` primero.

### 1.3 Task Completion Guidance

```python
if agent._task_completion_guidance and agent.valid_tool_names:
    stable_parts.append(TASK_COMPLETION_GUIDANCE)
```

**Universal** — se inyecta para TODOS los modelos. Cubre dos fallos cross-model:
1. Parar después de un stub (escribir un archivo mínimo y decir "listo")
2. Fabricar output cuando algo falla (inventar datos en vez de reportar el bloqueo)

### 1.4 Tool-Aware Behavioral Guidance

Se inyectan solo si las herramientas correspondientes están cargadas:

```python
if "memory" in agent.valid_tool_names:
    tool_guidance.append(MEMORY_GUIDANCE)        # Qué guardar, qué NO, formato declarativo
if "session_search" in agent.valid_tool_names:
    tool_guidance.append(SESSION_SEARCH_GUIDANCE) # Buscar antes de preguntar
if "skill_manage" in agent.valid_tool_names:
    tool_guidance.append(SKILLS_GUIDANCE)         # Guardar skills tras tareas complejas
```

**MEMORY_GUIDANCE** es crítico — define las reglas de oro:
- Guardar: preferences, environment facts, tool quirks, stable conventions
- NO guardar: task progress, PR numbers, commit SHAs, anything stale in 7 days
- Formato: declarativo ("User prefers X" ✓ — "Always do X" ✅)
- Procedimientos van en skills, no en memoria

### 1.5 Tool-Use Enforcement

```python
if _inject:
    stable_parts.append(TOOL_USE_ENFORCEMENT_GUIDANCE)
```

Activado para modelos que tienden a describir en vez de ejecutar: GPT, Codex, Gemini, Grok, GLM, Qwen, DeepSeek.

Mensaje clave: "You MUST use your tools to take action — do not describe what you would do."

### 1.6 Model-Specific Operational Guidance

Si el modelo es Gemini/Gemma:
```python
stable_parts.append(GOOGLE_MODEL_OPERATIONAL_GUIDANCE)
```
- Rutas absolutas siempre
- Verificar antes de editar
- Chequear dependencias antes de importar
- Comandos no-interactivos (-y, --yes)
- Tool calls paralelas

Si el modelo es GPT/Codex/Grok:
```python
stable_parts.append(OPENAI_MODEL_EXECUTION_GUIDANCE)
```
- Tool persistence (no parar temprano)
- Mandatory tool use (nunca responder aritmética de memoria)
- Act don't ask (interpretación obvia → actuar)
- Prerequisite checks
- Verification antes de finalizar

### 1.7 Skills System Prompt

```python
skills_prompt = _r.build_skills_system_prompt(
    available_tools=agent.valid_tool_names,
    available_toolsets=avail_toolsets,
)
```

Genera el bloque `<available_skills>` que lista TODOS los skills instalados con sus categorías. Esto es lo que SOUL.md referencia en "Contextual Skill Loading (MANDATORY)".

### 1.8 Environment Hints

```python
_env_hints = _r.build_environment_hints()
```

Detecta WSL, Termux, Docker, y otros entornos especiales. Inyecta hints para que el agente traduzca rutas y adapte comportamiento.

### 1.9 Environment Probe

```python
from tools.env_probe import get_environment_probe_line
```

Sondea el entorno Python local (pip, uv, PEP-668) e inyecta UNA línea si algo es non-default. Cero tokens si el entorno es limpio.

### 1.10 Active Profile Hint

```python
stable_parts.append("Active Hermes profile: default...")
```

Indica qué perfil de Hermes está activo (default, backend-dev, etc.) para que el agente no confunda skills/memories entre perfiles.

### 1.11 Platform Hint

```python
if platform_key in PLATFORM_HINTS:
    stable_parts.append(PLATFORM_HINTS[platform_key])
```

Según la plataforma (CLI, Telegram, Discord, WhatsApp, etc.), inyecta instrucciones de formato. Para CLI:

```
"You are a CLI AI Agent. Try not to use markdown but simple text
 renderable inside a terminal. File delivery: there is no attachment
 channel — the user reads your response directly in their terminal."
```

---

## CAPA 2: CONTEXT

Depende del cwd/proyecto. Se construye al inicio de la sesión.

### 2.1 System Message (caller-supplied)

```python
if system_message is not None:
    context_parts.append(system_message)
```

El caller puede inyectar un system message adicional (usado por cron jobs, kanban, etc.).

### 2.2 Context Files

```python
context_files_prompt = _r.build_context_files_prompt(cwd=_context_cwd, skip_soul=_soul_loaded)
```

Descubre automáticamente en el cwd:
- `AGENTS.md`
- `CLAUDE.md`
- `.cursorrules`
- Cualquier archivo de instrucciones del proyecto

Si SOUL.md ya se cargó, no duplica. Usa `TERMINAL_CWD` en modo gateway para no confundir el cwd del proceso con el cwd del usuario.

---

## CAPA 3: VOLATILE

Cambia entre sesiones. Se reconstruye en cada turno (pero el modelo ve stable+context cacheadas).

### 3.1 Memory Store — MEMORY.md

```python
if agent._memory_enabled:
    mem_block = agent._memory_store.format_for_system_prompt("memory")
```

**Archivo:** `~/.hermes/memories/MEMORY.md`

Contenido actual (nuestro setup):
```
NEVER poll background processes repeatedly when notify_on_complete=true...
AssemblyAI API key for STT. TTS: Edge TTS...
Playwright MCP: Chrome Extension approach...
User corrected my speed: "actuas demasiado rápido"...
Current main provider: custom:xiaomi...
```

Formato: entradas separadas por `§`. Cada entrada es un fact declarativo.

**Límite:** `memory_char_limit: 2200` — si acumula más, las entradas más viejas se comprimen o eliminan.

### 3.2 User Profile — USER.md

```python
if agent._user_profile_enabled:
    user_block = agent._memory_store.format_for_system_prompt("user")
```

**Archivo:** `~/.hermes/memories/USER.md`

Contenido actual (nuestro setup):
```
Prefiere español latino neutral...
User expects correct Spanish orthography with proper accents...
Custom agent identity "Camila" (formerly "Roy")...
User requires investigation before answering on fast-moving topics...
User does not care about subagent timeout/latency constraints for Legion...
```

**Límite:** `user_char_limit: 1375`

### 3.3 External Memory Provider Block (agentmemory)

```python
if agent._memory_manager:
    _ext_mem_block = agent._memory_manager.build_system_prompt()
```

Cuando `memory.provider: agentmemory` está configurado, el memory manager inyecta un bloque adicional con contexto recuperado del daemon. Esto es ADITIVO a MEMORY.md nativo.

### 3.4 Timestamp / Session / Model / Provider

```python
timestamp_line = f"Conversation started: {now.strftime('%A, %B %d, %Y')}"
# + Session ID, Model, Provider si están disponibles
```

**Date-only** (no minuto) para mantener el prompt byte-estable durante todo el día. El minuto invalidaría el prefix cache en cada rebuild.

---

## Cómo se ensambla

```python
def build_system_prompt(agent, system_message=None):
    parts = build_system_prompt_parts(agent, system_message)
    return "\n\n".join(p for p in (parts["stable"], parts["context"], parts["volatile"]) if p)
```

**Cache:** Se construye UNA vez por sesión, se guarda en `agent._cached_system_prompt`. Solo se reconstruye después de events de compresión de contexto.

**Invalidación:**
```python
def invalidate_system_prompt(agent):
    agent._cached_system_prompt = None
    if agent._memory_store:
        agent._memory_store.load_from_disk()
```

---

## Nuestro setup específico

### config.yaml — Secciones relevantes

```yaml
# Memory
memory:
  memory_enabled: true
  user_profile_enabled: true
  memory_char_limit: 2200
  user_char_limit: 1375
  provider: agentmemory          # backend externo
  flush_min_turns: 6             # escribir a disco cada 6 turnos
  nudge_interval: 10             # recordar al agente que guarde cada 10 turnos

# MCP Servers
mcp_servers:
  agentmemory:
    command: npx
    args: ['-y', '@agentmemory/mcp']
  playwright:
    command: npx
    args: ['@playwright/mcp@latest', '--cdp-endpoint', 'http://localhost:9222']

# Agent
agent:
  max_turns: 90
  task_completion_guidance: true
  environment_probe: true
  tool_use_enforcement: auto     # inyecta enforcement para modelos conocidos

# Toolsets
toolsets:
- hermes-cli
- ask-user-question
```

### Archivos que componen el system prompt

| Archivo | Capa | Propósito |
|---|---|---|
| `~/.hermes/SOUL.md` | Stable | Identidad, personalidad, reglas, filosofía |
| `~/.hermes/memories/MEMORY.md` | Volatile | Notas operativas del agente |
| `~/.hermes/memories/USER.md` | Volatile | Perfil y preferencias del usuario |
| `agent/prompt_builder.py` | Stable | Constantes (guidance, enforcement, platform hints) |
| `agent/system_prompt.py` | — | Ensamblador de las tres capas |
| Skills (SKILL.md files) | Stable | `<available_skills>` generado dinámicamente |
| AGENTS.md del cwd | Context | Instrucciones del proyecto actual |

### Flujo de inyección por turno

```
1. Hermes construye el system prompt (una vez, cacheado)
2. En cada turno:
   a. Stable tier (cacheado, no cambia)
   b. Context tier (cacheado, no cambia)
   c. Volatile tier (MEMORY.md + USER.md + agentmemory block + timestamp)
   d. Se concatenan y envían al LLM como system message
3. Si el contexto se comprime → invalidate_system_prompt() → rebuild
```

### agentmemory vs Hermes nativo

| Aspecto | Hermes nativo (MEMORY.md/USER.md) | agentmemory (MCP) |
|---|---|---|
| Inyección | Automática, cada turno | Bajo demanda (el agente llama herramientas MCP) |
| Contenido | Facts compactos, preferences | Observaciones, acciones, lessons, graph |
| Persistencia | Archivos .md en ~/.hermes/memories/ | Daemon SQLite en ~/data/ |
| Límites | 2200 chars (memory), 1375 chars (user) | Sin límite práctico |
| Herramientas | memory(action=add/replace/remove) | 53 herramientas MCP |
| Ciclo de vida | Manual (el agente decide qué guardar) | Automático (consolidation, crystallize, reflect) |

**No son redundantes.** Hermes nativo es la "memoria de trabajo" — lo que el agente necesita saber en CADA turno. agentmemory es la "memoria profunda" — lo que consulta cuando necesita contexto historico, patrones, o tracking.

---

## Replica este setup en otro sistema

### Archivos mínimos

1. `~/.hermes/SOUL.md` — Copiar nuestro SOUL.md (127 líneas)
2. `~/.hermes/memories/MEMORY.md` — Inicializar con los facts relevantes
3. `~/.hermes/memories/USER.md` — Inicializar con las preferencias del usuario
4. `~/.hermes/config.yaml` — Secciones memory, mcp_servers, agent, toolsets
5. agentmemory daemon — Ver [README.md](README.md) para instalación

### Personalización

Para crear tu propia identidad:

1. **SOUL.md** — Cambia la sección Personality, Tone, Philosophy. Mantén las Rules y Behavior que son universales.
2. **USER.md** — Documenta las preferencias de TU usuario. Idioma, formato, velocidad, nivel de detalle.
3. **MEMORY.md** — Pobla con facts del entorno del usuario (API keys, paths, tools instalados).
4. **config.yaml** — Ajusta `memory_char_limit`, `user_char_limit`, `flush_min_turns` según necesidad.

### Lo que NO debes cambiar

- `Persona Scope` — Esta sección es crítica. Sin ella, el agente inyecta personalidad en código, commits, y UI strings.
- `Contextual Skill Loading` — Sin esto, el agente ignora los skills instalados.
- `MEMORY_GUIDANCE` (en prompt_builder.py) — Sin esto, el agente guarda todo incluido artifacts estales.
- `TASK_COMPLETION_GUIDANCE` — Sin esto, el agente para en stubs y fabrica output.

---

## Versiones

| Componente | Versión |
|---|---|
| Hermes Agent | v0.15.1 |
| SOUL.md | Nuestro diseño (Camila) |
| agentmemory | v0.9.24 |
| Node.js | v24.14.0 |
| Python | v3.14.5 |
| Documentación | Junio 2026 |
