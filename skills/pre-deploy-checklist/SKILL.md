---
name: pre-deploy-checklist
description: Usar antes de cualquier deploy a staging o producción, antes de mergear a la rama que dispara deploys, o antes de crear un release tag. También al pasar una feature de "lista de implementar" a "lista de revisar".
---

# Pre-Deploy Checklist

## Overview

Hacer deploy sin verificar significa descubrir el problema en
producción — donde el costo es máximo. Esta skill es la checklist
explícita que separa "el código compila" de "el código está listo
para enfrentar usuarios reales".

**Principio:** nada va a producción sin haber pasado por una serie de
gates objetivos, cada uno con evidencia.

**Anunciar al inicio:** "Aplicando `pre-deploy-checklist` —
verificando gates antes del deploy."

## Step 1: Tests verdes (con output, no fe ciega)

Correr la suite completa y traer el output:

```bash
# Stack típico — adaptar al proyecto
npm test
pytest
go test ./...
cargo test
./gradlew test
```

Verificar:

- Conteo final: `X passed, 0 failed, Y skipped`.
- Si hay `skipped` nuevos, justificar cada uno.
- Si hay tests flakey conocidos, correrlos 2-3 veces para confirmar
  que pasan estable, no por suerte.

**No declarar "tests pasan" sin pegar el output**. Eso es afirmación
sin verificación.

## Step 2: Lint sin warnings nuevos

```bash
# Patrones típicos
npm run lint
ruff check .
golangci-lint run
cargo clippy
```

Reglas:

- **Errores**: bloquean. Si hay error, fix antes de deploy.
- **Warnings nuevos**: si el PR introduce warnings que no existían
  antes, fix o justificar explícitamente (no dejar deuda silenciosa).
- **Warnings preexistentes**: no son scope del PR, pero documentar
  si crecieron (señal de drift).

Si el repo tiene `pre-commit hooks` activos, el lint corre solo. Si
no, esta es la chance.

## Step 3: Format aplicado

```bash
# Patrones típicos
npm run format          # prettier, biome, etc.
ruff format .
gofmt -l . | wc -l      # debería ser 0
cargo fmt --check
```

Reglas:

- Format se aplica antes del commit, no después.
- Si el format change incluye archivos que no tocaste, separar en
  commit dedicado (`chore: format`) para no contaminar el diff del
  feature.

## Step 4: Typecheck pasando

Para stacks con tipos estáticos:

```bash
# TypeScript
npx tsc --noEmit

# Python con mypy
mypy src/

# Go: tipos siempre — pero confirmar build
go build ./...
```

Reglas:

- `any` / `dict` / `interface{}` nuevos requieren justificación.
- Si typecheck pasa local pero no en CI, es por versión distinta de
  la lib de tipos o flags distintos. Alinear `tsconfig.json`/`pyproject.toml`
  entre local y CI.

**Importante** (regla específica): **no correr `npm run build` con
`next dev` u otro server de desarrollo activo** — puede corromper el
`.next/`. Para typecheck en frontend, usar `npx tsc --noEmit`.

## Step 5: Migraciones revisadas

Si el PR incluye migrations de schema (ver skill `migrations-always`):

```bash
# Probar el ciclo completo local
<migration-cli> upgrade head
<migration-cli> downgrade -1
<migration-cli> upgrade head
```

Verificar:

- **No destructivas en tablas grandes** sin plan (drop column, change
  type, NOT NULL sin default).
- **Reversibles**: el down existe y funciona.
- **Idempotentes** si tu sistema permite re-run accidental.
- **Datos vs schema separados**: si la migration necesita backfill
  de datos, está en su propia migration data, no mezclada con el
  schema.

## Step 6: Env vars del nuevo deploy

Revisar:

- ¿Hay env vars **nuevas** que el deploy actual no tiene? Si sí,
  setearlas en staging/prod **antes** del deploy.
- ¿Cambió el nombre o el formato de alguna existente? Comunicar.
- ¿`.env.example` está actualizado con las nuevas?
- ¿Alguna env nueva es secreto? Configurarla por el panel del
  hosting, no en el código.

## Step 7: Smoke test post-deploy

Después del deploy (no antes), verificar que **lo más crítico
funciona**:

| Tipo de app | Smoke test mínimo |
|---|---|
| Backend API | `GET /health` o `GET /` retorna 200 + uno de los endpoints principales con auth |
| Frontend web | Cargar home + login + una ruta autenticada |
| Worker / queue | Verificar que se procesa al menos un job nuevo |
| Cron / scheduled | Esperar al próximo trigger y verificar logs |

Si el smoke test falla, **rollback inmediato** — no intentar fix en
caliente.

## Step 8: Logs y monitoring 5–30 min post-deploy

Después del smoke test, observar:

- Logs por errores nuevos que antes no aparecían.
- Métricas de latencia (no degradó dramáticamente).
- Sentry / error tracking por excepciones que no existían.
- Si hay canaries / blue-green, verificar el switch.

Documentar en el PR / Slack del equipo:

> "Deploy a producción de PR #X completado HH:MM. Smoke test
> passing. Logs estables 15 min post-deploy. Sin nuevos errores."

## Quick Reference

| Gate | Comando típico | Bloquea deploy si |
|---|---|---|
| Tests | `npm test` / `pytest` | Cualquier rojo |
| Lint | `npm run lint` / `ruff check` | Errores |
| Format | `npm run format` / `gofmt -l` | Diff no aplicado |
| Typecheck | `npx tsc --noEmit` / `mypy` | Errores nuevos |
| Migrations | `<cli> upgrade head + downgrade -1 + upgrade head` | Down rota o destructiva sin plan |
| Env vars | Manual vs `.env.example` | Nuevas no seteadas en prod |
| Smoke test | `curl /health` + login flow | 5xx o response no esperado |
| Monitoring | Sentry / dashboard | Errores nuevos en 15 min post |

## Common Mistakes

- **"Tests pasan" sin pegar el output.** No es verificación.
- **Saltar typecheck "porque CI lo va a correr".** Local + CI deben
  estar alineados; CI a veces tarda.
- **Format change masivo mezclado con feature.** Contamina el diff
  e impide revisar el cambio real.
- **Migration con `NOT NULL` sin default en tabla grande.** Lock de
  tabla en prod, downtime.
- **Env nueva no seteada en prod antes del deploy.** El servicio
  arranca y crashea inmediatamente.
- **No smoke test post-deploy.** Te enteras 2 horas después por un
  cliente.
- **Cerrar el laptop apenas se hace deploy.** Los problemas aparecen
  en los 15 minutos siguientes.

## Red Flags

**Nunca:**
- Hacer deploy con tests rojos, lint con errores, o typecheck
  fallando.
- Saltar gates con `--no-verify`, `--skip-checks`, `--force`.
- Hacer migration destructiva en tabla grande sin plan documentado.
- Hacer deploy sin haber actualizado env vars nuevas en el ambiente
  destino.
- Hacer deploy y no monitorear 15 minutos después.
- Hacer "fix en caliente" en producción para tapar un smoke test
  fallido — rollback primero.

**Siempre:**
- Pegar el output de tests (no decir "pasan").
- Lint, format y typecheck verdes antes de mergear.
- Ciclo `up + down + up` para migrations.
- Actualizar `.env.example` cuando agregás env nuevas.
- Smoke test mínimo post-deploy + 15 min de monitoring.
- Documentar el deploy en el canal del equipo (qué, cuándo, estado).
