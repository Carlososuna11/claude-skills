---
name: read-before-write
description: Usar antes de usar un componente, hook, función, endpoint, decorator, servicio o cualquier API ajena. También antes de sobreescribir un archivo, antes de modificar un módulo que no escribiste, y antes de asumir el shape de un objeto que viene de otra capa.
---

# Read Before Write

## Overview

Asumir la interfaz de algo en lugar de leerla es la fuente más común
de fallos silenciosos: el código compila, "parece" funcionar, pero el
componente renderiza vacío, el endpoint devuelve `undefined`, o el
hook nunca dispara su callback porque el nombre del prop estaba mal.

**Principio:** antes de **usar** algo, **leer** su contrato. Antes de
**sobreescribir** algo, leer su estado actual.

**Anunciar al inicio:** "Aplicando `read-before-write` antes de usar
`<lo-que-sea>`."

## Step 1: Identificar el contrato que vas a usar

Antes de invocar, listar explícitamente qué hace falta:

| Vas a usar... | Lee primero... |
|---|---|
| Componente React/Vue/Svelte | Su archivo y la interfaz de props |
| Hook personalizado | Su archivo: qué retorna, qué argumentos toma |
| Función/servicio del repo | Su signature (params + return type) |
| Endpoint del backend (frontend → API) | El controller / route handler |
| API externa (Stripe, Auth0, etc.) | Docs oficiales + tipos de la SDK |
| Decorator / annotation | Implementación o docs del framework |
| Módulo importado | El barrel/index para ver qué exporta |
| Variable de entorno | `.env.example` o config schema |

Sin ese paso, vas a inventar parámetros que no existen (`type`,
`onCancel`) cuando los reales son otros (`bankType`, `onSuccess`).

## Step 2: Leer hasta entender el contrato

No es "abrir el archivo". Es entender:

1. **Inputs**: ¿qué argumentos toma? ¿cuáles son requeridos vs
   opcionales? ¿qué tipo tiene cada uno?
2. **Output**: ¿qué retorna? ¿síncrono o asíncrono? ¿puede tirar
   excepción?
3. **Side effects**: ¿escribe a DB? ¿hace fetch? ¿muta argumentos?
4. **Estado interno**: si es un hook o componente, ¿qué estado guarda?
5. **Caso de error**: ¿qué devuelve si falla? ¿`null`, `undefined`,
   throw, `{ error: ... }`?

```bash
# Abrir el archivo del símbolo que vas a usar
grep -rn "export.*<NombreDelSimbolo>" src/
# Leer ese archivo entero, no solo la línea del export.

# Si es de un paquete, mirar los tipos
ls node_modules/<paquete>/dist/*.d.ts
# o
python -c "import <modulo>; help(<modulo>.<funcion>)"
```

## Step 3: Verificar antes de sobreescribir un archivo

Si vas a hacer `Write` sobre un archivo existente o `>` que reemplaza
contenido, **leer primero** lo que hay ahí. Razones:

- Puede haber cambios locales no commiteados que perderías.
- Puede haber código que otro agente/dev agregó que tu cambio borra.
- Puede haber comentarios o regiones que tu reemplazo elimina sin
  querer.

```bash
# Estado actual del archivo
cat <archivo>                # o Read tool

# Estado del archivo vs último commit
git diff <archivo>
git status <archivo>
```

Si hay cambios sin commitear o sin push, **detenerse y verificar con
el usuario** antes de sobreescribir.

## Step 4: Confirmar tipos cuando el repo no los enforza

Algunos repos no tienen TypeScript estricto o usan tipos genéricos
(`any`, `dict`, `object`). En esos casos, el typechecker NO te va a
avisar si pasas `type` en vez de `bankType` — el componente
simplemente renderiza vacío.

Compensar leyendo:

1. La interfaz/PropTypes del componente.
2. Tests del componente (suelen mostrar cómo se usa correctamente).
3. Storybook / playground si existe.
4. Otros lugares del repo donde el componente ya se usa.

```bash
# Otros usos del componente — copiar el patrón establecido
grep -rn "<BankConnect" src/   # JSX/TSX
grep -rn "useBankConnect"  src/  # hook
```

## Step 5: Tratar las APIs ajenas como un contrato firmado

Cuando integras con una API externa (Stripe, Twilio, payment provider,
LLM provider), **no inventar los nombres de campos**. Cada cliente SDK
tiene tipos oficiales — usarlos.

```typescript
// ❌ Asumir
const response = await stripe.paymentIntents.create({
  amount: 1000,
  currency: 'usd',
  metadata: { userId: '123' },
  email: 'foo@bar.com',  // ← inventado
});

// ✅ Mirar los tipos
import type { Stripe } from 'stripe';
// Stripe.PaymentIntentCreateParams te dice los campos reales
```

Si no estás seguro de un campo, mirar:
- Tipos del paquete (`.d.ts` para TS, `.pyi` para Python).
- Docs oficiales actualizadas.
- El changelog si la lib cambió versión recientemente.

## Quick Reference

| Situación | Acción |
|---|---|
| Voy a usar un componente nuevo | Step 1 + Step 2: leer su archivo y props |
| Voy a invocar un hook | Step 2: qué retorna y qué argumentos toma |
| Voy a llamar a una función del repo | Step 2: signature + side effects |
| Voy a sobreescribir un archivo | Step 3: leer estado actual + git diff |
| Voy a usar API externa | Step 5: mirar tipos oficiales |
| Repo sin TypeScript estricto | Step 4: leer interface + ejemplos + tests |
| Hay otros usos del mismo símbolo en el repo | Copiar el patrón establecido |

## Common Mistakes

- **Asumir nombres de props "obvios".** `type`, `onCancel`, `value` son
  comunes pero pueden no ser los del componente que usas. Verificar.
- **"Conozco esta API".** Las APIs cambian; los repos a veces wrappean
  con nombres distintos. Leer el wrapper local.
- **Pasar `any` o tipo genérico para evitar el chequeo.** Eso no
  resuelve el problema, lo posterga al runtime.
- **Reemplazar un archivo sin haberlo leído.** Borras cambios locales
  o de otro agente sin enterarte.
- **Confiar en autocompletado del IDE para "completar" un objeto.** El
  autocompletado a veces sugiere defaults o tipos genéricos, no los
  reales del componente.
- **Invocar un componente sin mirar tests u otros usos.** Esos lugares
  enseñan cómo el equipo lo usa.

## Red Flags

**Nunca:**
- Usar un componente sin leer su interfaz de props.
- Invocar un hook sin saber qué retorna.
- Sobreescribir un archivo sin haber leído su estado actual.
- Asumir el shape de un objeto que viene de otra capa.
- Pasar campos a una API externa sin verificar los tipos oficiales.
- Confiar en autocompletado para "rellenar" un objeto.

**Siempre:**
- Leer el archivo del símbolo que vas a usar antes de invocarlo.
- Mirar tests u otros usos en el repo para ver el patrón establecido.
- Verificar tipos oficiales en SDKs de terceros.
- Hacer `git diff` antes de sobreescribir archivos.
- Cuando dudes, leer.
