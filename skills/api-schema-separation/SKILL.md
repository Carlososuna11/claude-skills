---
name: api-schema-separation
description: Usar al definir un endpoint nuevo, al modelar entidades de backend, o al exponer datos por API. También al notar que un endpoint devuelve el modelo de DB directo, o que el mismo schema se usa para crear, actualizar y responder.
---

# API Schema Separation

## Overview

Exponer el modelo de la base de datos directo por la API es la fuente
de tres clases de bugs:

1. **Leak de campos internos** (password_hash, audit_fields,
   soft-delete, internal_id).
2. **Validación insuficiente** al crear/actualizar (mass assignment:
   el caller manda `isAdmin: true` y pasa).
3. **Acoplamiento rígido** entre cambios de DB y cambios de API
   (renombrar una columna rompe a todos los clientes).

**Principio:** el modelo de DB es interno. La API expone **schemas
distintos** para cada operación (Create, Update, Response), y un
mapeo explícito traduce entre ambos mundos.

**Anunciar al inicio:** "Aplicando `api-schema-separation` antes de
definir el contrato de este endpoint."

## Step 1: Nombrar los 3-4 schemas por entidad

Por cada entidad expuesta por API, definir explícitamente:

| Schema | Para | Contiene |
|---|---|---|
| `EntidadCreate` | POST (crear) | Solo campos que el caller PUEDE enviar al crear |
| `EntidadUpdate` | PATCH (actualizar) | Solo campos editables. Todos opcionales |
| `EntidadResponse` | GET / como output de POST/PATCH | Lo que el caller VE |
| `EntidadList` (opcional) | GET listados | Subset de Response, sin relaciones pesadas |

Tres reglas:

- **Create ≠ Response**: campos como `id`, `created_at`, `updated_at`,
  `slug` los genera el server, no el caller.
- **Update ≠ Create**: PATCH no requiere todos los campos. Y algunos
  campos (`email` verificado, `role`) pueden no ser editables.
- **Response ≠ Modelo DB**: el modelo tiene `password_hash`,
  `deleted_at`, `internal_id`, `tenant_id`, `embedding`... la
  Response no.

## Step 2: Schemas como tipo, no como diccionario suelto

Usar el sistema de schemas del stack — no diccionarios sin tipo:

| Stack | Herramienta |
|---|---|
| Python + FastAPI | Pydantic |
| Python + Django REST | Serializers / Pydantic |
| Node + NestJS | class-validator + class-transformer (DTOs) |
| Node + Express puro | Zod / Joi |
| Go | structs con tags `json:` y validate |
| Java/Kotlin + Spring | DTOs con `@Valid` + Jakarta annotations |

Los schemas DEBEN validar:
- Tipos (string, number, email, UUID).
- Constraints (length, range, regex).
- Required vs optional.

Sin validación en el schema, la validación se hace adentro del
service y queda dispersa.

## Step 3: Mapeo explícito DB ↔ Schema

Existe siempre una función que mapea el modelo de DB al schema de
response, y otra que mapea el schema de create al modelo de DB.
Mejor explícito que mágico:

```python
# Mapeo Response — explícito qué expone
def to_response(user: UserModel) -> UserResponse:
    return UserResponse(
        id=user.id,
        email=user.email,
        nombre=user.nombre,
        created_at=user.created_at,
        # NO incluir: password_hash, internal_id, deleted_at
    )

# Mapeo Create — explícito qué acepta
def from_create(data: UserCreate, hashed_password: str) -> UserModel:
    return UserModel(
        email=data.email,
        nombre=data.nombre,
        password_hash=hashed_password,
        # `is_admin`, `tenant_id` NO vienen de input — los setea el server
    )
```

Si tu ORM soporta `model_dump()` o equivalente automático, **pasarle
una whitelist explícita** — nunca `dict(model)` completo.

## Step 4: Campos prohibidos en Response

Lista mínima que **nunca** debería salir por API:

- `password`, `password_hash`, `salt`.
- Tokens internos (`api_key`, `refresh_token`, `oauth_state`).
- IDs internos paralelos (`internal_id`, `legacy_id`, `tenant_id` si
  el caller no es admin).
- Campos de auditoría (`created_by_user_id`, `deleted_at`,
  `audit_log`).
- Campos calculados costosos que solo sirven internamente
  (`embedding`, `score`, `denormalized_total`).
- Tokens de relación (`stripe_customer_id`, `s3_key_internal`).

Si alguno necesita salir (ej. `stripe_customer_id` para el cliente
admin), va a un schema separado (`UserAdminResponse`) que solo se
serializa desde endpoints autorizados.

## Step 5: Mass assignment — whitelist, no blacklist

En Create y Update, **enumerar los campos permitidos**, no enumerar
los prohibidos.

```typescript
// ❌ Mal — blacklist incompleta
const data = { ...req.body };
delete data.isAdmin;       // se olvidaron de role, tenantId, etc.
await db.users.create(data);

// ✅ Bien — whitelist explícita (el DTO)
const dto = UserCreateDto.parse(req.body);  // valida + filtra
const user = await usersService.create(dto);
```

Razón: si agregas un campo nuevo al modelo (ej. `permissions`), un
blacklist puede pasarlo por alto y exponerlo. Un whitelist lo deja
afuera automáticamente.

## Step 6: Versionado del contrato

Cuando un schema necesita cambio breaking (renombrar campo, cambiar
tipo, quitar campo requerido), no romper a los clientes existentes:

Opciones:
- **API versioning** (`/v1/users` vs `/v2/users`).
- **Field aliasing** durante un período (`fullName` reemplaza `name`,
  pero ambos siguen aceptados por 3 meses).
- **Deprecation header** + comunicación a clientes antes del cambio.

Cambios NO breaking que no requieren versión:
- Agregar campo opcional al Create.
- Agregar campo a la Response (clientes no listados lo ignoran).
- Hacer un campo opcional cuando antes era requerido.

Cambios breaking sí necesitan plan:
- Quitar campo de Response.
- Cambiar tipo de un campo.
- Hacer requerido un campo que era opcional.
- Renombrar un campo.

## Quick Reference

| Situación | Acción |
|---|---|
| Voy a crear endpoint nuevo | Step 1: definir 3-4 schemas por entidad |
| Voy a devolver un modelo de DB en la response | Step 3: mapear con whitelist explícita |
| Voy a aceptar `req.body` directo en create | Step 5: whitelist via DTO/schema |
| Mi modelo tiene campos `password_hash`, `internal_id`, audit | Step 4: NO en Response |
| Update toma todos los campos del Create | Step 1: Update ≠ Create (todos opcionales, no editable lo immutable) |
| Necesito cambiar shape de un endpoint en uso | Step 6: deprecation o versioning |

## Common Mistakes

- **Devolver el modelo de DB directo.** Filtra campos internos sin
  que nadie los pida.
- **Mismo schema para Create y Response.** El caller "tiene que
  enviar el id" cuando el server lo genera.
- **Blacklist en create.** Te olvidas de un campo nuevo y se filtra.
- **`model_dump()` / `dict(model)` sin filtrar.** Lo mismo que
  devolver el modelo.
- **Diccionarios sin tipo.** Validación dispersa por todo el service.
- **Versionar nada y renombrar campos directo.** Rompe a los
  clientes silenciosamente.

## Red Flags

**Nunca:**
- Exponer el modelo de DB directo como Response.
- Usar el mismo schema para Create, Update y Response.
- Aceptar `req.body` completo sin pasar por DTO/schema con whitelist.
- Devolver `password_hash`, tokens, IDs internos, audit fields en
  cualquier Response.
- Hacer breaking change al contrato sin deprecation o versioning.

**Siempre:**
- Definir Create, Update, Response (y List si aplica) explícitos por
  entidad.
- Usar la herramienta de schemas del stack (Pydantic, Zod, DTOs).
- Mapeo explícito DB → Response con whitelist.
- Validar input en el DTO/schema, no en el service.
- Versionar o deprecar antes de romper clientes.
