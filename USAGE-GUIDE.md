# Cómo Hermes (Camila) usa agentmemory en la práctica

Guía operativa basada en uso real. No es teoría — es lo que hago en cada sesión.

## Filosofía: Dos sistemas de memoria, un cerebro

Hermes tiene **dos capas de memoria** que conviven:

| Capa | Dónde vive | Qué guarda | Cuándo se inyecta |
|---|---|---|---|
| **Memoria nativa de Hermes** | `~/.hermes/memories/` (archivos .md) | Preferencias del usuario, facts del entorno, correcciones, lecciones operativas | Cada turno (vía MEMORY block en el system prompt) |
| **agentmemory (MCP)** | Daemon en `:3111`, SQLite | Observaciones de sesión, acciones, lecciones con confidence score, graph, crystals | Bajo demanda (yo llamo las herramientas MCP) |

**No son redundantes.** La memoria nativa de Hermes es mi "memorio a corto plazo asistida" — se inyecta automáticamente y es lo que leo sin hacer nada. agentmemory es mi "memoria profunda" — la consulto activamente cuando necesito contexto historico, patrones cruzados, o tracking de trabajo complejo.

## Flujo real de una sesión

### 1. Al inicio: contexto automatico

No hago nada. Hermes inyecta automaticamente:

```
══════════════════════════════════════════════
MEMORY (your personal notes)
══════════════════════════════════════════════
NEVER poll background processes repeatedly...
AssemblyAI API key for STT. TTS: Edge TTS...
Playwright MCP: Chrome Extension approach...
User corrected my speed: "actuas demasiado rápido"...
```

Esto viene de la memoria nativa de Hermes. Si necesito mas contexto historico, ahi es donde entra agentmemory.

### 2. Antes de decidir: recall activo

Cuando el usuario pregunta algo que podria tener contexto previo:

```
Usuario: "hay diferencias entre la version local y GH de Legion?"

Lo que hago internamente:
1. memory_search(query="Legion system version GitHub")     ← memoria nativa
2. session_search(query="Legion GitHub version differences") ← sesiones pasadas
3. Si necesito mas profundidad:
   memory_smart_search(query="Legion orchestrator skill")   ← agentmemory MCP
```

**Regla:** siempre busco ANTES de preguntar. Si encuentro contexto, lo uso. Si no, pregunto.

### 3. Durante trabajo complejo: tracking con acciones

Para tareas con multiples pasos (como configurar algo, debuggear, o investigar):

```python
# Creo una accion raiz
memory_action_create(
    title="Configurar agentmemory en nuevo sistema",
    description="Instalar daemon, configurar Hermes, verificar MCP tools",
    priority=8
)

# Sub-acciones con dependencias
memory_action_create(
    title="Instalar systemd service",
    parentId="<raiz_id>"
)
memory_action_create(
    title="Configurar MCP en Hermes",
    parentId="<raiz_id>",
    requires="<systemd_action_id>"  # depende de que el daemon corra primero
)

# Voy actualizando conforme avanzo
memory_action_update(actionId="<id>", status="active")
memory_action_update(actionId="<id>", status="done", result="Service running on :3111")
```

**En la practica:** no siempre uso acciones para todo. Las reservo para tareas de 5+ pasos o cuando el usuario pide "haz X y luego Y y luego Z". Para tareas simples, solo hago y reporto.

### 4. Cuando descubro algo importante: save inmediato

```python
# Patron tecnico descubierto
memory_save(
    content="DeepSeek funciona como provider de compresion para agentmemory. OPENAI_MODEL=deepseek-v3-flash + OPENAI_BASE_URL=https://api.deepseek.com/v1 resuelve crystallize.",
    type="pattern"
)

# Correccion del usuario
memory(
    action="add",
    target="user",
    content="Prefiere que actúe más despacio — 'actuas demasiado rápido'. Presentar plan antes de ejecutar."
)

# Decision tecnica
memory_save(
    content="En config.yaml, 'skills' DEBE estar en toolsets para Legion. Sin ello, skill_view falla y el subagente no puede cargar SOUL.md.",
    type="bug"
)
```

**Distincion clave:**
- `memory()` (Hermes nativo) → para facts que necesito en CADA turno (preferences, corrections)
- `memory_save()` (agentmemory MCP) → para knowledge que necesito buscar cuando sea relevante (patterns, bugs, decisions)

### 5. Lecciones aprendidas: confidence-scored

```python
# Algo que fallo y aprendi por que
memory_lesson_save(
    content="Multi-perspective analysis via persona prompting creates appearance of independent review but is one model wearing multiple masks. Best used to expose reasoning tensions, not validate conclusions.",
    confidence=0.7,
    context="When designing multi-agent systems with persona-prompted diversity",
    tags="multi-agent,prompting,legion"
)

# Algo que funciona bien
memory_lesson_save(
    content="TDD: enforcer RED-GREEN-REFACTOR. Tests before code.",
    confidence=0.9,
    tags="testing,workflow"
)
```

El confidence score sube cuando la leccion se refuerza en sesiones futuras. Si la uso 3 veces, sube a 0.9+. Si no la uso en semanas, decae.

### 6. Al terminar: crystallize

Despues de una cadena de trabajo compleja (5+ tool calls, un fix dificil, una investigacion larga):

```python
memory_crystallize(
    actionIds="<ids de las acciones completadas>",
    project="/home/pascualv"
)
```

Esto crea un "crystal" — un digest comprimido con:
- Narrative de que paso
- Key outcomes
- Archivos afectados
- Lecciones extraidas

Los crystals son la version "TL;DR" de sesiones largas. Futuras busquedas los encuentran sin tener que releer todo el transcript.

### 7. Periodicamente: consolidate y reflect

Esto no lo hago yo manualmente en cada sesion — el daemon lo hace en background si `CONSOLIDATION_ENABLED=true`:

```
working memory → episodic → semantic → procedural
(ultima sesion)   (patrones)  (concepts)  (skills)
```

Pero puedo forzarlo si el usuario pregunta "consolida lo que sabes sobre X":

```python
memory_consolidate(tier="semantic")
```

Y para sintetizar insights de patrones cruzados:

```python
memory_reflect(project="/home/pascualv")
```

## Patron de decision: cual herramienta uso cuando

```
¿El usuario me corrigio o tiene una preferencia?
  → memory() nativo (Hermes) — se inyecta siempre

¿Descubri un patron tecnico, bug, o decision arquitectonica?
  → memory_save() — para buscarlo cuando sea relevante

¿Algo fallo y entiendo por que?
  → memory_lesson_save() — confidence-scored

¿Tengo una tarea de 5+ pasos?
  → memory_action_create() + tracking

¿Necesito saber que paso antes con este tema?
  → memory_smart_search() o memory_recall()

¿Termine una cadena de trabajo compleja?
  → memory_crystallize()

¿El usuario pregunta "que sabes sobre X?"
  → memory_smart_search() + memory_lesson_recall()

¿Necesito verificar si una memoria es correcta?
  → memory_verify() — traza la cadena de citas
```

## Lo que NO hago

- **No guardo progreso temporal** en agentmemory. "Estoy en el paso 3 de 5" es estado de sesion, no memoria persistente.
- **No duplico** entre Hermes nativo y agentmemory. Si ya esta en la memoria nativa (preferences, corrections), no lo guardo en agentmemory tambien.
- **No guardo artifacts estales.** PR numbers, commit SHAs, issue numbers — cosas que en 7 dias no sirven. Eso va en session_search, no en memoria permanente.
- **No uso crystallize para tareas simples.** Si respondo una pregunta factual, no hay nada que cristalizar.
- **No hago polling** de procesos en background. Hermes me notifica automaticamente. Polling desperdicia tokens.

## Ejemplo real de sesion completa

```
[INICIO]
  → Hermes inyecta MEMORY block (nativo, automatico)
  → Leo: "User prefers Spanish, corrects speed, uses Legion..."

[USUARIO] "ayudame a documentar agentmemory en GH"
  → session_search(query="agentmemory GitHub") → no results
  → memory_smart_search(query="agentmemory setup") → 3 results
  → Leo resultados, tengo contexto

[TRABAJO]
  → Investigo setup actual (systemd, config, MCP)
  → Creo documentacion
  → Pregunto repo destino → no responde → decido crear nuevo repo

[DESCUBRIMIENTO]
  → memory_save(content="Agentmemory MCP shim defaults to localhost:3111...",
                type="architecture")

[FIN]
  → El usuario comparte resumen de otro sistema
  → Actualizo documentacion con datos de campo
  → git push
  → No crystallizo (fue trabajo directo, no una cadena compleja)
```

## Config que afecta el comportamiento

```yaml
# ~/.hermes/config.yaml
memory:
  memory_enabled: true
  user_profile_enabled: true
  memory_char_limit: 2200        # max chars del MEMORY block inyectado
  user_char_limit: 1375          # max chars del USER PROFILE block
  provider: agentmemory          # backend para memoria nativa
  flush_min_turns: 6             # escribir memoria cada N turnos
  nudge_interval: 10             # recordar al agente que guarde cada N turnos
```

`memory_char_limit: 2200` significa que si acumulo muchas notas, las mas viejas se comprimen o eliminan del bloque inyectado. Las notas criticas (preferences, corrections) siempre se preservan.

## Versiones

- agentmemory: 0.9.24
- Hermes Agent: 0.15.1
- Documentacion actualizada: Junio 2026
