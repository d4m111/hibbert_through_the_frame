# Plan: Integración de Claude Chat en Hibbert

## Objetivo

Agregar un panel de chat con Claude dentro de la app que pueda **controlar** el editor de queries y leer/interpretar los resultados. El usuario conversa en lenguaje natural y Claude:

- Escribe queries en [hibbert/ui/query_editor.py](../../hibbert/ui/query_editor.py)
- Las ejecuta
- Lee los resultados desde [hibbert/ui/results_panel.py](../../hibbert/ui/results_panel.py)
- Itera o explica al usuario

## Viabilidad

Totalmente factible usando **tool use** del SDK de Anthropic (o el Claude Agent SDK, que envuelve el loop de herramientas). El patrón es:

1. Definimos tools (funciones Python) que controlan la UI.
2. Claude conversa con el usuario y decide qué tool llamar.
3. La app ejecuta la tool sobre los widgets reales y devuelve el resultado.
4. Claude interpreta y responde.

## "Usar la cuenta de Claude del usuario"

**Limitación importante:** la suscripción de claude.ai (Pro/Max) no está disponible para apps de terceros. No existe un OAuth público que conecte esa suscripción.

**Solución estándar:** el usuario obtiene una API key en https://console.anthropic.com y la pega en una pantalla de Settings de Hibbert. El costo corre por esa cuenta API (separada de la suscripción a claude.ai).

- Guardar la key con `keyring` (keychain del SO) — no en plano en `settings.json`.
- Validar la key al guardarla con un request mínimo.

## Elección: Agent SDK vs SDK directo

| Opción | Ventaja | Desventaja |
|---|---|---|
| **Claude Agent SDK** | Loop de tool use ya armado, manejo de contexto, menos código | Dependencia más pesada, menos control fino |
| **`anthropic` SDK directo** | Liviano, control total del loop | Hay que implementar el bucle de tool use a mano |

**Recomendación:** empezar con Agent SDK. Si más adelante hace falta control fino (streaming custom, compaction propia, etc.), migrar al SDK directo es directo porque las tools se reusan tal cual.

## Arquitectura

### Componentes nuevos

```
hibbert/
├── ai/
│   ├── __init__.py
│   ├── client.py          # Wrapper sobre el SDK (async, manejo de API key)
│   ├── tools.py           # Definición de tools y dispatch
│   └── conversation.py    # Estado de la conversación (mensajes, tool calls)
└── ui/
    ├── chat_panel.py      # QDockWidget con el chat
    └── settings_dialog.py # (extender) campo para la API key
```

### Flujo de ejecución

```
Usuario escribe en chat
        │
        ▼
ChatPanel emite send_message(texto)
        │
        ▼
AiClient (en QThread) llama al SDK
        │
        ├─► Claude responde con texto  ──► ChatPanel muestra
        │
        └─► Claude pide tool call
                  │
                  ▼
            tools.dispatch(tool_name, args)
                  │
                  ▼ (en main thread vía signal)
            QueryEditor / ResultsPanel ejecutan acción
                  │
                  ▼
            Resultado serializado → SDK
                  │
                  ▼
            Claude continúa el loop
```

### Threading

- El cliente Claude corre en un `QThread` (o `QRunnable` con `QThreadPool`) — bloquear la UI con un request HTTP es inaceptable.
- Las tools que tocan widgets Qt **deben ejecutarse en el main thread**. Patrón:
  - Worker emite `tool_requested(name, args, reply_id)`.
  - Slot en main thread ejecuta y emite `tool_replied(reply_id, result)`.
  - Worker espera el reply (`QEventLoop` local o `QWaitCondition`).

## Set inicial de tools

| Tool | Args | Devuelve | Notas |
|---|---|---|---|
| `list_databases` | — | `[str]` | |
| `list_collections` | `database` | `[{name, type}]` | Incluye `type: "view"` |
| `get_collection_schema` | `database, collection, sample_size=20` | `{field: type}` | Sampling para inferir schema |
| `set_query` | `text` | `ok` | Reemplaza contenido del editor |
| `append_to_query` | `text` | `ok` | Útil para construir queries incrementales |
| `get_query` | — | `text` | Lee el editor actual |
| `execute_query` | — | `{count, elapsed_ms, sample, error?}` | Dispara `execute_requested` y espera |
| `get_results_sample` | `n=10` | `[doc]` | Lee `_current_data` truncado |
| `get_results_summary` | — | `{count, fields, types}` | Resumen sin enviar todos los docs |
| `get_indexes` | `database, collection` | `[index]` | |
| `get_collection_stats` | `database, collection` | `{...}` | Saltar si `type == "view"` |

### Por qué *no* mandar todos los resultados

Un `find` puede traer megas. Por defecto las tools devuelven **resúmenes** (`get_results_summary`) o **muestras truncadas** (`get_results_sample` con `n` limitado). Claude pide más solo si lo necesita. Esto baja costo de tokens y acelera respuestas.

### Confirmación del usuario para escrituras

Operaciones destructivas (`insertOne`, `insertMany`, `updateOne`, `updateMany`, `replaceOne`, `deleteOne`, `deleteMany`, `bulkWrite`, `dropIndex`, etc.) **deben pedir confirmación explícita** antes de ejecutarse.

Implementación: `execute_query` detecta el método (parser ya existente) y, si es de escritura, antes de ejecutar muestra un diálogo "Claude quiere ejecutar: `db.users.deleteMany({...})` — ¿Confirmar?". Sin confirmación, la tool devuelve `{error: "user_declined"}` y Claude se entera.

## Cambios mínimos a archivos existentes

### `hibbert/ui/query_editor.py`
- No requiere cambios — ya expone `setPlainText`, `toPlainText`, `execute_requested`.

### `hibbert/ui/results_panel.py`
- Exponer un getter público para `_current_data` (o usar el atributo directamente desde el dispatcher de tools).
- Posiblemente agregar `get_summary()` que devuelva `{count, fields, types}` para `get_results_summary`.

### `hibbert/ui/main_window.py` (o donde se arma el layout)
- Agregar `QDockWidget` con `ChatPanel` (cerrable, redockeable).
- Conectar el dispatcher de tools a las instancias activas de `QueryEditor` / `ResultsPanel` del tab actual.

### Settings
- Agregar campo `anthropic_api_key` (almacenado vía `keyring`).
- Agregar campo `claude_model` (default: `claude-opus-4-7` — el más capable; permitir cambiar a `claude-sonnet-4-6` o `claude-haiku-4-5` para usuarios cost-sensitive).

## UI del chat panel

```
┌────────────────────────────────┐
│ Claude  (opus-4.7) ⚙           │  ← header con modelo y settings
├────────────────────────────────┤
│                                │
│ [historial de mensajes]        │
│                                │
│ Usuario:                       │
│ > muéstrame los users sin email│
│                                │
│ Claude: voy a revisar el       │
│ schema primero…                │
│ ▸ get_collection_schema(users) │  ← tool calls colapsables
│ ▸ set_query({...})             │
│ ▸ execute_query() → 142 docs   │
│                                │
│ Encontré 142 usuarios sin      │
│ campo email. ¿Querés ver…      │
│                                │
├────────────────────────────────┤
│ [input multilínea]      [Send] │
│                         [Stop] │  ← cancelar request en curso
└────────────────────────────────┘
```

- Tool calls se muestran colapsadas ("▸ tool_name(args)") — expandibles para ver args + resultado.
- Markdown renderizado en respuestas de Claude (usar `QTextBrowser` o `QTextEdit` con HTML).
- Botón **Stop** que cancela el request en vuelo.
- El idioma de la GUI sigue siendo inglés (CLAUDE.md), pero el usuario puede chatear en cualquier idioma.

## Costos y caching

- **Prompt caching**: marcar el system prompt + descripción de tools como `cache_control: ephemeral`. Esto baja drásticamente el costo en conversaciones largas (los tokens cacheados cuestan ~10% del normal).
- **Streaming**: usar streaming para que el usuario vea la respuesta llegar token a token.
- **Modelo por defecto**: `claude-opus-4-7` para máxima capacidad de razonamiento sobre queries; ofrecer `claude-haiku-4-5` para usuarios que prioricen costo.

## Seguridad

- API key nunca en logs ni en exports.
- Confirmación obligatoria para writes y `drop*`.
- Modo "read-only" del chat: opción en settings para deshabilitar tools de escritura por completo.
- No exponer tools que toquen el sistema de archivos o ejecuten comandos.

## Fases sugeridas

1. **MVP read-only**: settings con API key, panel de chat, tools de lectura (`list_*`, `get_*`, `set_query`, `execute_query` solo para reads). Usuario ya puede pedir "explicame estos datos" o "armame un aggregate de X".
2. **Writes con confirmación**: agregar tools de escritura con diálogo de confirmación.
3. **Memoria de conversación por colección**: persistir el historial por `(connection, database, collection)` para que retomar una sesión sea natural.
4. **Slash commands en el chat**: `/explain`, `/optimize`, `/schema`, `/index-suggest` para flujos comunes con un click.

## Dependencias nuevas

- `anthropic>=0.40` (o `claude-agent-sdk` si vamos por ese camino)
- `keyring>=24` (almacenamiento seguro de la API key)

Actualizar `requirements.txt`, `README.md` y `BUILDING.md` según CLAUDE.md.

## Decisiones pendientes

- [ ] ¿Agent SDK o SDK directo? (recomendación: Agent SDK)
- [ ] ¿Panel lateral fijo o `QDockWidget` flotante? (recomendación: dock para que el usuario lo cierre/mueva)
- [ ] ¿Compartir conversación entre tabs/colecciones o una por tab? (recomendación: una por tab, con la colección actual como contexto inicial)
- [ ] ¿Permitir múltiples providers (OpenAI, etc.) o solo Claude? (recomendación: solo Claude por ahora — KISS)
