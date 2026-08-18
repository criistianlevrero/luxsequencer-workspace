> 🚚 **PENDIENTE DE MUDANZA** — Cada sección de este documento **pertenece a un proyecto
> distinto** y debe repartirse a los `docs/auditoria/` de cada repo en la próxima actualización
> de la base de conocimientos. Está acá porque esos repos todavía no tienen la carpeta.
>
> | Sección | A dónde va |
> |---|---|
> | ~~`copilot-instructions.md` desactualizado~~ | ✅ **Mudado 2026-08-12** a `luxsequencer-core/docs/auditoria/2026-08-06-drift-copilot-instructions.md` |
> | ~~`ui/README.md` inventa una capa~~ | ✅ **Mudado 2026-08-12** a `luxsequencer-core/docs/auditoria/2026-08-06-drift-copilot-instructions.md` |
> | ~~`MIGRATION_PLAN.md` desactualizado hacia atrás~~ | ✅ **Mudado 2026-08-11** a `lux-ui/docs/auditoria/2026-08-06-drift-migration-plan.md` |
> | ~~Versiones inconsistentes~~ | ✅ **Mudado 2026-08-11** a `core-renderers/docs/auditoria/2026-08-06-drift-versiones.md` |
>
> **Sólo queda la § 7 (README de cloud)**, que se muda cuando ese repo tenga su auditoría de
> Fase 2.
>
> **Estado**: VIGENTE · **Fecha**: 2026-08-06 · **Última verificación**: 2026-08-12

# Drift por proyecto — pendiente de repartir

Hallazgos de la auditoría de Fase 1 acotados a un solo proyecto. Cada uno tiene evidencia
verificada, no inferencia.

### 4. `copilot-instructions.md` de core describe una versión y una arquitectura viejas

🚚 **Mudado el 2026-08-12** a
[`luxsequencer-core/docs/auditoria/2026-08-06-drift-copilot-instructions.md`](../../luxsequencer-core/docs/auditoria/2026-08-06-drift-copilot-instructions.md),
reverificado en la auditoría de Fase 2 de ese repo y ampliado con un hallazgo más: el documento
también publica `type: 'custom'` como tipo de control válido, que es justamente lo que la política
de controles declarativos estrictos prohíbe.

### 5. `MIGRATION_PLAN.md` de lux-ui está desactualizado hacia atrás

🚚 **Mudado el 2026-08-11** a
[`lux-ui/docs/auditoria/2026-08-06-drift-migration-plan.md`](../../lux-ui/docs/auditoria/2026-08-06-drift-migration-plan.md),
verificado de nuevo y ampliado con tres hallazgos más en la auditoría de Fase 2 de ese repo.

### 6. `luxsequencer-core/src/components/ui/README.md` inventa una capa

🚚 **Mudado el 2026-08-12** al
[mismo documento que la § 4](../../luxsequencer-core/docs/auditoria/2026-08-06-drift-copilot-instructions.md),
por ser el mismo problema —documentación para agentes que describe arquitectura removida—
reverificado y ampliado: además de `foundation/`, que no existe, `primitives/` ya no contiene
ningún componente local. Es un barrel de re-exports de `@luxsequencer/ui`.

### 7. El README de cloud es una especificación de producto, no una descripción del código

510 líneas describiendo modelo de datos completo, Stripe, `revenue_ledger`, flujos de compra,
fases 1–4 de roadmap. Lo implementado en `luxsequencer-cloud/src/` son 8 archivos en `api/`,
auth con Supabase, dos páginas y grids. **No hay ninguna dependencia de Stripe** en
`package.json`. El `supabase-schema.sql` sí define las tablas (`purchases`, `revenue_ledger`,
etc.), pero sin código que las use.

Al auditar cloud: separar explícitamente *spec* de *implementación*. Hoy están mezcladas en el
mismo documento sin marcarlo.

### 8. Versiones inconsistentes en core-renderers

🚚 **Mudado el 2026-08-11** a
[`core-renderers/docs/auditoria/2026-08-06-drift-versiones.md`](../../core-renderers/docs/auditoria/2026-08-06-drift-versiones.md),
reverificado y ampliado en la auditoría de Fase 2 de ese repo: HEAD está 6 commits por delante del
último tag, y esos commits incluyen un refactor completo y un renderer nuevo sin que ningún número
haya subido.
