> **Estado**: VIGENTE · **Fecha**: 2026-08-06 · **Alcance**: ecosistema
>
> Informe de la Fase 1 de la auditoría: análisis general del monorepo. Cada afirmación fue
> verificada ejecutando comandos o leyendo código, no inferida de la documentación.
>
> **Alcance**: sólo hallazgos que cruzan repos. Lo que pertenece a un proyecto vive en el
> `docs/auditoria/` de ese proyecto.

# Auditoría Fase 1 — general del monorepo

## Modelo de distribución objetivo

Definido por el autor el 2026-08-06. **Es el objetivo, no el estado actual**: los bloqueantes
verificados contra este modelo están en
[docs/next-steps/bloqueantes-modelo-distribucion.md](../next-steps/bloqueantes-modelo-distribucion.md).

| Proyecto | Audiencia | Cómo se distribuye | Licencia objetivo |
|---|---|---|---|
| `luxsequencer-core` | Pública — cualquiera clona y desarrolla renderers | Repo público clonable | Abierta (a definir) |
| `core-renderers` | Pública — repo de referencia para autores de renderers | Repo público clonable | Abierta (a definir) |
| `lux-ui` | Consumidores de los tres proyectos | **Paquete npm publicado** | MIT |
| `luxsequencer-contracts` | Consumidores de los tres proyectos | **Paquete npm publicado** | MIT |
| `luxsequencer-cloud` | Interno | **Código cerrado**, no distribuido | Propietaria |

**Rol de cloud**: es el servicio que monetiza el ecosistema. Cuentas de usuario, guardado de
sesiones/performances, y el marketplace de plugins.


## Topología real

Repo raíz privado `criistianlevrero/luxsequencer-workspace` con los 5 proyectos como submódulos,
unidos por un npm workspace. Cada proyecto sigue siendo un repositorio independiente y clonable
por separado.

| Directorio | Paquete npm | Repo git | Branch | Versión |
|---|---|---|---|---|
| [luxsequencer-core/](../../luxsequencer-core/) | `luxsequencer` | `criistianlevrero/luxsequencer` | `0.6-beta` | `0.6-beta` |
| [lux-ui/](../../lux-ui/) | `@luxsequencer/ui` | `criistianlevrero/lux-ui` | `main` | `0.1.0-alpha.0` |
| [luxsequencer-cloud/](../../luxsequencer-cloud/) | `luxsequencer-cloud` | `criistianlevrero/luxsequencer-cloud` | `main` | `0.1.0` |
| [core-renderers/](../../core-renderers/) | `@luxsequencer/core-renderers-repo` | `criistianlevrero/luxsequencer-marketplace-official` | `main` | `0.1.0-beta.1` |
| [luxsequencer-contracts/](../../luxsequencer-contracts/) | `@luxsequencer/contracts` | `criistianlevrero/luxsequencer-contracts` | `main` | `0.1.0` |

Notas críticas de topología:

- **`luxsequencer-contracts` ya tiene repo y remoto** (creado y pusheado el 2026-08-06). Falta
  publicarlo en npm: hasta entonces los consumidores siguen dependiendo de él por `file:`, con lo
  cual un clon fresco necesita los cinco directorios como hermanos.
- **Autenticación git**: no hay credential helper configurado, pero **sí hay clave SSH funcionando**
  (`id_ed25519`, GitHub ya en `known_hosts`). `luxsequencer-contracts` usa remoto SSH; los otros
  cuatro repos usan HTTPS y por lo tanto piden credenciales en cada push — probablemente por eso
  hay commits sin pushear acumulados en `lux-ui` y `luxsequencer-core`.
- El [package.json](../../package.json) raíz **no declara `workspaces`**; sólo tiene `@ladle/react`
  como devDependency. El `node_modules/` raíz existe pero no orquesta nada.
- Cada proyecto tiene su propio `package-lock.json` y su propio `node_modules/`.
- El directorio `core-renderers/` y el repo remoto `luxsequencer-marketplace-official` **no
  se llaman igual**. La doc usa "core-renderers"; el remoto usa "marketplace-official".

### Grafo de dependencias real

```
@luxsequencer/contracts  (npm público 0.1.0 · fuente de tipos compartidos)
        ▲        ▲        ▲
        │        │        │  "^0.1.0"  → symlink local dentro del workspace,
        │        │        │              descarga del registro fuera de él
        │        │        └────────────── core-renderers   (devDependency)
        │        └────────────────────── luxsequencer-cloud
        └───────────────────────────────  luxsequencer-core

@luxsequencer/ui  (npm público 0.1.0 · consumido vía dist/ compilado)
        ▲        ▲
        └────────┴── luxsequencer-core, luxsequencer-cloud   "^0.1.0"

core-renderers  →  NO es dependencia npm de core.
                   Se consume en runtime por HTTP (proxy same-origin, puerto 4174).
```

Symlinks verificados en `node_modules/@luxsequencer/{contracts,ui}` **de la raíz** — el workspace
hoistea. Editar `lux-ui/src` **no** se refleja en los consumidores hasta correr `npm run build`
en `lux-ui` (consume `dist/`, no `src/`); el script `prepare` lo cubre en cada `npm install`.

### Tamaño relativo (líneas en `src/`)

| Proyecto | Archivos `.ts/.tsx` | Líneas en `src/` |
|---|---:|---:|
| luxsequencer-core | 127 | ~20.600 |
| lux-ui | 59 | ~3.800 |
| core-renderers | 8 | ~3.100 |
| luxsequencer-cloud | 46 | ~2.700 |
| luxsequencer-contracts | 4 | ~500 |

`luxsequencer-core` es ~70% del código. Es donde vive la complejidad real.

---


## Estado verificado (ejecutado el 2026-08-06)

| Proyecto | `tsc --noEmit` | Tests | Lint |
|---|---|---|---|
| luxsequencer-contracts | ✅ limpio | sin tests | sin script |
| luxsequencer-core | ✅ limpio | ❌ **5 fallan / 88** (3 archivos) | ⚠️ 0 errores, **260 warnings** |
| lux-ui | ✅ limpio | ✅ 8/8 | ✅ limpio |
| luxsequencer-cloud | ✅ limpio | ✅ 27/27 | sin script |
| core-renderers | sin script | sin tests | sin script — `npm run validate` ✅ `[catalog] ok` |

**Tests que fallan en `luxsequencer-core`** (todos con la misma causa raíz aparente: el mock de
`../../components/renderers` no expone `resolveRendererDefinition`, export nuevo tras el
refactor a worker-only):

- `src/components/sequencer/Sequencer.test.tsx` → 2 tests
- `src/store/slices/project.slice.test.ts` → 2 tests
- `src/store/slices/ui.slice.test.ts` → 1 test

**`npm run lint` de core falla aunque no haya errores**: el script usa `--max-warnings 0` y hay
260 warnings (mayoría `@typescript-eslint/no-explicit-any`). El script está efectivamente roto
como gate de CI.

**Ojo con los comandos de test**: `npm test` en `luxsequencer-core` y `lux-ui` arranca vitest en
**modo watch** (se cuelga). Usar `npx vitest run`.

---


---

## Drift confirmado que cruza repos

Cada punto tiene evidencia verificada, no inferencia. El drift acotado a un solo proyecto está
en el `docs/auditoria/` de cada repo.

### 1. La doc dice "cuatro repositorios"; son cinco proyectos

[luxsequencer-core/README.md](../../luxsequencer-core/README.md) y los README hermanos describen la
arquitectura como "cuatro repositorios principales" y no mencionan `luxsequencer-contracts`,
que hoy es la dependencia transversal del sistema. Las instrucciones de instalación
("clona los cuatro repositorios") **no producen un entorno funcional**.

### 2. Las instrucciones de instalación describen un flujo que ya no se usa

Los README indican `npm link lux-ui`. La realidad es `file:../lux-ui` declarado en
`package.json` con symlinks ya resueltos por npm. Además el nombre del paquete es
`@luxsequencer/ui`, no `lux-ui`.

### 3. El "marketplace" no es dinámico

[luxsequencer-core/src/components/renderers/index.ts](../../luxsequencer-core/src/components/renderers/index.ts)
contiene un array `ALLOWED_RENDERERS` **hardcodeado** con los 4 renderers y sus manifests
duplicados inline. Verificado:

- `HARDCODED_EXTERNAL_RENDERERS` está vacío (`[]`), así que
  `getMarketplaceRendererRegistry()` siempre devuelve `{}`.
- **`core-renderers/src/catalog.json` nunca se lee**: `grep -rn "catalog"` sobre
  `luxsequencer-core/src` devuelve **cero** resultados.
- Los `manifest.json` de cada renderer tampoco se fetchean; core sólo pide por HTTP los
  archivos `*.worker.ts` y `*-declarative-schema.ts`.

O sea: el catálogo, los manifests y el validador de catálogo son infraestructura **preparada
pero no conectada**. La doc de renderers los presenta como parte del flujo activo.

Corolario: los metadatos están duplicados en dos repos y **ya divergieron**. El nombre del
renderer concéntrico es `"Concéntrico"` en `catalog.json` y `"Concénctrico"` (typo) en el
allowlist de core.


---

## Puntos de integración a verificar en cada sesión

Estos son los contratos que atraviesan proyectos. Cuando audites uno, chequeá su lado del
contrato contra el otro lado:

| Contrato | Definido en | Consumido por | Riesgo |
|---|---|---|---|
| Tipos de controles declarativos | `contracts/src/declarativeControls.ts` | core (vía shim `src/types/declarativeControls.ts`), core-renderers | core **extiende** los tipos base con `Omit<>` + campos con funciones; fácil de desincronizar |
| Identidad/manifest de marketplace | `contracts/src/marketplace.ts` | core (`sdk/toolIdentity.ts`), catálogo de core-renderers | metadatos duplicados a mano, ya divergentes (ver drift #3) |
| API cloud | `contracts/src/api.ts` | `cloud/src/types/api.ts` | core **no** lo consume todavía |
| Protocolo worker de renderer | `core/src/graphics-pipeline/` | workers en `core-renderers/src/renderers/*` | contrato de runtime, cruza red; sin tests de integración |
| Componentes UI | `lux-ui/dist` | core, cloud | rota silenciosamente si `dist` está viejo |

---


---

## Correcciones a este informe

Hallazgos que se reportaron y después se probaron falsos. Se dejan registrados para que nadie los
vuelva a "descubrir".

### `ring-offset-*` no se genera nunca — **FALSO**

Se reportó que Tailwind v4 no generaba las utilidades `ring-offset-*` y que eran markup muerto en
el `Button` de `lux-ui`.

Era un error de medición: el test buscaba el selector `.ring-offset-2` sin prefijo, pero
`Button.tsx` sólo usa esas clases con la variante `focus:`, así que el selector real es
`.focus\:ring-offset-2`.

Verificado el 2026-08-07 con la app corriendo: el `boxShadow` computado del botón enfocado es
`oklch(0.278 0.033 256.848) 0 0 0 2px` (el offset) más
`oklch(0.715 0.143 215.221) 0 0 0 4px` (el anillo), y el hueco se ve en pantalla. Funciona.

### El CSS de core creció por escaneo cruzado entre proyectos — **FALSO**

Se sospechó que la detección automática de Tailwind subía hasta la raíz del workspace y escaneaba
proyectos hermanos. Verificado: las clases que sólo existen en `luxsequencer-cloud`
(`text-green-600`, `bg-green-100`, `border-indigo-600`) **no** aparecen en el CSS de core.

La causa real fue el upgrade de dependencias al borrar los lockfiles. Ver
[decisión de versionado](../decisiones/2026-08-07-versionado-dependencias.md).
