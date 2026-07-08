# Flujo de trabajo del agente

El agente de documentación está implementado en `src/agent/`. Toma un comando (`chat`, `init`, `update` o `review`), reúne el contexto del repositorio, construye los prompts, ejecuta una sesión de DeepAgents y registra los metadatos de actualización exitosa — pero solo si el contenido de la documentación cambió realmente.

## Flujo principal

`src/agent/index.ts` sigue esta secuencia para las ejecuciones que no son de chat:

1. Cargar `~/.openwiki/.env` en `process.env`.
2. Resolver el proveedor mediante `resolveConfiguredProvider()` y asegurar que la clave de API del proveedor exista.
3. Resolver el ID de modelo desde la entrada de la CLI, `OPENWIKI_MODEL_ID` o el modelo por defecto del proveedor.
4. Crear un contexto de ejecución a partir del estado de Git y los metadatos de actualización previos.
5. Tomar una instantánea del hash de contenido actual de `openwiki/` (antes de la ejecución).
6. Construir el prompt de sistema y el prompt de usuario.
7. Crear el cliente de modelo específico del proveedor (`ChatAnthropic`, `ChatOpenRouter` o `ChatOpenAI`).
8. Crear un `LocalShellBackend` de DeepAgents enraizado en el repositorio con un punto de control SQLite.
9. Transmitir mensajes y eventos de herramientas de vuelta a la CLI.
10. Para `init` y `update`, comparar la instantánea de contenido posterior con la anterior. Escribir `openwiki/.last-update.json` **solo si el contenido cambió**.

Las ejecuciones de chat omiten la escritura de metadatos por completo.

## Creación de modelo específica por proveedor

`createModel()` en `src/agent/index.ts` se ramifica según el proveedor:

- **anthropic**: `new ChatAnthropic(modelId, { apiKey, anthropicApiUrl? })` — usa `@langchain/anthropic` directamente. Cuando `ANTHROPIC_BASE_URL` está establecido, la URL base alternativa resuelta se pasa como `anthropicApiUrl` para que las peticiones puedan enrutarse a un endpoint compatible con Anthropic auto-alojado o proxied en lugar de la API por defecto.
- **openrouter**: `new ChatOpenRouter({ apiKey, baseURL, model, siteName: "OpenWiki" })` — usa el modelo de OpenRouter seleccionado directamente.
- **openai**: `new ChatOpenAI({ apiKey, model, useResponsesApi: true })` — usa la API de Responses de OpenAI para llamadas oficiales a OpenAI.
- **baseten / fireworks / openai-compatible**: `new ChatOpenAI({ apiKey, configuration: { baseURL? }, model })` — clientes compatibles con OpenAI que usan la URL base del proveedor cuando está configurada. El proveedor `openai-compatible` no tiene endpoint por defecto; su URL base la suministra el usuario mediante `OPENAI_COMPATIBLE_BASE_URL` y es obligatoria (`requiresBaseUrl: true`), lo que permite a OpenWiki apuntar a cualquier gateway compatible con OpenAI (por ejemplo, un gateway LiteLLM que fronta proveedores subyacentes).

Las URLs base se resuelven a través de `resolveProviderBaseUrl()` en `src/constants.ts`, que prefiere la variable de entorno de URL base alternativa del proveedor (`baseUrlEnvKey`) sobre el valor por defecto integrado antes de recurrir al endpoint por defecto del propio SDK. Los proveedores marcados con `requiresBaseUrl` son validados al arrancar por `ensureProviderBaseUrl()`.

## Estrategia de prompting

`src/agent/prompt.ts` codifica las reglas del producto directamente en el prompt de sistema. Se instruye al agente para que:

- inspeccione el código base actual y escriba documentación bajo `openwiki/`,
- use herramientas de descubrimiento de sistema de archivos e historial de Git en lugar de inventar hechos,
- mantenga el wiki inicial enfocado y navegable,
- evite páginas delgadas o escasas — fusionar esbozos en páginas más amplias en lugar de crear muchos directorios pequeños,
- documente el repositorio tanto para humanos como para agentes futuros,
- respete la raíz del repositorio como el único proyecto en alcance,
- evite leer secretos o archivos `.env`,
- use el historial de Git para las ejecuciones de init y update,
- respete el archivo de plan temporal y los requisitos de metadatos de actualización,
- asegure que los archivos `/AGENTS.md` y/o `/CLAUDE.md` de nivel superior referencien la guía rápida de OpenWiki (insertando o refrescando una sección estandarizada).

El prompt de usuario cambia según el comando:

- `init` incluye el resumen actual de Git y pide documentación nueva.
- `update` incluye los metadatos de la última actualización y un resumen de cambios de Git.
- `chat` simplemente reenvía el mensaje del usuario.
- `review` pide un resumen contextual de solo lectura sin escribir archivos.

## Evidencia de Git y metadatos de actualización

`src/agent/utils.ts` es responsable de la evidencia del repositorio que el prompt ve:

- el estado actual del árbol de trabajo,
- el HEAD actual,
- una ventana de cambios desde la última actualización exitosa cuando `.last-update.json` incluye un `gitHead` o `updatedAt`,
- los 20 commits más recientes con archivos modificados para ejecuciones de init (o actualizaciones sin metadatos previos),
- un resumen de diff contra HEAD.

En ejecuciones exitosas de init/update donde el contenido cambió, el agente escribe metadatos JSON con:

- `updatedAt`
- `command`
- `gitHead`
- `model`

Esos metadatos se usan posteriormente para acotar las ejecuciones de update.

### Instantánea de contenido

`createOpenWikiContentSnapshot()` calcula un hash SHA-256 de todo el árbol del directorio `openwiki/` (excluyendo `.last-update.json`). El tiempo de ejecución del agente toma una instantánea antes y después de la ejecución. Si coinciden — lo que significa que el modelo no hizo cambios en la documentación — el archivo de metadatos no se actualiza. Esto evita que los bucles de actualización programados recambien los metadatos cuando el wiki ya está actualizado.

## Errores de modelo

El tiempo de ejecución del agente usa solo el proveedor y modelo seleccionados para una ejecución. Si esa petición falla, OpenWiki muestra el error del proveedor y se detiene en lugar de reintentar con otro modelo.

## Por qué esto importa

El agente no es solo un envoltorio de chat genérico. Está intencionalmente restringido para que pueda:

- escribir documentación local del repositorio sin salirse del repo,
- preservar la continuidad entre ejecuciones mediante puntos de control y metadatos,
- mantener las actualizaciones basadas en evidencia de Git,
- evitar el recambio de metadatos mediante la verificación de instantánea de contenido,
- soportar tanto casos de uso interactivos como de mantenimiento programado.

## Aspectos a vigilar al cambiar el comportamiento del agente

- Mantenga el prompt sincronizado con las herramientas de sistema de archivos y las convenciones de ruta reales usadas por la CLI.
- Tenga cuidado con la semántica de `.last-update.json`, porque las ejecuciones de update lo usan para decidir qué cambió desde la ejecución exitosa anterior.
- La verificación de instantánea de contenido significa que una actualización nula no actualizará los metadatos. Si cambia la lógica de instantánea, asegúrese de que `.last-update.json` siga excluido.
- La carga de credenciales ocurre antes de la resolución del modelo; los cambios ahí afectan tanto la configuración inicial como el arranque del agente.
- Al agregar un proveedor, añada una rama en `createModel()` y asegúrese de que la clave de entorno de la API se verifique en `ensureProviderKey()`.
- El backend de DeepAgents está configurado con `virtualMode: true`, lo cual es importante para el comportamiento exclusivamente documental.
