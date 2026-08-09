> 🚚 **PENDIENTE DE MUDANZA** — Cada sección de este documento **pertenece a un proyecto
> distinto** y debe repartirse a los `docs/auditoria/` de cada repo en la próxima actualización
> de la base de conocimientos. Está acá porque esos repos todavía no tienen la carpeta.
>
> | Sección | A dónde va |
> |---|---|
> | `copilot-instructions.md` desactualizado | `luxsequencer-core/docs/auditoria/` |
> | `ui/README.md` inventa una capa | `luxsequencer-core/docs/auditoria/` |
> | `MIGRATION_PLAN.md` desactualizado hacia atrás | `lux-ui/docs/auditoria/` |
> | Versiones inconsistentes | `core-renderers/docs/auditoria/` |
>
> **Estado**: VIGENTE · **Fecha**: 2026-08-06

# Drift por proyecto — pendiente de repartir

Hallazgos de la auditoría de Fase 1 acotados a un solo proyecto. Cada uno tiene evidencia
verificada, no inferencia.

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
