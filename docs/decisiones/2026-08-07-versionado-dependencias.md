> **Estado**: VIGENTE · **Fecha**: 2026-08-07 · **Alcance**: ecosistema
>
> Si esta decisión cambia, se escribe un archivo nuevo que la supersede. **No se edita esta.**

# Estrategia de versiones de dependencias

## Contexto

Al unificar el workspace se borraron los lockfiles por proyecto, lo que provocó un **upgrade no
revisado** en los cinco:

| Paquete | Antes | Después |
|---|---|---|
| `tailwindcss` | 4.1.18 | 4.3.3 |
| `daisyui` | 5.5.19 | 5.7.16 |
| `@tailwindcss/postcss` | 4.1.18 | 4.3.3 |
| `react` | 19.2.0 | 19.2.8 |
| `vite` | 6.4.1 | 6.4.3 |

Nada se rompió. Eso explica además el crecimiento del CSS de core de 102.77 kB a 119.81 kB; se
descartaron con evidencia las hipótesis de falta de estilos y de escaneo cruzado entre proyectos.

## Decisión

- **Durante el desarrollo**, las versiones quedan fijadas por el `package-lock.json` de la raíz
  (991 entradas, commiteado). **No flotan solas**: actualizar es un acto deliberado
  (`npm update`), revisado y commiteado como cualquier otro cambio.
- **Al preparar una versión publicable**, se fijan versiones exactas en el `package.json` de las
  **aplicaciones**: `luxsequencer-core`, `core-renderers`, `luxsequencer-cloud`. Como ya no hay
  lockfile por repo, es lo único que acota qué instala un clon suelto.
- **Las librerías mantienen rangos permisivos**: `@luxsequencer/contracts` y `@luxsequencer/ui`
  **no se fijan nunca**. Fijar versiones en una librería publicada obliga a los consumidores a
  instalar copias duplicadas y genera conflictos irresolubles. Hoy el punto es teórico: ambas
  tienen cero dependencias de runtime.
- **Fijar es un paso revisado, no una foto**: se fija después de verificar la app, nunca antes.
  Es lo que evita repetir el salto ciego de la migración.

## Límite conocido

Fijar en `package.json` sólo fija el **primer nivel** (~40 entradas). Las dependencias
transitivas siguen resolviendo por sus propios rangos. Reproducibilidad total sólo la da un
lockfile.

**Dónde importa ese límite**: sólo donde se resuelven dependencias sin lockfile. Los artefactos
publicados no corren riesgo — `contracts` y `lux-ui` se publican como `dist/` compilado y sin
dependencias, y core/cloud/core-renderers se despliegan como bundles ya construidos. El riesgo
real está en **dónde corre el build de producción**: si corre desde este workspace, está fijado;
si algún día corre en CI clonando un repo suelto, hay que darle un lockfile.

Ese hueco lo cierra el [meta-repo de DX para terceros](../next-steps/meta-repo-dx-terceros.md).
