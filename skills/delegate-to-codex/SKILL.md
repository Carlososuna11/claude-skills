---
name: delegate-to-codex
description: Usar antes de invocar a Codex (OpenAI) como subagente — para ejecución mecánica (aplicar fixes con diff claro, refactors, tests, migrations siguiendo patrón) o para análisis que alimenta una decisión (code review profundo, second opinion adversarial, brainstorming generativo, investigación cross-repo). También al armar el prompt y al validar el resultado.
---

# Delegate to Codex

## Overview

Codex (OpenAI) puede correr Bash, git, tests, y trabajar de forma
autónoma sobre tareas con spec claro. Delegar a Codex libera tokens
del modelo principal para trabajo donde el contexto vivo importa:
coordinación, decisiones, conversación con el usuario.

**Principio:** Codex **genera material** (diffs, findings, alternativas,
hallazgos); el modelo principal **decide** con el usuario. Delegar el
análisis es bueno; delegar la decisión final no.

**Anunciar al inicio:** "Delegando a Codex: `<tarea>`. Razón: `<por qué
califica para delegación>`."

## Step 1: ¿Esta tarea califica para delegación?

### Sí delegar — Ejecución mecánica

- Aplicar fixes específicos de code review (cuando hay diff + reasoning
  claro del reviewer).
- Refactors mecánicos: rename, mover archivos, consolidar branches
  similares, extraer función.
- Implementar siguiendo un patrón ya establecido en el codebase ("hacé
  X siguiendo el patrón de Y").
- Escribir tests para código que ya existe.
- Migrations entre versiones de un patrón (ej. migrar de API v1 a v2).

### Sí delegar — Análisis que alimenta una decisión

- **Code review profundo**: pedirle a Codex que revise un PR completo
  o un flow end-to-end y devuelva findings concretos (con archivo:línea,
  severidad, sugerencia).
- **Second opinion**: cuando tienes una decisión ~70% definida, pedirle
  a Codex que la critique adversarialmente. Suele traer argumentos no
  considerados.
- **Brainstorming generativo**: "dame 3 enfoques alternativos para
  resolver X, con tradeoffs concretos". Codex genera, tú decides con
  el usuario cuál tomar.
- **Investigación profunda**: leer muchos archivos / docs y devolver
  un resumen estructurado de hallazgos. Útil cuando el costo de tokens
  de leer todo en el modelo principal es alto.

### No delegar

- La **decisión final** sobre arquitectura, scope, prioridades. Codex
  puede sugerir; la decisión la tomas tú con el usuario.
- Coordinación con humanos (mandar mensajes, abrir tickets, ping a
  maintainers).
- Curaduría de auto-memoria — eso depende de tu lectura de la
  conversación.
- Tareas que requieren mantener contexto de la conversación viva con
  el usuario (ej. "según lo que acabamos de discutir, hacé X").

**Test rápido:** "¿El resultado de Codex es material para que yo decida,
o estoy pidiéndole que decida por mí?" Si lo primero, delegá. Si lo
segundo, no.

## Step 2: Armar el prompt para Codex

Codex no tiene tu contexto de conversación. El prompt debe ser
**autocontenido**. Plantilla:

```markdown
## Tarea
<una frase: qué cambiar, en qué archivos>

## Contexto del repo
- Ruta absoluta: <path>
- Branch actual esperado: <branch>
- Comando de tests: <ej. npm test, pytest, cargo test>
- Comando de lint/format: <ej. ruff check, eslint .>

## Cambios específicos
1. En `<archivo:línea>`: <qué cambiar>
2. En `<archivo>`: <qué crear/borrar>

## Referencias
- Pattern a seguir: `<archivo>` líneas X-Y
- Docs relevantes: <link o path>
- Issue/PR relacionado: <#número>

## Verificación
- Correr `<comando>` y confirmar que pasa
- Verificar que `<comportamiento>` sigue funcionando

## Reporte esperado
- Archivos modificados (con SHA del commit si commiteaste)
- Tests que pasaron / fallaron
- Posibles efectos colaterales que detectaste
- ¿Pusheaste? (default: no, solo commit local para review)

## Reglas
- NO pasar secrets en outputs.
- NO hacer `git push --force`.
- NO `--no-verify` ni `--no-gpg-sign`.
- Si hay duda, parar y reportar — no improvisar.
```

Ajustar a la tarea. El "Contexto del repo" y el "Reporte esperado" son
no-negociables.

## Step 3: Elegir el modelo correcto

Codex (cuando se usa con cuenta ChatGPT) solo admite los modelos
listados en su cache local de modelos. Pasar uno fuera del cache
falla en <30 segundos con `invalid_request_error: The 'X' model is
not supported when using Codex with a ChatGPT account`.

**Cómo verificar los modelos válidos:** consultar la documentación
del plugin de Codex instalado en este entorno. Suele exponer un
helper o un archivo cache (ej. `~/.codex/models_cache.json`) que
lista los slugs admitidos para la cuenta activa. Si no encuentras
referencia, pídeselo al usuario antes de dispatchar.

Guía general (ajustar al cache actual):

| Tarea | Effort recomendado |
|---|---|
| Review arquitectural serio / decisiones críticas | Frontier disponible, effort xhigh |
| Code review tradicional | Modelo estándar, effort medium |
| Tarea mecánica / rápida | Variante mini |

## Step 4: Verificar que Codex realmente arrancó

El subagente puede reportar `Thread ready` y `Turn started` y aún así
fallar en <30s — modelo no soportado, workspace desactivado (billing),
o tarea malformada.

Consulta el status del job recién dispatchado **antes** de declarar
que está corriendo. La forma exacta depende del runtime del plugin
(suele haber un comando `status --json` en el companion script del
plugin de Codex). Mira:

- Conteo de jobs en `Running` → debe ser ≥1 si tu job arrancó.
- `latestFinished.status` para tu `jobId` → si es `failed`, leer el
  error y relanzar.

**Falla común — workspace desactivado (HTTP 402):** la cuenta ChatGPT
tiene el workspace de Codex desactivado (billing/estado). No es
problema de modelo. Mientras se reactiva, hacer el trabajo en el
modelo principal.

## Step 5: Recibir el reporte y validar

Cuando Codex termina, devolver al usuario su reporte verbatim si el
flujo de la skill que invocaste pide eso (ver `codex:rescue` skill
docs).

Antes de aceptar el trabajo:

1. Mirar el diff: `git log --oneline -5` + `git diff <sha-anterior>..HEAD`.
2. Confirmar que los tests realmente corrieron y pasaron (no solo que
   Codex lo afirmó).
3. Verificar que el commit identity es la correcta (ver
   `pr-discipline` Step 4).
4. Si Codex pusheó cuando le habías dicho que no, **flagearlo al
   usuario** — no aceptar silenciosamente.

Si el trabajo está mal, **no improvisar el fix**. Reportar al usuario
qué falló y decidir si:
- Re-dispatch con prompt corregido.
- Tomar el trabajo de Codex y completarlo en el modelo principal.
- Descartar y empezar de cero.

## Quick Reference

| Situación | Acción |
|---|---|
| Fix de review con diff claro | Delegar (ejecución) |
| Refactor mecánico (rename, mover) | Delegar (ejecución) |
| Implementar siguiendo pattern existente | Delegar (ejecución) |
| Tests para código existente | Delegar (ejecución) |
| Code review profundo de PR/flow | Delegar (análisis) |
| Second opinion adversarial | Delegar (análisis) |
| "Dame 3 enfoques con tradeoffs" | Delegar (brainstorm generativo) |
| Investigación cross-repo (leer mucho) | Delegar (análisis) |
| Decidir cuál enfoque tomar | NO delegar (decisión tuya + usuario) |
| Coordinación con humanos | NO delegar |
| Curaduría de memoria | NO delegar |
| Pedirle continuación de conversación viva | NO delegar |
| Modelo no en `models_cache.json` | NO usar, falla en 30s |
| `Thread ready` pero status `failed` | Verificar y relanzar |

## Common Mistakes

- **Delegar sin contexto del repo.** Codex no sabe la ruta absoluta,
  el branch, ni el comando de tests si no se lo dices.
- **Asumir que `Thread ready` = está corriendo.** Verificar status.
- **Pasar un modelo que no soporta la cuenta.** Falla silenciosa en 30s.
- **Aceptar el reporte sin verificar el diff.** Codex puede afirmar
  "tests pass" y no haberlos corrido. Validar siempre.
- **Delegar trabajo que requiere juicio.** El roundtrip de prompt +
  ejecución + validación cuesta más que hacerlo directo.
- **Pasarle un secret en el prompt.** Codex loguea; el output queda
  en transcript.

## Red Flags

**Nunca:**
- Delegar la **decisión final** (Codex puede sugerir; tú decides con
  el usuario).
- Delegar coordinación con humanos.
- Delegar curaduría de auto-memoria.
- Pasar secrets, tokens, passwords en el prompt o el contexto.
- Aceptar el reporte sin mirar el diff o los findings.
- Permitir `git push --force` o `--no-verify` en el dispatch.
- Usar un modelo que no esté en `models_cache.json`.

**Siempre:**
- Prompt autocontenido (Codex no tiene tu contexto).
- Pedir reporte estructurado (findings, archivos, comandos corridos,
  posibles efectos colaterales).
- Verificar status después de dispatchar.
- Mirar el diff o los findings antes de aceptar el trabajo.
- Confirmar atribución del commit.
- Reportar al usuario si Codex hizo algo distinto a lo pedido (ej.
  pusheó cuando dijiste que no).
