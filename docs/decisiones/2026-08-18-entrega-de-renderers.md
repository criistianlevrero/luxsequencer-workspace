> **Estado**: VIGENTE · **Fecha**: 2026-08-18 · **Alcance**: ecosistema
>
> Si esta decisión cambia, se escribe un archivo nuevo que la supersede. **No se edita esta.**
>
> **Completa**, sin superseder, a
> [Distribución de renderers: dos canales](2026-08-06-distribucion-renderers.md), que ya definió
> los dos canales y el modelo de entrega por URL firmada. Acá se define **cómo llega el renderer
> al core**, que aquella dejó abierto.

# Entrega de renderers: camino único, caché local, titularidad permanente

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

### 4. Titularidad permanente, revocación como canal aparte

**Lo comprado es del usuario, sin expiración y sin chequeo obligatorio.** Funciona offline para
siempre. No hay revalidación que pueda dejar a alguien sin sus renderers a mitad de un set.

Eso **no** implica renunciar a poder apagar un paquete. Son dos ejes distintos, y confundirlos es
lo que hace parecer que la promesa al usuario y la seguridad están en conflicto:

| Eje | Pregunta | Naturaleza | Alcance |
|---|---|---|---|
| **Titularidad** | ¿sigue teniendo derecho a lo que compró? | comercial | por usuario |
| **Revocación** | ¿este paquete todavía puede correr? | seguridad | **por paquete y clave, no por persona** |

La revocación alcanza incluso a paquetes ya adquiridos, y existe para un caso que no es de
piratería: **un renderer resulta malicioso, rompe con una actualización del core, o hay que bajarlo
por un problema legal.** Sin ella, el usuario sigue ejecutando algo que ya sabemos que no debería.

Se consulta de forma oportunista cuando hay red; sin red, sigue valiendo lo último conocido. La
maquinaria ya está construida: `communityTrustStoreRevocationUrl`,
`communityRevokedPublicKeyIds` y el período de gracia para rotación de claves.

**Revocar no es desposeer.**

#### Por qué no revalidar titularidad

Revalidar no protege contra el usuario decidido: esta misma arquitectura ya asume por escrito que
el worker entregado queda en su máquina. Sólo frena al casual, que es el que menos daño hace. Lo
que sí queda sin cubrir se atiende mejor por otras vías:

- **Reembolsos y contracargos** → política comercial con ventana, no enforcement técnico.
- **Cuentas compartidas a escala** → telemetría oportunista de uso cuando hay conexión. Mide sin
  bloquear a nadie; si una cuenta aparece en cuarenta máquinas, se actúa comercialmente.

La telemetría debe ser **agregada y declarada en los términos, nunca silenciosa**: qué renderers
usa alguien es dato de uso, y esto es una herramienta creativa.

### 5. El tercero sube fuente; cloud revisa, compila y firma

El desarrollador **sube su renderer desde la web**, no desde un repo. Se revisa y se aprueba antes
de publicarlo. Al principio el proceso puede ser manual; después, un pipeline.

Cloud compila la fuente y firma el JS resultante con su clave. El core verifica esa firma contra
el trust store antes de ejecutar.

Consecuencia deliberada: **el toolchain es de cloud, no del tercero.** Un renderer de terceros
nunca ejecuta un script de build en infraestructura propia, lo que elimina una clase entera de
ataque de supply chain antes de que exista.

#### La revisión sola no alcanza, y por eso el toolchain importa

Un worker aprobado que pueda hacer `importScripts()` o `fetch()` + `eval` **descarga su carga real
después de la revisión**: se revisó un archivo y el usuario ejecuta otro. Ni siquiera hace falta
mala fe elaborada — alcanza con que el paquete traiga una URL de configuración cuya respuesta
cambie tres meses después de aprobado.

Para que revisar signifique algo, **lo aprobado tiene que ser lo único que puede correr.** Como
cloud compila, cloud inyecta un bootstrap que neutraliza los globals no declarados —`fetch`,
`XMLHttpRequest`, `WebSocket`, `importScripts`— antes de la primera línea del renderer, y **el
tercero no puede optar por salirse**. Si mintió en el manifest, su código no viola la política:
directamente no funciona.

Ese mismo bootstrap es donde van los límites de recursos. Para un VJ, un renderer que hunde el
framerate a mitad de un set es tan grave como uno que roba datos.

## Qué implica para lo que ya está construido

| Pieza existente | Bajo esta decisión |
|---|---|
| Trust store, verificación de firma y checksum (~1.200 líneas con tests) | **Es la mitad del camino ya hecha.** Verificar autenticidad de código de terceros es requisito, no adorno |
| `requiredCapabilities` y `validateRendererSdkContract` | Base del modelo de capacidades que la revisión va a necesitar |
| `isMarketplaceLicenseTokenValid` — token de licencia validado en el cliente | **Superado.** Si cloud entrega los bytes sólo a quien compró, un chequeo de titularidad en el cliente no controla nada. Que no verifique la firma deja de ser un bug a arreglar: es una pieza que sobra |
| `HARDCODED_EXTERNAL_RENDERERS = []` | **Mecanismo equivocado**, no lista por llenar. Una allowlist fijada en build no puede expresar "este usuario compró estos tres" |

## Lo que esta decisión no resuelve

- **El diseño concreto del modelo de capacidades.** El punto 5 dice que sin sandbox la revisión no
  significa nada, pero el conjunto de capacidades, su representación en el manifest y el contenido
  del bootstrap son trabajo de diseño. Es H6 de
  [`docs/next-steps/marketplace-de-terceros.md`](../next-steps/marketplace-de-terceros.md).
- **La ventana de reembolso**, que el punto 4 delega a política comercial sin fijarla.

> La política de revalidación de titularidad figuraba acá como pregunta abierta y **se resolvió el
> mismo 2026-08-18**, en el punto 4: titularidad permanente, revocación por paquete como canal
> separado.

El plan para llegar hasta acá, con los huecos ordenados por bloqueo, está en
[`docs/next-steps/marketplace-de-terceros.md`](../next-steps/marketplace-de-terceros.md).
