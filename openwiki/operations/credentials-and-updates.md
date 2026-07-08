# Credenciales y actualizaciones

OpenWiki tiene dos preocupaciones operativas que importan tanto para usuarios como para mantenedores:

1. el almacenamiento local de credenciales en `~/.openwiki/.env`, y
2. los metadatos de actualización persistidos en `openwiki/.last-update.json`.

También incluye ejemplos de flujos de trabajo para GitHub Actions y GitLab CI para actualizaciones programadas.

## Almacenamiento local de credenciales

`src/env.ts` gestiona un archivo de entorno privado bajo el directorio home del usuario:

- directorio: `~/.openwiki` (modo `0o700`)
- archivo: `~/.openwiki/.env` (modo `0o600`)

El archivo almacena la configuración del proveedor y las claves de API:

- `OPENWIKI_PROVIDER` — el proveedor de modelo seleccionado
- `OPENWIKI_MODEL_ID` — el ID de modelo por defecto
- Claves de API de proveedores: `OPENROUTER_API_KEY`, `OPENAI_API_KEY`, `OPENAI_COMPATIBLE_API_KEY`, `ANTHROPIC_API_KEY`, `BASETEN_API_KEY`, `FIREWORKS_API_KEY`
- URLs base: `ANTHROPIC_BASE_URL` (opcional — enruta el proveedor anthropic a un endpoint compatible con Anthropic distinto de la API por defecto) y `OPENAI_COMPATIBLE_BASE_URL` (obligatoria para el proveedor openai-compatible, que no tiene endpoint por defecto)
- Configuración opcional de LangSmith: `LANGSMITH_API_KEY`, `LANGCHAIN_PROJECT`, `LANGCHAIN_TRACING_V2`

El cargador fusiona esos valores en `process.env`, dando prioridad a los valores existentes a nivel de proceso sobre los del archivo. Las claves obsoletas (`OPENAI_BASE_URL`, `OPENAI_ORG_ID`, `OPENAI_PROJECT`) se omiten al cargar y se eliminan al guardar.

`src/credentials.tsx` proporciona el flujo interactivo de configuración inicial cuando es necesario:

- solicita un proveedor (menú de selección con teclas de flecha),
- solicita la clave de API del proveedor,
- solicita una elección de modelo (menú de selección de la lista de modelos del proveedor, o un ID de modelo personalizado),
- opcionalmente solicita una clave de LangSmith,
- escribe los resultados con permisos de archivo restrictivos,
- elimina las variables de entorno obsoletas relacionadas con OpenAI al guardar.

El flujo de configuración se ejecuta para **todos** los comandos interactivos (chat, init y update) cuando faltan credenciales — no solo para chat. En modo no interactivo (sin TTY o con `--print`), las claves de proveedor faltantes producen un error en lugar de una solicitud.

## Resolución del proveedor

`resolveConfiguredProvider()` en `src/constants.ts` determina el proveedor activo:

1. Si `OPENWIKI_PROVIDER` está establecido y es válido, se usa.
2. De lo contrario, si `OPENROUTER_API_KEY` está presente, se usa por defecto `openrouter`.
3. De lo contrario, se recurre a `DEFAULT_PROVIDER` (`openrouter`).

`needsCredentialSetup()` en `src/credentials.tsx` verifica si la variable de entorno del proveedor es válida y si la clave de API del proveedor, un ID de modelo (salvo que se haya establecido explícitamente) y una clave de LangSmith están presentes. Cualquier valor de proveedor faltante o inválido desencadena el flujo interactivo.

## Diagnósticos de modelo y credenciales

La capa de entorno también produce diagnósticos para la interfaz de la CLI. Esos diagnósticos reportan:

- de dónde proviene cada credencial (`process.env`, `~/.openwiki/.env`, ambos, o `unset`),
- si el valor no está establecido,
- la longitud aparente,
- una vista previa enmascarada,
- advertencias por formato sospechoso como espacios, nuevas líneas, comillas o sufijos entre corchetes,
- IDs de modelo inválidos,
- valores de proveedor inválidos.

Los diagnósticos cubren las seis claves de proveedor más `OPENWIKI_PROVIDER`, `OPENWIKI_MODEL_ID`, las URLs base (`ANTHROPIC_BASE_URL`, `OPENAI_COMPATIBLE_BASE_URL`) y `LANGSMITH_API_KEY`. Esto facilita el diagnóstico de problemas de arranque sin exponer valores secretos (los valores no secretos como el proveedor, el ID de modelo y las URLs base se muestran completos).

## Metadatos de actualización

Tras ejecuciones exitosas de `init` o `update` donde el contenido de `openwiki/` cambió, `src/agent/utils.ts` escribe `openwiki/.last-update.json` con:

- `updatedAt`
- `command`
- `gitHead`
- `model`

La verificación de cambio de contenido usa `createOpenWikiContentSnapshot()`, que aplica un hash al directorio `openwiki/` (excluyendo `.last-update.json`). Si el hash es idéntico antes y después de la ejecución, los metadatos no se escriben. Esto evita que los bucles de actualización programados actualicen la marca de tiempo cuando no hubo cambios en la documentación.

Las ejecuciones de update usan estos metadatos para construir un resumen de cambios desde la ejecución exitosa anterior de OpenWiki — prefiriendo `gitHead` para un rango de commits preciso, recurriendo a `updatedAt` para un rango basado en tiempo.

## Flujos de trabajo de CI programados

El repositorio incluye `examples/openwiki-update.yml` como un flujo de trabajo de GitHub Actions para actualizaciones programadas. Este:

- se ejecuta en un horario (diariamente a las 08:00 UTC) y mediante dispatch manual,
- hace checkout del repositorio,
- instala Node.js 22,
- instala OpenWiki globalmente,
- ejecuta `openwiki --update --print`,
- pasa `OPENROUTER_API_KEY`, `OPENWIKI_MODEL_ID` y `LANGSMITH_API_KEY` desde los secretos de GitHub,
- abre un pull request con `peter-evans/create-pull-request` limitado al directorio `openwiki`.

El flujo de trabajo es una buena referencia para mantenimiento automatizado. El repositorio también contiene un flujo de trabajo `checks.yml` para CI (verificaciones de lint/formato).

El repositorio también incluye `examples/openwiki-update.gitlab-ci.yml` como un job de GitLab CI para actualizaciones programadas. Este:

- se ejecuta desde un pipeline programado o un pipeline web activado manualmente,
- instala OpenWiki globalmente en un contenedor de Node.js 22,
- ejecuta `openwiki --update --print`,
- omite el resto del job cuando `openwiki/` no cambió,
- confirma los cambios en una rama generada `openwiki/update-$CI_PIPELINE_ID`,
- empuja esa rama de vuelta al proyecto de GitLab, y
- crea un merge request apuntando a la rama por defecto del proyecto a través de la API de GitLab.

Los usuarios de GitLab deben configurar variables CI/CD protegidas para la clave del proveedor de modelo, por ejemplo `OPENROUTER_API_KEY`, y `OPENWIKI_GITLAB_TOKEN`. El token de GitLab necesita permiso para empujar una rama y crear merge requests en el proyecto objetivo.
