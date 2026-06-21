---
name: design-rigor-alternatives
description: Usar antes de afirmar que una decisión arquitectónica, de storage, de tabla, de contrato, de abstracción o de patrón "es la correcta" o "es necesaria". También antes de declarar que un patrón existente no aplica al caso actual. Y antes de empezar a codear cualquier diseño no trivial.
---

# Design Rigor — Alternatives First

## Overview

Saltar directo a "voy a hacer X" sin enumerar al menos un par de
alternativas con tradeoffs es la fuente principal de PRs sobre-
diseñados, abstracciones equivocadas, y reviews que cambian la
postura completa post-facto.

**Principio:** antes de afirmar una decisión de diseño, listar 2–3
alternativas reales con tradeoffs concretos y recomendar una. La
elección puede ser obvia, pero el ejercicio de listar fuerza a ver
costos que de otra forma se ignoran.

**Anunciar al inicio:** "Aplicando `design-rigor-alternatives` antes
de proponer el diseño."

## Step 1: Encuadrar el problema en una oración

Antes de proponer, escribir el problema en una oración concreta —
mismo formato que `no-premature-abstraction` Step 1:

> "Necesitamos `<resultado>` para `<contexto>`, con la restricción de
> `<constraint>`."

Sin esa oración, las alternativas terminan resolviendo problemas
distintos entre sí y la comparación no sirve.

## Step 2: Listar al menos 2–3 alternativas reales

Cada alternativa debe ser **algo que realmente podrías ejecutar** —
no un strawman para que tu opción gane. Cada una con:

1. **Mecanismo**: cómo funciona en una frase.
2. **Tradeoff principal**: lo que ganas y lo que cedes.
3. **Costo de cambio**: ¿qué hay que tocar?
4. **Costo de mantenimiento**: ¿qué se rompe en 6 meses?

Plantilla:

```
Opción A — <nombre corto>
  Mecanismo: <una frase>
  Gana: <cosa concreta>
  Cede: <cosa concreta>
  Toca: <archivos / capas>
  Riesgo a 6m: <qué se vuelve doloroso si esto crece>

Opción B — ...
Opción C — ...
```

## Step 3: Buscar evidencia, no opinión

Las alternativas se sustentan con evidencia, no con preferencia:

- **Patrones existentes en el repo** (`archivo:línea` que muestra cómo
  el equipo ya resuelve algo similar).
- **Documentación oficial** de la lib/framework cuando aplica.
- **ADRs / decisiones previas** del proyecto si existen.
- **Métricas reales** si el problema tiene componente de performance.

Sin evidencia, una alternativa es una opinión disfrazada.

## Step 4: Recomendar UNA y declarar nivel de certeza

Después de las 2–3 alternativas, **decir cuál recomiendas y por qué**.
Y declarar el nivel de certeza:

- **Alta (90%+)**: hay evidencia clara y la decisión es reversible.
- **Media (60–80%)**: las alternativas tienen mérito, esta gana por
  poco.
- **Baja (<60%)**: estoy adivinando, conviene consultar al maintainer
  o probar con prototipo.

Frase tipo:

> "Recomiendo A. Certeza media. Si en 2 semanas vemos que `<X>`,
> migrar a B es barato porque `<razón>`."

No decir "100% necesario" salvo que tengas evidencia para sostenerlo.
La sobre-certeza es un anti-patrón.

## Step 5: Si un review te cambia la postura, identifica el filtro que faltó

Si después de proponer alguien (reviewer, maintainer, segundo
opinión de Codex) te cambia la postura completa, **no aplicar el
cambio en silencio**. Pausar y preguntar:

> "¿Qué filtro le faltó a mi lista original? ¿Era una alternativa
> que descarté demasiado rápido, o una restricción que no consideré?"

Anotar la respuesta — esa es la lección concreta que evita repetir el
mismo análisis pobre la próxima vez.

## Cuándo NO aplica esta skill

- **Bug fixes pequeños** donde la solución es obvia y reversible.
- **Refactor mecánico** dentro de un módulo (rename, mover archivo).
- **Implementar un patrón ya establecido** del repo (ahí aplica
  `reuse-existing-patterns`, no esta skill).
- **Spike exploratorio** donde explícitamente acordaste con el usuario
  que es throwaway.

## Quick Reference

| Situación | Acción |
|---|---|
| Voy a empezar diseño no trivial | Step 1 + Step 2 + Step 4 antes de codear |
| Tengo una sola "opción obvia" | Listar al menos 1 alternativa para verla rechazada |
| Quiero decir "100% necesario" | Step 4: justificar con evidencia o bajar la certeza |
| El reviewer pide cambiar la postura | Step 5: identificar el filtro que faltó |
| Es un bug fix obvio | No aplica esta skill |
| Es spike throwaway | No aplica esta skill |

## Common Mistakes

- **Strawman alternatives**: poner 2 opciones malas para que la tuya
  gane. La comparación no sirve si las otras no son ejecutables.
- **Opinión sin evidencia**: "esta opción es mejor" sin citar patrones
  del repo, docs o métricas.
- **"100% necesario"** cuando la certeza real es media. Si te
  equivocas, perdes credibilidad.
- **Aceptar el cambio del reviewer en silencio.** Sin entender qué
  filtro faltó, vas a repetir el mismo error.
- **Listar alternativas solo para parecer riguroso** sin haberlas
  considerado de verdad. El proceso solo sirve si la elección está
  abierta antes de empezar.

## Red Flags

**Nunca:**
- Afirmar una decisión arquitectónica sin haber enumerado al menos
  2 alternativas reales.
- Decir "es la única opción" sin haber buscado evidencia.
- Decir "100% necesario" si no puedes citar la evidencia que lo
  sostiene.
- Aceptar un cambio de postura del reviewer sin entender qué filtro
  te faltó.
- Hacer el ejercicio de alternativas como teatro después de haber
  decidido.

**Siempre:**
- Encuadrar el problema en una oración antes de listar opciones.
- Listar 2–3 alternativas ejecutables con tradeoffs concretos.
- Sustentar cada alternativa con evidencia (archivo, docs, métrica).
- Declarar nivel de certeza explícito al recomendar una.
- Si un review cambia tu postura, anotar el filtro que faltó.
