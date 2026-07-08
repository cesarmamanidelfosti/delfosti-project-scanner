# Guía de inicio rápido de OpenWiki

OpenWiki es una CLI escrita en TypeScript que genera y mantiene la documentación de un repositorio mediante un flujo impulsado por un agente. El paquete expone un único binario `openwiki`, almacena las credenciales locales en `~/.openwiki/.env` y registra los metadatos de cada actualización exitosa en `openwiki/.last-update.json`.

## Qué hace este repositorio

- Inicia una aplicación de terminal interactiva basada en Ink para conversar con el agente de OpenWiki.
- Admite ejecuciones puntuales de documentación con `--init`, `--update` y `--print`.
- Compatible con múltiples proveedores de modelos — OpenRouter (por defecto), Anthropic, OpenAI, Baseten y Fireworks — cada uno con su propia clave de API y lista de modelos.
- Usa un backend de shell local de DeepAgents con rutas virtuales de sistema de archivos enraizadas en el repositorio objetivo.
- Crea o renueva la documentación dentro del directorio `openwiki/` del repositorio objetivo.
- Se cierra automáticamente tras una ejecución exitosa de `--init` o `--update` en una terminal interactiva, de modo que la CLI funciona tanto en modo interactivo como en modo de un solo disparo.
- Permite opcionalmente programar actualizaciones automatizadas mediante GitHub Actions o GitLab CI.

## Puntos de partida

- [Descripción de la arquitectura](./architecture/overview.md) — estructura de ejecución, módulos principales y flujo de procesamiento.
- [Uso de la CLI](./cli/usage.md) — comandos, opciones, selección de modelo/proveedor y configuración inicial de credenciales.
- [Flujo de trabajo del agente](./agent/workflow.md) — cómo se ensamblan y persisten las ejecuciones de documentación.
- [Credenciales y actualizaciones](./operations/credentials-and-updates.md) — almacenamiento local de entorno, metadatos y actualizaciones programadas.

## Archivos fuente clave

- `README.md` — resumen de instalación y uso orientado al usuario.
- `package.json` — punto de entrada binario, scripts y dependencias.
- `src/cli.tsx` — interfaz de usuario en Ink, ejecución de comandos, auto-salida y ciclo de vida de las ejecuciones.
- `src/commands.ts` — análisis de la línea de comandos y contenido de la ayuda.
- `src/agent/index.ts` — tiempo de ejecución del agente, creación de modelos específica por proveedor, respaldo y escritura de metadatos.
- `src/agent/prompt.ts` — ensamblaje del prompt, instrucciones de las ejecuciones de documentación y reglas de inserción en AGENTS.md/CLAUDE.md.
- `src/agent/utils.ts` — recopilación de evidencia de Git, instantánea de contenido y manejo de `.last-update.json`.
- `src/agent/types.ts` — tipos compartidos del agente (`OpenWikiCommand`, `RunContext`, `UpdateMetadata`, opciones/eventos de ejecución).
- `src/env.ts` — persistencia de `~/.openwiki/.env` y diagnósticos de credenciales.
- `src/credentials.tsx` — flujo interactivo de configuración inicial para selección de proveedor, claves de API y selección de modelo.
- `src/constants.ts` — configuraciones de proveedores, opciones de modelos, claves de entorno y funciones de validación.
- `examples/openwiki-update.yml` — ejemplo de automatización programada con GitHub Actions.
- `examples/openwiki-update.gitlab-ci.yml` — ejemplo de automatización programada con GitLab CI.

## Mapa de documentación

- [Arquitectura](./architecture/overview.md)
- [CLI](./cli/usage.md)
- [Agente](./agent/workflow.md)
- [Operaciones](./operations/credentials-and-updates.md)

## Notas para agentes futuros

- El repositorio es intencionalmente enfocado: la superficie principal del producto es la CLI más el agente generador de documentación.
- Trate el directorio `openwiki/` de este repositorio como salida de documentación generada por una ejecución futura de OpenWiki, no como código fuente de la aplicación.
- Al modificar el comportamiento, verifique tanto el analizador de la CLI como el prompt/tiempo de ejecución del agente, porque la semántica visible para el usuario está dividida entre `src/commands.ts`, `src/cli.tsx` y `src/agent/*`.
- El soporte de proveedores está centralizado en `src/constants.ts`. Agregar o cambiar un proveedor implica actualizar `PROVIDER_CONFIGS`, el tipo `OpenWikiProvider` y la rama de creación de modelo en `src/agent/index.ts`.

## Mapa de fuentes

- `README.md`
- `package.json`
- `src/cli.tsx`
- `src/commands.ts`
- `src/agent/index.ts`
- `src/agent/prompt.ts`
- `src/agent/utils.ts`
- `src/agent/types.ts`
- `src/env.ts`
- `src/credentials.tsx`
- `src/constants.ts`
- `examples/openwiki-update.yml`
- `examples/openwiki-update.gitlab-ci.yml`
