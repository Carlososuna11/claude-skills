---
name: careful-destructive-ops
description: Usar antes de ejecutar cualquier operación de blast radius alto que sea difícil o imposible de revertir — rm -rf, git reset --hard, git push --force/--force-with-lease, git branch -D, DROP TABLE/DATABASE, sobreescribir archivos sin leer, killall, o cualquier flag --no-verify/--no-gpg-sign/--skip-checks. También aplica cuando un reviewer recomienda revertir una decisión ya acordada con el usuario.
---

# Careful Destructive Ops

## Overview

Las operaciones destructivas se ejecutan con confirmación explícita,
preview cuando aplique, y diagnóstico antes que bypass. Nunca usar un
shortcut destructivo para "hacer desaparecer" un obstáculo.

**Principio:** investigar el root cause antes que rodear el síntoma.

**Anunciar al inicio:** "Operación destructiva detectada (`<comando>`).
Aplicando `careful-destructive-ops`."

## Step 0: ¿La operación es realmente destructiva?

Esta skill aplica a comandos cuyo undo es: imposible, costoso, o
visible para otros (commits en main, force-push a branch protegida,
DELETE de filas, archivos sobreescritos).

```
Destructivo:                          NO destructivo:
- rm / rm -rf                         - mv, cp
- git reset --hard                    - git reset --soft, git stash
- git push --force / -f               - git push normal
- git branch -D                       - git branch -d (refuses if unmerged)
- git checkout -- / git restore .     - git checkout <archivo específico>
- DROP TABLE / TRUNCATE / DELETE all  - SELECT, UPDATE WHERE
- Sobreescribir archivo > out         - Append >>
- killall, kill -9                    - kill (SIGTERM)
- --no-verify, --no-gpg-sign          - flags normales
- gh repo delete                      - gh repo archive
```

Si está en la columna izquierda, seguir los steps. Si está en la
derecha, no aplica.

## Step 1: Confirmar con el usuario antes de ejecutar

Salvo que el usuario haya autorizado explícitamente este tipo de
operación en la conversación actual, **detenerse y preguntar**. Mostrar:

1. El comando exacto que se va a ejecutar.
2. Qué se va a perder/cambiar (archivos, branches, registros).
3. Por qué hace falta esa operación destructiva específicamente — no
   un equivalente más seguro.
4. Si existe alternativa reversible, mencionarla.

> "Voy a ejecutar `git reset --hard origin/main` — esto descarta los
> 3 commits locales en `<branch>` (X, Y, Z). Alternativa reversible:
> `git reset --keep` que mantiene los archivos no commiteados.
> ¿Confirmás `--hard`?"

Si la respuesta no es un sí explícito, no ejecutar.

## Step 2: Investigar antes que bypass

Cuando aparece un blocker (hook que falla, test que no pasa, lock file,
permission denied), la primera reacción **no puede ser** un flag que
lo evite (`--no-verify`, `--force`, `rm <lockfile>`). El blocker
suele señalar un problema real.

Antes de cualquier bypass:

1. Leer el output completo del error. No skimear.
2. Reproducir manualmente lo que falló (correr el hook a mano,
   ejecutar el test individual).
3. Documentar el mecanismo: qué hace, con qué input, qué output
   produce vs. el esperado.
4. Solo si el mecanismo está confirmado y el bypass es realmente
   inevitable, proceder con confirmación explícita (Step 1).

Confundir recencia con causalidad es el error más común. "Mi cambio
X coincide con que falla Y" no prueba que X cause Y — puede solo
haberlo expuesto.

## Step 3: Manejo de force-push de otros sobre tu branch

Si un `git push` falla con "rejected (non-fast-forward)" después de
que otro contribuidor empujó cambios sobre la misma branch (review
absorbing, rebase, squash):

1. **NO** hacer `git push --force` ciego ni rebase ciego.
2. `git fetch origin <branch>` — traer su HEAD nuevo.
3. Inspeccionar:
   ```bash
   git log <local-base>..origin/<branch> --oneline
   git show <each-sha> --stat
   ```
4. Para cada commit nuevo: ¿toca los mismos archivos que tú? ¿cambia
   signaturas que asumiste? ¿resuelve algo que ya tenías?
5. Si todo OK: `git reset --hard origin/<branch>` + `git cherry-pick
   <tus-shas>`. Resolver conflictos puntuales.
6. Re-correr tests para confirmar que tus cambios siguen aplicables
   al HEAD nuevo.
7. Mencionar en el PR qué cherry-pickeaste para que el otro
   contribuidor tenga contexto.

## Step 4: Verificar branch antes de commit en checkout compartido

Si el repo tiene un checkout principal que comparten varios procesos o
agentes (típico en monorepos con scripts concurrentes), antes de cada
`git add` / `git commit`:

```bash
git rev-parse --abbrev-ref HEAD
```

Confirmar que el branch actual es el esperado. Un proceso paralelo
puede haber cambiado el HEAD entre tus edits y tu commit. Recovery
si el commit cayó en branch ajeno:

```bash
git reflog                              # reconstruir secuencia
git branch -f <mi-branch> <mi-SHA>     # llevar commit a mi branch
git checkout <mi-branch>                # liberar el branch ajeno
git branch -f <branch-ajeno> <SHA-original>  # restaurar el ajeno
```

Para evitarlo de raíz: usar un git worktree aislado (ver skill
`use-worktrees`).

## Step 5: No revertir decisiones del usuario por recomendación de un reviewer

Si un reviewer (Codex, otra Claude, externo) recomienda revertir algo
que ya fue acordado explícitamente con el usuario en brainstorming o
spec, **flagear el conflicto antes de aplicar**:

> "El reviewer recomienda X (config A), pero acordamos Y (config B)
> porque <razón documentada>. ¿Cambiamos o mantenemos Y?"

Reviewers operan sobre código aislado, sin contexto de las decisiones
previas. Aplicar a ciegas revierte trabajo.

Esto aplica a: config de DataSource/migrations, env behavior, deployment
strategy, integration patterns, cualquier decisión arquitectónica del
design doc.

No aplica a: typos, bug fixes, refactors locales, tests — esos van sin
preguntar.

## Quick Reference

| Situación | Acción |
|---|---|
| `rm -rf` / `git reset --hard` / `git push --force` | Step 1 (confirmar) |
| Hook falla → tentación de `--no-verify` | Step 2 (investigar root cause) |
| Test rojo → tentación de `--skip-checks` | Step 2 (reproducir manual) |
| Push rechazado "non-fast-forward" | Step 3 (fetch + inspect, NO force) |
| Commit cayó en branch ajeno | Step 4 (reflog + branch -f) |
| Reviewer pide revertir decisión acordada | Step 5 (flagear, no aplicar) |
| `gh repo delete` / `DROP DATABASE` | Confirmar 2 veces, backup primero |
| Sobreescribir archivo (`>` o Write sin Read previo) | Read primero, después Write |

## Common Mistakes

- **`--no-verify` por defecto.** Los hooks pre-commit existen por razón —
  ejecutarlos manualmente revela el problema real.
- **`git push --force` en branches compartidas.** Reescribe historia que
  otros ya pulled. Usar `--force-with-lease` solo si entendiste qué
  prevee y por qué.
- **`git reset --hard` "para empezar de cero".** Pierde cambios no
  commiteados sin recovery. Usar `git stash` o `git reset --keep`.
- **Diagnosticar por correlación temporal.** "Mi commit fue antes del
  fallo → es mi causa." Pedirle el mecanismo a tu yo del futuro.
- **Aplicar fix de reviewer sin contextualizar.** Si contradice un
  acuerdo previo, frenar y consultar (Step 5).

## Red Flags

**Nunca:**
- Ejecutar `rm -rf`, `git reset --hard`, `git push --force`, `DROP
  TABLE` sin confirmación explícita del usuario en la conversación
  actual.
- Usar `--no-verify`, `--no-gpg-sign`, `--skip-checks` para "hacer
  pasar" un commit. Si el hook falla, hay un problema real.
- Asumir que un reviewer tiene contexto que tú sí tienes. Si la
  recomendación contradice un acuerdo previo, frenar.
- Hacer un segundo hotfix si el primero falló sin entender por qué
  falló el primero.
- Pushear a `main` directamente si el repo tiene staging — verificar
  el base branch siempre antes de mergear.

**Siempre:**
- Reproducir el problema localmente antes de hotfix a prod.
- Verificar el branch con `git rev-parse --abbrev-ref HEAD` antes de
  commit cuando el checkout es compartido.
- Mostrar el comando exacto + qué se pierde + alternativa reversible
  antes de pedir confirmación.
- Investigar el mecanismo del error antes de cualquier bypass.
