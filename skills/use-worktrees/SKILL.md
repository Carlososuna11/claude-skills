---
name: use-worktrees
description: Usar antes de iniciar cualquier fix, feature, refactor o chore que vaya a producir sus propios commits y PR — especialmente en repos donde la rama de integración no es main (puede ser staging, master, dev, develop, trunk). Cubre desde detectar aislamiento existente hasta abrir el PR contra la base correcta.
---

# Use Worktrees

## Overview

Cada trabajo vive en su propio workspace aislado y termina con un PR
hacia la rama de integración real del repo (no necesariamente `main`).

**Principio:** detectar aislamiento primero, preferir herramientas nativas,
caer a `git worktree` solo si hace falta. Nunca asumir la base — resolverla,
guardarla, y usarla hasta el final.

**Anunciar al inicio:** "Usando la skill `use-worktrees` para preparar el
workspace y resolver la rama base."

## Step 0: Detectar aislamiento existente

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
git rev-parse --show-superproject-working-tree 2>/dev/null   # vacío = no submódulo
```

- Si `GIT_DIR != GIT_COMMON` y no es submódulo → **ya estás en un worktree**.
  Saltar a Step 2.
- Si hay una tool nativa (`EnterWorktree`, `/worktree`, `--worktree`),
  usarla y saltar a Step 2. NO correr `git worktree add` cuando hay tool
  nativa — crea estado fantasma que la plataforma no maneja.

## Step 1: Resolver la rama base

Esta es la decisión que cambia entre proyectos. Probar en este orden,
**parar en el primer paso que dé respuesta sin ambigüedad**:

1. **Memoria previa.** ¿Existe `reference_base_branch_<repo>.md` en el
   auto-memory dir? → usar ese valor.
2. **Convención documentada del repo.** Buscar mención explícita de la
   rama de PRs en `CLAUDE.md`, luego `CONTRIBUTING.md`, luego `README.md`.
3. **Triggers de CI.** Revisar `.github/workflows/*.yml`: la rama que
   dispara los deploys suele ser la base.
4. **Default del remoto.**
   `gh repo view --json defaultBranchRef -q '.defaultBranchRef.name'`.
   Buen candidato pero no definitivo (algunos equipos mergean a `staging`
   aunque `main` sea el default).
5. **Preguntar al usuario** — único paso obligatorio si los anteriores
   dejaron duda. Ver abajo.

**Preguntar al usuario** solo si los 4 pasos anteriores no produjeron una
respuesta sin ambigüedad. Mostrar las ramas remotas y preguntar literal:

```bash
git branch -r | grep -v HEAD | head -10
```

> "Para abrir PRs en este repo, ¿cuál es la rama de integración?
> (suele ser `main`, `master`, `staging`, `dev`, `develop` o `trunk`)"

## Step 2: Persistir la base en memoria

Una vez resuelta, guardar como auto-memoria para que sesiones futuras no
vuelvan a preguntar. Convención exacta (matchea `reference_n8n_instances.md`,
`reference_ve_tools.md`, etc.):

**Ruta del archivo de memoria** (auto-memory dir; Claude conoce la ruta
exacta en su system prompt):

```
<auto_memory_dir>/reference_base_branch_<repo>.md
```

`<repo>` es el nombre del directorio del repo, en minúsculas y con guiones.

**Contenido exacto:**

```markdown
---
name: reference-base-branch-<repo>
description: Rama de integración del repo <repo>. Feature branches parten de aquí y PRs apuntan acá.
metadata:
  type: reference
---

**Repo:** `<owner>/<repo>` (path local: `<absolute path>`)
**Rama de integración:** `<branch>`
**Cómo se determinó:** <"usuario confirmó YYYY-MM-DD" | "CLAUDE.md" | "gh default + usuario confirmó">

Aplicar: `git worktree add` parte de `origin/<branch>`. `gh pr create --base <branch>`.
```

**Append a `MEMORY.md`** una sola línea, mismo estilo que las existentes:

```
- [Base branch <repo>](reference_base_branch_<repo>.md) — <branch>
```

Si no se puede resolver el auto-memory dir, escribir a `.claude/memory/`
en la raíz del repo y avisarle al usuario.

## Step 3: Crear el worktree

```bash
git fetch origin
BASE="<rama resuelta en Step 1>"
SLUG="<tipo>/<descriptive-slug>"   # feature/, fix/, refactor/, chore/, docs/, perf/, test/
PATH_WT="../$(basename "$PWD")-${SLUG##*/}"

# Precondiciones
git ls-remote --exit-code --heads origin "$BASE" >/dev/null || { echo "Base $BASE no existe en origin"; exit 1; }
[ -e "$PATH_WT" ] && { echo "$PATH_WT ya existe"; exit 1; }
git show-ref --verify --quiet "refs/heads/$SLUG" && { echo "Branch $SLUG ya existe local"; exit 1; }

git worktree add -b "$SLUG" "$PATH_WT" "origin/$BASE"
cd "$PATH_WT"
```

Si el repo usa otra convención de prefijos (`bugfix/`, ID de ticket),
respetarla — verificar `git log --oneline -20` y `CONTRIBUTING.md`.

## Step 4: Setup + baseline

```bash
[ -f package.json ]    && npm install
[ -f pyproject.toml ]  && (poetry install 2>/dev/null || pip install -e .)
[ -f requirements.txt ] && pip install -r requirements.txt
[ -f Cargo.toml ]      && cargo build
[ -f go.mod ]          && go mod download

# Baseline: tests verdes antes de tocar nada
<comando de tests del proyecto>
```

Si los tests fallan en baseline, **parar y reportar** — no se puede
distinguir un bug nuevo de uno preexistente.

## Step 5: Trabajar

- Todos los edits y commits ocurren en el worktree.
- Si por error hay cambios en el checkout principal, `git stash` ahí y
  `git stash pop` en el worktree.

## Step 6: Cerrar con PR

```bash
# Precondiciones
gh auth status >/dev/null 2>&1 || { echo "gh no autenticado"; exit 1; }
git fetch origin

# Default seguro: fast-forward merge. Rebase SOLO si la branch nunca fue
# pusheada o el equipo lo manda explícitamente.
git merge --ff-only "origin/$BASE" || { echo "No FF-able. Resolver manual."; exit 1; }

git push -u origin "$SLUG"

# El --base es OBLIGATORIO. No confiar en lo que gh infiera.
gh pr create --base "$BASE" --head "$SLUG" \
  --title "<tipo>: <descripción corta>" \
  --body  "<descripción según convención del repo>"

# Verificar antes de declarar éxito
gh pr view --json url,baseRefName -q '"PR " + .url + " → " + .baseRefName'
```

## Report (al terminar)

```
Worktree: <full path>
Branch:   <slug> (base origin/<BASE>)
Setup:    <ok | falló>
Baseline: <N tests pasando | falló>
PR:       <url o "pendiente">
```

## Quick Reference

| Situación | Acción |
|---|---|
| Ya en worktree linked | Saltar a Step 2 |
| Hay tool nativa de worktree | Usarla, no `git worktree add` |
| Memoria `reference_base_branch_<repo>.md` existe | Usar ese valor sin preguntar |
| Base declarada en `CLAUDE.md` o `CONTRIBUTING.md` | Usarla, no preguntar |
| Ningún paso del waterfall fue concluyente | Preguntar al usuario |
| `BASE` no existe en `origin` | Abortar antes de crear worktree |
| Path del worktree ya existe | Abortar, elegir otro slug |
| Branch local ya existe | Abortar, elegir otro slug |
| Tests fallan en baseline | Parar y reportar |
| `gh auth status` falla | Pedirle al usuario que autentique |
| `merge --ff-only` falla | Resolver manual; no rebasar automático |

## Common Mistakes

- **Asumir `main` como base.** Resolver siempre con el waterfall del Step 1.
- **`git rebase` automático.** Cambia historia y puede romper colaboradores;
  usar `merge --ff-only` por default.
- **`gh pr create` sin `--base`.** GitHub usa el default del repo, que no
  siempre es la base correcta.
- **Crear worktree dentro del repo principal.** Usar carpeta hermana
  (`../<repo>-<slug>`) o `.worktrees/` solo si está en `.gitignore`.
- **Saltar Step 0.** Crear worktree dentro de otro worktree rompe estado.
- **Escribir la memoria con guiones (`reference-base-branch-...`)** cuando
  el dir usa underscores (`reference_base_branch_...`). Match exacto.

## Red Flags

**Nunca:**
- Correr `git worktree add` cuando hay tool nativa disponible.
- Hacer commit en el checkout principal sin invocar esta skill primero.
- Abrir un PR sin `--base` explícito.
- Rebasear automáticamente — siempre `merge --ff-only` salvo orden explícita.
- Declarar el PR abierto antes de verificar con `gh pr view`.

**Siempre:**
- Step 0 (detectar aislamiento) antes que nada.
- Resolver la base con el waterfall y persistirla en memoria.
- Verificar precondiciones (`gh auth`, base existe en remoto, path libre).
- Cerrar reportando el resultado de `gh pr view` literal.
