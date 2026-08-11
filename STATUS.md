# Estado del ecosistema LuxSequencer

**Última verificación**: 2026-08-11 · **Protocolo**: [STATUS-PROTOCOL.md](STATUS-PROTOCOL.md)

Estado a nivel ecosistema. El estado de cada proyecto vive en su propio `STATUS.md`.

## Verificación ejecutada

| Proyecto | `tsc --noEmit` | Tests | Lint | Build |
|---|---|---|---|---|
| `luxsequencer-contracts` | limpio | sin suite (paquete de tipos) | sin script | limpio |
| `lux-ui` | limpio | 8/8 | limpio | limpio |
| `luxsequencer-core` | limpio | **5 fallan / 88** (3 archivos) | 0 errores, **260 warnings** | limpio |
| `luxsequencer-cloud` | limpio | 27/27 | sin script | limpio |
| `core-renderers` | sin script | sin suite | sin script | `npm run validate` ok |

Notas:

- **Los 5 tests que fallan en core son preexistentes** a la migración a workspace, con la misma
  causa raíz: el mock de `../../components/renderers` no expone `resolveRendererDefinition`,
  export nuevo tras el refactor a worker-only. Archivos: `Sequencer.test.tsx` (2),
  `project.slice.test.ts` (2), `ui.slice.test.ts` (1).
- **`npm run lint` de core falla aunque no haya errores**: usa `--max-warnings 0` y hay 260
  warnings (mayoría `no-explicit-any`). Está roto como gate de CI.
- **`npm test` en core y lux-ui arranca vitest en modo watch** y se cuelga. Usar `npx vitest run`.
- **`lux-ui` tiene un quinto gate que la tabla no cubre y está en rojo**: `npx playwright test`
  falla. Diagnosticado el 2026-08-11: el test apunta a un id de story que no existe y captura la
  página de error de Ladle; el snapshot base es un canvas vacío. **Nunca testeó nada.** Los cuatro
  verdes de la fila también piden matices: el lint corre con `no-explicit-any` apagado y el
  type-check cubre sólo `src/`. Ver [`lux-ui/STATUS.md`](lux-ui/STATUS.md).

## Capacidades del ecosistema

| Capacidad | Estado | Evidencia | Notas |
|---|---|---|---|
| Workspace unificado | IMPLEMENTADO | `package.json` → `workspaces`, `package-lock.json` (991 entradas) | Verificado con clon limpio |
| Submódulos git | IMPLEMENTADO | `.gitmodules` | 5 proyectos, core en `0.6-beta` |
| Clon de proyecto suelto funcional | IMPLEMENTADO | — | Verificado: `core-renderers` solo baja contracts de npm y valida |
| `@luxsequencer/contracts` en npm | IMPLEMENTADO | `0.1.0`, MIT | Publicado 2026-08-06 |
| `@luxsequencer/ui` en npm | IMPLEMENTADO | `0.1.0`, MIT | Publicado 2026-08-06 |
| Protocolo de estado | PARCIAL | [STATUS-PROTOCOL.md](STATUS-PROTOCOL.md) | Rollout en 2 de 5 repos |
| Verificación visual de la app | IMPLEMENTADO | — | 2026-08-07: app levantada, 0 errores de consola, pipeline de renderers dibujando |
| Licencias abiertas en core y core-renderers | PLANEADO | — | Ver [bloqueantes](docs/next-steps/bloqueantes-modelo-distribucion.md) |
| Lado servidor de cloud | PLANEADO | — | Ver [bloqueantes](docs/next-steps/bloqueantes-modelo-distribucion.md) |
| Meta-repo de DX para terceros | PLANEADO | — | Ver [next-steps](docs/next-steps/meta-repo-dx-terceros.md) |
| CI compartido | PLANEADO | — | No existe |

## Progreso de la auditoría

| Fase | Estado | Fecha |
|---|---|---|
| General del monorepo | ✅ hecho | 2026-08-06 |
| `luxsequencer-contracts` | ✅ hecho — auditado, publicado, con `STATUS.md` | 2026-08-06 |
| `luxsequencer-cloud` | 🟡 parcial — `STATUS.md` y README hechos; falta auditar el código a fondo | 2026-08-06 |
| `lux-ui` | ✅ hecho — auditado, con `STATUS.md`. README pendiente de recorte | 2026-08-11 |
| `core-renderers` | ⬜ pendiente | — |
| `luxsequencer-core` | ⬜ pendiente — es el 70% del código, el más grande | — |

## Rollout del protocolo de estado

| Repo | `STATUS.md` | README recortado | `docs/` con scope propio |
|---|---|---|---|
| `luxsequencer-cloud` | ✅ | ✅ | ✅ |
| `luxsequencer-contracts` | ✅ | ✅ | — (no necesita) |
| `lux-ui` | ✅ | ⬜ | ✅ |
| `luxsequencer-core` | ⬜ | ⬜ | ⬜ |
| `core-renderers` | ⬜ | ⬜ | ⬜ |

`lux-ui` tiene el README pendiente de recorte: describe una estructura y un flujo de instalación
que no existen (ver su auditoría, § 3, filas D8–D12). No se tocó en la sesión de auditoría por la
regla de no arreglar documentación sobre la marcha.

~~Pendiente de unificación: `lux-ui` usa `docs/nextsteps/` (sin guion).~~ **Resuelto 2026-08-11**:
renombrado a `docs/next-steps/`.

## Pendientes abiertos

1. **Republicar `lux-ui` como `0.1.1`** — POSTERGADO (2026-08-07). El `@source` de Tailwind
   agregado a `styles.css` viaja en el paquete, así que los consumidores externos no lo reciben
   hasta esa versión. Mientras se trabaje en local, el symlink del workspace ya lo entrega.

2. **Bloqueantes del modelo de distribución** — reservados para una conversación aparte
   (2026-08-07). Ver [docs/next-steps/bloqueantes-modelo-distribucion.md](docs/next-steps/bloqueantes-modelo-distribucion.md).

3. **🚚 Mudanza pendiente de contenido con scope acotado.** Hay documentos en la raíz que
   pertenecen a un proyecto. Están marcados con 🚚 en su encabezado. **Al actualizar la base de
   conocimientos de esos repos, moverlos**:

   | Documento en la raíz | A dónde va |
   |---|---|
   | [`docs/decisiones/2026-08-06-flag-desarrollo-renderers.md`](docs/decisiones/2026-08-06-flag-desarrollo-renderers.md) | `luxsequencer-core/docs/decisiones/` |
   | [`docs/auditoria/2026-08-06-drift-por-proyecto.md`](docs/auditoria/2026-08-06-drift-por-proyecto.md) § copilot-instructions y ui/README | `luxsequencer-core/docs/auditoria/` |
   | ~~§ MIGRATION_PLAN~~ | ✅ **mudado 2026-08-11** a [`lux-ui/docs/auditoria/2026-08-06-drift-migration-plan.md`](lux-ui/docs/auditoria/2026-08-06-drift-migration-plan.md) |
   | [`docs/auditoria/2026-08-06-drift-por-proyecto.md`](docs/auditoria/2026-08-06-drift-por-proyecto.md) § versiones inconsistentes | `core-renderers/docs/auditoria/` |
   | Decisión "Backend de cloud: Supabase", hoy dentro de `luxsequencer-cloud/STATUS.md` | `luxsequencer-cloud/docs/decisiones/` |

   Están en la raíz sólo porque esos repos todavía no tienen la carpeta. **No es contenido de
   ecosistema**: viola la regla de scope y está acá de forma transitoria.

4. **Sin lockfile en clones sueltos** — costo asumido del lockfile único en la raíz. Lo cierra el
   meta-repo de DX para terceros.
