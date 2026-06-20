---
name: use-worktrees
description: Use BEFORE starting any code change (fix or feature). Creates an isolated git worktree from the project's integration branch, opens a descriptive feature branch in it, and when the work is done helps open the PR back to that base. Asks the user for the integration branch the first time in each repo (main, master, staging, dev, develop, trunk...) and remembers it for the rest of the session.
---

# Use Worktrees (con detección de base + apertura de PR)

> Regla: nunca tocar el checkout principal para implementar fixes o
> features. Cada trabajo vive en su propio `git worktree` con su propio
> feature branch, y termina con un PR hacia la rama de integración del
> repo (la que el usuario haya definido como base).

## Por qué

- **Aisla** cada fix/feature de los demás — puedes tener 3-4 trabajos en
  paralelo sin tocar el árbol principal.
- **Permite paralelizar**: tests corriendo en un worktree, revisión en
  otro, desarrollo en un tercero.
- **Mantiene el árbol limpio**: si hay que volver a la rama de integración
  rápido para algo urgente, no hay cambios sucios bloqueando.
- **Cierra el ciclo correctamente**: el PR va a la rama que el equipo
  realmente usa como integración (no siempre `main` — puede ser `master`,
  `staging`, `dev`, `develop`, `trunk`, etc.).

## Cuándo se invoca

Antes de cualquiera de estas acciones:

- Empezar a implementar un fix o feature solicitado por el usuario.
- Empezar a ejecutar un plan generado por una skill de planeación.
- Aceptar una tarea que va a generar commits propios.

## Cuándo NO aplicar

- Edición de archivos meta (workspace tasks, memorias `~/.claude/`,
  configs personales) — esos viven fuera de los repos de código.
- Cambios triviales de un solo archivo que no necesitan PR (typo en
  README, comentario). En la duda, sí worktree.
- Hotfix urgente al checkout principal que se va a commitear y mergear
  en menos de 5 minutos, con el usuario presente confirmando.

## Procedimiento

### Paso 1 — Determinar la rama de integración (base) del repo

Esta es **la decisión más importante** y la única que cambia entre
proyectos. La skill busca la base en tres lugares, en orden:

1. **Memoria previa**. Buscar en el sistema de auto-memoria de Claude un
   archivo con la convención `reference_base_branch_<repo-name>.md`. Si
   existe, leerlo y usar el valor que indique. No preguntar de nuevo.

2. **Convenciones documentadas del repo**. Si no hay memoria, revisar
   en este orden:
   - El `CLAUDE.md` del repo (suele decir cuál es la rama de PRs).
   - El `CONTRIBUTING.md` o `README.md`.
   - Workflow files en `.github/workflows/` (la rama que dispara deploys
     suele ser la de integración).

3. **Default branch del remoto**. Si nada documenta la convención:
   ```bash
   gh repo view --json defaultBranchRef -q '.defaultBranchRef.name'
   ```
   Esto da un buen candidato, pero no es definitivo (algunos equipos
   usan `staging` como integración aunque `main` sea el default).

4. **Preguntar al usuario** (paso obligatorio si los anteriores no dan
   certeza). Mostrar las ramas remotas para que el usuario elija:
   ```bash
   git branch -r | head -10
   ```
   Y preguntar literalmente:
   > "Para abrir PRs en este repo, ¿cuál es la rama de integración
   > a la que apuntan los feature branches? (suele ser `main`,
   > `master`, `staging`, `dev`, `develop` o `trunk`)"

   Esperar la respuesta. Confirmar antes de seguir.

### Paso 2 — Guardar la decisión en memoria

Una vez confirmada la base, **guardar como auto-memory** para no volver
a preguntar. Crear el archivo de memoria con este contenido:

```markdown
---
name: reference-base-branch-<repo-name>
description: Rama de integración del repo <repo-name>. Todos los feature branches parten de aquí y los PRs apuntan acá.
metadata:
  type: reference
---

**Repo**: `<owner>/<repo-name>` (path local: `<absolute path>`)
**Rama de integración**: `<branch>`

**Cómo se determinó**: <"el usuario lo confirmó el YYYY-MM-DD" |
"está documentado en CLAUDE.md" | "es el default del remoto y el
usuario confirmó">

**Cómo aplicar**:
- `git worktree add` parte de `origin/<branch>`.
- Los PRs se abren con `gh pr create --base <branch>`.
- Si el equipo cambia la convención, actualizar esta memoria y notificar.
```

Y agregar la línea correspondiente al `MEMORY.md` index.

### Paso 3 — Crear el worktree con su feature branch

```bash
# Desde la raíz del checkout principal:
git fetch origin

# Carpeta hermana, nombre descriptivo del fix/feature
WORKTREE_PATH="../<repo>-<feature-slug>"
BRANCH_NAME="<feature|fix|chore>/<descriptive-slug>"
BASE_BRANCH="<el branch resuelto en Paso 1>"

git worktree add -b "$BRANCH_NAME" "$WORKTREE_PATH" "origin/$BASE_BRANCH"
cd "$WORKTREE_PATH"
```

Convenciones de naming:

| Tipo de trabajo | Prefijo del branch | Ejemplo |
|---|---|---|
| Feature nueva | `feature/` | `feature/onboarding-revamp` |
| Bug fix | `fix/` | `fix/cache-invalidation-stale-data` |
| Refactor | `refactor/` | `refactor/extract-payment-validator` |
| Tarea menor | `chore/` | `chore/upgrade-node-22` |
| Docs solamente | `docs/` | `docs/api-reference-v3` |

Si el repo usa otra convención (`bugfix/`, `task/`, IDs de ticket,
etc.), respetarla. Verificar en commits previos o `CONTRIBUTING.md`.

Carpeta del worktree:
- Hermana del checkout principal (`../<repo>-<feature-slug>`).
- Nombre descriptivo del trabajo, no del autor.
- Ejemplos: `api-cache-invalidation-bug`, `frontend-onboarding-revamp`,
  `infra-db-migration-v2`.

### Paso 4 — Trabajar en el worktree

- Todos los edits, commits y tests se hacen en `$WORKTREE_PATH`.
- Si el proyecto tiene un ambiente especial para tests (servidor de CI
  remoto, contenedor compartido), los tests siguen corriendo donde
  corresponda — el worktree solo aísla código, no infraestructura.
- Si por error se hicieron cambios en el checkout principal antes de
  invocar esta skill, mover con `git stash` al worktree nuevo y aplicar.
- Cada commit en su propio mensaje claro. Si el repo tiene una skill o
  convención de mensajes de commit, respetarla.

### Paso 5 — Cerrar con PR a la base correcta

Cuando el trabajo está listo (tests pasan, lint limpio, review propio
hecho):

```bash
# Asegurar que estamos al día con la base
git fetch origin
git rebase "origin/$BASE_BRANCH"   # o merge, según convención del repo

# Push del feature branch al remoto
git push -u origin "$BRANCH_NAME"

# Abrir el PR apuntando a la base correcta (NO al default branch a ciegas)
gh pr create --base "$BASE_BRANCH" --head "$BRANCH_NAME" \
  --title "<tipo>: <descripción corta>" \
  --body "<descripción siguiendo la convención del proyecto>"
```

> **Crítico**: el `--base` es la rama de integración guardada en memoria
> (Paso 1), no el default que `gh` use por su cuenta. Verificar antes
> de presionar enter.

Si el proyecto tiene una skill o convención específica para abrir PRs
(checklist de seguridad, formato de descripción, mención de tickets),
invocarla o seguirla antes del `gh pr create`.

### Paso 6 — Limpieza al terminar

Cuando el PR se mergea (o se descarta):

```bash
cd <ruta del checkout principal>
git worktree remove "../<repo>-<feature-slug>"

# Y si la rama local también queda obsoleta:
git branch -D "<feature|fix|chore>/<descriptive-slug>"
```

## Detalles que la skill respeta

- **No commitea en el principal**: si por error se hicieron cambios en
  el checkout principal antes de invocar esta skill, se mueven con
  `git stash` al worktree nuevo y se aplican ahí.
- **No mezcla bases**: si un worktree fue creado desde la base resuelta
  en Paso 1, todos los commits siguientes parten de ahí. Rebasar contra
  otra base es una decisión consciente, no un accidente.
- **Worktree por trabajo, no por sesión**: si Claude se reinicia y vuelve
  al checkout principal, antes de seguir trabajando, recordar la regla
  y retomar en el worktree apropiado.
- **Memoria de la base sobrevive sesiones**: una vez que el usuario
  confirmó "este repo usa `dev` como integración", esa decisión queda
  en `reference_base_branch_<repo>.md` y la próxima sesión la respeta
  sin volver a preguntar. Si el equipo migra (ej. `master` → `main`),
  actualizar la memoria.

## Resultado esperado

Cuando esta skill se invoca correctamente:

1. El usuario sabe (y la memoria registra) cuál es la rama de
   integración del repo.
2. Se crea un worktree hermano con un nombre claro.
3. Se crea un feature branch con prefijo y nombre descriptivos,
   partiendo de la base correcta.
4. Todo el trabajo vive ahí; el checkout principal queda intacto.
5. Al finalizar, el PR se abre apuntando a la base correcta (no al
   default del remoto a ciegas).
