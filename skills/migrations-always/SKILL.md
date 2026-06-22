---
name: migrations-always
description: Usar antes de cambiar el schema de la base de datos — agregar/quitar/renombrar columnas, tablas, índices, constraints, enums. También al notar que alguien hizo un cambio de schema sin migration (manual, en consola, vía GUI), o al modificar un modelo ORM que se refleja en DB.
---

# Migrations Always

## Overview

Cambios de schema sin migration son la causa principal de bugs de
prod que "funcionaban en mi máquina" — diferencias silenciosas entre
los esquemas de cada ambiente, columnas que existen en dev pero no
en prod, índices que faltan en staging.

**Principio:** **cero cambios de schema manuales**. Todo cambio va en
un archivo de migration versionado, ejecutado de forma reproducible,
y reversible.

**Anunciar al inicio:** "Aplicando `migrations-always` antes de
modificar el schema."

## Step 1: Confirmar la herramienta de migrations del repo

| Stack | Herramienta típica |
|---|---|
| Python + SQLAlchemy | Alembic |
| Python + Django | Django migrations (`makemigrations` / `migrate`) |
| Node + Prisma | `prisma migrate` |
| Node + TypeORM | TypeORM migrations |
| Node + Knex | Knex migrations |
| Ruby on Rails | Rails ActiveRecord migrations |
| Go | golang-migrate, sqlc + goose |
| Java/Kotlin + Spring | Flyway o Liquibase |
| Sin ORM (SQL puro) | Flyway, dbmate, sql-migrate |

Si el repo no tiene migrations configuradas, **detenerse antes de
tocar el schema** — primero hay que setupear la herramienta. No hay
excepción "es la primera tabla, hacelo manual".

## Step 2: Crear el archivo de migration, no editar la DB

```bash
# Patrones típicos
alembic revision --autogenerate -m "add_user_phone_column"
python manage.py makemigrations users
npx prisma migrate dev --name add_user_phone
npx typeorm migration:generate -n AddUserPhone
knex migrate:make add_user_phone
flyway -X migrate
```

Reglas:

- **Nombre descriptivo**: `add_user_phone_column`, no `update_v2`.
- **Timestamp en el filename**: lo agrega la herramienta automáticamente.
- **Un cambio lógico por migration**: agregar 5 tablas no relacionadas
  en una sola migration dificulta el rollback.
- **Revisar el SQL generado**: especialmente con autogenerate, que a
  veces detecta mal el cambio.

## Step 3: Up y Down (o `forwards` y `backwards`)

Cada migration debe ser **reversible**:

```python
# Alembic ejemplo
def upgrade():
    op.add_column('users', sa.Column('phone', sa.String(20), nullable=True))

def downgrade():
    op.drop_column('users', 'phone')
```

Excepciones donde down razonable es difícil:
- Migraciones de datos con transformación destructiva.
- Cambios de tipo que pierden info.

En esos casos, documentar en un comentario del down:

```python
def downgrade():
    # WARNING: this drops the column. Data in `users.phone` is not
    # recoverable. Manual restore from backup required if rolling back
    # past production users.
    op.drop_column('users', 'phone')
```

## Step 4: Probar el ciclo completo localmente

Antes de commit:

```bash
# Aplicar adelante
<cli> upgrade head

# Probar reverso
<cli> downgrade -1

# Volver adelante
<cli> upgrade head
```

Verificar que **los 3 pasos funcionan sin error**. Si el down falla,
ese es un problema que se descubre en producción durante un incident
— mejor encontrarlo ahora.

## Step 5: Nunca editar migrations ya aplicadas en otros ambientes

Una vez que una migration corrió en staging o prod, **es inmutable**:

```bash
# ❌ Mal
vim alembic/versions/abc123_add_user_phone.py   # editar contenido
git commit -am "fix typo en migration"          # repushear

# Causa: prod ya tiene la migration aplicada con el contenido viejo.
# El checksum/hash deja de coincidir → tooling rompe en próximo run.
```

Si necesitas corregir algo de una migration que ya está aplicada:

1. **Crear una migration nueva** que aplique el corrective.
2. **Nunca** editar la vieja.

Las únicas migrations editables son las que **aún no se aplicaron en
ningún ambiente compartido** (solo en tu local).

## Step 6: Separar schema migrations de data migrations

Mezclar cambio de schema con backfill de datos en una sola migration
es problemático:

- Las herramientas modernas (Prisma, Alembic) suelen aplicar la
  migration en una sola transacción. Una migration mixta puede
  trabar la transacción por largo rato si el backfill es grande.
- Down de una migration mixta es ambiguo (¿revertir solo schema?
  ¿borrar el dato?).
- Logs y debugging se complican.

Patrón recomendado: **dos migrations consecutivas**.

```
2026_03_15_120000_add_user_phone_column.py   # schema: add column
2026_03_15_120001_backfill_user_phone.py     # data: populate from old field
```

Para backfills grandes (millones de filas), considerar:
- Hacer el backfill **fuera del transaction lock** — un script de
  data migration que corre por batches.
- Usar `WHERE phone IS NULL` para idempotencia.

## Step 7: Cuidados con tablas grandes en producción

Operaciones que parecen inocentes pero locks o downtime en tablas
grandes:

| Operación | Riesgo |
|---|---|
| `ADD COLUMN ... NOT NULL` sin default | Reescribe toda la tabla, lock largo |
| `ALTER COLUMN ... TYPE` (cambio de tipo) | Reescribe toda la tabla |
| `DROP COLUMN` | En Postgres es rápido, pero el storage no se libera hasta vacuum |
| `CREATE INDEX` sin `CONCURRENTLY` (Postgres) | Lock de escritura mientras se construye |
| `ADD CONSTRAINT NOT NULL` | Verifica toda la tabla |

Patrón seguro para `NOT NULL` en tabla grande:

```python
# Migration 1: agregar columna nullable con default
op.add_column('orders', sa.Column('payment_method', sa.String(50),
                                  nullable=True, server_default='legacy'))

# Migration 2 (separada, después): backfill si hace falta
op.execute("UPDATE orders SET payment_method = 'card' WHERE created_at > '...'")

# Migration 3 (separada, después que el backfill terminó): NOT NULL
op.alter_column('orders', 'payment_method', nullable=False)
```

3 migrations en lugar de 1, pero sin downtime.

## Step 8: Histórico es sagrado — todas las migrations se commitean

- `alembic/versions/`, `migrations/`, `prisma/migrations/` — todo va
  al repo, sin excepción.
- El archivo **autogenerado** (.sql o .py) va versionado, no solo el
  modelo ORM.
- Nunca borrar migrations viejas para "limpiar". El histórico es el
  registro de cómo el schema llegó a su estado actual.

Si una migration es vergonzosa (typo, mal diseño), está bien — queda
ahí como parte del histórico. Lo único que NO se hace es editarla
después de aplicada (Step 5).

## Quick Reference

| Situación | Acción |
|---|---|
| Voy a agregar/quitar columna o tabla | Step 2: crear migration con la herramienta del repo |
| Edité una migration que ya está en staging/prod | Step 5: revertir el cambio, crear migration nueva |
| Tabla con millones de filas + `NOT NULL` nuevo | Step 7: patrón de 3 migrations |
| Necesito hacer backfill de datos | Step 6: data migration separada del schema |
| Migration generada por la herramienta tiene bug | Editarla SI aún no fue aplicada en ningún ambiente compartido |
| Setup nuevo de proyecto sin herramienta de migrations | Setupear primero — no hacer cambio manual |

## Common Mistakes

- **Cambio de schema manual** (en consola, GUI, "rápido para probar").
  Crea drift entre ambientes.
- **Editar migration aplicada en prod.** Rompe el tracking interno
  de la herramienta.
- **Schema + data en la misma migration.** Difícil de revertir,
  difícil de debuggear.
- **`NOT NULL` sin default en tabla grande.** Lock + downtime.
- **`CREATE INDEX` sin `CONCURRENTLY` en Postgres.** Bloquea
  escrituras durante el build del índice.
- **Borrar migrations viejas.** Rompe el histórico.
- **Down con `pass`** ("nadie va a rollback").** Llegará el día que
  lo necesites.

## Red Flags

**Nunca:**
- Cambiar el schema directo en la DB (sin migration).
- Editar una migration que ya está aplicada en staging o prod.
- Hacer `NOT NULL` con default no determinístico en tabla grande sin
  el patrón de 3 migrations.
- Borrar archivos de migration viejos.
- Commitear migration sin haberla corrido localmente (up + down + up).
- Mezclar cambio de schema y backfill grande de datos en la misma
  migration.

**Siempre:**
- Crear archivo de migration vía la herramienta oficial del repo.
- Probar `up + down + up` localmente antes de commit.
- Nombre descriptivo + un cambio lógico por migration.
- Schema migrations y data migrations en archivos separados.
- Para tablas grandes, planear pasos múltiples para evitar downtime.
- Versionar todas las migrations, nunca borrar histórico.
