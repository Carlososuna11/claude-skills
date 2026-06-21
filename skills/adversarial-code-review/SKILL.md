---
name: adversarial-code-review
description: Usar al revisar código (propio o ajeno) — al abrir un PR, al hacer review a otro PR, al cerrar una feature antes de mergear, o al auditar un módulo existente. Combina cuatro lentes: abstracciones correctas, separación de responsabilidades por capa, security-thinking ("¿cómo lo vulnero?"), y mantenibilidad a largo plazo.
---

# Adversarial Code Review

## Overview

Un code review que solo busca bugs evidentes es la mitad del trabajo.
Los problemas que matan repos a 2 años no son bugs — son
abstracciones equivocadas, capas que se mezclan, validaciones que se
saltan, y código que nadie quiere tocar.

**Principio:** revisar el código como si fueras el atacante, el dev
que llega el año que viene, y el responsable de mantenerlo todo
junto. Las cuatro lentes se aplican en orden y cada hallazgo se
prioriza por costo en el futuro.

**Anunciar al inicio:** "Aplicando `adversarial-code-review` —
revisando con 4 lentes."

## Las 4 lentes

### Lente 1 — Abstracciones correctas

Pregunta: **¿la abstracción modela el dominio o lo distorsiona?**

Síntomas de abstracción mal:

- **Parámetros que solo aplican a algunos casos.** Si la función
  `processPayment(method, options)` ignora `options.bankCode` cuando
  `method === 'card'`, la abstracción está forzando dos dominios
  distintos en una sola firma. Mejor: tipo discriminado.
- **`if (kind === 'A') { ... } else if (kind === 'B') { ... }` que
  se repite en N lugares.** Cada nuevo kind obliga a tocar N
  archivos. Indica polimorfismo faltante.
- **Helpers genéricos que solo se usan en un sitio.** Si
  `formatNumber(value, precision, locale, currency, separator)`
  tiene 5 parámetros y un solo caller, la generalización es teatro.
  Inline el código.
- **Tipos `any` / `dict` / `object` en interfaces críticas.** El
  diseño no decidió qué shape tiene el dato — lo difirió al runtime,
  donde es más caro.
- **Abstracciones que envuelven otras abstracciones sin agregar
  semántica.** `UserService` que solo hace `return userRepo.find()`.
  Ruido.

Acción al detectar:

```
Hallazgo: <ubicación>
Síntoma: <cuál de los anteriores>
Costo si se queda: <qué se rompe al escalar / cambiar dominio>
Sugerencia: <colapsar / partir / discriminar / inline>
```

### Lente 2 — Separación de responsabilidades por capa

Pregunta: **¿cada archivo/clase/función hace UNA cosa de UNA capa?**

Capas típicas (ajustar al stack):

| Capa | Responsabilidad |
|---|---|
| Controller / route handler | Recibir request, validar shape, devolver response |
| Service / use-case | Orquestar lógica de negocio |
| Repository / DAO | Hablar con persistencia |
| Domain / entity | Reglas invariantes del dominio |
| Infrastructure | Adapters a externos (HTTP, queue, DB driver) |
| UI / view | Renderizar, no decidir |

Violaciones a buscar:

- **Controller con lógica de negocio.** Llamadas a la DB directo,
  cálculos, decisiones de dominio. Esa lógica pertenece al service.
- **Service que arma HTTP responses.** El service no debería saber
  que existe HTTP.
- **Repository que toma decisiones.** Si el repo hace
  `if (user.tier === 'premium') return discounted()`, está
  decidiendo. Esa decisión va arriba.
- **UI que conoce schema de DB.** El componente que renderiza no
  debería referenciar `users.password_hash`.
- **Entidad anémica con lógica en services.** Si el modelo `User`
  solo tiene campos y toda la lógica vive en `UserService`, la
  capa de dominio no existe — es un DTO disfrazado.
- **Capas que se saltan.** Controller llamando a repository sin
  pasar por service. Cada salto perdido es una invariante que se
  puede violar en otro caller futuro.

Acción al detectar: nombrar la capa que está siendo violada y mover
el código a su capa correcta. Si la separación no se respeta en N
lugares, es señal de que el modelo de capas del repo está mal
diseñado o no se enseñó al equipo.

### Lente 3 — Security thinking ("¿cómo lo vulnero?")

Pregunta: **si fuera un atacante / usuario malintencionado / agente
con prompt injection, ¿qué rompo?**

Vectores estándar a chequear:

- **Input validation:** ¿se validan los inputs en el borde del
  sistema (controller/route)? ¿O confían en que el frontend filtró?
- **Authentication:** ¿el endpoint está protegido por algún guard?
  ¿El guard se ejecuta en ESTE path concreto (no solo "existe")?
- **Authorization:** ¿se verifica que el usuario X tenga permiso
  sobre el recurso Y? ¿O solo que esté autenticado?
- **IDOR:** ¿los IDs son secuenciales y verificables? Si soy usuario
  123, ¿puedo pedir el recurso del usuario 124 cambiando el id en
  la URL?
- **Injection:** SQL/NoSQL/command injection en strings que vienen
  del input. ¿Hay queries construidas con string concat?
- **Mass assignment:** ¿el endpoint acepta un payload con TODOS los
  campos del modelo, incluyendo `is_admin` o `tier`? Hay que
  whitelist.
- **Secrets en respuesta:** ¿se devuelven hashes, tokens, internal
  IDs, stack traces en error responses?
- **Race conditions:** ¿hay operaciones tipo "check then act" que
  pueden ser interleaved? (Ej: verificar balance, después debitar —
  sin lock pueden ejecutarse 2 retiros del mismo balance.)
- **Rate limiting / DoS:** ¿el endpoint tiene límites? ¿La query
  permite pedir 1M registros?
- **Logs con info sensible:** passwords, tokens, PII en logs es leak.
- **CORS / CSRF:** ¿los endpoints sensibles aceptan requests de
  cualquier origen?

Para cada hallazgo, **escribir el ataque concreto**:

> "Si soy usuario A con token válido, hago `GET /api/orders/42`
> donde la orden 42 pertenece al usuario B, y el response trae los
> datos. No hay verificación de ownership en el handler."

No abstracto ("falta auth"). Concreto (el request que rompe).

### Lente 4 — Mantenibilidad a largo plazo

Pregunta: **el dev que llega en 12 meses sin contexto, ¿puede tocar
esto sin romper algo?**

Señales de problemas futuros:

- **Funciones largas sin estructura interna.** > 80 líneas suele
  indicar que hay múltiples responsabilidades mezcladas.
- **Archivos largos sin estructura.** > 500 líneas suele ser señal
  de partir en módulos.
- **Nombres mentirosos.** `getUser` que también crea el usuario si no
  existe. `validateInput` que también escribe a la DB.
- **Estado global mutable.** Singletons, variables módulo-level,
  caches sin invalidación clara.
- **Dependencias implícitas.** El módulo X funciona solo si Y se
  invocó antes, pero la dependencia no está en el código.
- **Configuración mágica.** Booleanos hardcoded, números mágicos sin
  constantes nombradas, env vars leídas en el medio del código.
- **Falta de errores tipados.** `throw new Error('...')` genérico
  obliga a parsear strings para diferenciar fallas.
- **Tests acoplados a implementación.** Si refactor del prod
  rompe tests sin cambiar comportamiento, los tests están mal
  diseñados (ver `test-layer-discipline`).
- **Comentarios contradictorios con el código.** Indica que el
  comentario se quedó cuando el código cambió. Borrar o actualizar.
- **Código muerto.** Branches que ningún caller toma, funciones
  exportadas sin uso. Eliminar.

Para cada hallazgo:

```
Hallazgo: <ubicación>
Síntoma: <cuál>
Cuándo va a doler: <qué cambio futuro se va a complicar por esto>
Sugerencia: <refactor concreto, no abstracto>
```

## Step 1: Revisar en el orden de las lentes

No mezclar las 4 lentes en un mismo pase. Hacer una pasada por
lente, en orden:

1. **Abstracciones** primero — si el diseño está mal, todo lo demás
   se mueve cuando arregles esto.
2. **Capas** segundo — separación correcta hace que security sea
   más fácil de auditar.
3. **Security** tercero — con capas claras, los choke points donde
   validar inputs son explícitos.
4. **Mantenibilidad** al final — pulir lo que sobrevivió las 3
   primeras lentes.

## Step 2: Priorizar hallazgos

Por cada hallazgo, etiquetar severidad:

- 🔴 **Bloqueante**: bug, vulnerabilidad explotable, capa rota que
  afecta producción. No mergear sin fix.
- 🟠 **Importante**: abstracción equivocada que va a doler en 3
  meses, falta de validación que aún no es exploit pero puede
  serlo. Fix antes de scale.
- 🟡 **Mejora**: refactor que vale la pena pero no bloquea. Documentar
  como deuda técnica con costo claro (ver `comments-discipline`
  Regla 3 — no usar `TODO` vacío).

## Step 3: Comunicar como peer técnico, no como auditor

El review se entrega como peer, no como auditoría:

- Lead con la razón antes de la observación.
- Engagear con el reasoning del autor si lo documentó en el PR
  body.
- Si el autor explicó por qué eligió X, responder a Y específicamente,
  no reinventar desde primeros principios.
- Bullets por severidad. Sin tags artificiales (`B1`, `M2`). Sin
  code blocks gigantes cuando no aportan.

Ver skill `pr-discipline` Step 6 para el formato detallado.

## Quick Reference

| Lente | Pregunta |
|---|---|
| 1 — Abstracciones | ¿Modela el dominio o lo distorsiona? |
| 2 — Capas | ¿Cada archivo/función hace una cosa de una capa? |
| 3 — Security | Como atacante, ¿qué rompo y con qué request concreto? |
| 4 — Mantenibilidad | ¿Un dev sin contexto puede tocar esto en 12 meses sin romper? |

| Síntoma | Lente |
|---|---|
| Parámetros que solo aplican a algunos casos | 1 (abstracción) |
| Controller con lógica de negocio | 2 (capas) |
| Endpoint sin verificación de ownership | 3 (security — IDOR) |
| Función de 200 líneas | 4 (mantenibilidad) |
| `any` en interfaz crítica | 1 (abstracción) |
| Service armando HTTP responses | 2 (capas) |
| String concat para queries SQL | 3 (security — injection) |
| Nombre que miente sobre lo que hace | 4 (mantenibilidad) |

## Common Mistakes

- **Solo buscar bugs evidentes.** Los problemas caros no son bugs
  visibles; son abstracciones y capas equivocadas.
- **Saltar la lente de security en código "interno".** El próximo
  cambio puede exponer ese endpoint sin que nadie revise auth.
- **Quejarse de mantenibilidad sin proponer refactor concreto.**
  "Esto es difícil de mantener" no es accionable. "Partir en
  módulos A y B porque X" sí.
- **Mezclar las 4 lentes en un mismo pase.** Tu cerebro pierde
  consistencia. Una lente por pasada.
- **Audit-style review.** Bullets sin contexto, tono externo. Mejor
  peer-style: razón + observación, engageaer con el autor.
- **Bloquear PR por hallazgos de mantenibilidad solamente.** Eso es
  para documentar como deuda, no para bloquear.

## Red Flags

**Nunca:**
- Mergear con hallazgo 🔴 sin fix.
- Aprobar review sin haber pasado las 4 lentes.
- Asumir "alguien ya validó la auth" sin trazar el guard en este
  path.
- Hacer review sin mirar el PR body / reasoning del autor.
- Reescribir el código del autor desde primeros principios sin
  responder a su reasoning documentado.

**Siempre:**
- Pasar las 4 lentes en orden (abstracciones → capas → security →
  mantenibilidad).
- Escribir el ataque concreto al reportar hallazgos de security.
- Llevar refactor sugerido al reportar hallazgos de mantenibilidad.
- Priorizar 🔴 / 🟠 / 🟡 y separar lo que bloquea merge de lo que
  no.
- Entregar el review como peer técnico, no como auditor externo.
