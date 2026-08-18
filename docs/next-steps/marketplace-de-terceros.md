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

### H6 — Falta el modelo de capacidades 🟠

**El problema.** Un renderer es JavaScript arbitrario corriendo en el browser del usuario, **en el
mismo origen que la app** —tiene que ser same-origin, por eso existe el proxy del 4174 para evitar
el `SecurityError` al construir el `Worker`—. Es decir: código de terceros ejecutándose dentro de
la frontera de confianza, con `fetch`, WebSockets, IndexedDB del origen, `importScripts` y todo el
CPU que quiera.

No es hipotético. `dvd-screensaver.worker.ts:143` hace `await fetch(trimmed)` sobre una URL que el
usuario pega en un control, para bajar un logo custom. **Es una feature legítima**, y es también
una llamada de red sin declarar ni restringir. Hoy no existe diferencia entre "necesito red para
una feature" y "necesito red para exfiltrar".

**Por qué la revisión sola no cierra el hueco.** Ver el punto 5 de
[la decisión de entrega](../decisiones/2026-08-18-entrega-de-renderers.md): un paquete aprobado
que pueda cargar código en runtime descarga su carga real después de la revisión.

**Las dos mitades del modelo.**

*Declarar* — existe a medias, y **sobre el eje equivocado**. Los valores actuales de
`requiredCapabilities` son `offscreen-canvas`, `webgl2`, `canvas2d`, `uniform-updates`: capacidades
de *tecnología de dibujo*, que responden "¿el browser lo soporta?", no "¿se lo permito?". Además
se declaran en la allowlist del core (`src/components/renderers/index.ts:153` y siguientes), **no
en los `manifest.json` de los renderers** — verificado el 2026-08-18: los manifests no las traen.
Para un tercero no hay hoy ningún lugar donde declarar nada.

*Hacer cumplir* — no existe. Declarar `webgl2` no impide llamar a `fetch`.

**El eje correcto.** La enorme mayoría de los renderers necesitan canvas, uniforms que entran,
`ImageBitmap` que sale. Nada más: sin red, sin storage, sin código dinámico. Un renderer que pide
red es la excepción que se mira con lupa en la revisión — como el caso legítimo de
`dvd-screensaver`.

**Cómo hacerlo cumplir**, en orden de fuerza:

| Nivel | Qué es | Se lo saltea |
|---|---|---|
| Sólo revisión | alguien lee el código | ofuscación, o carga diferida |
| Análisis estático en el build | cloud rechaza fuente que referencie los globals prohibidos | `globalThis['fe'+'tch']` |
| **Sandbox en runtime** | un bootstrap neutraliza esos globals antes del código del renderer | nada, si lo inyecta cloud al compilar |
| Aislamiento por origen | servir el worker desde otro origen | pelea con el same-origin del `Worker` |

**Enfoque elegido: estático + sandbox.** El análisis en el build como filtro barato y señal para
el revisor; el sandbox como la garantía real. Sin sandbox, la revisión es teatro.

**Qué falta diseñar**: el conjunto cerrado de capacidades, su representación en el manifest, el
contenido del bootstrap, y los límites de recursos que van en ese mismo lugar.

Medio camino hecho: `validateRendererSdkContract` y el campo `requiredCapabilities` ya existen —
hay que reorientarlos de compatibilidad a permiso.

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

- **Ventana de reembolso.** La decisión de entrega delega los reembolsos a política comercial en
  vez de enforcement técnico, pero no fija la ventana.
- **Comisión y precios.** La tabla `platform_config` tiene un `global_commission_percent` con
  default 20% y ningún código que lo lea.

> La **política de revalidación de titularidad** figuraba acá y se resolvió el 2026-08-18:
> titularidad permanente, con revocación por paquete como canal separado. Ver el punto 4 de la
> decisión de entrega.

## Huecos que aparecen de esas decisiones

- **Canal de revocación consultable desde el core.** La maquinaria del trust store existe
  (`communityTrustStoreRevocationUrl`, `communityRevokedPublicKeyIds`, gracia de rotación), pero
  cloud no expone ningún endpoint que la alimente.
- **Telemetría oportunista de uso**, agregada y declarada en los términos, para detectar cuentas
  compartidas sin bloquear a nadie. No existe nada: `posthog` está instalado y el módulo que lo
  envuelve es huérfano.
