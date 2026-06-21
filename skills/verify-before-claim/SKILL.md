---
name: verify-before-claim
description: Usar antes de declarar causa raíz de un bug, antes de afirmar que un fix funciona, antes de decir "esto es seguro" o "no es explotable", antes de empujar un hotfix a producción, o antes de marcar una tarea como "ya verificado" o "no bloqueante". También al recibir presión para diagnosticar rápido.
---

# Verify Before Claim

## Overview

Afirmar sin verificar es la fuente principal de hotfixes ruidosos,
diagnósticos errados, y pérdida de confianza con el equipo. La regla
es simple: **no afirmar antes de poder explicar el mecanismo y haber
visto evidencia directa**.

**Principio:** correlación temporal no es causalidad. "Mi commit X
coincide con que falla Y" no prueba que X cause Y — puede solo
haberlo expuesto.

**Anunciar al inicio:** "Aplicando `verify-before-claim` antes de
declarar `<la afirmación>`."

## Qué cuenta como "afirmación que requiere verificación"

- "Esto es la causa del bug."
- "El fix funciona."
- "Ya está verificado / no es bloqueante."
- "No es explotable / no hay riesgo de seguridad."
- "Los tests pasan."
- "El gate cubre este caso."
- "El validator se ejecuta en este path."
- "El cambio no afecta producción."

Si el output que estás a punto de generar contiene una de estas, esta
skill aplica.

## Step 1: Explicar el mecanismo antes de afirmar

Antes de declarar causa raíz o que un fix funciona, **escribir el
mecanismo paso a paso**:

1. ¿Qué función entra?
2. ¿Con qué input concreto?
3. ¿Qué output produce vs el esperado?
4. ¿En qué archivo:línea ocurre la divergencia?

Si no puedes contestar las cuatro, no tienes el diagnóstico todavía —
solo una hipótesis. Decirlo así: *"hipótesis: X. Mecanismo pendiente
de confirmar."*

## Step 2: Reproducir localmente antes de hotfix

Si tienes acceso a stack local (Docker compose, dev server, contenedor),
**reproducir el bug ahí antes de empujar el fix a producción**. 10
minutos de reproducción local ahorran horas de hotfixes fallidos.

```bash
# Patrones comunes según stack
docker compose up -d              # stack completo local
npm run start:dev / pnpm dev      # backend en hot reload
pytest tests/path/test_bug.py     # test específico que reproduce
```

Si no puedes reproducir local (env-specific, prod data, race
condition), **decirlo explícitamente** antes de afirmar el fix:
*"no reproduje local; deploy a staging primero y verifico ahí"*.

## Step 3: Confirmar que el gate corre en ESE path

Existir un gate (validator, guard, check, branch en `if`) no es lo
mismo que correr en el path que te interesa. Verificar con:

```bash
# Buscar todos los call sites del gate
grep -rn "<NombreDelGate>" src/

# Trazar el path desde el entry point
# (route handler → service → repository → DB call)
```

Errores típicos del "el gate existe":
- El validator está registrado globalmente pero un controller usa
  `@SkipAuth` o equivalente.
- La función helper se importa en N archivos pero solo se invoca en
  N-1.
- El check pasa por short-circuit antes de llegar (`if (!user) return`
  antes de validar permisos).

## Step 4: Leer docs oficiales antes de adivinar

Cuando un bug depende del comportamiento de un framework/library
(NestJS, Django, Next.js, FastAPI, React, etc.), **leer las docs
oficiales primero** — 5 minutos de docs suelen ganar a 2 horas de
trial-and-error.

Reglas:
- Si la lib tiene config con varias opciones (`transform: true`,
  `strict: true`, `whitelist: true`), las docs explican qué hace cada
  una.
- Si una función tiene un comportamiento sorprendente, buscar el
  issue/PR en GitHub donde se discutió ese comportamiento.
- Si la lib cambió mayor versión, las docs viejas mienten — verificar
  versión actual del repo.

## Step 5: No encadenar hotfixes sin entender el anterior

Si el primer fix **no funcionó**, **no empujes un segundo de inmediato**.
Cada hotfix fallido erosiona confianza con el equipo y suele indicar
que el diagnóstico inicial estaba equivocado.

Pasos correctos cuando un fix falla:

1. Volver a Step 1: ¿el mecanismo que afirmaste era correcto?
2. Reabrir la posibilidad de que la causa sea otra.
3. Consultar al maintainer del módulo si hay asimetría de conocimiento
   (otro dev sabe la arquitectura mejor que tú).
4. Reproducir local con el nuevo entendimiento.

## Step 6: Reportar con evidencia, no con afirmación

Cuando reportas al usuario, llevar **evidencia** junto a la afirmación:

❌ "Los tests pasan."
✅ "Los tests pasan: `pytest tests/auth/` → 24 pasaron, 0 fallaron."

❌ "El fix está aplicado."
✅ "El fix está aplicado en `src/auth/guard.ts:45`. `git diff HEAD~1
   src/auth/guard.ts` muestra el cambio. Tests de regresión: `npm
   test -- --grep auth` → verde."

❌ "No es explotable."
✅ "No es explotable porque `<mecanismo>`. Verificado intentando
   `<vector>` y obteniendo `<rejection>`."

## Quick Reference

| Situación | Acción antes de afirmar |
|---|---|
| "Esta es la causa del bug" | Step 1: escribir mecanismo paso a paso |
| "El fix funciona" | Step 2: reproducir local + Step 6: evidencia |
| "El gate cubre este caso" | Step 3: trazar el path, no asumir |
| "No es bloqueante" | Step 6: justificar con evidencia |
| "Los tests pasan" | Step 6: pegar el output del comando |
| Bug depende de framework/lib | Step 4: leer docs oficiales primero |
| Primer fix no funcionó | Step 5: rediagnosticar, no encadenar hotfix |
| No puedes reproducir local | Decirlo explícito antes de afirmar |

## Common Mistakes

- **Recencia = causalidad.** "Mi commit fue antes del fallo → soy la
  causa." Falso si tu commit solo expuso un bug latente.
- **Hipótesis sin mecanismo.** "Creo que es por X" sin poder explicar
  cómo X produce el síntoma. No es diagnóstico.
- **Confiar en que un gate existe sin trazar su invocación.** Existir
  ≠ ejecutarse en ese path.
- **Hotfix encadenado sin entender por qué falló el primero.** Cada
  fallo nuevo erosiona confianza más que el primero.
- **Adivinar comportamiento del framework en vez de leer docs.** Las
  docs lo explican y tardan 5 minutos.
- **Reportar "tests pasan" sin pegar el output.** Si lo afirmaste, el
  usuario lo va a verificar igual — mejor traerlo de una.

## Red Flags

**Nunca:**
- Declarar causa raíz sin escribir el mecanismo paso a paso.
- Empujar hotfix a producción sin haber reproducido local (o sin
  haber dicho explícitamente que no pudiste).
- Afirmar "el gate cubre este caso" sin trazar las invocaciones del
  gate desde el entry point.
- Empujar un segundo hotfix de inmediato si el primero falló.
- Decir "los tests pasan" sin haber corrido los tests en este
  contexto.

**Siempre:**
- Diferenciar "hipótesis" de "diagnóstico confirmado" en lo que dices
  al usuario.
- Llevar evidencia junto a la afirmación (output de comando,
  archivo:línea, captura).
- Leer docs oficiales cuando el bug depende del framework.
- Reabrir el diagnóstico si el fix falla, no doblar la apuesta.
- Reconocer asimetría de conocimiento y consultar al experto del
  módulo antes de afirmar.
