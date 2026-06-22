---
name: paginate-and-filter
description: Usar al diseñar o implementar un endpoint que devuelve una colección, una vista de tabla/lista en frontend, o al notar que un endpoint devuelve N+ resultados sin paginar. También al modelar filtros para una pantalla de exploración de datos.
---

# Paginate and Filter

## Overview

Devolver listas completas de golpe es la causa #1 de endpoints que se
caen cuando los datos crecen, vistas que tardan segundos en cargar, y
quejas del usuario "no encuentro nada". La regla es simple: ninguna
vista, ningún endpoint, ningún query devuelve "todo".

**Principio:** todo listado se pagina y se filtra. El usuario no
debería tener que scrollear 10.000 items para encontrar uno.

**Anunciar al inicio:** "Aplicando `paginate-and-filter` antes de
diseñar este listado."

## Step 1: Elegir el modelo de paginación

Dos opciones — elegir antes de implementar:

| Modelo | Cuándo |
|---|---|
| **Offset/Limit** (`?page=2&pageSize=20`) | Listas con números de página visibles, tablas administrativas, datos que cambian poco |
| **Cursor-based** (`?cursor=abc&limit=20`) | Feeds infinite scroll, datos que cambian rápido (orden por `created_at`), datasets grandes |

Reglas:

- **Offset** es más simple pero **degrada con datasets grandes** (`OFFSET
  10000` es lento) y **muestra duplicados** si hay inserts mientras
  scrolleas.
- **Cursor** es estable y rápido pero **no permite "ir a la página 50"
  directo**.
- Si el caller necesita "total count + page X de Y", solo offset
  funciona. Si necesita "siguiente página estable", cursor.

## Step 2: Response shape estándar

Todos los endpoints paginados devuelven la misma forma. Ejemplo:

```json
{
  "data": [ ... ],
  "pagination": {
    "page": 2,
    "pageSize": 20,
    "total": 347,
    "totalPages": 18,
    "nextCursor": "abc123"
  }
}
```

Reglas:

- `data` siempre es array, nunca el item directamente.
- Incluir `total` solo si el modelo es offset (cursor no lo necesita).
- `nextCursor: null` indica fin de la lista.
- `pageSize` default razonable (20–50). Max enforced (típico 100) —
  rechazar requests con `pageSize > max` con 400.

## Step 3: Filtros server-side, no client-side

Los filtros se aplican en el query de DB, no en JavaScript después de
cargar todo. **Filtrar en cliente requiere cargar todo primero — el
anti-patrón que estás evitando.**

Tipos comunes de filtros que tu endpoint debería soportar:

| Filtro | Param de query |
|---|---|
| Búsqueda full-text | `?search=texto` |
| Igualdad | `?status=active` |
| Múltiples valores | `?status=active,pending` |
| Rango fecha | `?from=2026-01-01&to=2026-06-30` |
| Rango numérico | `?minAmount=100&maxAmount=500` |
| Ordenamiento | `?orderBy=created_at&order=desc` |
| Inclusión de relaciones | `?include=user,items` |

Cada uno se valida (whitelist de campos ordenables y filtrables — nunca
aceptar `orderBy=cualquierColumna` sin restricción, eso permite leak
o injection).

## Step 4: Evitar N+1 al paginar

Paginación + relaciones = trampa clásica. Si tu endpoint devuelve
`orders` y cada uno tiene `user`, paginar a 20 órdenes y hacer 20
queries para los usuarios es N+1.

Patrones de fix por ORM:

```python
# SQLAlchemy
stmt = select(Order).options(selectinload(Order.user)).offset(...).limit(...)

# Django ORM
qs = Order.objects.select_related('user').prefetch_related('items')[offset:offset+limit]

# Prisma
const orders = await prisma.order.findMany({
  include: { user: true, items: true },
  skip, take,
});

# TypeORM
const orders = await repo.find({ relations: ['user', 'items'], skip, take });
```

Verificar con query log o con tool del ORM (`EXPLAIN`, debug toolbar)
que el endpoint hace 1–3 queries, no `2 + N`.

## Step 5: UI side — query state en la URL

Los filtros y la página actual viven en la URL, no solo en estado del
componente:

```
/orders?status=pending&from=2026-01-01&page=3
```

Razones:
- El usuario puede compartir el link y otra persona ve lo mismo.
- Refresh de página no pierde el estado.
- Back/forward del browser funcionan.
- Marcar como favorito un set de filtros es trivial.

En React/Next: usar `useSearchParams` o `nuqs`/`use-query-state`. En
Vue: `useRoute`. En Angular: `ActivatedRoute`. En frameworks de UI
sin router: query string parsing manual.

## Step 6: Skeleton + empty state + error

Cada vista paginada tiene cuatro estados visuales claros:

| Estado | UI |
|---|---|
| Loading | Skeleton con la forma de la tabla/lista (ver `frontend-screen-and-loading`) |
| Empty (no data) | Mensaje intencional + CTA si aplica ("aún no tienes órdenes — crear la primera") |
| Empty (filtered) | "Ningún resultado con estos filtros" + botón "limpiar filtros" |
| Error | Mensaje claro + retry button |

No usar el mismo "No hay datos" para los dos casos de empty — son
problemas distintos.

## Quick Reference

| Situación | Acción |
|---|---|
| Voy a crear endpoint que devuelve lista | Step 1 (elegir modelo) + Step 2 (shape estándar) |
| Voy a mostrar tabla en frontend | Step 3 (filtros server-side) + Step 5 (URL state) |
| Endpoint devuelve >100 items sin paginar | Refactor inmediato — agregar paginación |
| Filtros aplicados con `.filter()` en JS después del fetch | Mover al backend |
| Endpoint con relaciones devuelve 20 items en 2 segundos | Step 4 (probable N+1) |
| Vista sin loading state | Step 6 (skeleton) |
| Vista de "sin datos" idéntica con/sin filtros | Step 6 (separar empty vs empty-filtered) |

## Common Mistakes

- **Devolver "todo" porque "son pocos registros por ahora".** El día
  que crezca, el endpoint se rompe y tu vista también.
- **Filtrar en cliente.** Trae todo primero, filtra después — el
  antipatrón que esta skill previene.
- **`orderBy` sin whitelist.** Permite que el caller ordene por
  `password_hash` (leak) o cause queries lentas a propósito.
- **N+1 oculto.** El endpoint "funciona" en dev con 5 registros y
  explota en prod con 500.
- **Estado de filtros solo en componente.** El usuario pierde
  contexto al refrescar o compartir.
- **`pageSize` sin max.** El caller pide `?pageSize=1000000` y rompe
  el server.
- **Empty state genérico para "sin datos" y "filtrado sin matches".**
  Confunde al usuario.

## Red Flags

**Nunca:**
- Diseñar un endpoint que devuelve "todo" sin paginar.
- Filtrar en cliente cuando los datos pueden crecer.
- Aceptar `orderBy` arbitrario sin whitelist.
- Olvidar el max de `pageSize` (deja la puerta abierta a DoS barato).
- Devolver lista paginada con N+1 en relaciones.

**Siempre:**
- Elegir el modelo (offset vs cursor) antes de codear.
- Validar y limitar `pageSize`.
- Filtros server-side, validados contra whitelist.
- Verificar con query log que no hay N+1.
- Filtros + página en la URL, no solo en componente.
- Loading + empty (sin datos) + empty (sin matches) + error como
  estados separados.
