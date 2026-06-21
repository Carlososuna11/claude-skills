---
name: public-artifact-hygiene
description: Usar antes de escribir cualquier contenido en sistemas visibles para el equipo o terceros — repos (CLAUDE.md, README, código, mensajes de commit, descripciones de PR), Linear/Jira, Notion, GitHub issues/discussions, Slack público, documentación. Especialmente al final de un flujo de trabajo privado (workspace local, memoria personal, scratchpads) cuando vas a publicar el resultado.
---

# Public Artifact Hygiene

## Overview

Lo que escribes en sistemas compartidos queda visible para gente que
no tiene tu contexto local: tus paths absolutos, tu workspace privado,
tus memorias, tus notas previas. Referenciar artefactos privados en
sistemas públicos genera ruido, confusión, y a veces expone
información sensible.

**Principio:** lo privado se queda en lo privado. En lo compartido,
referencias a artefactos que el lector PUEDE abrir.

**Anunciar al inicio:** "Aplicando `public-artifact-hygiene` antes
de publicar en `<sistema>`."

## Qué cuenta como "sistema visible"

| Sistema | Visible por |
|---|---|
| Repos (CLAUDE.md, README, código, commits, PRs) | Todo el equipo + público si el repo es open source |
| Linear / Jira (descripciones, comentarios) | Equipo entero |
| Notion público | Toda la organización |
| GitHub issues / discussions / comments | Público si el repo es open source |
| Slack canales públicos | Toda la organización |
| Documentación externa | Clientes, stakeholders |

Lo opuesto — **privado** — incluye:

- Tu home local (`/home/carlos/...`, `~/`).
- Tu workspace de tareas (`workspace/tasks/`, `scratchpads/`).
- Tu auto-memoria (`~/.claude/projects/.../memory/`).
- Tus notas personales en archivos fuera del repo.
- Conversaciones de Claude/ChatGPT/Codex en tu máquina.

## Step 1: Identificar qué clase de artefacto vas a escribir

| Vas a escribir... | Es visible? |
|---|---|
| `git commit -m ...` | Sí — queda en historia |
| `gh pr create --body ...` | Sí — visible en GitHub |
| README.md, CLAUDE.md, CONTRIBUTING.md | Sí — versionado |
| Linear issue description | Sí — equipo |
| Comentario en Notion | Sí — equipo o más |
| Mensaje a Slack `#general` | Sí — toda la org |
| Comentario `//` en código del repo | Sí — versionado |
| Memoria local en `~/.claude/` | NO — privado |
| Notas en `workspace/tasks/` | NO — privado |
| Conversación viva con el agente | Privado por defecto |

Si la respuesta es "sí", aplica esta skill.

## Step 2: No incluir paths locales

Patrones a evitar en cualquier contenido visible:

- `/home/<usuario>/...` o `~/Documents/...` o `~/projects/...`
- `/Users/<usuario>/...` (macOS)
- `C:\Users\...` (Windows)
- Rutas absolutas a archivos en tu máquina.

En lugar de:

> "Ver `/home/carlos/Documents/Trabajo/proyecto/scripts/migrate.py`
> para el detalle."

Usar:

> "Ver `scripts/migrate.py` en este repo." (path relativo desde la
> raíz del repo)

O si no está en el repo:

> "El script de migración vive fuera de este repo por convención del
> proyecto."

## Step 3: No referenciar artefactos privados

Patrones a evitar:

- `workspace/tasks/<tarea>/TASK.md` (tu cuaderno privado).
- `~/.claude/projects/.../memory/<archivo>.md` (tu memoria de agente).
- `scratchpads/<algo>.md` (notas locales).
- `docs/superpowers/specs/<spec>.md` si la carpeta `docs/superpowers/`
  está gitignored.
- Cualquier referencia a una sesión de Claude/Codex/ChatGPT como
  fuente de autoridad — esas conversaciones no son consultables por
  otros.

En lugar de:

> "Origen: workspace/tasks/auth-rewrite/TASK.md"

Hacer una de dos:

1. **Pegar el contenido relevante inline** en el artefacto público:
   > "Origen: tras incidente de 2025-11-20 donde el guard de auth no
   > validaba el token en endpoints internos, decidimos reescribir
   > el módulo. Scope: ..."

2. **Referenciar un artefacto público equivalente**:
   > "Origen: GitHub issue #245, ADR #12, PR #1203."

## Step 4: Quick test antes de publicar

Antes de enviar/commitear/abrir, leer el contenido completo y
preguntarse:

> "Si alguien sin acceso a mi laptop, sin acceso a mis memorias, sin
> haber estado en mi conversación con el agente, leyera esto, ¿lo
> entendería?"

Si la respuesta es no porque hay referencias a paths locales,
workspace privado, memorias o conversaciones — **reescribir hasta que
sí**.

## Step 5: Cómo pegar contexto privado de forma pública

Si la información útil está en tu workspace privado, **traerla al
artefacto público de forma autocontenida**:

| Vive en privado | Lleva a lo público como... |
|---|---|
| Análisis de causa raíz en notas locales | Resumen del análisis, no link a las notas |
| Decisión documentada en memoria | Razón inline + enlace a ADR público si existe |
| Discusión con el agente | Conclusión final + datos, no historia de la conversación |
| Spec borrador en scratchpad | Spec finalizado en `docs/` del repo, sin referenciar el borrador |

## Quick Reference

| Situación | Acción |
|---|---|
| Voy a escribir en repo / PR / commit / Linear / Slack | Step 1: confirmar que es visible + Step 2-3: limpiar |
| Tengo paths absolutos en lo que escribí | Step 2: convertir a relativos o quitar |
| Quiero referenciar workspace/tasks/, scratchpads/, ~/.claude/ | Step 3: pegar contexto inline en su lugar |
| Quiero citar una conversación con el agente | Step 5: traer la conclusión, no la conversación |
| El contenido es útil pero solo cierra con mi contexto | Step 4: reescribir para que cierre solo |

## Common Mistakes

- **Paths absolutos en CLAUDE.md / README.** "Ver
  `/home/carlos/Documents/...`" no significa nada para otro dev.
- **Referenciar `workspace/tasks/X/TASK.md` en Linear.** El equipo no
  tiene acceso a tu cuaderno privado.
- **"Origen: conversación con Claude el martes pasado".** No es
  consultable.
- **Asumir que `docs/superpowers/` es público.** Si está en
  `.gitignore`, no lo es.
- **Pegar texto que tiene `~/.claude/` o paths locales en un commit
  message.** Queda para siempre en la historia.

## Red Flags

**Nunca:**
- Pegar paths absolutos (`/home/...`, `~/...`, `/Users/...`,
  `C:\...`) en contenido visible.
- Referenciar `workspace/tasks/`, scratchpads, memorias personales, o
  carpetas gitignored desde sistemas compartidos.
- Citar conversaciones con un agente como fuente de autoridad para
  otros.
- Asumir que algo en `docs/` es público sin verificar `.gitignore`.

**Siempre:**
- Antes de publicar, leer el contenido entero pensando en un lector
  sin tu contexto local.
- Convertir paths absolutos a relativos al repo, o quitarlos.
- Pegar contexto privado inline cuando aporta al artefacto público,
  no como referencia.
- Cuando hay artefacto público equivalente (issue, PR, ADR, página
  Notion), referenciarlo a ese.
