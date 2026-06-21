---
name: reuse-existing-patterns
description: Usar antes de añadir un archivo, abstracción, validator, guard, servicio, helper, componente, constante, tipo o test nuevo al repo. También al diseñar la solución a un problema cuando el repo lleva tiempo en producción y probablemente ya tenga el mecanismo que estás por inventar.
---

# Reuse Existing Patterns

## Overview

Crear paralelos cuando ya existe un patrón establecido es la causa
número uno de los comentarios de revisión del tipo *"esto ya existe,
usa el patrón actual"* — y obliga a rehacer trabajo. Más allá del
review, los repos con N formas distintas de hacer lo mismo se
vuelven costosos de mantener y difíciles de onboardear.

**Principio:** buscar primero, crear después. El reflejo correcto al
empezar cualquier cambio es buscar 30 segundos si el mecanismo ya
existe.

**Anunciar al inicio:** "Aplicando `reuse-existing-patterns` antes de
crear `<lo-que-sea>`."

## Por qué importa más allá del review

- **Consistencia interna**: si el repo tiene tres formas de validar
  lo mismo, el próximo dev (humano o agente) no sabe cuál usar.
- **Costo de mantenimiento**: si la regla cambia, hay que tocar N
  lugares en vez de uno. Bugs se duplican.
- **Curva de onboarding**: leer un repo con múltiples patrones
  equivalentes cuesta más que uno con uno solo bien definido.

## Step 1: Identificar qué vas a crear

Antes de codear, nombrar explícitamente la **categoría** de lo que vas
a crear. La búsqueda cambia por categoría:

| Vas a crear... | Categoría |
|---|---|
| Reglas para validar un input | Validator / schema |
| Lógica antes de ejecutar un endpoint | Guard / interceptor / middleware |
| Lógica de negocio compartida | Service / repository / use-case |
| Endpoint admin u operativo | Tool controller |
| Formateo de dinero/fechas/números | Helper / util |
| Renderizado de algo en UI | Componente |
| Test nuevo | Test (mirar convención) |
| Valor constante o configuración | Constant / config / enum |
| Forma de datos | Type / interface / DTO |
| Algo de orquestación entre módulos | Module / facade / mediator |

Sin esa categoría explícita, la búsqueda termina siendo difusa y
encuentra cualquier cosa.

## Step 2: Buscar antes de crear

### Qué buscar según categoría

| Categoría | Dónde mirar primero |
|---|---|
| Validator / schema | `validators/`, archivos `*.validator.*`, schemas Zod/Joi/Pydantic/class-validator |
| Guard / interceptor / middleware | Carpetas equivalentes (`guards/`, `interceptors/`, `middleware/`, `decorators/`) |
| Service / repository | Patrones de inyección (`@Injectable`, factories, providers, módulos del framework) |
| Tool controller | Controllers en `tools/`, `admin/`, `ops/` |
| Helper / util | Archivos en `utils/`, `helpers/`, `lib/` por nombre similar |
| Componente UI | `components/` por funcionalidad similar, no solo por nombre exacto |
| Test | Convención del repo: `__tests__/`, `*.spec.*`, `*.test.*` colocados como vecinos vs. en carpeta dedicada |
| Constant / config / enum | Archivos consolidados de constants, enums, config |
| Type / interface / DTO | Tipos compartidos en `types/`, `dto/`, `models/`, `interfaces/` |
| Module / facade | Index files, barrels, README del módulo |

### Cómo buscar — comandos universales

```bash
# Por concepto en el código
grep -rni "<concepto>" src/ | head -20
rg -i "<concepto>" src/ | head -20      # si tienes ripgrep

# Por exportaciones de símbolos similares
grep -rn "export.*<concepto>" src/ | head -20

# Por estructura de carpetas
fd -t d "<nombre>" src/                  # si tienes fd
find src -type d -iname "*<nombre>*"

# Si el repo usa barrel files / index re-exports
grep -rn "<concepto>" src/**/index.* 2>/dev/null
```

Adaptar a la herramienta disponible (`rg` si está, `fd`, o `grep`/
`find` estándar). El comando importa menos que el reflejo de buscar
antes de crear.

### Ampliar la búsqueda si el primer pase no encontró nada

- Sinónimos: si buscaste "auth" sin éxito, probar "guard", "permission",
  "token", "credential".
- Singular y plural.
- En inglés y en el idioma del repo (algunos repos tienen mezcla).
- En tests: a veces el helper que buscas vive en `tests/helpers/`.
- En docs/README: a veces hay un mapa del módulo que apunta al pattern
  oficial.

## Step 3: Decidir — usar el existente o crear paralelo

### Usar el existente si:

- Cumple el caso de uso con cambios menores (parámetro opcional, nueva
  variante).
- El equipo lo invoca consistentemente en código reciente (`git log
  --since=3.months <archivo>` no muestra movimiento de huida).
- Está documentado o tiene tests.

### Crear paralelo solo si:

- El patrón existente está **explícitamente deprecado** y el equipo
  migró a uno nuevo. Verificar en `CHANGELOG`, ADRs, o preguntar al
  maintainer.
- El caso de uso es **genuinamente distinto** y forzar el patrón
  actual lo deformaría. Esto debe documentarse en el body del PR
  para que el revisor no lo marque como duplicado.
- El patrón existente tiene un bug conocido y la migración es parte
  del scope del cambio actual.

Cuando creas paralelo, dejarlo escrito en el PR:

> "No reuso `validators/AmountValidator` porque <razón>. El nuevo
> `MultiCurrencyAmountValidator` cubre <X> sin tocar el caso que
> resuelve el original."

## Quick Reference

| Situación | Acción |
|---|---|
| Voy a crear archivo nuevo | Step 1 (categoría) → Step 2 (buscar) → Step 3 (decidir) |
| Primer pase de búsqueda no encontró | Ampliar (sinónimos, idiomas, tests, docs) |
| Encontré algo parecido pero no exacto | Step 3: ¿basta con cambio menor? |
| Encontré exactamente lo que iba a hacer | Usar el existente, no crear paralelo |
| Existente está deprecado | Verificar en CHANGELOG/ADR y crear con justificación |
| No estoy seguro si crear paralelo | Preguntar al maintainer antes de codear |

## Common Mistakes

- **Saltar Step 1 (categoría) y buscar directo.** La búsqueda sin
  categoría es difusa y termina en cualquier resultado.
- **Buscar solo por el nombre exacto que tenías en mente.**
  Probar sinónimos, plural/singular, inglés/idioma del repo.
- **Asumir "no hay nada parecido" sin haber buscado en tests/docs.**
  Mucho código de soporte vive ahí.
- **Crear paralelo "por las dudas"** sin documentar por qué en el PR.
  El revisor lo marca como duplicado.
- **Tomar el primer match como definitivo** sin chequear si está
  deprecado.

## Red Flags

**Nunca:**
- Crear archivo, helper, validator, componente o constante nueva sin
  haber buscado primero si ya existe.
- Crear paralelo sin documentar la razón en el PR.
- Tomar el primer match sin verificar si está activo o deprecado.

**Siempre:**
- Nombrar la categoría antes de buscar.
- Probar sinónimos y singular/plural cuando el primer pase falla.
- Documentar en el PR la razón si decides crear paralelo.
- Prueba final antes de codear: *"si hubiera buscado treinta segundos
  antes de empezar, ¿lo habría encontrado?"*. Si sí, había que buscar.
