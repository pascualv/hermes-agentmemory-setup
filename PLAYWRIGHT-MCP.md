# Playwright MCP — Skill y Flujo de Trabajo

Documentación completa de cómo Hermes usa Playwright MCP para automatización de navegador. Dos modos, configuración, script de conveniencia, pitfalls reales, y flujo de trabajo operativo.

## Arquitectura

```
┌─────────────┐     MCP (stdio)     ┌──────────────────┐     CDP/Extension    ┌─────────────┐
│ Hermes Agent │ ◄─────────────────► │ @playwright/mcp  │ ◄──────────────────► │ Chrome      │
│ (Python)     │                     │ (Node.js)        │                      │ (navegador) │
└─────────────┘                      └──────────────────┘                      └─────────────┘
```

Playwright MCP es un servidor MCP que expone herramientas de automatización de navegador al agente. El agente puede navegar, hacer click, escribir, tomar snapshots, y extraer contenido — todo a través de herramientas MCP estructuradas (no screenshots).

**Herramientas disponibles:**
- `browser_navigate` — navegar a una URL
- `browser_snapshot` — obtener árbol de accesibilidad (texto estructurado)
- `browser_click` — hacer click en un elemento por ref ID
- `browser_type` — escribir en un campo
- `browser_press` — presionar teclas (Enter, Tab, Escape)
- `browser_scroll` — scroll arriba/abajo
- `browser_vision` — screenshot + análisis visual
- `browser_console` — obtener logs/errores del console
- `browser_get_images` — listar imágenes de la página
- `browser_back` — navegar atrás

**Dos modos de conexión:**

| Aspecto | CDP Mode | Extension Mode |
|---|---|---|
| Config flag | `--cdp-endpoint http://localhost:9222` | `--extension` |
| Chrome setup | Lanzar con `--remote-debugging-port` | Instalar extensión del Chrome Web Store |
| Perfil | Perfil dedicado (`~/chrome-hermes`) | Perfil default del usuario |
| Sesiones | Login manual en perfil dedicado | Hereda TODAS las sesiones existentes |
| Visibilidad | Acciones invisibles al usuario | El usuario puede ver las tabs |
| Auth | Sin auth (localhost only) | Token de la extensión |
| Recomendado para | Autonomía total, scraping | Acceso a sesiones existentes |

---

## Modo 1: Chrome Extension (Recomendado)

Conecta al Chrome real del usuario. Hereda todas las sesiones activas (Google, etc.) sin login adicional.

### Instalación

**Paso 1. Instalar la extensión**

Desde Chrome Web Store: [Playwright Extension](https://chromewebstore.google.com/detail/playwright-extension/mmlmfjhmonkocbjadbfplnigmagldckm)

**Paso 2. Obtener el token de autenticación**

1. Click en el ícono de la extensión en Chrome
2. Copiar el valor de `PLAYWRIGHT_MCP_EXTENSION_TOKEN`
3. Este token es único por perfil de Chrome

**Paso 3. Configurar Hermes**

En `~/.hermes/config.yaml`:

```yaml
mcp_servers:
  playwright:
    command: npx
    args:
    - '@playwright/mcp@latest'
    - --extension
    env:
      PLAYWRIGHT_MCP_EXTENSION_TOKEN: "tu-token-aqui"
    connect_timeout: 60
    timeout: 120
    sampling:
      enabled: false
```

**Paso 4. Restart Hermes**

```bash
hermes restart
```

### Uso

Cuando el agente interactúa con el navegador por primera vez, se carga una página de selección de tab. El agente elige qué tab controlar.

Sin el token, cada conexión requiere aprobación manual del usuario. Con el token, las conexiones son automáticas.

### Ventajas
- Hereda TODAS las sesiones del usuario (Google, GitHub, etc.)
- No requiere Chrome especial ni script de lanzamiento
- El usuario puede ver lo que el agente hace en Chrome
- Funciona con el perfil default de Chrome

---

## Modo 2: CDP (Perfil Dedicado)

Lanza un Chrome separado con perfil dedicado y remote debugging. Control total, pero requiere login manual en el perfil dedicado.

### Instalación

**Paso 1. Crear script de conveniencia**

```bash
cat > ~/.local/bin/chrome-hermes << 'SCRIPT'
#!/bin/bash
# Chrome con remote debugging para MCP Playwright
# Uso: chrome-hermes [--kill|--status]

PROFILE="$HOME/chrome-hermes"
PORT=9222

case "$1" in
  --kill)
    pkill -f "remote-debugging-port=$PORT" && echo "Chrome cerrado" || echo "No había Chrome corriendo"
    exit 0
    ;;
  --status)
    if curl -s http://localhost:$PORT/json/version >/dev/null 2>&1; then
      echo "✓ Chrome activo en puerto $PORT"
      curl -s http://localhost:$PORT/json/version | python3 -c "import sys,json; d=json.load(sys.stdin); print(f'  {d[\"Browser\"]}')"
    else
      echo "✗ Chrome no está escuchando en puerto $PORT"
    fi
    exit 0
    ;;
esac

# Verificar si ya está corriendo
if curl -s http://localhost:$PORT/json/version >/dev/null 2>&1; then
  echo "Chrome ya está activo en puerto $PORT"
  curl -s http://localhost:$PORT/json/version | python3 -c "import sys,json; d=json.load(sys.stdin); print(f'  {d[\"Browser\"]}')"
  exit 0
fi

# Crear directorio si no existe
mkdir -p "$PROFILE"

# Lanzar Chrome
echo "Iniciando Chrome con remote debugging..."
echo "  Perfil: $PROFILE"
echo "  Puerto: $PORT"
/opt/google/chrome/chrome \
  --remote-debugging-port=$PORT \
  --user-data-dir="$PROFILE" \
  "$@" &

# Esperar a que el puerto esté disponible
for i in $(seq 1 10); do
  if curl -s http://localhost:$PORT/json/version >/dev/null 2>&1; then
    echo "✓ Chrome listo"
    exit 0
  fi
  sleep 1
done

echo "✗ Chrome no respondió en el puerto $PORT después de 10s"
exit 1
SCRIPT

chmod +x ~/.local/bin/chrome-hermes
```

**Paso 2. Configurar Hermes**

En `~/.hermes/config.yaml`:

```yaml
mcp_servers:
  playwright:
    command: npx
    args:
    - '@playwright/mcp@latest'
    - --cdp-endpoint
    - http://localhost:9222
    connect_timeout: 60
    timeout: 120
    sampling:
      enabled: false
```

**Paso 3. Lanzar Chrome y restart Hermes**

```bash
chrome-hermes          # lanzar Chrome con perfil dedicado
hermes restart         # restart para que MCP se conecte
```

### Uso diario

```bash
chrome-hermes          # lanzar (o verificar que ya corre)
chrome-hermes --status # verificar estado
chrome-hermes --kill   # cerrar Chrome
```

### Perfil persistente

El perfil vive en `~/chrome-hermes/` y persiste entre reinicios:
- Login una vez a cualquier sitio → las sesiones sobreviven reinicios de Chrome
- Cookies, localStorage, y extensiones se preservan
- Para empezar de cero: `rm -rf ~/chrome-hermes` y relanzar

---

## Pitfalls (Reales, Encontrados en Uso)

### Chrome debe matarse primero
`--remote-debugging-port` se ignora si Chrome ya está corriendo. El nuevo proceso se adjunta al existente sin debugging. Solución: `pkill -f google-chrome` antes de lanzar.

### Lock file residual
Si `~/.config/google-chrome/SingletonLock` existe después de matar Chrome:
```bash
rm ~/.config/google-chrome/SingletonLock
```

### Chrome 147+ requiere user-data-dir non-default
`--remote-debugging-port` requiere `--user-data-dir` con ruta NO default. Usar `~/chrome-hermes`, NO `~/.config/google-chrome`.

### Las acciones de Playwright son invisibles al usuario (CDP mode)
Playwright controla Chrome via CDP pero NO trae la ventana al frente. Las páginas cargan silenciosamente en tabs de background.

**Workaround para tabs visibles:**
```bash
curl -s -X PUT "http://localhost:9222/json/new?https://example.com"
```
Crea una tab que aparece inmediatamente en el Chrome del usuario.

**Cuándo usar qué:**
- Playwright MCP → automatización/headless donde la visibilidad no importa
- CDP curl directo → cuando el usuario NECESITA ver el navegador

### MCP muere cuando Chrome se reinicia (CDP mode)
Si el usuario mata y relanza Chrome, la conexión WebSocket de Playwright MCP se rompe permanentemente. La sesión actual de Hermes NO puede recuperarse — el usuario debe iniciar una nueva sesión `hermes`.

### Sampling causa HTTP 400 en sesiones background
Si `auxiliary.mcp.model` está vacío (default) y sampling está habilitado, sesiones cron/background fallan con HTTP 400. Por eso:
```yaml
sampling:
  enabled: false
```

### Security: Remote debugging no tiene auth
Solo usar en localhost. Nunca exponer a la red.

### Brave/TradingView/Electron NO son Chrome
`pkill -f google-chrome` no mata Brave. Buscar `/opt/google/chrome/chrome` específicamente en `ps aux`.

---

## Flujo de Trabajo del Agente

### Patrón general: Navigate → Snapshot → Act → Verify

```
1. browser_navigate(url)     → ir a la página
2. browser_snapshot()        → obtener árbol de accesibilidad
3. Analizar snapshot         → identificar elementos por ref ID
4. browser_click/@e5)        → interactuar
5. browser_snapshot()        → verificar resultado
6. Repetir 3-5 según necesidad
```

### Cuando el agente usa Playwright

| Escenario | Herramientas típicas |
|---|---|
| Investigar un sitio web | navigate → snapshot → extract content |
| Llenar un formulario | navigate → snapshot → type → press(Enter) |
| Hacer login (CDP) | navigate → snapshot → type → click → snapshot |
| Scraping de datos | navigate → snapshot → console(expression) |
| Verificar un deploy | navigate → snapshot → buscar texto específico |
| Debug de una UI | navigate → vision(screenshot) → console |
| Crear una tab visible | terminal(curl CDP) |

### Ejemplo real: investigar un sitio

```python
# 1. Navegar
browser_navigate(url="https://api.xiaomimimo.com/docs")

# 2. Obtener estructura
browser_snapshot()
# → devuelve árbol con ref IDs: @e1=header, @e2=nav, @e5=link "API Reference"

# 3. Interactuar
browser_click(ref="@e5")

# 4. Verificar
browser_snapshot(full=True)
# → contenido completo de la página de API Reference

# 5. Extraer datos específicos
browser_console(expression="document.querySelectorAll('h2').length")
```

### Ejemplo real: llenar formulario

```python
browser_navigate(url="https://example.com/login")
browser_snapshot()
# → @e3=input "email", @e4=input "password", @e7=button "Submit"

browser_type(ref="@e3", text="user@example.com")
browser_type(ref="@e4", text="password123")
browser_click(ref="@e7")

# Verificar login exitoso
browser_snapshot()
# → buscar "Welcome" o el dashboard
```

### Cuándo NO usar Playwright

- **web_extract** es suficiente — para contenido estático, web_extract es más rápido y barato
- **web_search** es suficiente — para buscar información, no necesitas un navegador
- **Terminal + curl** — para APIs REST, curl es más directo
- **read_file** — para archivos locales, no necesita navegador

**Usar Playwright cuando:**
- La página es JavaScript-heavy (SPA, React, Vue)
- Necesitas interactuar con la página (click, scroll, forms)
- Necesitas ver qué hay en pantalla (snapshot de accesibilidad)
- La página requiere autenticación (CDP con perfil persistente o Extension con sesiones existentes)

---

## MCP Config Reference

### Extension mode
```yaml
mcp_servers:
  playwright:
    command: npx
    args:
    - '@playwright/mcp@latest'
    - --extension
    env:
      PLAYWRIGHT_MCP_EXTENSION_TOKEN: "tu-token"
    connect_timeout: 60
    timeout: 120
    sampling:
      enabled: false
```

### CDP mode
```yaml
mcp_servers:
  playwright:
    command: npx
    args:
    - '@playwright/mcp@latest'
    - --cdp-endpoint
    - http://localhost:9222
    connect_timeout: 60
    timeout: 120
    sampling:
      enabled: false
```

### Opciones útiles adicionales

| Flag | Env var | Default | Descripción |
|---|---|---|---|
| `--headless` | `PLAYWRIGHT_MCP_HEADLESS` | false | Sin ventana visible |
| `--browser <name>` | — | chrome | chrome, firefox, webkit, msedge |
| `--caps <caps>` | — | — | vision, pdf, devtools |
| `--user-data-dir <path>` | `PLAYWRIGHT_MCP_USER_DATA_DIR` | temp | Perfil persistente |
| `--timeout-action <ms>` | `PLAYWRIGHT_MCP_TIMEOUT_ACTION` | 5000 | Timeout por acción |
| `--timeout-navigation <ms>` | `PLAYWRIGHT_MCP_TIMEOUT_NAVIGATION` | 60000 | Timeout de navegación |
| `--viewport-size <size>` | `PLAYWRIGHT_MCP_VIEWPORT_SIZE` | — | Tamaño del viewport |
| `--console-level <level>` | — | error | error, warning, info, debug |
| `--save-session` | — | false | Guardar sesión en output dir |

---

## Nuestro Setup Actual

| Componente | Valor |
|---|---|
| Modo | CDP (perfil dedicado) |
| Chrome script | `~/.local/bin/chrome-hermes` |
| Perfil | `~/chrome-hermes/` |
| Puerto | 9222 |
| Playwright MCP | `@playwright/mcp@latest` (v0.0.75+) |
| Hermes config | `mcp_servers.playwright` con `--cdp-endpoint` |
| Sampling | disabled |

**Nota:** La memoria indica que el Chrome Extension approach es "recommended". El modo CDP funciona bien para nuestro caso (control total, perfil dedicado), pero el Extension mode es más simple y hereda sesiones existentes sin login adicional.

---

## Versiones

| Componente | Versión |
|---|---|
| @playwright/mcp | 0.0.75+ |
| Playwright (dependencia) | 1.61.0-alpha |
| Chrome | estable (Fedora 43) |
| Documentación | Junio 2026 |
