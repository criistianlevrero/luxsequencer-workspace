# LuxSequencer — contexto para sesiones de trabajo

> **Este archivo es el índice.** Léelo entero antes de tocar nada; lo demás se consulta cuando
> hace falta.
>
> **Última verificación**: 2026-08-07

## Qué estamos haciendo

Auditoría técnica del conjunto LuxSequencer, en dos fases:

1. **Fase 1 (hecha, 2026-08-06)** — Análisis general del monorepo. Informe en
   [docs/auditoria/2026-08-06-fase-1-monorepo.md](docs/auditoria/2026-08-06-fase-1-monorepo.md).
2. **Fase 2 (en curso)** — Una sesión de análisis por proyecto, con la misma metodología.
   Progreso en [STATUS.md](STATUS.md).

## Regla número uno: la documentación tiene drift confirmado

**No confíes en la documentación de este repositorio.** No es una sospecha: está verificado con
evidencia. La documentación describe en varios puntos arquitectura **planeada** presentada como
**implementada**, migraciones con tareas hechas marcadas como pendientes, e inventarios de
módulos que ya no existen.

**Método obligatorio en cada sesión:**

1. Leer el código primero, la documentación después.
2. Cada afirmación de la doc es una *hipótesis a verificar*, no un hecho.
3. Correr los comandos (type-check, test, lint, build) en vez de asumir que pasan.
4. Cuando doc y código discrepan, **el código es la verdad**; anotar la discrepancia.
5. No "arreglar" la doc sobre la marcha sin pedirlo.

## Dónde va cada cosa que escribas

**El scope de un documento es el del repo al que pertenece.** Si lo que estás escribiendo aplica
sólo a cloud, va en cloud. La raíz es únicamente para conocimiento que **excede el scope de un
proyecto**.

| Contenido | Dónde va |
|---|---|
| Cómo usar el workspace | [README.md](README.md) |
| Estado del ecosistema y progreso de auditoría | [STATUS.md](STATUS.md) |
| Cómo se separa plan de implementado | [STATUS-PROTOCOL.md](STATUS-PROTOCOL.md) |
| Decisión que cruza repos | `docs/decisiones/AAAA-MM-DD-tema.md` |
| Decisión de un solo proyecto | `<proyecto>/docs/decisiones/` |
| Hallazgo de auditoría que cruza repos | `docs/auditoria/` |
| Hallazgo de un solo proyecto | `<proyecto>/docs/auditoria/` |
| Trabajo planeado | `docs/next-steps/` o `<proyecto>/docs/next-steps/` |

**Al revertir una decisión se escribe un archivo nuevo que supersede al anterior. No se edita el
viejo.** Editar la historia en el lugar es el mismo mecanismo de drift que estamos combatiendo.

## Decisiones vigentes

| Decisión | Fecha | Alcance |
|---|---|---|
| [Topología: submódulos + npm workspace](docs/decisiones/2026-08-06-topologia-repos.md) | 2026-08-06 | ecosistema |
| [Distribución de paquetes: npm público](docs/decisiones/2026-08-06-distribucion-npm-publico.md) | 2026-08-06 | ecosistema |
| [Distribución de renderers: dos canales](docs/decisiones/2026-08-06-distribucion-renderers.md) | 2026-08-06 | ecosistema |
| [Versionado de dependencias](docs/decisiones/2026-08-07-versionado-dependencias.md) | 2026-08-07 | ecosistema |
| [Flag de desarrollo de renderers](docs/decisiones/2026-08-06-flag-desarrollo-renderers.md) | 2026-08-06 | 🚚 `luxsequencer-core` |
| Backend de cloud: Supabase para todo | 2026-08-06 | 🚚 en `luxsequencer-cloud/STATUS.md` |

**No relitigar sin motivo nuevo.**

🚚 = pendiente de mudanza al repo que le corresponde. Ver [STATUS.md](STATUS.md).

## Trampas operativas (leer antes de tocar nada)

1. **Orden de arranque en dev:** `core-renderers` (`npm run dev`, puerto 4174, `strictPort`)
   **antes** que `luxsequencer-core`, que lo proxea en `/marketplace-core-renderers`. El atajo es
   `npm run dev:all` desde core. El 4174 devuelve 404 en `/` a propósito: no tiene `index.html`,
   sólo sirve archivos bajo `/src/`.

2. **Hay trabajo sin commitear en dos repos, y es coherente entre sí.** `core-renderers` tiene el
   renderer `diagnostic-fps/` entero sin trackear más `catalog.json` modificado, y
   `luxsequencer-core` tiene 5 archivos modificados incluido el `ALLOWED_RENDERERS` que lo
   registra. Es un cambio atómico cross-repo a medio terminar. **Confirmar con el autor antes de
   commitear, resetear o interpretar esto como abandonado.**

3. **Submódulos**: por defecto quedás en HEAD detached — hacé `git checkout <rama>` antes de
   trabajar. Y **nunca commitees un puntero a un commit que no pusheaste**, porque nadie más va a
   poder clonar.

4. **El caret en `0.x` es restrictivo**: `^0.1.0` es `>=0.1.0 <0.2.0`. Si subís
   `@luxsequencer/contracts` a `0.2.0`, npm **baja la `0.1.0` del registro en silencio** en vez de
   enlazar la carpeta local. Al bumpear una minor, actualizá los rangos de los consumidores en el
   mismo commit.

5. **`lux-ui` y `contracts` se consumen desde `dist/`, no desde `src/`.** El script `prepare` los
   recompila en cada `npm install`, pero si editás sus fuentes en caliente hay que rebuildear.

6. **Hay archivos `.env` reales** en `luxsequencer-core/` y `luxsequencer-cloud/`. Están en
   `.gitignore` (verificado). No volcar su contenido en informes ni en salidas de comandos.

## Metodología para las sesiones por proyecto

Para cada proyecto, producir el mismo entregable:

1. **Estado real ejecutado** — type-check, tests, lint, build. Números concretos, no "pasa".
2. **Mapa de arquitectura desde el código** — módulos, capas, flujo de datos, puntos de entrada.
3. **Tabla de drift** — afirmación de la doc │ realidad en el código │ evidencia (archivo:línea).
4. **Código muerto / infraestructura desconectada.**
5. **Deuda y riesgos**, ordenados por impacto.
6. **Preguntas abiertas para el autor** — decisiones que no se pueden inferir del código.

El resultado va al `docs/auditoria/` y al `STATUS.md` **del proyecto**, no acá.
