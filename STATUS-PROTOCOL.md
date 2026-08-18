# Protocolo de estado — plan vs. en proceso vs. hecho

Mecanismo unificado para todos los proyectos de LuxSequencer. Declarado acá, implementado en
cada repo.

**Problema que resuelve**: la documentación del ecosistema mezclaba especificación con
descripción. El README de cloud tenía 510 líneas describiendo Stripe, ledger de ingresos y un
stack Next.js que nunca existió, en presente y sin marcar. El plan de migración de `lux-ui` tenía
tareas hechas marcadas como pendientes. Nada obligaba a la documentación a seguir al código.

## Los cuatro archivos

| Archivo | Contiene | Regla |
|---|---|---|
| `README.md` | **Sólo lo que existe hoy** | Si no lo podés señalar en el código, no va acá |
| `STATUS.md` | Tabla índice de capacidades con su estado | Uno por repo, en la raíz del repo |
| `docs/next-steps/*.md` | El detalle de cada cosa planeada | Un archivo por tema |
| `docs/archive/*.md` | Lo descartado o superado | No se borra: se archiva |

## Vocabulario

| Estado | Significado |
|---|---|
| `IMPLEMENTADO` | Existe, funciona y se puede señalar en el código |
| `PARCIAL` | Existe código, pero no cumple todavía lo que promete el nombre |
| `PLANEADO` | Decidido, sin construir |
| `DESCARTADO` | Se evaluó o se empezó, y se abandonó. Se registra para no volver a proponerlo |

## La regla de evidencia

**Toda fila `IMPLEMENTADO` o `PARCIAL` tiene que citar al menos un archivo.**
Si no se puede citar, no está implementado — se marca `PLANEADO`.

Es lo que le da filo al mecanismo: convierte cada afirmación en algo falsable en segundos, en vez
de una promesa que nadie puede auditar sin leer todo el código.

`PLANEADO` lleva `—` en evidencia. `DESCARTADO` cita el código muerto, si quedó alguno.

Un caso frecuente que conviene marcar bien: **módulo huérfano** — el archivo existe pero nadie lo
importa. Es `PARCIAL`, no `IMPLEMENTADO`, y la nota debe decirlo.

## Formato de `STATUS.md`

```markdown
# Estado de <proyecto>

**Última verificación**: AAAA-MM-DD · **Protocolo**: ver `STATUS-PROTOCOL.md` de la raíz.

## Verificación ejecutada

| Comando | Resultado | Fecha |
|---|---|---|
| `npx tsc --noEmit` | limpio | 2026-08-06 |
| `npx vitest run` | 27/27 en 5 archivos | 2026-08-06 |

## Capacidades

| Capacidad | Estado | Evidencia | Notas |
|---|---|---|---|
| Guardar performances | IMPLEMENTADO | `src/api/savePerformance.ts` | |
| Marketplace (UI) | PARCIAL | `src/features/marketplace/PluginGrid.tsx` | Datos hardcodeados |
| Pagos con Stripe | PLANEADO | — | Ver `docs/next-steps/stripe.md` |
| Stack Next.js | DESCARTADO | — | Nunca se construyó |

## Deuda crítica

Bloqueantes reales, no mejoras cosméticas.

## Deuda no crítica
```

La tabla de **verificación ejecutada** lleva resultados de comandos corridos de verdad, con
números. "Pasa" no es un resultado; "27/27 en 5 archivos" sí.

## Encabezado de documentos

Todo archivo dentro de `docs/` abre con una línea de estado:

```markdown
> **Estado**: PLANEADO · **Última verificación**: 2026-08-06
```

La fecha es lo que hace visible el drift: un documento verificado hace cinco meses es sospechoso
por default, sin que nadie tenga que descubrirlo leyendo el código.

## Cuándo se actualiza

- Al cambiar comportamiento: se actualiza la fila correspondiente **en el mismo cambio**.
- Al terminar una sesión de auditoría: se actualiza `Última verificación` y la tabla de comandos.
- Al descartar algo: pasa a `DESCARTADO` con una nota del porqué. No se borra la fila.

## Para sesiones con agentes

El `CLAUDE.md` de cada repo debe indicar:

1. Antes de afirmar que algo funciona, mirar `STATUS.md`.
2. Después de cambiar comportamiento, actualizar la fila correspondiente.
3. Si el código contradice a `STATUS.md`, **gana el código** — y se corrige la fila.

## Validación automática (pendiente)

`scripts/check-status.mjs`: verificar que todo path citado en la columna Evidencia exista.
Caza la mayor parte de la pudrición (archivos renombrados o borrados) por muy poco costo. Mismo
patrón que `core-renderers/scripts/validate-catalog.mjs`.

## Estado del rollout

| Repo | `STATUS.md` | README recortado |
|---|---|---|
| luxsequencer-cloud | ✅ 2026-08-06 | ✅ 2026-08-18 |
| luxsequencer-contracts | ✅ 2026-08-06 | ✅ 2026-08-18 |
| lux-ui | ✅ 2026-08-11 | ✅ 2026-08-18 |
| core-renderers | ✅ 2026-08-11 | ✅ 2026-08-18 |
| luxsequencer-core | ✅ 2026-08-12 | ✅ 2026-08-18 |

**Rollout completo en los 5 repos desde el 2026-08-18.**

Sobre las fechas de la segunda columna: `cloud` y `contracts` figuraban ✅ desde el 2026-08-06 y
**estaba mal**. Los dos recortes fueron reales —cloud pasó de 510 a 123 líneas— pero ninguno tocó
la sección de instalación, que se desactualizó **ese mismo día** con la decisión de topología.
`contracts` llegó a afirmar que el paquete no estaba publicado cuando ya lo estaba.

La lección para la próxima verificación: **un recorte puede ser real y quedar incompleto**, y un
✅ puesto el mismo día que cambia la topología es sospechoso por construcción.

## Una fuente única para la orquestación

Los cuatro READMEs con drift de instalación no tenían cuatro textos distintos: `luxsequencer-core`
y `core-renderers` compartían un bloque **idéntico de 58 líneas**, y los otros dos versiones
abreviadas del mismo. El drift no derivó cuatro veces; derivó una y estaba copiado.

Desde el 2026-08-18, la topología, la instalación, el orden de arranque y la resolución de
dependencias viven **sólo en el `README.md` de la raíz**. Los cinco proyectos apuntan ahí.

Lo único que se repite a propósito es el orden de arranque 4174 → 3000, presente también en
`luxsequencer-core` y `core-renderers`: es lo que rompe si no lo sabés, y son los dos repos donde
importa.

~~Nota de unificación pendiente: `lux-ui` usa `docs/nextsteps/` (sin guion).~~
**Resuelto 2026-08-11**: renombrado a `docs/next-steps/`.
