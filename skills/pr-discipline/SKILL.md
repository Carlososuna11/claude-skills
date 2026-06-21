---
name: pr-discipline
description: Usar antes y durante cualquier flujo de PR — abrir, revisar, comentar, mergear. Cubre alineación previa con el maintainer en cambios arquitectónicos, formato de title/body, atribución de commits (`-c user.email`), markdown limpio en gh CLI, estilo de comentarios de review, estrategia de merge (squash vs merge), y la regla de nunca mergear directo a main en repos con staging.
---

# PR Discipline

## Overview

Los PRs cuestan tiempo del equipo. La disciplina acá es prevenir el
desperdicio (re-review por scope sorprendente, merge a la rama
equivocada, atribución incorrecta) antes que producirlo.

**Principio:** consenso previo cuando toca arquitectura, formato
consistente siempre, merge target verificado dos veces.

**Anunciar al inicio:** "Aplicando `pr-discipline` para este PR."

## Step 1: Alinear con el maintainer ANTES de codear (si aplica)

Mandar un párrafo de proposal al maintainer del módulo **antes** de
abrir el PR si el cambio toca:

- Forma de respuesta de endpoints existentes.
- Estructura de DTOs o validators.
- Patrones cross-repo (validators, services, manejo de errores).
- Abstracciones arquitectónicas (decoradores nuevos, capas de
  servicio, patrones a nivel de módulo).

Plantilla:

> "[Maintainer], voy a hacer X para feature Y. Scope: a, b, c. Otros
> X quedan como están por Z razón. ¿OK?"

Esperar acknowledgment. Si hay push-back, iterar en texto (barato).
Si aprueba, el PR pasa sin debate post-facto.

**No aplica a:** bug fixes que no cambian contratos, features que
siguen patrones existentes, refactors internos a un módulo, docs o
tests solos.

## Step 2: Verificar la base correcta del PR

Antes de crear la branch, confirmar la rama de integración del repo
(no asumir `main`). Ver skill `use-worktrees` Step 1 — waterfall:

1. Memoria `reference_base_branch_<repo>.md`.
2. `CLAUDE.md` / `CONTRIBUTING.md`.
3. `.github/workflows/`.
4. `gh repo view --json defaultBranchRef`.
5. Preguntar al usuario.

Repos comunes: feature → staging → main (backend con rolling buffer),
feature → main (mobile/web simple), feature → dev → master (gitflow).

## Step 3: Reutilizar antes que crear

Antes de añadir un archivo, abstracción, helper, validator, guard,
servicio, componente o constante **nueva**, buscar primero si el
repositorio ya tiene algo equivalente.

Crear paralelos cuando ya existe un patrón establecido es la causa
número uno de los comentarios de revisión del tipo "esto ya existe,
usa el patrón actual" — y obliga a rehacer el trabajo. Más allá del
review, importa por tres razones:

- **Consistencia interna**: si hay tres formas de validar lo mismo,
  el próximo dev no sabe cuál usar.
- **Mantenimiento**: si la regla cambia, hay que tocar N lugares en
  vez de uno.
- **Curva de onboarding**: leer un repo con múltiples patrones
  equivalentes cuesta más que uno con uno solo.

### Qué buscar según el tipo de cambio

| Vas a crear... | Busca primero... |
|---|---|
| Validator de input / schema | `validators/`, archivos `*.validator.*`, schemas Zod/Joi/Pydantic existentes |
| Guard, interceptor, middleware | Carpetas equivalentes (`guards/`, `interceptors/`, `middleware/`) |
| Servicio / repositorio | Patrones de inyección (`@Injectable`, factories, providers, módulos) |
| Endpoint admin u operativo | Controllers de tools / admin existentes |
| Helper de formato (dinero, fechas, números) | Archivos en `utils/`, `helpers/`, `lib/` por nombre similar |
| Componente UI | `components/` por funcionalidad similar, no solo por nombre |
| Test | Convención del repo: `__tests__/`, `*.spec.*`, `*.test.*` |
| Constante / config / enum | Archivos de constants, enums, config consolidados |
| Tipo / interfaz / DTO | Tipos compartidos en `types/`, `dto/`, `models/` |

### Cómo buscar — comandos universales

```bash
# Por concepto en el código
grep -rni "<concepto>" src/ | head -20

# Por exportación de símbolos similares
grep -rn "export.*<concepto>" src/ | head -20

# Por estructura de carpetas
fd -t d "<nombre>" src/        # o: find src -type d -iname "*<nombre>*"

# Si el repo usa barrel files / index re-exports
grep -rn "<concepto>" src/**/index.* 2>/dev/null
```

Adaptar a la herramienta disponible (`rg` si está, `fd`, o `grep`/
`find` estándar). El comando importa menos que el reflejo de buscar
antes de crear.

### Cuándo sí está bien crear paralelos

- El patrón existente está deprecado y el equipo migró a uno nuevo.
  Verificar en `CHANGELOG`, ADRs, o preguntar al maintainer.
- El caso de uso es genuinamente distinto y el patrón actual no
  aplica. Esto se documenta en el body del PR para que el revisor
  no lo marque como duplicado.

**Prueba rápida antes de codear:** "Si hubiera buscado treinta
segundos antes de empezar, ¿lo habría encontrado?" Si la respuesta
es sí, había que buscar.

## Step 4: Crear el PR — title, body, atribución

### Title

- Bajo 70 caracteres. Lo largo va al body.
- Formato del repo (Conventional Commits si está; si no, frase corta).
- En el idioma del equipo (si el equipo escribe en español, el title
  va en español).

### Body

Plantilla mínima:

```markdown
## Summary
<1-3 bullets de QUÉ cambia y POR QUÉ>

## Test plan
- [ ] <comando o flujo a verificar>
- [ ] <otro>

## Risk
<edge cases, rollback plan, dependencias>
```

Si el repo tiene template (`.github/PULL_REQUEST_TEMPLATE.md`), usarlo
como base.

### Atribución (commit identity)

**Nunca pasar `-c user.email=...` ni `-c user.name=...`** a `git
commit`. El repo del trabajo tiene config local correcta; el global
suele ser personal. Override accidentalmente atribuye commits del
trabajo a la cuenta personal de GitHub.

Verificar antes de pushear:

```bash
git log -1 --format='%an <%ae>'
```

Si el email no es el de trabajo, **no pushear** — corregir antes:

```bash
git commit --amend --reset-author   # usa la config local actual
```

No reescribir historia de `main`/`staging` (branches protegidas) solo
por atribución — el costo del force-push supera el beneficio.

## Step 5: Subir markdown con backticks limpios

`gh pr edit --body-file` y `gh pr comment --body` **escapan los
backticks** a `\``, rompiendo el formato. Solución: `gh api -X PATCH`
con JSON payload.

```bash
python3 <<'PY'
import json, subprocess
with open('/tmp/body.md') as f:
    body = f.read()
subprocess.run(
    ['gh', 'api', '-X', 'PATCH', 'repos/<org>/<repo>/pulls/<num>', '--input', '-'],
    input=json.dumps({'body': body}), text=True, check=True
)
PY
```

Verificar:

```bash
gh pr view <num> --json body --jq .body | head -10
```

Si los backticks salen literales (no escapados), está limpio.

## Step 6: Estilo de comentarios de review

- Agrupar por severidad: **Bloqueante** / **Mejoras menores** / **Nits**.
- Un bullet por hallazgo, en una línea: qué + por qué + fix sugerido.
- Formato `archivo.ts:42` para referenciar líneas (clickable en
  GitHub).
- Lead con la razón antes que con la observación. Como notas en
  margen de un peer, no como auditoría externa.
- Si el autor documentó su reasoning en el PR body, **engagear con
  ese reasoning** — no reinventarlo desde primeros principios.
- Sin tags artificiales tipo "B1", "M2". Sin code blocks de ejemplo
  cuando no aportan.

Lo que un reviewer humano experimentado nota como "AI dump":
estructura demasiado simétrica, mismas frases de transición, bullets
sin contexto del autor.

## Step 7: Mergear — squash vs merge según el flujo del equipo

Verificar la convención del repo. Patrones comunes:

| Repo type | feature → integration | integration → main |
|---|---|---|
| Backend con staging | **squash** | **merge commit** |
| Mobile/web sin staging | **squash** o merge | N/A |
| Gitflow tradicional | merge a dev | merge a master |

**Por qué la asimetría en repos con staging:** si haceés squash de
staging → main, el PR parece gigante (repite N features ya
squasheados). El merge commit preserva la lista de features tal
cual cayeron en staging.

```bash
# feature → staging
gh pr merge --squash <num>

# staging → main (en repos que separan)
gh pr merge --merge <num>
```

**NUNCA** mergear directo a main si el repo tiene staging:

- `git revert` undoes el código pero los commits quedan en historia
  para siempre — visibles para todo el equipo.
- Triple-checkear el `--base` de `gh pr create` y el target del merge
  en GitHub UI antes de presionar.
- Preguntarle al usuario `"¿mergeo a staging?"` antes de ejecutar
  `gh pr merge` si hay duda.

## Quick Reference

| Situación | Acción |
|---|---|
| Cambio toca contratos o arquitectura | Step 1: alinear con maintainer |
| Crear branch | Step 2: resolver base con waterfall |
| Empezar implementación | Step 3: buscar si el patrón ya existe en el repo |
| Listo para abrir PR | Step 4: title <70, body con plantilla, verificar atribución |
| Body o comment con código | Step 5: `gh api PATCH` con JSON |
| Dejar review | Step 6: bullets por severidad, lead con razón |
| Mergear | Step 7: squash vs merge según convención |
| Repo con staging | NUNCA mergear directo a main |

## Common Mistakes

- **Abrir PR arquitectónico sin pre-alinear.** Produce 2-3 rondas
  de re-review y debate post-facto. Mandar el párrafo de proposal
  antes de codear.
- **Crear archivo o abstracción nueva cuando ya existe equivalente.**
  Buscar primero (Step 3). El revisor va a apuntarlo.
- **Title > 70 chars con la descripción completa.** Title corto, el
  detalle va al body.
- **Pasar `-c user.email` para commits scriptados.** Sobreescribe la
  config local de trabajo con el global personal.
- **`gh pr edit --body-file file.md` con backticks.** Los escapa.
  Usar `gh api PATCH` con JSON.
- **Reviews en prosa narrativa.** El usuario quiere bullets por
  severidad — escaneable, no narrativa.
- **Squash de staging → main.** Repite history que ya fue squasheado
  en cada feature. Usar merge commit.
- **Mergear directo a main "porque es chico".** Los commits quedan
  en historia para siempre, visibles para todos.

## Red Flags

**Nunca:**
- Abrir PR que toca contratos/arquitectura sin alineación previa.
- Crear archivo o abstracción paralela cuando ya existe equivalente.
- Pasar `-c user.email` / `-c user.name` a `git commit`.
- Pushear sin verificar `git log -1 --format='%ae'`.
- Usar `gh pr edit --body-file` para markdown con backticks.
- Mergear directo a main en repo con staging.
- Squashear staging → main.
- Dejar review sin agrupar por severidad.

**Siempre:**
- Pre-alinear cambios arquitectónicos con el maintainer.
- Resolver la base del PR con el waterfall, no asumir `main`.
- Buscar si el patrón ya existe en el repo antes de crear uno nuevo.
- Title <70 + body con Summary + Test plan + Risk.
- Verificar atribución del commit antes de push.
- `gh api PATCH` con JSON para markdown con código.
- Bullets por severidad en reviews, lead con razón.
- Squash en feature → integration, merge commit en integration → main.
