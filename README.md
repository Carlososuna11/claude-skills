# claude-skills

Skills personales para [Claude Code](https://claude.com/claude-code)
refinadas en el trabajo diario sobre varios proyectos.

No reemplazan los plugins oficiales (`superpowers`, `code-review`, etc.) —
los complementan con prácticas específicas que se repiten tanto que tiene
sentido versionarlas como skills.

## Instalación

```bash
# Como plugin Claude Code
claude code plugin install Carlososuna11/claude-skills

# O clonarlas directamente a tu carpeta de skills personales:
cd ~/.claude/skills
git clone https://github.com/Carlososuna11/claude-skills.git
```

Después, Claude Code carga las skills automáticamente cuando coincide su
`description`.

## Skills disponibles

| Skill | Cuándo se invoca |
|---|---|
| [`use-worktrees`](./skills/use-worktrees/SKILL.md) | Antes de implementar un fix o feature en cualquier repo — resuelve la rama base, crea worktree, abre PR |
| [`careful-destructive-ops`](./skills/careful-destructive-ops/SKILL.md) | Antes de operaciones difíciles de revertir — `rm -rf`, `git reset --hard`, `git push --force`, `DROP TABLE`, `--no-verify` |
| [`pr-discipline`](./skills/pr-discipline/SKILL.md) | Antes y durante cualquier flujo de PR — alineación previa, atribución, markdown limpio, estilo de review, estrategia de merge |
| [`delegate-to-codex`](./skills/delegate-to-codex/SKILL.md) | Cuando una tarea es mecánica/bien definida y conviene delegarla a Codex para conservar tokens del modelo principal |
| [`secrets-handling`](./skills/secrets-handling/SKILL.md) | Antes de comandos que puedan exponer credenciales — leer config de CLI, dumps de env, comandos con secrets inline |

> **Estado**: en construcción. Más skills se van agregando a medida que se
> destilan patrones útiles del trabajo diario.

## Cómo se estructura una skill aquí

```
skills/<nombre>/
  SKILL.md      # frontmatter YAML (name, description) + body markdown
  [extras]      # scripts, sub-prompts, assets si la skill los necesita
```

El `description` del frontmatter es lo que Claude usa para decidir cuándo
invocar la skill. Mantenerlo claro y accionable.

## Convención de naming

- `kebab-case`.
- Verbo en imperativo o describiendo qué se hace, no quién la hizo.
  - ✅ `use-worktrees`, `careful-destructive-ops`, `pr-discipline`
  - ❌ `carlos-worktrees`, `<proyecto>-canary` (si solo aplica a un proyecto, va en su repo)
- Si una skill solo aplica a un proyecto, vive en el repo de ese proyecto
  (en `.claude/skills/`), no acá.

## Licencia

MIT. Usalas, copialas, adaptalas a tu gusto.
