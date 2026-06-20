---
name: use-worktrees
description: Use BEFORE starting to implement any fix or feature in a code repository. Creates an isolated git worktree from the project's integration branch (staging/main) instead of editing the primary checkout. Keeps the main working tree clean and allows parallel feature work.
---

# Use Worktrees Always

> **Regla**: nunca implementar fixes o features en el checkout principal.
> Cada trabajo vive en su propio `git worktree`.

## Por qué

- **Aisla** cada fix/feature de los demás — vos podés tener 3-4 trabajos
  en paralelo sin tocar el árbol principal.
- **Permite paralelizar** trabajos: tests corriendo en uno, review en otro,
  desarrollo en un tercero.
- **Mantiene el árbol limpio**: si hay que volver a `main` rápido para un
  hotfix, no hay cambios sucios bloqueando.
- **El patrón ya está establecido** en los repos donde se usa
  (`repositorios/`, carpetas hermanas con worktrees nombradas por feature).

## Cuándo se invoca

Esta skill **debe invocarse antes** de cualquiera de estas acciones:

- Empezar a implementar un fix o feature solicitado por el usuario.
- Empezar a ejecutar un plan de `superpowers:writing-plans`.
- Aceptar una tarea que va a generar commits propios.

## Cuándo NO aplicar

- **Edición de archivos meta** (workspace tasks, memorias `~/.claude/`,
  configs personales) — esos viven fuera de los repos de código.
- **Cambios de un solo archivo trivial** que no necesitan PR propio
  (ej. fix de typo en un README). En la duda, sí worktree.
- **Hotfix urgente al checkout principal** que se va a commitear y mergear
  en menos de 5 minutos, con el usuario presente confirmando.

## Procedimiento

### 1. Identificar la base correcta

- Por defecto, la rama de integración del repo (verificar el `CLAUDE.md`
  del proyecto). En Kontigo: `staging`. En la mayoría de proyectos
  Innovares: `main`. En proyectos abiertos: lo que el README diga.
- Si la skill `feedback_branch_from_staging` o equivalente aplica al
  repo, respetarla.

### 2. Crear el worktree

Hay dos formas equivalentes:

**Opción A — via skill superpowers** (preferida si está disponible):

```
Invoke superpowers:using-git-worktrees
```

Esa skill se encarga de crear el worktree, validar que la base esté
actualizada, y dejarlo listo.

**Opción B — manual con git**:

```bash
# Desde la raíz del repo principal:
git fetch origin
git worktree add ../<repo>-<feature-name> origin/<base-branch>
cd ../<repo>-<feature-name>
git checkout -b feature/<descriptive-name>
```

Convención de naming de la carpeta:
- Hermana del checkout principal (`../<repo>-<feature>`).
- Nombre descriptivo del fix/feature, no del autor.
- Ejemplos reales del patrón Kontigo: `core-api-alchemy-resilience`,
  `core-api-megasoft-pci`.

### 3. Trabajar en el worktree

- Todos los edits, commits y tests se hacen ahí.
- Si la skill de testing aplica (`dev_caracas_core_api_tests` u
  equivalente del proyecto), los tests siguen corriendo en el ambiente
  que corresponda — el worktree solo aísla código, no infrastructure.

### 4. Limpieza al terminar

Cuando el PR se mergea (o se descarta):

```bash
cd <ruta del repo principal>
git worktree remove ../<repo>-<feature-name>
```

Si la rama local también queda obsoleta:

```bash
git branch -D feature/<descriptive-name>
```

## Detalles que la skill respeta

- **No commitea en el principal**: si por error se hicieron cambios en
  el checkout principal antes de invocar esta skill, primero se mueven
  con `git stash` al worktree nuevo, después se aplican.
- **No mezcla bases**: si un worktree fue creado desde `staging`, todos
  los commits siguientes parten de ahí; rebasar contra otra base es una
  decisión consciente, no un accidente.
- **Worktree por trabajo, no por sesión**: si Claude se reinicia y vuelve
  al checkout principal, antes de seguir trabajando, recordar la regla y
  retomar en el worktree apropiado.

## Resultado esperado

Cuando la skill se invoca correctamente, el siguiente mensaje del usuario
ve a Claude trabajando en un worktree con nombre claro, la rama correcta,
y el árbol del checkout principal limpio.
