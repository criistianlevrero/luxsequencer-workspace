> **Estado**: VIGENTE · **Fecha**: 2026-08-06 · **Alcance**: ecosistema
>
> Si esta decisión cambia, se escribe un archivo nuevo que la supersede. **No se edita esta.**

# Distribución de paquetes: npm público

## Decisión

`@luxsequencer/ui` y `@luxsequencer/contracts` van a **npm público**, no a un registro propio ni
a GitHub Packages.

## Razón decisiva

Un autor externo de renderers necesita `@luxsequencer/contracts` para tipar su schema
declarativo. Cualquier registro con autenticación le exige credenciales antes de escribir la
primera línea, lo que contradice el objetivo de que "cualquiera puede desarrollar renderers".

**GitHub Packages exige token incluso para paquetes públicos**, así que queda descartado pese a
parecer la opción gratis.

`lux-ui` también tiene que ser público: si `luxsequencer-core` es abierto y depende de ella,
cualquiera que clone core tiene que poder instalarla.

## Estado

Ambos publicados el 2026-08-06 en la org `luxsequencer`:

- `@luxsequencer/contracts@0.1.0` — MIT, cero dependencias de runtime
- `@luxsequencer/ui@0.1.0` — MIT, cero dependencias de runtime, peers `react >=18`

## Consecuencias

- Se arrancó en `0.x`: `unpublish` sólo se puede dentro de las 72hs y sin dependientes.
- Las versiones prerelease (`0.1.0-alpha.0`) **no quedan como `latest`** y exigen `--tag`, así
  que un `npm install` sin versión explícita no las recibe. Por eso ambos salieron como `0.1.0`.
- Publicar con 2FA activo requiere OTP interactivo: no se puede automatizar sin un token de
  automatización, que a su vez saltea el 2FA.
