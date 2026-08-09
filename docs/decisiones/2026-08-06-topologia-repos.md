> **Estado**: VIGENTE · **Fecha**: 2026-08-06 · **Alcance**: ecosistema
>
> Si esta decisión cambia, se escribe un archivo nuevo que la supersede. **No se edita esta.**

# Topología: submódulos + npm workspace

## Contexto

Los cinco proyectos eran repos independientes en una carpeta, unidos a mano con dependencias
`file:../`. Un clon limpio no compilaba, y no existía registro de qué combinación de commits
funcionaba junta — algo que se hizo evidente cuando apareció un cambio de `diagnostic-fps`
partido entre dos repos sin nada que los vinculara.

## Decisión

Repo raíz **privado** `criistianlevrero/luxsequencer-workspace`, con los cinco proyectos como
**submódulos** y un **npm workspace** que los abarca.

Las dos capas son independientes y resuelven cosas distintas:

- **Submódulos (git)**: cómo se clona. El repo raíz no contiene el código, sólo punteros a
  commit. Cada proyecto conserva repo, historia y rama propias (core en `0.6-beta`). El aporte
  real es registrar qué combinación de commits funciona junta.
- **Workspaces (npm)**: cómo se resuelven las dependencias. Los consumidores declaran `^0.1.0`
  en vez de `file:../`. Dentro del workspace npm enlaza las carpetas locales; fuera, baja del
  registro.

## Verificación

Probado end-to-end con clones desde cero el 2026-08-06:

| Escenario | Resultado |
|---|---|
| `clone --recurse-submodules` + `npm install` | Symlinks a las carpetas locales; core buildea; clases de lux-ui en el CSS |
| Clonar `core-renderers` solo + `npm install` | Baja `@luxsequencer/contracts@0.1.0` del registro; imports y `validate` OK |

El segundo escenario era imposible antes: contracts sólo existía en el disco del autor.

## Alternativas descartadas

**`cloud` como subcarpeta simple.** Si la raíz es pública, la subcarpeta expone el código
cerrado; si es privada, el externo no puede clonarla. Además cloud es el que más va a necesitar
deploy y secretos propios, y una subcarpeta acopla eso al repo raíz.

**Submódulos como mecanismo de pinning para `lux-ui` y `contracts`.** Una vez publicados en npm,
el pin es semver en `package.json`. Mantener ambos generaría drift silencioso: el puntero
apuntando a un SHA que el build no usa.

## Consecuencias

- Los submódulos son una clase conocida de errores: HEAD detached, commitear un puntero a un SHA
  no pusheado, `submodule update` descartando trabajo. Las trampas concretas están en el
  [README.md](../../README.md).
- El tamaño de contexto para agentes **no cambia**: la topología git no altera qué archivos hay
  en disco. La palanca sigue siendo el scoping vía `CLAUDE.md` y `STATUS.md` por proyecto.
