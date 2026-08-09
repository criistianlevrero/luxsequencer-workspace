> **Estado**: PLANEADO · **Última verificación**: 2026-08-07 · **Alcance**: ecosistema
>
> Reservado para una conversación aparte, por decisión del autor (2026-08-07).

# Bloqueantes del modelo de distribución

Tres bloqueantes verificados contra el [modelo de distribución objetivo](../auditoria/2026-08-06-fase-1-monorepo.md).
Ninguno es de topología de repos.

## 1. "Core abierto" no es cierto legalmente hoy

- `luxsequencer-core/LICENSE` es un archivo de **0 bytes**, commiteado así.
- `core-renderers` no tiene archivo LICENSE ni campo `license` en `package.json`.

Sin licencia explícita el default es *todos los derechos reservados*: nadie puede forkear ni
redistribuir legalmente.

`contracts` y `lux-ui` sí quedaron resueltos (MIT con archivo `LICENSE`) al publicarlos.

## 2. "Cloud cerrado" hoy no protege nada

`luxsequencer-cloud` es una SPA que habla directo con Supabase desde el browser. La lógica de
negocio se ejecuta en el navegador del usuario y se descarga en cada visita. Cerrar el repo
oculta el historial, no el código.

Detalle en el `STATUS.md` de cloud.

## 3. La aplicación de licencias está rota en ambos extremos

- **Lado core**: `isMarketplaceLicenseTokenValid()` decodifica el payload del JWT y valida
  `pluginKey`/`exp`/`iat`, pero **nunca verifica la firma**. Un token forjado a mano pasa.
- **Lado cloud**: firma con el secreto que reciba por parámetro; si no recibe ninguno usa un
  literal inseguro. El único llamador que inyecta un secreto real es un plugin del dev server.

**Problema estructural, más allá de los dos bugs**: no se puede hacer cumplir una licencia paga
con una validación dentro de un cliente open source — cualquiera forkea core y borra el chequeo.

La aplicación tiene que estar en la **entrega** (cloud sirve los bytes del plugin sólo a cuentas
con derecho), no en la ejecución. Eso cambia qué es un renderer pago: no un repo que se clona,
sino un artefacto que cloud entrega. Ver
[decisión de distribución de renderers](../decisiones/2026-08-06-distribucion-renderers.md).
