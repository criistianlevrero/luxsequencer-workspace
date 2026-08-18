> **Estado**: VIGENTE · **Fecha**: 2026-08-18 · **Alcance**: ecosistema
>
> Si esta decisión cambia, se escribe un archivo nuevo que la supersede. **No se edita esta.**
>
> **Completa**, sin superseder, a
> [Distribución de renderers: dos canales](2026-08-06-distribucion-renderers.md), que ya definió
> los dos canales y el modelo de entrega por URL firmada. Acá se define **cómo llega el renderer
> al core**, que aquella dejó abierto.

# Entrega de renderers: camino único, caché local, fuente revisada

## Contexto

El producto es un marketplace: el usuario se loguea en cloud, adquiere renderers de terceros, y al
abrir el core puede usar los que están registrados a su nombre. La decisión del 2026-08-06 ya
estableció que la distribución es object storage + URL firmada de corta vida emitida por cloud
según titularidad.

Lo que faltaba definir es de dónde carga el core, en qué formato viaja un renderer de terceros, y
cómo se desarrolla uno sin tener que publicarlo.

## Decisión

### 1. En producción hay un solo camino de carga: el marketplace

**No hay renderers empaquetados con el core.** Los cuatro oficiales tampoco: viven en el Storage
de cloud como cualquier otro y se entregan por el mismo mecanismo.

El motivo es no sostener dos rutas de carga con dos modelos de confianza distintos. Un solo camino
significa un solo lugar donde puede fallar la validación de firma, y un solo lugar donde arreglarla.

### 2. Los renderers adquiridos quedan cacheados localmente

La titularidad se resuelve contra cloud cuando hay conexión; **los bytes ya están en la máquina**.

Esto no es una concesión de seguridad. La decisión del 2026-08-06 ya asume por escrito que *"una
vez entregado, el worker está en la máquina del usuario; no hay forma de impedir que alguien
decidido lo extraiga"*. Cachear no regala nada que no estuviera regalado.

Lo que sí resuelve son dos problemas reales de uso:

- **LuxSequencer es una herramienta de VJ.** Se usa en clubes y festivales, con wifi de venue. Un
  set no puede depender de que la conexión aguante. Sin caché, un corte de red deja al usuario sin
  ningún renderer: no degradado, negro.
- **El arranque en frío.** Sin caché hay que bajar cada renderer antes de dibujar el primer frame.

El caché es almacenamiento, no un segundo camino de carga: el mecanismo de entrega sigue siendo
uno solo.

### 3. El desarrollo local no pasa por el marketplace

Un desarrollador de renderers —propio o tercero— levanta `core-renderers` en local y el core lo
consume por el proxy same-origin ya existente. No hace falta publicar nada para iterar.

Ese camino se habilita con un `source: 'local-dev'` explícito en el manifest, **no** con un flag
global que apague la validación. La diferencia importa: un flag global es una perilla que alguien
puede dejar encendida en un build de producción; un `source` distinto sólo afecta a los paquetes
que lo declaran.

Esto ya estaba recomendado en la "Propuesta de evolución" de `luxsequencer-core/docs/renderers.md`
§ 9, y en
[la decisión del flag de desarrollo](../../luxsequencer-core/docs/decisiones/2026-08-06-flag-desarrollo-renderers.md).
Queda confirmado como el camino.

### 4. El tercero sube fuente; cloud revisa, compila y firma

El desarrollador **sube su renderer desde la web**, no desde un repo. Se revisa y se aprueba antes
de publicarlo. Al principio el proceso puede ser manual; después, un pipeline.

Cloud compila la fuente y firma el JS resultante con su clave. El core verifica esa firma contra
el trust store antes de ejecutar.

Consecuencia deliberada: **el toolchain es de cloud, no del tercero.** Un renderer de terceros
nunca ejecuta un script de build en infraestructura propia, lo que elimina una clase entera de
ataque de supply chain antes de que exista.

## Qué implica para lo que ya está construido

| Pieza existente | Bajo esta decisión |
|---|---|
| Trust store, verificación de firma y checksum (~1.200 líneas con tests) | **Es la mitad del camino ya hecha.** Verificar autenticidad de código de terceros es requisito, no adorno |
| `requiredCapabilities` y `validateRendererSdkContract` | Base del modelo de capacidades que la revisión va a necesitar |
| `isMarketplaceLicenseTokenValid` — token de licencia validado en el cliente | **Superado.** Si cloud entrega los bytes sólo a quien compró, un chequeo de titularidad en el cliente no controla nada. Que no verifique la firma deja de ser un bug a arreglar: es una pieza que sobra |
| `HARDCODED_EXTERNAL_RENDERERS = []` | **Mecanismo equivocado**, no lista por llenar. Una allowlist fijada en build no puede expresar "este usuario compró estos tres" |

## Lo que esta decisión no resuelve

- **La política de revalidación de titularidad.** Cada cuánto se rechequea contra cloud, y qué
  pasa si el usuario está offline tres semanas. Va desde "revalida o no anda" hasta "una vez
  comprado, tuyo". Es decisión de producto y queda abierta.
- **El modelo de capacidades para la revisión.** Un worker puede hacer `fetch()` a cualquier lado
  —`dvd-screensaver` ya lo hace—, así que revisar fuente ajena buscando malicia necesita un
  modelo de amenazas escrito.

El plan para llegar hasta acá, con los huecos ordenados por bloqueo, está en
[`docs/next-steps/marketplace-de-terceros.md`](../next-steps/marketplace-de-terceros.md).
