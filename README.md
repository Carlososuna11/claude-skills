# carlos-skills

Skills personales para [Claude Code](https://claude.com/claude-code) refinadas
trabajando en proyectos de Innovares (Kontigo, DataGo/Mercadato) y otros.

No reemplazan los plugins oficiales (`superpowers`, `code-review`, etc.) —
las complementan con prácticas específicas que repito tanto que tiene sentido
versionarlas como skills.

## Instalación

```bash
# Como plugin Claude Code
claude code plugin install carlosalvaroosuna1/carlos-skills

# O clonarlas directamente a tu carpeta de skills personales:
cd ~/.claude/skills
git clone https://github.com/carlosalvaroosuna1/carlos-skills.git
```

Después, Claude Code carga las skills automáticamente cuando coincide su
`description`.

## Skills disponibles

| Skill | Cuándo se invoca |
|---|---|
| [`use-worktrees`](./skills/use-worktrees/SKILL.md) | Antes de implementar un fix o feature en un repo de Innovares |

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
  - ❌ `carlos-worktrees`, `kontigo-canary` (la generalizamos primero)
- Si una skill solo aplica a un proyecto, vive en el repo de ese proyecto
  (en `.claude/skills/`), no acá.

## Licencia

MIT. Usalas, copialas, adaptalas a tu gusto.
