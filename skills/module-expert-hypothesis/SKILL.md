---
name: module-expert-hypothesis
description: Usar cuando el maintainer o dueño de un módulo da una pista, hipótesis u objeción casual sobre un bug, diseño o decisión que estás tomando. También cuando hay asimetría de conocimiento (alguien sabe la arquitectura mejor que tú) y vas a afirmar o empujar algo.
---

# Module Expert Hypothesis

## Overview

Cuando el dueño de un módulo deja caer una hipótesis casual ("yo
probaría con X", "esto suele ser por Y", "fijate si Z está activo"),
es información de alta señal: viene de alguien que conoce la
arquitectura, los bugs históricos y las trampas del código que tú
no viste.

**Principio:** la hipótesis del experto del módulo es la **primera a
refutar**, no a ignorar. Tratarla como hipótesis nula del análisis.

**Anunciar al inicio:** "Aplicando `module-expert-hypothesis` —
tomando la pista de `<persona>` como punto de partida."

## Qué cuenta como "experto del módulo"

- Quien escribió la primera versión del módulo.
- Quien hace la mayoría de los reviews ahí (`git log --format='%an'
  <archivo>` agrupado por autor).
- Quien debuggeó los últimos 2-3 incidentes de ese módulo.
- Quien es el on-call rotation owner del servicio.
- En equipos chicos: el tech lead o el dev senior del área.

Cuando esa persona te da una pista, **el costo de ignorarla y
equivocarte es mayor** que el costo de tomarla y verificarla.

## Step 1: Anotar la hipótesis textual antes de procesarla

Anotar exactamente lo que dijo el experto, sin reformular:

> "Massi dijo: 'fijate que `transform: true` esté en el
> ValidationPipe, eso suele faltar cuando se rompe la deserialización
> de fechas'."

Reformularla a tu lenguaje pierde precisión y sesga el análisis. Si
hace falta clarificar, preguntar al experto, no reformular.

## Step 2: Mapear cómo podría ser cierta

Antes de buscar refutar la hipótesis, hacer el ejercicio inverso:
**suponiendo que la hipótesis es correcta, ¿qué evidencia esperaríamos
ver?**

- Si "falta `transform: true`" → esperaríamos: `class-transformer` no
  convirtiendo objetos, propiedades `@Type()` ignoradas, `Date` objects
  llegando como strings.
- Si "el guard no se está ejecutando en ese path" → esperaríamos: el
  log del guard no aparece para esa request, o el endpoint no tiene
  el decorator del guard registrado.

Si las observaciones del bug **encajan** con lo que esperaríamos
bajo la hipótesis, la hipótesis es probablemente correcta. Verificar.

## Step 3: Verificar la hipótesis ANTES de buscar otras

Verificar el camino que indicó el experto:

1. Reproducir lo que dijo (correr el comando, mirar el archivo, leer
   el log que mencionó).
2. Confirmar si el mecanismo se cumple o no.
3. Solo si la hipótesis se refuta con evidencia clara, explorar otras
   causas.

Anti-patrón: ignorar la hipótesis del experto, gastar 2 horas
explorando otros caminos, descubrir que tenía razón. Costo doble:
las 2 horas + la pérdida de credibilidad cuando vuelves a decirle
"tenías razón".

## Step 4: Si la hipótesis es incorrecta, llevarle evidencia

Si después de verificar, la hipótesis del experto no aplicaba,
**no decir "ya lo probé" en abstracto**. Llevarle evidencia
concreta:

> "Probé tu pista. `transform: true` ya está activo en
> `app.module.ts:42`. El bug persiste igual. ¿Qué otra cosa
> probarías?"

Esto:
- Le ahorra el viaje mental de pensar que no le hiciste caso.
- Le da material para la siguiente hipótesis.
- Mantiene la conversación productiva.

## Step 5: Consultar antes de hotfix cuando la asimetría es grande

Si vas a empujar un hotfix en un módulo donde el experto sabe mucho
más que tú, **consultar antes**:

> "Estoy viendo `<síntoma>` en `<módulo>`. Mi hipótesis es `<X>`.
> ¿Te suena razonable o estoy en un camino equivocado?"

Mensaje de 30 segundos al experto, esperar respuesta de 1 minuto.
Mejor eso que empujar un hotfix que no resuelve y deja al equipo
debuggeando algo que no era.

Esto compone con `verify-before-claim` Step 5 (no encadenar hotfixes)
y con `pr-discipline` Step 1 (pre-alinear cambios arquitectónicos).

## Step 6: Documentar la lección

Cuando un experto del módulo te da una pista que termina siendo
correcta y tú la habías considerado de baja probabilidad,
**documentar la lección**. Plantilla:

> "En `<módulo>`, cuando aparece `<síntoma>`, la causa más probable
> es `<X>` — confirmado por `<persona>` en `<contexto>`."

Esto puede vivir en:
- Auto-memoria del agente si trabajas con uno.
- README del módulo (si la lección es estable y compartible).
- Tu propio playbook personal.

## Quick Reference

| Situación | Acción |
|---|---|
| El experto del módulo te da una pista casual | Step 1: anotar textual + Step 2-3: verificar primero |
| Estás por empujar hotfix en módulo ajeno | Step 5: consultar al experto antes |
| La hipótesis del experto no aplicaba | Step 4: llevarle evidencia, no "ya lo probé" |
| Acertaron y tú habías descartado la idea | Step 6: documentar la lección |
| No estás seguro quién es el experto | `git log --format='%an' <archivo>` + on-call rotation |

## Common Mistakes

- **Ignorar la pista por orgullo o prisa.** El costo de "ya lo
  estaba pensando distinto" suele ser mayor que el de verificar la
  hipótesis del experto.
- **Reformular la hipótesis a tu lenguaje.** Pierde precisión.
  Anotarla textual.
- **Decir "ya lo probé" sin evidencia.** El experto se queda sin
  material para la siguiente hipótesis.
- **Empujar hotfix sin consultar cuando hay asimetría.** Si te
  equivocas, el experto te va a corregir igual — mejor preguntar
  antes.
- **No registrar la lección** cuando tu modelo mental estaba
  equivocado. Vas a repetir el mismo error.

## Red Flags

**Nunca:**
- Ignorar la hipótesis del experto del módulo sin haberla verificado
  primero.
- Reformular la pista a tu lenguaje antes de procesarla.
- Decir "ya lo probé" sin evidencia concreta.
- Empujar hotfix en módulo ajeno con asimetría de conocimiento sin
  consultar.
- Insistir contra el experto sin haber refutado su hipótesis con
  evidencia.

**Siempre:**
- Anotar la hipótesis textual antes de procesarla.
- Verificar el camino del experto antes de explorar otros.
- Llevar evidencia concreta cuando refutes la hipótesis del experto.
- Consultar antes de hotfix cuando la asimetría de conocimiento es
  grande.
- Documentar la lección cuando tu modelo mental estaba equivocado.
