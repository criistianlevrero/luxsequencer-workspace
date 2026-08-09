> **Estado**: VIGENTE · **Fecha**: 2026-08-06 · **Alcance**: `core-renderers` + `luxsequencer-cloud`
>
> Si esta decisión cambia, se escribe un archivo nuevo que la supersede. **No se edita esta.**

# Distribución de renderers: dos canales

## Decisión

- **`core-renderers` (público)**: renderers oficiales, template y ejemplo. Es el repo "aprendé a
  hacer un renderer".
- **Renderers de comunidad**: la **curación** es un workflow de revisión (PR a repo privado o
  upload por la plataforma). La **distribución** es object storage + URL firmada de corta vida
  emitida por cloud según titularidad.

## Por qué no un repo git privado

Se propuso originalmente un repo privado para los renderers de comunidad. Se descartó:

1. **El usuario final no clona nada.** Un VJ que compra un renderer necesita que le llegue a la
   app en runtime, no un `git clone`.
2. **Curar es un workflow de revisión, no un lugar de almacenamiento.** Se puede curar con PRs,
   pero el artefacto tiene que vivir detrás de un chequeo de autorización.
3. **Un repo no sabe de titularidad por usuario.** Es todo-o-nada; no puede entregarle a una
   cuenta sí y a otra no.

## Límite aceptado explícitamente

Un renderer es un worker JS que corre en el browser del usuario. Una vez entregado, está en su
máquina. **No hay forma de impedir que alguien decidido lo extraiga.**

El objetivo realista es encarecer la copia casual, no volverla imposible — y eso acota cuánta
ingeniería vale la pena meterle.
