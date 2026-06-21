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

### Workflow

| Skill | Cuándo se invoca |
|---|---|
| [`use-worktrees`](./skills/use-worktrees/SKILL.md) | Antes de iniciar trabajo de código con commits y PR — resuelve la rama base, crea worktree, abre PR contra la base correcta |
| [`pr-discipline`](./skills/pr-discipline/SKILL.md) | Antes y durante cualquier flujo de PR — alineación previa, atribución, markdown limpio, estilo de review, estrategia de merge |
| [`delegate-to-codex`](./skills/delegate-to-codex/SKILL.md) | Antes de invocar a Codex — ejecución mecánica o análisis que alimenta una decisión |

### Disciplinas de código

| Skill | Cuándo se invoca |
|---|---|
| [`read-before-write`](./skills/read-before-write/SKILL.md) | Antes de usar un componente, hook, función o API ajena — y antes de sobreescribir un archivo |
| [`reuse-existing-patterns`](./skills/reuse-existing-patterns/SKILL.md) | Antes de crear un archivo, validator, guard, helper, componente o constante nueva — buscar si ya existe |
| [`no-premature-abstraction`](./skills/no-premature-abstraction/SKILL.md) | Antes de proponer una nueva tabla/clase/helper/capa, o cuando el impulso es "ya que estoy acá refactorizo esto otro" |
| [`verify-before-claim`](./skills/verify-before-claim/SKILL.md) | Antes de afirmar causa raíz de un bug, que un fix funciona, o que algo "no es bloqueante / no es explotable" |
| [`design-rigor-alternatives`](./skills/design-rigor-alternatives/SKILL.md) | Antes de afirmar una decisión arquitectónica/storage/contrato como "la correcta" — enumerar 2–3 alternativas con tradeoffs primero |
| [`module-expert-hypothesis`](./skills/module-expert-hypothesis/SKILL.md) | Cuando el maintainer/dueño de un módulo da una pista u objeción casual — tratarla como primera hipótesis a refutar, no a ignorar |
| [`public-artifact-hygiene`](./skills/public-artifact-hygiene/SKILL.md) | Antes de escribir en sistemas visibles (repos, PRs, commits, Linear, Notion, Slack) — quitar paths locales, workspace privado, referencias a artefactos no compartibles |

### Seguridad

| Skill | Cuándo se invoca |
|---|---|
| [`careful-destructive-ops`](./skills/careful-destructive-ops/SKILL.md) | Antes de operaciones difíciles de revertir — `rm -rf`, `git reset --hard`, `git push --force`, `DROP TABLE`, `--no-verify` |
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
