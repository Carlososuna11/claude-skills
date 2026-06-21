---
name: comments-discipline
description: Usar antes de escribir o editar un comentario en el código (`//`, `/* */`, `#`, `"""..."""`, docstrings), un comentario en TECH_DEBT/BACKLOG, descripciones de tests, o cualquier texto que viva DENTRO de archivos del repo y se vaya a versionar. NO aplica a PR descriptions, commit messages, chat, ni mensajes a otros devs.
---

# Comments Discipline

## Overview

Los comentarios en código viven para siempre. Mal escritos, mienten:
quedan desactualizados, parafrasean código que ya se lee solo, o
referencian tareas que nadie va a buscar.

Bien escritos, son la única forma de transmitir **por qué** el código
hace algo no obvio — algo que ningún identificador puede expresar.

**Principio:** los comentarios son en **inglés**, **explicativos del
WHY**, y **no específicos** de la tarea que los originó.

**Anunciar al inicio:** "Aplicando `comments-discipline` antes de
escribir este comentario."

## Las 3 reglas no negociables

### Regla 1 — Siempre en inglés

Independiente del idioma del equipo, los comentarios en código se
escriben en **inglés**:

```typescript
// ❌ Mal: en español
// validamos que el monto sea positivo antes de continuar

// ✅ Bien: en inglés
// Reject negative amounts here so downstream can assume normalized input
```

Razones:
- El código sobrevive a la rotación del equipo. El próximo dev puede
  no hablar el idioma actual.
- Si el repo va open source, los comentarios en español tienen que
  reescribirse.
- Las APIs, librerías, error messages y stack traces están en inglés —
  mezclar idiomas crea fricción cognitiva al leer.

**Excepciones legítimas:**
- Strings que se muestran al usuario final (UI text).
- Comentarios sobre dominio cultural específico (ej. "tasa BCV venezolana").
- Tests de localización donde el idioma es parte del test.

### Regla 2 — Explicativos del WHY, no del WHAT

Los comentarios deben explicar lo que el código **no puede decir por
sí solo**: intención, restricciones ocultas, decisiones de diseño,
trade-offs.

```python
# ❌ Mal: parafrasea el código
# Loop through users and check if active
for user in users:
    if user.is_active:
        ...

# ✅ Bien: explica el WHY
# Skip inactive users — they may have stale OAuth tokens that crash
# the downstream sync (see incident 2025-09-12).
for user in users:
    if user.is_active:
        ...
```

Qué SÍ comentar:
- **Constraints ocultas:** "must be called before X, otherwise Y."
- **Invariantes:** "this list is sorted by created_at ascending."
- **Workarounds:** "library bug #123: setting this to undefined
  triggers a crash; null is safe."
- **Decisiones de diseño no obvias:** "we deliberately read from
  replica even though it's slightly stale, because primary has lock
  contention here."
- **Edge cases sutiles:** "empty string is intentional — the API
  treats null and '' differently."

Qué NO comentar:
- Lo que el nombre de la función/variable ya dice (`// fetch user`
  encima de `function fetchUser()`).
- El flujo de control obvio (`// loop over items`).
- Cualquier explicación que se podría inferir leyendo 5 líneas.

### Regla 3 — No específicos de la tarea que los originó

Un comentario es para quien venga a leer el código dentro de 6 meses,
no para tu reviewer de hoy.

```typescript
// ❌ Mal: referencias a contexto ephemeral
// TODO: arreglar para el ticket KON-1234
// Fix por la review de Massi (PR #4521)
// Cambio temporal mientras esperamos respuesta de Stripe

// ✅ Bien: el porqué permanente
// Stripe webhook payload may arrive before our internal record is
// committed; retry once with exponential backoff before failing.
```

Razones:
- Las IDs de tickets, PRs, sprints, devs se vuelven irrelevantes
  rápido.
- `TODO` sin acción concreta vive para siempre y nadie lo limpia.
- Decir "temporal" cuando no hay deadline lo vuelve permanente.

**Si necesitas referenciar un ticket o incidente**, hacerlo de forma
que sobreviva al contexto:

```typescript
// Workaround for Stripe API issue, see GitHub issue #1234.
// Safe to remove once Stripe ships the fix mentioned in their roadmap.
```

## Step 1: Antes de escribir el comentario, preguntarse

3 preguntas en orden:

1. **¿Lo que voy a escribir es WHY o WHAT?** Si es WHAT, borrarlo.
2. **¿Está en inglés (salvo excepción legítima)?** Si no, traducir.
3. **¿Va a tener sentido en 6 meses sin contexto de esta semana?**
   Si no, reescribirlo sin referencias ephemeral.

Si las tres son sí, el comentario aporta. Si alguna falla,
reescribir o no escribir.

## Step 2: Descripciones de tests

Los `describe`/`it`/`test` strings son comentarios visibles. Misma
regla:

```typescript
// ❌ Mal
describe('verifica que el usuario activo se procese', () => {
  it('debería retornar exito', () => { ... });
});

// ✅ Bien
describe('processActiveUser', () => {
  it('returns success when token is valid', () => { ... });
  it('rejects when token has expired tier mismatch', () => { ... });
});
```

## Step 3: TECH_DEBT, BACKLOG, ROADMAP en repo

Cuando se escribe en archivos de deuda técnica o backlog que viven
en el repo:

- Misma regla de inglés (estos archivos también sobreviven al
  equipo).
- Describir el **problema y el costo**, no el dev que lo flageó:

```markdown
# ❌ Mal
- TODO (Carlos): refactorizar el AuthGuard

# ✅ Bien
- AuthGuard duplicates token parsing logic that already lives in
  AuthService. Refactor to delegate, removes ~40 lines and one source
  of drift.
```

## Quick Reference

| Situación | Acción |
|---|---|
| Voy a parafrasear el código | No escribir el comentario |
| Voy a explicar el WHY de algo no obvio | Escribirlo, en inglés, sin referencias ephemeral |
| Voy a poner `TODO` con id de ticket | Reescribir como problema + costo, sin ID volátil |
| Comentario sobre dominio cultural específico | Excepción legítima: puede ir en idioma local |
| Test description en español | Traducir a inglés |
| `// fix de la review de X` | Reescribir como WHY permanente o borrar |
| Comentario "temporal mientras..." | Si no hay deadline, no es temporal — reescribir como permanente o borrar |

## Common Mistakes

- **Parafrasear el código.** "Loop over users" encima de
  `for user in users`. Aporta cero.
- **Comentar en el idioma del equipo.** Cuesta nada escribir en
  inglés y le sirve a todo lector futuro.
- **Pegar IDs de tickets sin contexto.** `// KON-1234` no significa
  nada cuando Linear/Jira cambia o el ticket se cierra.
- **Decir "temporal" sin condición de eliminación.** Se vuelve
  permanente.
- **Test descriptions narrativas en español.** Mezcla idiomas en
  un mismo archivo y pierde búsqueda por keyword.
- **Comentar lo que se infiere en 5 líneas.** Si el lector lo va a
  ver leyendo igual, el comentario ocupa espacio sin agregar.

## Red Flags

**Nunca:**
- Escribir comentarios en español/idioma local cuando el código se
  versiona (salvo excepciones legítimas).
- Parafrasear el código en el comentario.
- Incluir IDs de tickets, PRs, sprints, nombres de devs en
  comentarios versionados.
- Decir "temporal", "TODO", "FIXME" sin condición concreta de
  eliminación o issue público linkeado.
- Comentar el WHAT que el nombre de la función ya dice.

**Siempre:**
- Antes de escribir, preguntar: ¿WHY o WHAT? ¿Inglés? ¿Útil en 6
  meses?
- Explicar la intención, restricción oculta, invariante, workaround.
- Si necesitas referenciar un problema, linkear a issue/PR público,
  no a ticket interno o nombre de dev.
- Aplicar la misma regla a TECH_DEBT, BACKLOG, ROADMAP y test
  descriptions que viven en el repo.
