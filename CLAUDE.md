# LuxSequencer — Contexto de auditoría del monorepo

> **Este archivo es la base para todas las sesiones de análisis.** Léelo completo antes de
> analizar cualquier subproyecto.

## Qué estamos haciendo

Auditoría técnica del conjunto LuxSequencer, en dos fases:

1. **Fase 1 (hecha, 2026-08-06)** — Análisis general del monorepo: topología, dependencias
   reales, salud de build/test/lint, y detección de drift documental a nivel sistema.
   El resultado está en este archivo.
2. **Fase 2 (pendiente)** — Una sesión de análisis por proyecto, con la misma metodología.
   Ver [Registro de progreso](#registro-de-progreso) al final.

## Regla número uno: la documentación tiene drift confirmado

**No confíes en la documentación de este repositorio.** No es una sospecha: está verificado
con evidencia concreta más abajo. La documentación (`README.md`, `docs/**`,
`.github/copilot-instructions.md`, `MIGRATION_PLAN.md`) describe en varios puntos:

- arquitectura **planeada** presentada como arquitectura **implementada**,
- estado de migraciones **desactualizado** (tareas hechas marcadas como pendientes),
- inventarios de módulos y capas que **ya no existen**.

**Método obligatorio en cada sesión:**

1. Leer el código primero, la documentación después.
2. Cada afirmación de la doc se trata como *hipótesis a verificar*, no como hecho.
3. Correr los comandos (type-check, test, lint, build) en vez de asumir que pasan.
4. Cuando doc y código discrepan, **el código es la verdad**; anotar la discrepancia en el
   informe del proyecto.
5. No "arreglar" la doc sobre la marcha sin pedirlo — la auditoría primero mapea, después se
   decide qué corregir.

---

## Modelo de distribución objetivo

Definido por el autor el 2026-08-06. **Es el objetivo, no el estado actual** — ver bloqueantes
abajo.

| Proyecto | Audiencia | Cómo se distribuye | Licencia objetivo |
|---|---|---|---|
| `luxsequencer-core` | Pública — cualquiera clona y desarrolla renderers | Repo público clonable | Abierta (a definir) |
| `core-renderers` | Pública — repo de referencia para autores de renderers | Repo público clonable | Abierta (a definir) |
| `lux-ui` | Consumidores de los tres proyectos | **Paquete npm publicado** | MIT |
| `luxsequencer-contracts` | Consumidores de los tres proyectos | **Paquete npm publicado** | MIT |
| `luxsequencer-cloud` | Interno | **Código cerrado**, no distribuido | Propietaria |

**Rol de cloud**: es el servicio que monetiza el ecosistema. Cuentas de usuario, guardado de
sesiones/performances, y el marketplace de plugins.

### Bloqueantes verificados contra este objetivo

Ninguno es de topología de repos; son previos a esa decisión.

1. **"Core abierto" no es cierto legalmente hoy.**
   `luxsequencer-core/LICENSE` es un archivo de **0 bytes** y está commiteado así.
   `core-renderers` no tiene archivo LICENSE ni campo `license` en `package.json`. Sin licencia
   explícita el default es *todos los derechos reservados*: nadie puede forkear ni redistribuir
   legalmente.

2. ~~**`@luxsequencer/contracts` no se puede publicar en npm.**~~ **Resuelto parcialmente el
   2026-08-06**: se creó el repo git, se removió `"private": true`, y se agregaron `LICENSE`,
   `publishConfig.access: public`, `repository` y script `prepare`. **Queda pendiente**: crear el
   remoto, y limpiar `src/api.ts` antes de publicar — mezcla dos generaciones de contratos con 13
   tipos sin consumidores, y publicar los ata a semver. Ver
   [luxsequencer-contracts/STATUS.md](luxsequencer-contracts/STATUS.md).
   `lux-ui` sigue listo para publicar (MIT, sin `private`), pero le falta archivo `LICENSE`.

3. **"Cloud cerrado" hoy no protege nada.**
   `luxsequencer-cloud` es una SPA que habla directo con Supabase desde el browser. La lógica de
   negocio —incluida `src/api/validateLicense.ts`, que decide si un usuario compró un plugin—
   **se ejecuta en el navegador del usuario** y por lo tanto se descarga en cada visita. Cerrar
   el repo oculta el historial, no el código. Para que el modelo de monetización sea real, cloud
   necesita un lado servidor (Edge Functions de Supabase o backend propio).

4. **La aplicación de licencias del marketplace está rota en ambos extremos.**
   - `luxsequencer-core/src/components/renderers/sdk/licenseToken.ts:49` —
     `isMarketplaceLicenseTokenValid()` parte el JWT en 3 partes, decodifica el payload en
     base64 y valida `pluginKey`/`exp`/`iat`. **Nunca verifica la firma.** Un token forjado a
     mano pasa la validación.
   - `luxsequencer-cloud/src/api/validateLicense.ts:62` — firma con el secreto recibido por
     parámetro, y si no recibe ninguno usa el literal `'dev-insecure-license-secret-change-me'`.
     El único llamador que inyecta un secreto real es `mock-api/plugin.ts`, un plugin del dev
     server de Vite.
   - **Problema estructural, más allá de los dos bugs**: no se puede hacer cumplir una licencia
     paga con una validación dentro de un cliente open source — cualquiera forkea core y borra
     el chequeo. La aplicación tiene que estar en la **entrega** (cloud sirve los bytes del
     plugin sólo a cuentas con derecho), no en la ejecución. Esto cambia qué es un renderer pago:
     no un repo que se clona, sino un artefacto que cloud entrega.

---

## Decisiones tomadas (2026-08-06)

Cerradas en discusión con el autor. **No relitigar sin motivo nuevo**; registrar acá cualquier
cambio de rumbo con su fecha.

### Backend de cloud: Supabase para todo

Se evaluó reemplazar Supabase por un backend propio en Docker sobre Railway, y **se revirtió en
la misma sesión**. Razón: la capa `src/api/` ya está desacoplada de React (firma
`(auth, payload, options)`), así que la migración queda disponible a futuro a bajo costo, y hoy
no justifica el trabajo — sobre todo porque reimplementar auth es la parte más grande y menos
visible.

Consecuencia que **no** es opcional: "Supabase para todo" tiene que incluir **Edge Functions**
para lo que maneje secretos o emita credenciales. Si esas rutas quedan en código de browser, los
bloqueantes 3 y 4 de arriba siguen abiertos y el modelo de monetización no cierra. Supabase
Storage cubre además la entrega controlada de renderers privados (bucket privado + URL firmada).

### Distribución de paquetes: npm público

`@luxsequencer/ui` y `@luxsequencer/contracts` van a **npm público**, no a un registro propio ni
a GitHub Packages.

Razón decisiva: un autor externo de renderers necesita `@luxsequencer/contracts` para tipar su
schema declarativo. Cualquier registro con autenticación le exige credenciales antes de escribir
la primera línea, lo que contradice el objetivo de que "cualquiera puede desarrollar renderers".
GitHub Packages exige token **incluso para paquetes públicos**, así que queda descartado pese a
parecer la opción gratis.

`lux-ui` también tiene que ser público: si `luxsequencer-core` es abierto y depende de ella,
cualquiera que clone core tiene que poder instalarla.

Pendientes operativos: reservar el scope `@luxsequencer` en npm cuanto antes; arrancar en `0.x`
porque `unpublish` sólo se puede dentro de las 72hs y sin dependientes.

### Flag de desarrollo de renderers: selección de origen, no desactivación de control

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

### Distribución de renderers: dos canales

- **`core-renderers` (público)**: renderers oficiales, template y ejemplo. Es el repo "aprendé a
  hacer un renderer".
- **Renderers de comunidad**: la **curación** es un workflow de revisión (PR a repo privado o
  upload por la plataforma). La **distribución** es object storage + URL firmada de corta vida
  emitida por cloud según titularidad. **No un repo git privado**: un repo es todo-o-nada, no
  sabe de titularidad por usuario, y el usuario final no clona nada.

Límite aceptado explícitamente: un renderer es un worker JS que corre en el browser del usuario.
Una vez entregado, está en su máquina. No hay forma de impedir que alguien decidido lo extraiga.
El objetivo realista es encarecer la copia casual, no volverla imposible — y eso acota cuánta
ingeniería vale la pena meterle.

### Mecanismo de estado: plan vs. en proceso vs. hecho

Definido en [STATUS-PROTOCOL.md](STATUS-PROTOCOL.md). Cada repo lleva un `STATUS.md` con tabla de
capacidades, y **toda fila `IMPLEMENTADO`/`PARCIAL` debe citar un archivo**. El `README.md`
describe sólo lo que existe; lo planeado vive en `docs/next-steps/`.

Rollout: hecho en `luxsequencer-cloud`. Pendiente en los otros cuatro.

---

## Decisión abierta: topología de repositorios

**Sin resolver al 2026-08-06.** No ejecutar cambios de topología sin confirmación explícita.

**Lo que ya quedó resuelto**: al decidirse npm público para `lux-ui` y `contracts`, el pin de
versiones de esos dos pasa a ser semver en `package.json`. Eso elimina el argumento principal
para tenerlos como submódulos, y reduce la discusión pendiente a: (a) ¿la raíz se vuelve repo
git?, y (b) ¿`core-renderers` y `cloud` entran como submódulos o quedan sueltos?

**Prerrequisito bloqueante**: `luxsequencer-contracts` no tiene repo git y tiene
`"private": true`. Sin resolver eso no hay publicación en npm ni topología posible que lo
incluya. Es el primer paso de cualquiera de los caminos.

Propuesta inicial del autor: convertir la raíz en repo git, cada proyecto como submódulo, y
`luxsequencer-cloud` como subcarpeta simple.

Puntos de análisis registrados:

- **La raíz como repo git tiene consenso**: versiona el `CLAUDE.md`, los scripts de bootstrap y
  las ADRs. Hoy este archivo vive en un directorio sin versionar.
- **`cloud` como subcarpeta tiene una contradicción**: si la raíz es pública, la subcarpeta es
  pública y se pierde el código cerrado; si la raíz es privada, el desarrollador externo no
  puede clonarla y se pierde la DX unificada. Además, cloud es el proyecto que más va a
  necesitar pipeline de deploy y secretos independientes, cosa que una subcarpeta acopla.
- **Submódulos y npm publishing son redundantes para `lux-ui` y `contracts`**: resuelven el mismo
  problema (estado reproducible entre repos). Publicado en npm, el `package.json` del consumidor
  *es* el pin. Mantener ambos genera drift silencioso: el puntero del submódulo apunta a un SHA
  que el build no usa.
- **El contrato core ↔ core-renderers no lo resuelve ningún submódulo**: es HTTP en runtime. Su
  versionado real es `packageVersion` del manifest, hoy desconectado (ver drift #3).
- **Impacto en agentes IA**: el tamaño de contexto **no cambia** — la topología git no altera qué
  archivos hay en disco. La palanca de contexto es el scoping vía `CLAUDE.md` por subproyecto.
  Lo que sí impacta es que los submódulos son una clase conocida de errores para agentes (HEAD
  detached, commitear un puntero a un SHA no pusheado, `submodule update` descartando trabajo).
  Riesgo actualmente activo: hay commits sin pushear en `lux-ui` y en `luxsequencer-core`.
- **Combinación sugerida si se va por submódulos**: agregar **npm workspaces** en la raíz listando
  los directorios de los submódulos. Da un `npm install` único y symlinks automáticos, y elimina
  el cableado manual de `file:../lux-ui` que causa el problema de `dist` desactualizado. Costo a
  vigilar: convivencia entre lockfile raíz y lockfiles por proyecto.

---

## Topología real

No es un monorepo npm. Es **una carpeta contenedora con 5 proyectos independientes**, de los
cuales **4 son repositorios git separados** y **1 no está bajo control de versiones**.

| Directorio | Paquete npm | Repo git | Branch | Versión |
|---|---|---|---|---|
| [luxsequencer-core/](luxsequencer-core/) | `luxsequencer` | `criistianlevrero/luxsequencer` | `0.6-beta` | `0.6-beta` |
| [lux-ui/](lux-ui/) | `@luxsequencer/ui` | `criistianlevrero/lux-ui` | `main` | `0.1.0-alpha.0` |
| [luxsequencer-cloud/](luxsequencer-cloud/) | `luxsequencer-cloud` | `criistianlevrero/luxsequencer-cloud` | `main` | `0.1.0` |
| [core-renderers/](core-renderers/) | `@luxsequencer/core-renderers-repo` | `criistianlevrero/luxsequencer-marketplace-official` | `main` | `0.1.0-beta.1` |
| [luxsequencer-contracts/](luxsequencer-contracts/) | `@luxsequencer/contracts` | `criistianlevrero/luxsequencer-contracts` | `main` | `0.1.0` |

Notas críticas de topología:

- **`luxsequencer-contracts` ya tiene repo y remoto** (creado y pusheado el 2026-08-06). Falta
  publicarlo en npm: hasta entonces los consumidores siguen dependiendo de él por `file:`, con lo
  cual un clon fresco necesita los cinco directorios como hermanos.
- **Autenticación git**: no hay credential helper configurado, pero **sí hay clave SSH funcionando**
  (`id_ed25519`, GitHub ya en `known_hosts`). `luxsequencer-contracts` usa remoto SSH; los otros
  cuatro repos usan HTTPS y por lo tanto piden credenciales en cada push — probablemente por eso
  hay commits sin pushear acumulados en `lux-ui` y `luxsequencer-core`.
- El [package.json](package.json) raíz **no declara `workspaces`**; sólo tiene `@ladle/react`
  como devDependency. El `node_modules/` raíz existe pero no orquesta nada.
- Cada proyecto tiene su propio `package-lock.json` y su propio `node_modules/`.
- El directorio `core-renderers/` y el repo remoto `luxsequencer-marketplace-official` **no
  se llaman igual**. La doc usa "core-renderers"; el remoto usa "marketplace-official".

### Grafo de dependencias real (`file:` + symlinks)

```
luxsequencer-contracts  (git local sin remoto, fuente de tipos compartidos)
        ▲        ▲        ▲
        │        │        │  file:../luxsequencer-contracts
        │        │        └────────────── core-renderers   (devDependency)
        │        └────────────────────── luxsequencer-cloud
        └───────────────────────────────  luxsequencer-core

lux-ui  (@luxsequencer/ui, consumido vía dist/ compilado)
        ▲        ▲
        └────────┴── luxsequencer-core, luxsequencer-cloud   file:../lux-ui

core-renderers  →  NO es dependencia npm de core.
                   Se consume en runtime por HTTP (proxy same-origin, puerto 4174).
```

Symlinks verificados en `*/node_modules/@luxsequencer/{contracts,ui}` — apuntan a los
directorios hermanos. Editar `lux-ui/src` **no** se refleja en los consumidores hasta correr
`npm run build` en `lux-ui` (consume `dist/`, no `src/`).

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

## Drift confirmado a nivel monorepo

Cada punto tiene evidencia verificada, no inferencia.

### 1. La doc dice "cuatro repositorios"; son cinco proyectos

[luxsequencer-core/README.md](luxsequencer-core/README.md) y los README hermanos describen la
arquitectura como "cuatro repositorios principales" y no mencionan `luxsequencer-contracts`,
que hoy es la dependencia transversal del sistema. Las instrucciones de instalación
("clona los cuatro repositorios") **no producen un entorno funcional**.

### 2. Las instrucciones de instalación describen un flujo que ya no se usa

Los README indican `npm link lux-ui`. La realidad es `file:../lux-ui` declarado en
`package.json` con symlinks ya resueltos por npm. Además el nombre del paquete es
`@luxsequencer/ui`, no `lux-ui`.

### 3. El "marketplace" no es dinámico

[luxsequencer-core/src/components/renderers/index.ts](luxsequencer-core/src/components/renderers/index.ts)
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

### 4. `copilot-instructions.md` de core describe una versión y una arquitectura viejas

- Afirma *"package.json shows v0.0.0"*; la versión real es `0.6-beta`.
- Documenta `RendererDefinition.component` como el React FC que dibuja. Tras el refactor a
  worker-only, todos los renderers usan `EmptyExternalRenderer` (`() => null`) y el dibujado
  ocurre en el worker externo.

### 5. `MIGRATION_PLAN.md` de lux-ui está desactualizado hacia atrás

Tareas marcadas `[ ]` que **ya están hechas**:

- Fase 1, "Reemplazar imports en `luxsequencer-core`" → hecho
  (`luxsequencer-core/src/components/ui/primitives/index.ts` es un re-export puro de
  `@luxsequencer/ui`).
- Fase 2 completa, "Migrar `SliderInput`, `CollapsibleSection`, `AdvancedSelect`,
  `RangeSlider`" → los cuatro existen en `lux-ui/src/composites/`.
- Fase 4, "Storybook con casos canónicos" y "Tests de regresión visual" → hay stories de Ladle
  para prácticamente todos los componentes y `lux-ui/src/test/visual/` con Playwright.

El "Avance actual" fechado 2026-03-03 quedó congelado.

### 6. `luxsequencer-core/src/components/ui/README.md` inventa una capa

Documenta cuatro capas (`primitives/`, `composites/`, `patterns/`, `foundation/`).
**`foundation/` no existe** en `luxsequencer-core/src/components/ui/`; se fue a lux-ui.
También lista `AdvancedSelect` como composite de core, pero core sólo conserva `ColorPicker` y
`Vector2DPicker`.

### 7. El README de cloud es una especificación de producto, no una descripción del código

510 líneas describiendo modelo de datos completo, Stripe, `revenue_ledger`, flujos de compra,
fases 1–4 de roadmap. Lo implementado en `luxsequencer-cloud/src/` son 8 archivos en `api/`,
auth con Supabase, dos páginas y grids. **No hay ninguna dependencia de Stripe** en
`package.json`. El `supabase-schema.sql` sí define las tablas (`purchases`, `revenue_ledger`,
etc.), pero sin código que las use.

Al auditar cloud: separar explícitamente *spec* de *implementación*. Hoy están mezcladas en el
mismo documento sin marcarlo.

### 8. Versiones inconsistentes en core-renderers

Existe `RELEASE_NOTES_v0.1.0-beta.2.md`, pero `package.json` y `catalog.json` siguen en
`0.1.0-beta.1`. Y los `manifest.json` de los renderers declaran `packageVersion: "0.6.0-beta"`
—la versión de *core*, no la del repo de renderers.

---

## Trampas operativas (leer antes de tocar nada)

1. **`lux-ui/dist` está desactualizado ahora mismo.** `lux-ui/src/primitives/Slider.tsx` es más
   nuevo que `lux-ui/dist/index.js`. Como core y cloud consumen `dist/`, están viendo una
   versión vieja del Slider. Correr `npm run build` en `lux-ui` antes de diagnosticar cualquier
   bug de UI.
2. **Hay trabajo sin commitear repartido en tres repos, y es coherente entre sí.** No se puede
   evaluar un repo en aislamiento:
   - `core-renderers`: renderer `diagnostic-fps/` **entero sin trackear**, `catalog.json`
     modificado (+6), y `scales-declarative-schema.ts` con **−140 líneas**.
   - `luxsequencer-core`: 5 archivos modificados, incluido el `ALLOWED_RENDERERS` que agrega
     `diagnostic-fps` (+27) y `GraphicsPipelineHost.tsx` (+66).
   - `lux-ui`: 2 commits sin pushear (rebuild del Slider).
   - `luxsequencer-core`: 1 commit sin pushear.
   Es un cambio atómico cross-repo a medio terminar. **Confirmar con el usuario antes de
   commitear, resetear o interpretar esto como abandonado.**
3. **Cambios cross-repo no tienen mecanismo de coordinación.** Cuatro repos, sin submodules,
   sin workspace, sin CI compartido. Un cambio de contrato requiere N commits manuales
   sincronizados a mano. Este es probablemente el mayor riesgo estructural del sistema.
4. **Orden de arranque en dev:** `core-renderers` (`npm run dev`, puerto 4174, `strictPort`)
   **antes** que `luxsequencer-core`, que lo proxea en `/marketplace-core-renderers`. El atajo
   es `npm run dev:all` desde core.
5. **Si tocás `luxsequencer-contracts`**: hay que correr `npm run build` ahí, porque los
   consumidores importan `dist/`, no `src/`.
6. **Hay archivos `.env` reales** en `luxsequencer-core/` y `luxsequencer-cloud/`. Están en
   `.gitignore` (verificado). No volcar su contenido en informes ni en salidas de comandos.

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

## Metodología para las sesiones por proyecto

Para cada proyecto, producir el mismo entregable:

1. **Estado real ejecutado** — type-check, tests, lint, build. Números concretos, no "pasa".
2. **Mapa de arquitectura desde el código** — módulos, capas, flujo de datos, puntos de entrada.
3. **Tabla de drift** — afirmación de la doc │ realidad en el código │ evidencia (archivo:línea).
4. **Código muerto / infraestructura desconectada** — como el catálogo del punto 3 de arriba.
5. **Deuda y riesgos**, ordenados por impacto.
6. **Preguntas abiertas para el usuario** — decisiones que no se pueden inferir del código
   (¿el marketplace dinámico es objetivo activo o abandonado? ¿contracts se va a publicar?).

Orden sugerido, de contrato hacia afuera:
`luxsequencer-contracts` → `lux-ui` → `core-renderers` → `luxsequencer-core` → `luxsequencer-cloud`.

---

## Registro de progreso

| Fase | Estado | Fecha |
|---|---|---|
| General del monorepo | ✅ hecho | 2026-08-06 |
| luxsequencer-contracts | ⬜ pendiente | — |
| lux-ui | ⬜ pendiente | — |
| core-renderers | ⬜ pendiente | — |
| luxsequencer-core | ⬜ pendiente | — |
| luxsequencer-cloud | ⬜ pendiente | — |

Al cerrar cada fase: actualizar esta tabla y agregar el informe del proyecto (o un enlace a él).
