> **Estado**: PLANEADO · **Fecha**: 2026-08-18 · **Alcance**: ecosistema
>
> Plan para llegar al marketplace de terceros. Las decisiones que lo definen están en
> [Distribución de renderers](../decisiones/2026-08-06-distribucion-renderers.md) y
> [Entrega de renderers](../decisiones/2026-08-18-entrega-de-renderers.md); acá va **qué falta y
> en qué orden**.
>
> Todo lo que se afirma sobre el estado actual está verificado contra el código el 2026-08-18.

# Marketplace de terceros — arquitectura objetivo y huecos

## El flujo completo, cuando esté terminado

```
DESARROLLADOR                CLOUD                          USUARIO / CORE
─────────────                ─────                          ──────────────
sube fuente por web  ──▶  revisión y aprobación
                          compila a JS
                          firma con clave de cloud
                          guarda en Storage
                                 │
                          catálogo público  ◀────────  navega el marketplace
                                 │
                          registra compra    ◀────────  adquiere
                                 │
                          Edge Function:     ◀────────  abre el core
                          ¿tiene derecho?              pide sus renderers
                          emite URL firmada
                                 │
                                 └──────────────▶  descarga, verifica firma,
                                                   cachea local y ejecuta
```

## Dónde estamos

| Etapa | Estado |
|---|---|
| Verificación de firma y checksum en el core | ✅ construido, con tests (~1.200 líneas) |
| Contrato de worker versionado, identidad canónica, manifests | ✅ construido |
| Esquema de base de datos (`plugins`, `purchases`, RLS) | ✅ construido |
| Catálogo de marketplace en la UI de cloud | 🟡 UI con datos hardcodeados; `listPlugins` existe y funciona pero no lo consume nadie |
| Registro dinámico de renderers por usuario | ⬜ no existe |
| Pipeline de compilación y firma | ⬜ no existe |
| Edge Function de URL firmada | ⬜ no existe. **No hay ninguna Edge Function** |
| Flujo de compra | ⬜ no existe. La tabla `purchases` no la escribe nadie |
| Caché local de renderers | ⬜ no existe |

## Los huecos, ordenados por qué bloquea a qué

### H1 — El core hardcodea los uniforms de cada renderer 🔴 **bloquea todo lo demás**

`applyRendererUniforms` (`luxsequencer-core/src/components/renderers/pipeline/GraphicsPipelineHost.tsx:233`)
es una cadena de comparaciones por id: una rama para `webgl`, otra para `diagnostic-fps`, otra
para `concentric`, otra para `dvd-screensaver`.

**Un tercero no puede publicar un renderer funcional aunque supere toda la cadena de validación**,
porque sus uniforms no tienen rama en ese `if`. Cada renderer nuevo obliga a editar el core,
recompilarlo y publicarlo — que es exactamente lo que un marketplace existe para evitar.

Es el techo real del marketplace, y no tiene nada que ver con seguridad. Como lo resume la
auditoría de core: *el sistema tiene la puerta blindada y la pared abierta.*

Hace falta un **contrato genérico de uniforms**: que el schema declarativo que ya viaja en el
manifest alcance para que los valores del store lleguen al worker sin que el core sepa qué
renderer es. Todo lo demás de este documento es inútil hasta que esto exista.

### H2 — No hay formato de artefacto definido 🔴

Hoy `core-renderers` publica `.worker.ts` crudo y el Vite del consumidor lo transpila. Para un
repo de ejemplo en desarrollo está bien. Como canal de producción no sirve: no se compila
TypeScript de un tercero en el browser del usuario.

La decisión ya fijó el formato de salida —JS compilado y firmado por cloud— pero falta definir el
de entrada: qué se acepta subir, con qué límites, y contra qué versión del SDK de renderer se
compila.

### H3 — Cloud no tiene lado servidor 🔴

Emitir una URL firmada según titularidad es, por definición, código de servidor. No hay ninguna
Edge Function en `luxsequencer-cloud`. La capa `src/api/*` tiene la forma correcta
—`(auth, payload, options)`— pero corre en el browser.

Es el corolario de la [decisión de backend](../../luxsequencer-cloud/docs/decisiones/2026-08-06-backend-supabase.md)
que nunca se ejecutó.

### H4 — El registro de renderers es una allowlist de build 🟠

`HARDCODED_EXTERNAL_RENDERERS` es `[]` y está fijada en tiempo de compilación
(`luxsequencer-core/src/components/renderers/index.ts:265`). No es una lista por llenar: una
allowlist compilada no puede expresar titularidad por usuario. Hay que reemplazarla por un
registro que se arme en runtime con lo que cloud responda para esa cuenta.

### H5 — No hay flujo de compra 🟠

La tabla `purchases` existe con su RLS de sólo lectura —correcta: el usuario no puede fabricarse
una compra— pero ningún código la escribe. Requiere H3, y Stripe sigue sin instalarse.

### H6 — Falta el modelo de capacidades para la revisión 🟠

Un worker puede hacer `fetch()` a cualquier lado; `dvd-screensaver` ya lo hace (R8 de la auditoría
de `core-renderers`). Revisar fuente ajena buscando malicia sin un modelo de amenazas escrito no
escala más allá de unos pocos paquetes conocidos.

Medio camino hecho: `requiredCapabilities` en el manifest y `validateRendererSdkContract` ya
existen.

### H7 — Caché local de renderers adquiridos 🟡

Decidido el 2026-08-18, sin implementar. Depende de H2: no se puede cachear un artefacto cuyo
formato no está definido.

## Deuda que este plan vuelve obsoleta

- **`isMarketplaceLicenseTokenValid`** y el token de licencia validado en el cliente. Que no
  verifique la firma criptográfica figuraba como bug en la deuda crítica de cloud; bajo el modelo
  de entrega por URL firmada, es una pieza que sobra. **No arreglarlo: retirarlo** cuando H3 esté.
- El flag `VITE_MARKETPLACE_ENFORCE_LICENSE_TOKENS` y su bypass global, reemplazados por
  `source: 'local-dev'`.

## Decisiones de producto que siguen abiertas

- **Política de revalidación de titularidad** (ver la decisión de entrega): cada cuánto se
  rechequea contra cloud y qué pasa con un usuario offline por semanas.
- **Comisión y precios.** La tabla `platform_config` tiene un `global_commission_percent` con
  default 20% y ningún código que lo lea.
