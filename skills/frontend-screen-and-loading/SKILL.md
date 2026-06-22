---
name: frontend-screen-and-loading
description: Usar al diseñar/implementar una vista, dashboard, formulario o cualquier pantalla que muestre datos asíncronos. También al revisar una vista que deja mucho espacio vacío, usa spinners genéricos en lugar de skeletons, o tiene mala UX de carga.
---

# Frontend Screen and Loading

## Overview

Una vista bien diseñada hace dos cosas que rara vez se hacen bien:
**aprovecha el espacio disponible** sin verse apretada ni vacía, y
**comunica claramente el estado de carga** sin spinners genéricos
que dejan al usuario sin contexto.

**Principio:** la pantalla del usuario es un recurso. No la
desperdicies con padding excesivo en escritorio ni la satures en
mobile. Y nunca dejes al usuario sin información sobre qué está
cargando.

**Anunciar al inicio:** "Aplicando `frontend-screen-and-loading`
antes de definir esta vista."

## Step 1: Aprovechar el ancho disponible

### Anti-patrón típico

```tsx
// ❌ Mal — todo el contenido apretado en una columna chica
<main style={{ maxWidth: '600px', margin: '0 auto' }}>
  <Form />
</main>
// En pantalla de 1920px, queda 660px usable y 1260px de fondo gris
```

### Reglas

- **`max-width` sensible al contenido**, no al gusto:
  - Texto largo (artículos, docs): 65–75 caracteres por línea (~700px).
  - Formularios simples: 500–700px.
  - **Dashboards, tablas, vistas de datos**: usar 100% con padding
    lateral (`px-6` o `px-8` en Tailwind, equivalente).
  - Forms complejos: 2 columnas si el ancho lo permite.

- **Grid de 2-3 columnas para forms** cuando la pantalla es ancha:

```tsx
// ✅ Bien
<form className="grid grid-cols-1 md:grid-cols-2 gap-4">
  <Input label="Nombre" />
  <Input label="Apellido" />
  <Input label="Email" className="md:col-span-2" />
  <Input label="Teléfono" />
  <Input label="Ciudad" />
</form>
```

- **Dashboards** usan grid responsivo:

```tsx
<div className="grid grid-cols-1 md:grid-cols-2 xl:grid-cols-4 gap-4">
  <KpiCard ... />
  <KpiCard ... />
  ...
</div>
```

### Test rápido

Abrir la vista en 1440px de ancho. Si el contenido ocupa menos del
60% del viewport sin razón (no es texto largo), está
sub-aprovechado.

## Step 2: Layout responsivo, no centrado a cualquier precio

| Breakpoint | Comportamiento |
|---|---|
| Mobile (<640px) | Una columna, todo full-width con padding lateral mínimo (12-16px) |
| Tablet (640-1024px) | 2 columnas para forms y dashboards; tablas con scroll horizontal si hace falta |
| Desktop (>1024px) | Aprovechar el ancho — 2-4 columnas según contenido |
| Wide (>1440px) | Considerar `max-w-7xl` o similar para que el contenido no se estire ridículamente |

Tablas en mobile **nunca** se ven bien con todas las columnas — o:
- Scroll horizontal explícito (`overflow-x-auto`).
- Card-view alternativa para mobile.
- Esconder columnas secundarias.

## Step 3: Skeletons, no spinners

### Anti-patrón

```tsx
// ❌ Mal — usuario no sabe qué viene
{isLoading ? <Spinner /> : <UserProfile data={data} />}
```

### Reglas

- **Skeleton matches el layout real**: misma forma, mismo tamaño,
  mismas posiciones que el contenido final.
- **Animación sutil** (shimmer/pulse), no rotación constante.
- **Sin contador de tiempo** — el skeleton no debe decir "loading
  3s" — eso se siente lento.

```tsx
// ✅ Bien — skeleton específico
{isLoading ? <UserProfileSkeleton /> : <UserProfile data={data} />}

function UserProfileSkeleton() {
  return (
    <div className="space-y-4 animate-pulse">
      <div className="flex items-center gap-4">
        <div className="w-16 h-16 rounded-full bg-gray-200" />
        <div className="space-y-2">
          <div className="h-4 w-40 bg-gray-200 rounded" />
          <div className="h-3 w-24 bg-gray-200 rounded" />
        </div>
      </div>
      <div className="h-32 bg-gray-200 rounded" />
    </div>
  );
}
```

### Cuándo SÍ usar spinner

- Acciones puntuales con feedback claro: botón "Guardar" → spinner
  inline mientras se procesa.
- Operaciones que toman <500ms — el skeleton flashea feo, mejor
  spinner pequeño o nada.
- Cargas secundarias en una vista que ya tiene contenido.

## Step 4: Estados intencionales — empty, error, loading

Cada vista que carga datos tiene **5 estados** mínimos. Diseñar los
5 desde el principio, no agregarlos como afterthought.

| Estado | UI |
|---|---|
| Loading | Skeleton matching el layout |
| Data presente | El render principal |
| Empty (sin datos) | Mensaje intencional + ilustración o icono + CTA si aplica |
| Empty (filtros sin resultados) | "Ningún resultado" + "Limpiar filtros" |
| Error | Mensaje claro + retry button + opcional: detalles colapsables |

Anti-patrón: `{data?.length ? <List /> : <Spinner />}` — el usuario
con data vacía ve spinner infinito.

Patrón correcto:

```tsx
if (isLoading) return <ListSkeleton />;
if (error) return <ErrorState onRetry={refetch} />;
if (!data?.length && hasFilters) return <EmptyFilteredState />;
if (!data?.length) return <EmptyState onCreate={createNew} />;
return <List items={data} />;
```

## Step 5: Forms — full-width inputs, label arriba

### Anti-patrón

```tsx
// ❌ Mal — input chico, label al lado, mucho espacio vacío
<div style={{ display: 'flex', gap: '12px' }}>
  <label style={{ width: '120px' }}>Nombre:</label>
  <input style={{ width: '200px' }} />
</div>
```

### Reglas

- Inputs ocupan el ancho de su columna (`w-full` en Tailwind).
- Label arriba del input, no al lado. Más rápido de escanear.
- Helper text debajo del input.
- Botón principal grande, ancho completo en mobile, ancho generoso
  en desktop.

```tsx
// ✅ Bien
<div className="space-y-1">
  <label htmlFor="email" className="text-sm font-medium">Email</label>
  <input id="email" type="email" className="w-full px-3 py-2 border rounded" />
  <p className="text-xs text-gray-500">Usaremos esto para enviarte el comprobante.</p>
</div>
```

## Step 6: Verificación rápida

Antes de declarar una vista lista, abrir y revisar:

- [ ] **Desktop wide (1440px+)**: ¿el contenido aprovecha al menos
  60-70% del ancho?
- [ ] **Mobile (375px)**: ¿se ve bien sin scroll horizontal? ¿padding
  lateral OK?
- [ ] **Loading**: ¿hay skeleton, no spinner?
- [ ] **Empty sin filtros**: ¿hay mensaje intencional?
- [ ] **Empty con filtros**: ¿se distingue del anterior?
- [ ] **Error**: ¿hay mensaje + retry?
- [ ] **Forms**: ¿inputs full-width? ¿labels arriba?

## Quick Reference

| Situación | Acción |
|---|---|
| Vista con `max-width` chico que deja pantalla vacía | Step 1: ampliar max-width o usar grid de columnas |
| Form con inputs de 200px en pantalla de 1920px | Step 5: inputs full-width + grid de 2 columnas |
| `<Spinner />` mientras carga la vista principal | Step 3: cambiar a skeleton |
| Tabla con todas las columnas en mobile | Step 2: scroll horizontal o card view |
| "No hay datos" idéntico con/sin filtros | Step 4: estados separados |
| Vista sin estado de error | Step 4: agregar error + retry |
| Spinner para acción puntual (botón Guardar) | OK — spinner inline en el botón |

## Common Mistakes

- **`max-width: 600px` por default** en vistas de datos. Desperdicia
  pantalla.
- **Spinner genérico para toda la vista**. El usuario no sabe qué
  viene.
- **Mismo "No hay datos" para empty sin filtros y filtros sin
  resultados.** Confunde.
- **Forms con labels al lado e inputs chicos.** Más lento de
  escanear y desperdicia espacio.
- **Olvidar el estado de error.** Cuando la API falla, el usuario ve
  spinner infinito.
- **Skeleton que no matchea el layout final.** Causa shift visual al
  cargar.
- **Mobile como afterthought.** Tablas con todas las columnas se
  rompen.

## Red Flags

**Nunca:**
- Hacer vista de datos con `max-width` que deje >40% de la pantalla
  vacío sin razón (texto largo es la única excepción).
- Usar `<Spinner />` genérico para la carga inicial de una vista.
- Tener un solo estado "no data" para casos distintos (sin nada vs
  sin matches de filtro).
- Olvidar el estado de error o el retry button.
- Diseñar solo desktop ignorando mobile, o solo mobile ignorando
  desktop.

**Siempre:**
- Layout responsivo con grid o flex que aprovecha el ancho cuando
  hay.
- Skeleton con la misma forma que el contenido final.
- 5 estados explícitos: loading, data, empty, empty-filtrado, error.
- Inputs full-width, labels arriba en forms.
- Verificar la vista en 375px (mobile) y 1440px+ (desktop) antes de
  declararla lista.
