# Descripción de la arquitectura

OpenWiki tiene una arquitectura pequeña pero en capas:

1. `src/cli.tsx` proporciona la aplicación de terminal interactiva y orquesta las ejecuciones, incluyendo la auto-salida para init/update.
2. `src/commands.ts` analiza argv y define el texto de ayuda y las opciones admitidas.
3. `src/credentials.tsx` gestiona la configuración inicial interactiva para selección de proveedor, claves de API, selección de modelo y trazado opcional con LangSmith.
4. `src/env.ts` lee y escribe `~/.openwiki/.env` y muestra diagnósticos de credenciales para todos los proveedores admitidos.
5. `src/agent/index.ts` ejecuta el agente de documentación, resuelve el proveedor, crea el cliente de modelo apropiado, recopila el contexto de Git y escribe los metadatos de actualización.
6. `src/agent/prompt.ts` construye los prompts de sistema y de usuario que indican al modelo cómo comportarse.
7. `src/agent/utils.ts` reúne la evidencia de Git, calcula una instantánea del contenido de OpenWiki y registra `.last-update.json` tras las ejecuciones exitosas de init/update.
8. `src/constants.ts` centraliza las configuraciones de proveedores, las opciones de modelos, las claves de entorno, las funciones de validación y los nombres de los directorios del wiki.
9. `src/agent/types.ts` define los tipos compartidos: `OpenWikiCommand`, `RunContext`, `UpdateMetadata` y las interfaces de opciones/eventos de ejecución.

## Estructura en tiempo de ejecución

La CLI arranca en `src/cli.tsx`, analiza el comando y luego:

- imprime la ayuda y termina,
- abre la interfaz de chat interactiva,
- ejecuta un comando init/update contra el repositorio actual, o
- realiza una ejecución en seco (dry-run) en modo desarrollo.

Para las ejecuciones que no son de chat, el agente recibe un `RunContext` que incluye los metadatos de la última actualización y un resumen de Git generado a partir de:

- `git status --short`
- `git rev-parse HEAD`
- `git log --max-count=20 --name-status --oneline` (init, o update sin metadatos previos)
- `git log <lastHead>..HEAD --name-status --oneline` (update con un `gitHead` registrado)
- `git log --since <updatedAt> --name-status --oneline` (update con solo marca de tiempo)
- `git diff --name-status HEAD`

### Resolución de proveedor y modelo

El tiempo de ejecución del agente resuelve el proveedor mediante `resolveConfiguredProvider()` en `src/constants.ts`:

1. Si `OPENWIKI_PROVIDER` está establecido y es válido, se usa.
2. De lo contrario, si `OPENROUTER_API_KEY` está presente, se usa por defecto `openrouter`.
3. De lo contrario, se recurre a `DEFAULT_PROVIDER` (`openrouter`).

La creación del modelo se ramifica según el proveedor en `src/agent/index.ts` (`createModel`):

- **anthropic** → `ChatAnthropic` con la clave de API de Anthropic.
- **openrouter** → `ChatOpenRouter` con el ID de modelo seleccionado.
- **openai** → `ChatOpenAI` con `useResponsesApi: true`.
- **baseten / fireworks / openai-compatible** → `ChatOpenAI` con la clave de API del proveedor y un `baseURL` personalizado opcional proveniente de `PROVIDER_CONFIGS`.

### Backend de DeepAgents

El agente usa un `LocalShellBackend` de DeepAgents enraizado en el repositorio, configurado con `virtualMode: true`, `maxOutputBytes: 100_000` y un tiempo de espera de 120 segundos. Un punto de control SQLite (`~/.openwiki/openwiki.sqlite`) persiste los hilos de conversación identificados por un hash de la ruta del repositorio.

### Instantánea de contenido y escritura de metadatos

Tras completar una ejecución que no es de chat, `src/agent/utils.ts` calcula una instantánea SHA-256 del directorio `openwiki/` (excluyendo `.last-update.json`). Los metadatos se escriben **solo si la instantánea cambió** — una actualización nula que deja la documentación intacta no actualizará `.last-update.json`. Esto evita bucles infinitos de actualización en los flujos de trabajo programados.

### Comportamiento de auto-salida

`shouldAutoExitStartupRun()` en `src/cli.tsx` determina si una ejecución de inicio debe salir automáticamente al completarse con éxito. Esto aplica a los comandos `--init` y `--update` (sin `--print`) cuando se ejecutan en una TTY: la CLI lanza la ejecución, muestra la salida en streaming y termina con código 0 al successar. Las ejecuciones de chat y las de `--print` no se ven afectadas.

## Por qué la arquitectura tiene esta forma

El diseño actual refleja un producto de documentación más que un marco de agentes de propósito general:

- La CLI es propietaria de la experiencia del usuario y la configuración inicial de credenciales para que la herramienta sea lista para instalar y ejecutar.
- La evidencia de Git se recopila en el proceso anfitrión antes de que el agente arranque, de modo que el modelo vea un contexto estable del repositorio.
- El soporte de proveedores está centralizado en `src/constants.ts` para que agregar un proveedor sea un cambio de configuración único más una rama de creación de modelo.
- La ejecución del modelo es de fallo rápido: si la petición del proveedor/modelo seleccionado falla, OpenWiki muestra ese error en lugar de continuar con otro modelo.
- La verificación de instantánea de contenido evita el recambio de metadatos cuando una actualización no produce cambios en la documentación, lo cual es importante para los flujos de trabajo programados en CI.
- La auto-salida para init/update hace que la CLI sea utilizable tanto en contextos interactivos como de un solo disparo sin requerir `--print`.

## Principales puntos de extensión

- Agregar o refinar comandos de la CLI en `src/commands.ts` y el comportamiento correspondiente de la interfaz en `src/cli.tsx`.
- Cambiar la configuración inicial o el almacenamiento local de credenciales en `src/credentials.tsx` y `src/env.ts`.
- Agregar un nuevo proveedor de modelo extendiendo `PROVIDER_CONFIGS` y `OpenWikiProvider` en `src/constants.ts`, y luego añadiendo una rama en `createModel` en `src/agent/index.ts`.
- Ajustar los modelos por defecto, la validación o las listas de respaldo en `src/constants.ts`.
- Extender el prompt de documentación o la evidencia de Git en `src/agent/prompt.ts` y `src/agent/utils.ts`.
- Modificar la persistencia de ejecuciones o el comportamiento de la instantánea en `src/agent/utils.ts`.

## Aspectos a vigilar al editar

- `src/cli.tsx` y `src/commands.ts` deben mantenerse alineados; el texto de ayuda y el comportamiento del analizador están acoplados intencionalmente.
- La configuración de credenciales escribe en un archivo real del directorio home, por lo que el manejo de permisos es importante.
- Se espera que el agente funcione con rutas virtuales locales al repositorio como `/README.md` y `/openwiki/quickstart.md`; el prompt lo advierte explícitamente.
- `openwiki/` en el repositorio objetivo es tanto la ubicación de salida de la documentación como la ubicación de metadatos para `.last-update.json`.
- Al agregar un proveedor, actualice `managedEnvKeys` en `src/env.ts` para que los diagnósticos y el formato de entorno cubran la nueva clave.
- La lógica de instantánea de contenido excluye `.last-update.json`; si se agregan nuevos archivos de metadatos bajo `openwiki/`, decida si también deben excluirse.

## Mapa de fuentes

- `src/cli.tsx`
- `src/commands.ts`
- `src/credentials.tsx`
- `src/env.ts`
- `src/agent/index.ts`
- `src/agent/prompt.ts`
- `src/agent/utils.ts`
- `src/agent/types.ts`
- `src/constants.ts`
- `package.json`
