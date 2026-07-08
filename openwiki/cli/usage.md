# Uso de la CLI

OpenWiki se distribuye como un único binario `openwiki` y está diseñado para funcionar tanto como una aplicación de terminal interactiva como como un ejecutor de documentación de un solo disparo.

## Comandos y modos

Según `src/commands.ts` y `README.md`, los patrones de entrada admitidos son:

- `openwiki` — abre la interfaz de chat interactiva.
- `openwiki "mensaje"` — envía un mensaje de chat inmediatamente y luego permanece abierto.
- `openwiki --init [mensaje]` — genera la documentación inicial de OpenWiki.
- `openwiki --update [mensaje]` — renueva la documentación existente de OpenWiki.
- `openwiki -p, --print` — ejecuta una vez e imprime la salida final del asistente (no interactivo).
- `openwiki --modelId <id>` / `--model-id <id>` — elige un ID de modelo para la ejecución.
- `openwiki --help` / `-h` — imprime el uso, las opciones y los ejemplos.
- `openwiki --dry-run` — opción solo de desarrollo que evita invocar al agente.

El analizador rechaza combinaciones incompatibles como `--init` y `--update` juntos, y requiere un mensaje o un comando cuando se usa `--print`.

### Auto-salida para init/update

Cuando `--init` o `--update` se ejecutan en una TTY (sin `--print`), la CLI inicia la ejecución, transmite la salida del agente y **se cierra automáticamente al successar** (`shouldAutoExitStartupRun` en `src/cli.tsx`). Esto significa que `openwiki --init` se comporta como un comando de un solo disparo mientras sigue mostrando una interfaz en vivo. Las ejecuciones de chat y las de `--print` no se ven afectadas — el chat permanece abierto para mensajes de seguimiento, y `--print` escribe a stdout y termina.

### Modo no interactivo

Si stdin no es una TTY (por ejemplo, CI), o se usa `--print`, la CLI requiere que una clave de API del proveedor ya esté guardada en `~/.openwiki/.env` o presente en el entorno. Mostrará un error con un mensaje claro si la clave falta, en lugar de pedirlo interactivamente.

## Comportamiento interactivo

`src/cli.tsx` es la cáscara de la aplicación basada en Ink. Maneja:

- el envío de chat y mensajes de seguimiento,
- los lanzamientos de comandos `init` / `update` (incluyendo desde los comandos slash `/init` y `/update`),
- la selección de proveedor y modelo durante la sesión (`/provider`, `/model`),
- la configuración interactiva de credenciales cuando es necesaria (incluyendo para init/update, no solo chat),
- la transmisión de texto del agente y eventos de herramientas,
- el historial de ejecuciones completadas y la visualización de errores,
- el manejo de salida para ayuda, errores y mensajes explícitos de `/exit`.

La interfaz persiste la selección de proveedor y modelo de vuelta a `~/.openwiki/.env` a través de `saveOpenWikiEnv()`.

## Credenciales y configuración inicial

La primera ejecución interactiva puede solicitar:

- un **proveedor** (`OPENWIKI_PROVIDER`) — openrouter, baseten, fireworks, openai, openai-compatible o anthropic,
- la **clave de API del proveedor** (por ejemplo `OPENROUTER_API_KEY`, `OPENAI_API_KEY`, `OPENAI_COMPATIBLE_API_KEY`, `ANTHROPIC_API_KEY`, `BASETEN_API_KEY`, `FIREWORKS_API_KEY`),
- una **URL base** para los proveedores que la requieren (el proveedor openai-compatible pide `OPENAI_COMPATIBLE_BASE_URL`),
- un **ID de modelo** almacenado como `OPENWIKI_MODEL_ID` — elegido de la lista de modelos del proveedor o un ID personalizado,
- opcionalmente `LANGSMITH_API_KEY` para trazado.

Si se proporciona una clave de LangSmith, la configuración inicial también habilita `LANGCHAIN_PROJECT=openwiki` y `LANGCHAIN_TRACING_V2=true`.

`src/credentials.tsx` determina si se necesita configuración y guía al usuario a través de los valores faltantes usando menús de selección con teclas de flecha para proveedor y modelo. Consulte [Credenciales y actualizaciones](../operations/credentials-and-updates.md) para más detalles.

## Selección de proveedor y modelo

Los proveedores y sus opciones de modelo se definen en `PROVIDER_CONFIGS` en `src/constants.ts`:

| Proveedor          | Clave de entorno            | URL base                                | Modelos                                                                |
| ------------------ | --------------------------- | --------------------------------------- | --------------------------------------------------------------------- |
| openrouter         | `OPENROUTER_API_KEY`        | `https://openrouter.ai/api/v1`          | GLM 5.2, Fusion, Kimi K2.7 Code, Claude Opus/Sonnet, GPT 5.4 mini/5.5 |
| baseten            | `BASETEN_API_KEY`           | `https://inference.baseten.co/v1`       | GLM 5.2, Kimi K2.7 Code                                               |
| fireworks          | `FIREWORKS_API_KEY`         | `https://api.fireworks.ai/inference/v1` | GLM 5.2, Kimi K2.7 Code                                               |
| openai             | `OPENAI_API_KEY`            | (por defecto)                           | GPT 5.4 mini, GPT 5.5                                                 |
| openai-compatible  | `OPENAI_COMPATIBLE_API_KEY` | `OPENAI_COMPATIBLE_BASE_URL` (obligatoria) | solo ID de modelo personalizado                                       |
| anthropic          | `ANTHROPIC_API_KEY`         | (por defecto, o `ANTHROPIC_BASE_URL`)   | Haiku, Sonnet, Opus                                                   |

El proveedor por defecto es `openrouter`. `resolveConfiguredProvider()` elige el proveedor a partir de `OPENWIKI_PROVIDER`, recurriendo a openrouter si `OPENROUTER_API_KEY` está establecido, y luego a `DEFAULT_PROVIDER`.

### URLs base alternativas

Establezca `ANTHROPIC_BASE_URL` para enrutar el proveedor anthropic hacia un endpoint alternativo compatible con Anthropic (por ejemplo, un gateway auto-alojado o proxied) en lugar de la API por defecto. Cuando está establecido, se pasa a `ChatAnthropic` como `anthropicApiUrl`; la `ANTHROPIC_API_KEY` sigue enviándose como credencial de la petición.

### Proveedor OpenAI-compatible

El proveedor `openai-compatible` apunta a cualquier endpoint de chat-completions compatible con OpenAI. No tiene un endpoint por defecto, por lo que `OPENAI_COMPATIBLE_BASE_URL` es **obligatoria** (la configuración interactiva la solicita, y una ejecución se aborta temprano si falta). Esto es útil para endpoints LLM compatibles con OpenAI como los expuestos por un gateway LiteLLM, que permite alcanzar cualquier proveedor subyacente a través de una única API con forma OpenAI.
Como el proveedor no tiene una lista de modelos preestablecida, establezca `OPENWIKI_MODEL_ID` (o elija "ID de modelo personalizado" en la configuración) al nombre que el gateway exponga.

```bash
OPENWIKI_PROVIDER=openai-compatible
OPENAI_COMPATIBLE_API_KEY=<clave del gateway>
OPENAI_COMPATIBLE_BASE_URL=https://<gateway>/v1
OPENWIKI_MODEL_ID=<nombre del modelo que expone el gateway>
```

Las URLs base se resuelven mediante `resolveProviderBaseUrl()` en `src/constants.ts`, que prefiere la variable de entorno `baseUrlEnvKey` del proveedor sobre el valor por defecto integrado.
