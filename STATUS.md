# Estado del ecosistema LuxSequencer

**Última verificación**: 2026-08-12 · **Protocolo**: [STATUS-PROTOCOL.md](STATUS-PROTOCOL.md)

Estado a nivel ecosistema. El estado de cada proyecto vive en su propio `STATUS.md`.

## Verificación ejecutada

| Proyecto | `tsc --noEmit` | Tests | Lint | Build |
|---|---|---|---|---|
| `luxsequencer-contracts` | limpio | sin suite (paquete de tipos) | sin script | limpio |
| `lux-ui` | limpio | 8/8 | limpio | limpio |
| `luxsequencer-core` | limpio | **5 fallan / 88** (3 archivos) | 0 errores, **260 warnings** | limpio |
| `luxsequencer-cloud` | limpio | 27/27 | sin script | limpio |
| `core-renderers` | **no existe** | **no existe** | **no existe** | no hay build; `npm run validate` ok |

Notas:

- **Los 5 tests que fallan en core son preexistentes** a la migración a workspace, y tienen
  **dos causas raíz, no una** (corregido el 2026-08-12; antes esta nota decía que compartían
  causa). `Sequencer.test.tsx` (2) y `ui.slice.test.ts` (1) fallan porque el mock de
  `../../components/renderers` no expone `resolveRendererDefinition`, export nuevo tras el
  refactor a worker-only. `project.slice.test.ts` (2) falla por otra cosa: el harness de store
  del test no incluye `hydrateRendererAnimatableProperties` (`project.slice.ts:451`). El arreglo
  es distinto en cada familia. Ver [`luxsequencer-core/STATUS.md`](luxsequencer-core/STATUS.md).
- **`npm run lint` de core falla aunque no haya errores**: usa `--max-warnings 0` y hay 260
  warnings (mayoría `no-explicit-any`). Está roto como gate de CI.
- **`npm test` en core y lux-ui arranca vitest en modo watch** y se cuelga. Usar `npx vitest run`.
- **`lux-ui` tiene un quinto gate que la tabla no cubre y está en rojo**: `npx playwright test`
  falla. Diagnosticado el 2026-08-11: el test apunta a un id de story que no existe y captura la
  página de error de Ladle; el snapshot base es un canvas vacío. **Nunca testeó nada.** Los cuatro
  verdes de la fila también piden matices: el lint corre con `no-explicit-any` apagado y el
  type-check cubre sólo `src/`. Ver [`lux-ui/STATUS.md`](lux-ui/STATUS.md).
- **`core-renderers` no tiene gates porque no compila.** No es configuración faltante: no hay
  `tsconfig.json`, y los `.ts` se sirven crudos para que Vite los transpile del lado del
  consumidor. Tampoco hay build: `npm run preview` arranca y devuelve 404 en todo. Diagnosticado
  el 2026-08-11, ver [`core-renderers/STATUS.md`](core-renderers/STATUS.md).

## Capacidades del ecosistema

| Capacidad | Estado | Evidencia | Notas |
|---|---|---|---|
| Workspace unificado | IMPLEMENTADO | `package.json` → `workspaces`, `package-lock.json` (991 entradas) | Verificado con clon limpio |
| Submódulos git | IMPLEMENTADO | `.gitmodules` | 5 proyectos, core en `0.6-beta` |
| Clon de proyecto suelto funcional | IMPLEMENTADO | — | Verificado: `core-renderers` solo baja contracts de npm y valida |
| `@luxsequencer/contracts` en npm | IMPLEMENTADO | `0.1.0`, MIT | Publicado 2026-08-06 |
| `@luxsequencer/ui` en npm | IMPLEMENTADO | `0.1.0`, MIT | Publicado 2026-08-06 |
| Protocolo de estado | PARCIAL | [STATUS-PROTOCOL.md](STATUS-PROTOCOL.md) | Rollout en 2 de 5 repos |
| Verificación visual de la app | IMPLEMENTADO | — | 2026-08-11: los 4 renderers dibujan. **`dvd-screensaver` sale invertido verticalmente**; ver [`core-renderers/STATUS.md`](core-renderers/STATUS.md) |
| Licencias abiertas en core y core-renderers | PLANEADO | — | Ver [bloqueantes](docs/next-steps/bloqueantes-modelo-distribucion.md) |
| Lado servidor de cloud | PLANEADO | — | Ver [bloqueantes](docs/next-steps/bloqueantes-modelo-distribucion.md) |
| Meta-repo de DX para terceros | PLANEADO | — | Ver [next-steps](docs/next-steps/meta-repo-dx-terceros.md) |
| CI compartido | PLANEADO | — | No existe |

## Progreso de la auditoría

| Fase | Estado | Fecha |
|---|---|---|
| General del monorepo | ✅ hecho | 2026-08-06 |
| `luxsequencer-contracts` | ✅ hecho — auditado, publicado, con `STATUS.md` | 2026-08-06 |
| `luxsequencer-cloud` | ✅ hecho — auditado, con `STATUS.md`. README pendiente de recorte | 2026-08-18 |
| `lux-ui` | ✅ hecho — auditado, con `STATUS.md`. README pendiente de recorte | 2026-08-11 |
| `core-renderers` | ✅ hecho — auditado, con `STATUS.md`. README pendiente de recorte | 2026-08-11 |
| `luxsequencer-core` | ✅ hecho — auditado, con `STATUS.md`. README pendiente de recorte | 2026-08-12 |

**Fase 2 cerrada en los 5 repos** (2026-08-18). El ciclo de auditoría por proyecto está completo.

Lo que queda abierto del ciclo no es auditar: es **aplicar**. Los cuatro repos con README sin
recortar comparten un mismo drift, y cada auditoría dejó su lista de deuda priorizada en el
`STATUS.md` del repo.

## Rollout del protocolo de estado

| Repo | `STATUS.md` | README recortado | `docs/` con scope propio |
|---|---|---|---|
| `luxsequencer-cloud` | ✅ | ✅ 2026-08-18 | ✅ |
| `luxsequencer-contracts` | ✅ | ✅ 2026-08-18 | — (no necesita) |
| `lux-ui` | ✅ | ✅ 2026-08-18 | ✅ |
| `core-renderers` | ✅ | ✅ 2026-08-18 | ✅ |
| `luxsequencer-core` | ✅ | ✅ 2026-08-18 | ✅ |

**Rollout del protocolo completo en los 5 repos.**

✅ **Cerrado el 2026-08-18.** Los cinco READMEs quedaron recortados y verificados.

Resultó ser **un** problema, no cuatro: `luxsequencer-core` y `core-renderers` compartían un
bloque **idéntico de 58 líneas** describiendo la topología anterior al 2026-08-06 —repos clonados
sueltos, `npm link lux-ui`—, y `lux-ui` y `cloud` tenían versiones abreviadas del mismo. El drift
no derivó cuatro veces: derivó una y estaba copiado.

**La orquestación del ecosistema ahora vive sólo en el [`README.md`](README.md) de esta raíz** y
los cinco proyectos apuntan ahí. Lo único que se repite a propósito es el orden de arranque
4174 → 3000, en `core` y `core-renderers`, porque es lo que rompe si no se sabe.

Además, dos repos que esta tabla daba por recortados **no lo estaban**:

- `luxsequencer-cloud`: el recorte del 2026-08-06 fue real —de 510 a 123 líneas, con la spec
  movida a `plataforma-vision.md`— pero no tocó Requisitos ni Instalación.
- `luxsequencer-contracts`: afirmaba que el paquete **no estaba publicado en npm**. Lo está desde
  el 2026-08-06; verificado contra el registro el 2026-08-18 (`0.1.0`, MIT).

Los dos se desactualizaron **el mismo día en que se los marcó ✅**, por la decisión de topología.

Lo propio de cada repo, también corregido:

- `lux-ui`: documentaba una capa `patterns/` que no existe, y el alcance de extracción como plan
  cuando ya hay una decisión que lo supersede.
- `core-renderers`: `npm run preview` como funcional (devuelve 404 en todo), 3 renderers cuando
  son 4, y un plan de contratos presentado entero como futuro con la fase 1 hecha. El plan se
  movió a su `docs/next-steps/`.
- `luxsequencer-core`: cuatro repos cuando son cinco, 3 renderers cuando son 4, **GPL-3.0 con un
  `LICENSE` de 0 bytes**, y `npm run test` publicado sin avisar que se cuelga en watch.

Sobre la licencia: no se eligió ninguna. El README ahora dice que está sin definir y que el
default legal es *todos los derechos reservados*, en vez de afirmar algo falso. Sigue siendo un
[bloqueante abierto](docs/next-steps/bloqueantes-modelo-distribucion.md).

Ninguno se tocó en su sesión de auditoría, por la regla de no arreglar documentación sobre la
marcha.

~~`luxsequencer-core` arrastra además dos documentos que **inducen a construir contra arquitectura
removida**: `.github/copilot-instructions.md` y `src/components/ui/README.md`.~~
✅ **Corregidos el 2026-08-18**, empezando por ahí porque el primero es el que los agentes leen
automáticamente. Ver
[su drift](luxsequencer-core/docs/auditoria/2026-08-06-drift-copilot-instructions.md), que quedó
como registro de qué decían.

Los dos casos dejaron un criterio aplicable al resto del recorte: **donde el conocimiento ya vive
en `docs/`, el documento apunta en vez de duplicar**. El drift de `copilot-instructions.md` se
sostenía justamente porque mantenía copia propia del contrato de renderers sin mencionar
`docs/renderers.md`, que estaba sano y era la fuente de verdad.

~~Pendiente de unificación: `lux-ui` usa `docs/nextsteps/` (sin guion).~~ **Resuelto 2026-08-11**:
renombrado a `docs/next-steps/`.

## Pendientes abiertos

1. **Republicar `lux-ui` como `0.1.1`** — POSTERGADO (2026-08-07). El `@source` de Tailwind
   agregado a `styles.css` viaja en el paquete, así que los consumidores externos no lo reciben
   hasta esa versión. Mientras se trabaje en local, el symlink del workspace ya lo entrega.

2. **Bloqueantes del modelo de distribución** — reservados para una conversación aparte
   (2026-08-07). Ver [docs/next-steps/bloqueantes-modelo-distribucion.md](docs/next-steps/bloqueantes-modelo-distribucion.md).

3. ~~**🚚 Mudanza pendiente de contenido con scope acotado.**~~ ✅ **Cerrado el 2026-08-18.** La
   raíz ya no tiene contenido acotado a un solo proyecto. Se deja la tabla como registro de dónde
   fue cada cosa:

   | Documento en la raíz | A dónde va |
   |---|---|
   | ~~`docs/decisiones/2026-08-06-flag-desarrollo-renderers.md`~~ | ✅ **mudado 2026-08-12** a [`luxsequencer-core/docs/decisiones/`](luxsequencer-core/docs/decisiones/2026-08-06-flag-desarrollo-renderers.md) |
   | ~~§ copilot-instructions y ui/README~~ | ✅ **mudado 2026-08-12** a [`luxsequencer-core/docs/auditoria/2026-08-06-drift-copilot-instructions.md`](luxsequencer-core/docs/auditoria/2026-08-06-drift-copilot-instructions.md) |
   | ~~§ MIGRATION_PLAN~~ | ✅ **mudado 2026-08-11** a [`lux-ui/docs/auditoria/2026-08-06-drift-migration-plan.md`](lux-ui/docs/auditoria/2026-08-06-drift-migration-plan.md) |
   | ~~§ versiones inconsistentes~~ | ✅ **mudado 2026-08-11** a [`core-renderers/docs/auditoria/2026-08-06-drift-versiones.md`](core-renderers/docs/auditoria/2026-08-06-drift-versiones.md) |
   | ~~Decisión "Backend de cloud: Supabase"~~ | ✅ **mudado 2026-08-18** a [`luxsequencer-cloud/docs/decisiones/2026-08-06-backend-supabase.md`](luxsequencer-cloud/docs/decisiones/2026-08-06-backend-supabase.md) |

   `docs/auditoria/2026-08-06-drift-por-proyecto.md` **se retiró el 2026-08-18**: sus cinco
   secciones están repartidas. Cuatro se mudaron a los repos correspondientes; la § 7 (README de
   cloud como spec de producto) **no se mudó porque ya estaba resuelta** desde el 2026-08-06, con
   el recorte del README de 510 a 123 líneas y la spec a `docs/next-steps/plataforma-vision.md`.
   Verificado el 2026-08-18.

4. **Sin lockfile en clones sueltos** — costo asumido del lockfile único en la raíz. Lo cierra el
   meta-repo de DX para terceros.
