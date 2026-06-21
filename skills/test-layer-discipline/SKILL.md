---
name: test-layer-discipline
description: Usar antes de escribir un test nuevo, refactorizar una suite, o eliminar tests existentes. También cuando el impulso es "agregar un test porque sí" para subir cobertura, o al notar que un test mockea más de lo que prueba.
---

# Test Layer Discipline

## Overview

Tests escritos sin necesidad real son peor que no tener test:
agregan tiempo de CI, fallan flakeando, mockean lógica que cambia y
arrastran a refactor cuando el código de producción se reorganiza.

**Principio:** los tests existen para **proteger comportamiento que
importa**, en la **capa correcta**. Sin un "por qué" claro, no se
escribe el test. Si el test mockea más UI que lógica, está en la capa
equivocada.

**Anunciar al inicio:** "Aplicando `test-layer-discipline` antes de
agregar este test."

## Step 1: ¿Por qué este test, ahora?

Antes de escribir, contestar **una de estas razones reales**:

| Razón | Test justificado |
|---|---|
| Cubrir un bug que acaba de pasar en producción | Regression test |
| Proteger lógica de negocio que va a cambiar mucho | Unit test sobre la función pura |
| Verificar un contrato entre módulos que el equipo va a romper sin querer | Integration test |
| Documentar el flujo crítico end-to-end de un usuario | E2E |
| Bloquear regresiones en seguridad/auth/dinero | Test específico al riesgo |

Razones que **NO** justifican un test:

- "Para subir cobertura."
- "Porque el TDD purist dice que sí."
- "Porque CI me obliga al 80%."
- "Por las dudas."
- "Para tener algo verde en el PR."

Si la razón es una de la lista de abajo, **no escribir el test**.

## Step 2: Elegir la capa correcta

Si tu test pasa más tiempo **configurando mocks** que ejecutando la
lógica que prueba, **está en la capa equivocada**.

| Lo que quieres probar | Capa correcta |
|---|---|
| Cálculo puro (formato de moneda, validador, parseo) | **Unit** — función pura, sin mocks |
| Decisión de negocio (regla "si X y Y, entonces Z") | **Unit** sobre la función que toma la decisión |
| Que dos módulos hablen bien entre sí | **Integration** — usar implementaciones reales, no mocks |
| Que un endpoint cumpla su contrato HTTP | **Integration** sobre el handler real |
| Flujo crítico end-to-end del usuario | **E2E** sobre el sistema vivo |
| Que un componente renderice con props correctos | **Component test** mínimo, sin probar lógica de negocio |

### Anti-patrón: testear lógica desde UI

```typescript
// ❌ Mal — probar la regla de descuento desde el componente
render(<Cart items={items} />);
expect(screen.getByText('Total: $90')).toBeInTheDocument();
// Para llegar acá tuviste que mockear: API, hooks, contexto, router...
// La regla "10% off > 5 items" se está validando a través de 5 capas.

// ✅ Bien — extraer la regla y testearla pura
test('applyDiscount: 10% when items > 5', () => {
  expect(applyDiscount({ subtotal: 100, itemCount: 6 })).toBe(90);
});
// La UI prueba que el cálculo se muestra (snapshot mínimo). La lógica se prueba sola.
```

**Regla del mock ratio:** si tu test tiene más líneas de `mock(...)` /
`jest.fn(...)` / setup que líneas de `assert(...)`, mover la lógica
abajo y probarla pura.

## Step 3: Extraer antes de testear

Cuando el código de producción mezcla lógica con I/O o UI, **refactor
del prod va primero, test del lógica va después**:

```typescript
// Antes:
async function handleSubmit() {
  const { name, age } = formState;
  if (age < 18) return showError('too young');
  if (name.length < 2) return showError('name too short');
  await api.createUser({ name, age });
  router.push('/welcome');
}

// Refactor: extraer la validación
function validateUser({ name, age }): ValidationResult { ... }

// Ahora el test es trivial:
test('validateUser rejects under 18', () => {
  expect(validateUser({ name: 'Ana', age: 17 })).toEqual({ ok: false, reason: 'too_young' });
});
```

Sin la extracción, el test es 30 líneas de mocks. Con la extracción,
3 líneas que valen.

## Step 4: Tests E2E solo para flujos críticos

Los E2E son caros (lentos, flakean, requieren entorno). Reservar para:

- Flujos donde un fallo cuesta dinero real (checkout, pago, transferencia).
- Auth completo (login, logout, refresh token).
- Permisos críticos (admin no puede borrar lo del usuario, usuario no
  puede ver lo del otro).
- "Golden path" de la feature principal del producto.

NO usar E2E para:
- Validaciones de formulario (unit sobre la función validadora).
- Renders correctos (component test mínimo).
- Lógica de UI condicional (hook test).
- Cualquier cosa que pueda probarse 2 capas más abajo.

## Step 5: Eliminar tests que no protegen

Auditar la suite cada cierto tiempo y borrar:

- Tests que llevan más de 1 año sin fallar — probablemente prueban
  algo trivial o están desconectados del código real.
- Tests que se modificaron 3+ veces para "que pasen" sin cambiar
  el comportamiento que prueban — están sobre-acoplados a la
  implementación.
- Tests con `skip` o `xit` de más de un mes — no protegen nada.
- Snapshot tests gigantes que se regeneran cada PR sin que nadie
  los lea — son ruido.

Eliminar es decisión activa, no negligencia. Documentar en el commit
**por qué** se eliminó: "Removed snapshot test that regenerated on
every CSS change and was never read by humans in 8 months."

## Quick Reference

| Situación | Acción |
|---|---|
| Voy a escribir un test | Step 1: razón real ≠ "subir cobertura" |
| El test va a tener más mocks que asserts | Step 2-3: extraer la lógica, probar pura |
| Quiero probar regla de negocio desde el componente UI | Step 2: extraer regla, probar unit |
| Voy a escribir E2E para validación de form | Bajar 2 capas: unit sobre el validador |
| Test lleva 1 año sin fallar | Step 5: candidato a eliminar |
| Test se modifica cada PR para "que pase" | Step 5: sobre-acoplado, candidato a eliminar |
| Solo quiero subir % cobertura | No escribir el test |

## Common Mistakes

- **Test por cobertura.** El número en el dashboard no protege nada
  si lo que se prueba es trivial.
- **Mock-heavy unit tests.** Si la mitad del test es setup de mocks,
  estás probando los mocks, no el código.
- **Probar lógica de negocio desde la UI.** El render del componente
  no es el lugar para validar reglas de descuento, permisos, etc.
- **E2E para todo "para estar seguros".** Lento, flaky, no escala.
- **`it.skip` permanente.** Si no se va a arreglar, borrar. Si se va
  a arreglar, ticketearlo con fecha.
- **Snapshots gigantes.** Se regeneran y nadie los lee. Ruido.
- **No borrar tests que no protegen.** La suite es código — el código
  muerto se elimina.

## Red Flags

**Nunca:**
- Escribir un test sin saber qué comportamiento concreto está
  protegiendo.
- Escribir tests solo para subir el porcentaje de cobertura.
- Probar lógica de negocio desde la capa UI cuando la lógica vive
  más abajo.
- Usar E2E para cosas que un unit/integration puede probar.
- Dejar `it.skip` / `xit` por más de un sprint sin acción.

**Siempre:**
- Justificar el test con una razón real antes de escribirlo.
- Extraer la lógica del prod si el test requiere mucho mock.
- Probar en la capa más baja posible que cubra el comportamiento.
- Reservar E2E para flujos críticos (dinero, auth, permisos, golden
  path).
- Eliminar tests que no protegen — la suite es código.
