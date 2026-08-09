> **Estado**: PLANEADO · **Última verificación**: 2026-08-07 · **Alcance**: ecosistema

# Meta-repo de DX para terceros

## Qué es

Un segundo meta-repo, análogo a `luxsequencer-workspace` pero **público**, para desarrolladores
externos de renderers. Agrupa los proyectos abiertos (`luxsequencer-core`, `core-renderers`) y
lleva su propio **`package-lock.json` commiteado**.

## Qué problema cierra

Hoy quien clona un repo suelto no tiene lockfile, así que resuelve dependencias a ciegas. Con
este meta-repo el externo clona, corre `npm ci`, y obtiene **el árbol exacto** que se validó.

Resuelve la reproducibilidad mejor que fijar versiones a mano en `package.json`: un lockfile fija
el árbol completo (~991 entradas), fijar versiones exactas fija sólo el primer nivel (~40).

## Nota operativa

**No editar el lockfile a mano.** Es un grafo resuelto: las entradas se referencian entre sí y
las claves son rutas que dependen del layout.

El procedimiento que sí funciona:

1. Copiar el `package-lock.json` **entero** desde `luxsequencer-workspace`.
2. Correr `npm install` (no `ci`) en el repo nuevo.
3. npm prefiere las versiones que ya están en el lockfile cuando satisfacen los rangos: poda las
   entradas de workspaces que no existen, reescribe la estructura y **conserva las versiones**.
4. Verificar con `npm ls` que coincidan.
5. Commitear el lockfile resultante.

La instrucción para el externo tiene que ser `npm ci`, no `npm install` — `ci` respeta el
lockfile estrictamente.

## Alcance a definir

- ¿Incluye sólo `core` + `core-renderers`, o también un template de renderer nuevo?
- ¿Los proyectos entran como submódulos, igual que en el workspace interno?
