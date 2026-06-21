---
name: no-premature-abstraction
description: Usar antes de proponer una nueva tabla, clase, helper, capa, decorator, estado global, URL param, o cualquier abstracción que vaya más allá del scope del problema concreto. También cuando el impulso es "ya que estoy acá, refactorizo esto otro" o "agrego este flag por si en el futuro".
---

# No Premature Abstraction

## Overview

Agregar features, refactors o abstracciones más allá de lo que el
problema concreto pide es la causa principal de PRs sobre-scopeados,
deuda escondida, y código que nadie usa.

**Principio:** YAGNI. Tres líneas similares es mejor que una
abstracción prematura. Diseñar para hoy, no para el "por si acaso"
hipotético.

**Anunciar al inicio:** "Aplicando `no-premature-abstraction` —
evaluando si esta abstracción es necesaria ahora."

## Qué cuenta como abstracción prematura

- Crear una clase/función genérica cuando hay un solo caller.
- Agregar un parámetro opcional "por si lo necesitamos después".
- Crear una nueva tabla cuando el campo cabe en una existente.
- Crear un módulo nuevo para tres líneas que viven bien en el actual.
- Generalizar un helper antes de tener el segundo caso de uso.
- Agregar un feature flag para algo que el usuario no pidió.
- Refactorizar código ajeno al fix actual "ya que estoy acá".
- Crear capas de indirección (service → repository → mapper → DTO)
  cuando el problema cabe en una.

## Step 1: ¿El problema concreto pide esto?

Antes de proponer la abstracción, escribir **el problema concreto**
en una oración:

> "El bug es: cuando un usuario X hace Y, el sistema Z falla con W."

Luego escribir la solución **mínima**:

> "Mínimo: cambiar línea N para que Z no falle cuando Y."

Si tu propuesta agrega más cosas que esa solución mínima, **detenerse
y justificar cada extra**. "Lo agregué de paso" no es justificación.

## Step 2: ¿Realmente hay un segundo caso de uso?

Regla de tres: **no abstraer hasta el tercer caso real**.

| Cantidad de casos | Acción correcta |
|---|---|
| 1 caso | Hardcoded, sin abstracción |
| 2 casos | Duplicar; revisar si vale la pena abstraer en el tercero |
| 3+ casos | Considerar abstraer, evaluando si los casos comparten *forma* además de *nombre* |

Casos que parecen iguales pero divergen rápido producen abstracciones
peores que la duplicación. Ejemplo típico: "estos dos forms son
iguales" → al mes son diferentes y la abstracción común te traba.

## Step 3: ¿Quién es el dueño del dato / lógica?

Si el problema toca datos que tiene otro módulo, **no copiar el dato
para resolverlo localmente** — usar el módulo dueño.

Ejemplo de antipatrón: agregar `userTier` en la tabla `transactions`
para "evitar el join" cuando `users.tier` ya existe. Eso duplica
estado y crea problemas de sincronización.

Reglas:
- Si el dato vive en X, X es la fuente de verdad. No copiar a Y.
- Si la lógica vive en X, invocarla desde Y. No reimplementar.
- Si el patrón existente cubre el caso (con ajuste menor), usarlo
  (ver skill `reuse-existing-patterns`).

## Step 4: Distinguir cleanup oportuno vs scope creep

A veces tocar código relacionado **sí** es válido — pero hay una
distinción clara:

| Sí: cleanup oportuno | No: scope creep |
|---|---|
| Renombrar un símbolo que tu cambio rompe | Renombrar todo el módulo "ya que estoy" |
| Borrar un import que tu cambio dejó inútil | Reformatear el archivo entero |
| Agregar un test para el bug que arreglas | Agregar tests para todo lo que esté cerca |
| Actualizar un comentario que ahora es incorrecto | Reescribir comentarios viejos del archivo |
| Corregir un typo que ves en una línea que tocas | Hacer typo-sweep al repo |

Regla: si lo que vas a tocar **no tiene relación causal** con el
problema concreto, **no lo toques en este PR**. Abre un PR aparte.

## Step 5: Half-finished implementations son peores que no hacerlas

Si el problema requiere una abstracción completa y no tienes tiempo
para hacerla bien, **mejor no empezarla**. Una abstracción a medias:

- Implica que existe, pero no la cubre.
- Otros devs la encuentran y la usan creyendo que está completa.
- Genera bugs cuando se descubre que el caso N no estaba cubierto.

Opciones cuando no hay tiempo:
- **Hardcoded ahora, abstraer cuando exista el tercer caso real.**
- **Abrir issue documentando el diseño pendiente** para no perder
  el contexto.
- **No abstraer, no abrir issue, no dejar nada a medias.**

## Quick Reference

| Situación | Acción |
|---|---|
| Mi propuesta agrega cosas al scope original | Step 1: justificar cada extra |
| Es el primer caso de uso | No abstraer; hardcoded está bien |
| Es el segundo caso | Duplicar; evaluar abstraer en el tercero |
| Los dos casos "parecen iguales" | Verificar si comparten forma o solo nombre |
| Voy a copiar un dato de otro módulo | Step 3: usar el módulo dueño en su lugar |
| Estoy renombrando porque mi cambio rompe | Cleanup oportuno: OK |
| Voy a reformatear todo el archivo | Scope creep: separar PR |
| No tengo tiempo para hacer la abstracción bien | Step 5: hardcoded ahora o no empezar |

## Common Mistakes

- **"Lo agregué de paso."** Cualquier extra fuera del scope necesita
  justificación explícita.
- **Abstraer al segundo caso.** Esperar al tercero — los dos primeros
  suelen divergir.
- **Duplicar un dato del módulo dueño "para evitar el join".** Crea
  problemas de sincronización peores que el join.
- **Refactorizar código ajeno "ya que estoy".** Eso es un PR aparte.
- **Half-finished implementations.** Mejor no empezar que dejar a
  medias.
- **Feature flag para algo que el usuario no pidió.** YAGNI.

## Red Flags

**Nunca:**
- Agregar features, refactors o helpers que no estén en el scope
  del problema concreto.
- Abstraer al segundo caso de uso sin verificar que comparten forma.
- Duplicar datos que ya tiene otro módulo en su tabla.
- Dejar implementaciones a medias creyendo "el siguiente las
  completa".
- Agregar parámetros opcionales "por si los necesitamos".
- Hacer cleanup masivo en un PR que tiene scope acotado.

**Siempre:**
- Escribir el problema concreto en una oración antes de proponer la
  solución.
- Justificar cada extra fuera de la solución mínima.
- Usar el módulo dueño cuando necesites su dato/lógica.
- Separar cleanups en PRs aparte si no son causalmente parte del fix.
- Preferir tres líneas similares a una abstracción prematura.
