> 🚚 **PENDIENTE DE MUDANZA** — Este documento **pertenece a `luxsequencer-core`** y debe moverse
> a `luxsequencer-core/docs/decisiones/` en la próxima actualización de la base de conocimientos.
> Está acá sólo porque ese repo todavía no tiene la carpeta. Ver `STATUS.md` de la raíz.
>
> **Estado**: VIGENTE · **Fecha**: 2026-08-06 · **Alcance**: `luxsequencer-core` + `core-renderers`
>
> Si esta decisión cambia, se escribe un archivo nuevo que la supersede. **No se edita esta.**

# Flag de desarrollo de renderers: selección de origen, no desactivación de control

`luxsequencer-core` va a llamar a la API de cloud, con un flag para que un autor de renderers
trabaje sin cuenta en la nube. El flag es **imprescindible**, pero se diseña al revés de como
está hoy:

- ❌ **Mal**: el flag desactiva la validación de licencia. En una app open source, el flag *es*
  el bypass — cualquiera lo pone en `false` y buildea.
- ✅ **Bien**: el flag cambia **de dónde vienen** los renderers. En dev vienen del
  `core-renderers` local (que es lo que ya hace el proxy al 4174). Los renderers pagos
  simplemente no están disponibles: no hay chequeo que saltear porque el artefacto nunca llega.

Consecuencia: **`VITE_MARKETPLACE_ENFORCE_LICENSE_TOKENS` debe eliminarse.** Es un bypass del
tipo malo, y hoy además no hace nada porque `HARDCODED_EXTERNAL_RENDERERS` está vacío.
