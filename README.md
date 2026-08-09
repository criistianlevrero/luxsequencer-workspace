# LuxSequencer — entorno de desarrollo unificado

Repositorio **privado e interno**. No contiene código propio: orquesta los cinco proyectos del
ecosistema como submódulos y los une en un único workspace de npm.

> Este README describe **únicamente lo que existe hoy**.
> Protocolo de estado: [`STATUS-PROTOCOL.md`](STATUS-PROTOCOL.md).
> Contexto para sesiones con agentes: [`CLAUDE.md`](CLAUDE.md).
>
> **Última verificación**: 2026-08-06

## Para qué existe

Los cinco proyectos son repositorios independientes y se pueden clonar por separado. Este repo
resuelve dos cosas que ninguno resuelve solo:

1. **Un `npm install`** que instala y enlaza los cinco, con iteración en vivo entre ellos.
2. **Un registro de qué combinación de commits funciona junta**, vía los punteros de submódulo.

No es necesario para desarrollar renderers ni para contribuir a un proyecto suelto.

## Clonar

```bash
git clone --recurse-submodules git@github.com:criistianlevrero/luxsequencer-workspace.git
cd luxsequencer-workspace
npm install
```

> **`--recurse-submodules` no es opcional.** Sin eso las carpetas quedan vacías y `npm install`
> falla, porque los workspaces no existen. Si ya clonaste sin el flag:
> `git submodule update --init --recursive`

## Los proyectos

| Directorio | Paquete | Repositorio | Rama | Distribución |
|---|---|---|---|---|
| `luxsequencer-core` | `luxsequencer` | `criistianlevrero/luxsequencer` | `0.6-beta` | Repo público |
| `core-renderers` | `@luxsequencer/core-renderers-repo` | `criistianlevrero/luxsequencer-marketplace-official` | `main` | Repo público |
| `lux-ui` | `@luxsequencer/ui` | `criistianlevrero/lux-ui` | `main` | **npm público** |
| `luxsequencer-contracts` | `@luxsequencer/contracts` | `criistianlevrero/luxsequencer-contracts` | `main` | **npm público** |
| `luxsequencer-cloud` | `luxsequencer-cloud` | `criistianlevrero/luxsequencer-cloud` | `main` | Cerrado |

## Cómo se resuelven las dependencias

Los consumidores declaran rangos semver, no rutas:

```json
"@luxsequencer/contracts": "^0.1.0",
"@luxsequencer/ui": "^0.1.0"
```

- **Dentro de este workspace**: npm ve que las versiones locales satisfacen el rango y
  **enlaza las carpetas locales**. Editás contracts y core lo ve al rebuildear.
- **Clonando un proyecto suelto**: npm baja los paquetes del registro. Funciona sin tener los
  otros cuatro repos.

Ese doble comportamiento es todo el motivo del workspace.

### El lockfile vive acá

Hay **un solo `package-lock.json`, en esta raíz**. Los proyectos no tienen el suyo: dentro de un
workspace npm los ignora, y mantenerlos sólo generaba divergencia.

Contrapartida asumida: quien clone un proyecto suelto no tiene lockfile, así que su instalación
no es reproducible.

## Trampas conocidas

**El caret en `0.x` es restrictivo.** `^0.1.0` significa `>=0.1.0 <0.2.0`. Si subís
`@luxsequencer/contracts` a `0.2.0`, el rango deja de cumplirse y npm **baja la `0.1.0` del
registro en silencio** en vez de enlazar tu carpeta local: vas a editar código que nadie consume,
sin ningún error. Al bumpear una minor, actualizá los rangos de los consumidores en el mismo
commit.

**Los submódulos apuntan a commits, no a ramas.** Si trabajás dentro de un submódulo, hacés
`git checkout <rama>` primero: por defecto quedás en HEAD detached. Y **nunca commitees un puntero
a un commit que no pusheaste**, porque nadie más va a poder clonar.

**Actualizar los punteros** tras avanzar en los subproyectos:

```bash
git submodule update --remote          # trae la punta de la rama declarada
git add <proyecto> && git commit       # registra la combinación nueva
```

## Documentación

| Documento | Contenido |
|---|---|
| [`CLAUDE.md`](CLAUDE.md) | Índice y reglas de trabajo. **Empezar por acá** |
| [`STATUS.md`](STATUS.md) | Estado del ecosistema y progreso de la auditoría |
| [`STATUS-PROTOCOL.md`](STATUS-PROTOCOL.md) | Cómo se separa lo planeado de lo implementado |
| [`docs/decisiones/`](docs/decisiones/) | Decisiones que cruzan repos, una por archivo y con fecha |
| [`docs/auditoria/`](docs/auditoria/) | Informes de auditoría a nivel ecosistema |
| [`docs/next-steps/`](docs/next-steps/) | Trabajo planeado que excede un proyecto |
| `<proyecto>/STATUS.md` | Estado por capacidad de cada proyecto |
| `<proyecto>/docs/` | Decisiones y hallazgos acotados a ese proyecto |

**Regla de scope**: un documento vive en el repo al que pertenece. La raíz es sólo para lo que
excede el scope de un proyecto.
