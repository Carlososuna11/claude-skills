---
name: secrets-handling
description: Usar antes de ejecutar comandos que puedan exponer credenciales — leer archivos de config de CLI (~/.aws/credentials, ~/.netrc, ~/.config/gh/hosts.yml, ~/.docker/config.json, ~/.npmrc, ~/.pypirc, ~/.pgpass, ~/.railway/config.json), dumps de env (env, printenv, set), comandos con password/token/api-key inline, o cualquier operación cuyo output incluya values de variables que contengan secrets.
---

# Secrets Handling

## Overview

Los secrets (passwords, tokens, API keys, session tokens, DB passwords,
private keys) nunca deben entrar al contexto del modelo. Una vez en el
transcript, quedan persistidos y pueden filtrarse a logs, allow-lists,
o memorias.

**Principio:** la autenticación vive en el shell parent. El modelo solo
invoca comandos que **referencian** las credenciales (vía variable de
entorno, profile, agente SSH), nunca las imprime.

**Anunciar al inicio:** "Detectada operación con potencial de exponer
credenciales. Aplicando `secrets-handling`."

## Step 0: Regla #0 — el shell parent ya está autenticado

Antes de iniciar el agente (Claude, Codex, etc.), el usuario corre los
comandos de auth en su shell:

```bash
aws sso login
ssh-add ~/.ssh/id_ed25519
gh auth login
docker login
# etc.
```

El proceso del modelo hereda env vars y archivos de credenciales **sin
verlos nunca como texto**. Cualquier comando que el modelo ejecute
después usa esas credenciales sin que tengan que aparecer en el
prompt o el output.

Si el agente arrancó sin auth previa, **decirle al usuario que lo haga
en su shell** — no intentar autenticarse desde adentro.

## Step 1: Archivos que NUNCA hay que leer

Estos archivos siempre contienen tokens, passwords, o claves. **No
abrir con Read, no `cat`, no `head`, no `tail`:**

| Archivo | Contiene |
|---|---|
| `~/.aws/credentials` | AWS access key + secret |
| `~/.aws/config` (sso_session blocks) | Tokens de SSO |
| `~/.netrc` | Passwords HTTP/FTP por host |
| `~/.pgpass` | Passwords de PostgreSQL |
| `~/.config/gh/hosts.yml` | GitHub OAuth tokens |
| `~/.docker/config.json` | Registry credentials |
| `~/.npmrc` (con `_authToken`) | npm tokens |
| `~/.pypirc` | PyPI tokens |
| `~/.railway/config.json` | Railway access + refresh tokens |
| `~/.config/fly/auth.toml` | Fly.io tokens |
| `~/.vercel/auth.json` | Vercel tokens |
| `~/.ssh/id_*` (sin `.pub`) | Private keys |
| `~/.kube/config` | Cluster tokens |
| Cualquier archivo `.env` en repos | Posibles secrets |

Si necesitás info que está en uno de estos archivos, **pedírsela al
usuario**, no leer el archivo.

## Step 2: Comandos que NUNCA hay que ejecutar

Estos comandos imprimen secrets aunque sea por accidente:

```bash
# Dump completo de env del proceso
env
printenv
set

# Dump filtrado — IGUAL peligroso porque PEM keys son multi-line y se cortan
env | grep <pattern>
env | awk -F= '...'
printenv | sort

# Leer envs de servicios remotos
railway run env
railway variables
heroku config
flyctl secrets list  (los nombres están OK, los values NO)
kubectl get secret <name> -o yaml
docker exec <container> env

# Comandos con password inline
psql postgresql://user:password@host/db
mysql -uuser -ppassword
curl -u user:password ...
```

Filtros como `grep` o `awk` **no protegen** contra leaks: cuando un
secret es multi-line (private keys PEM, JSON blobs, certs), el corte
expone fragmentos.

## Step 3: Alternativas reference-based

En vez de imprimir el secret, usar tools que lo referencian sin leerlo:

| En vez de | Usar |
|---|---|
| `psql postgresql://user:password@host/db` | `psql -U user -h host db` (con `.pgpass`) |
| `aws s3 ls --access-key X --secret Y` | `aws s3 ls` (con profile activo) |
| `curl -u user:pass URL` | `gh api URL` o headers via env var |
| Leer `~/.config/gh/hosts.yml` | `gh auth status` |
| Leer `~/.railway/config.json` para env ID | `railway status` / `railway environment` |
| `env | grep DB_PASSWORD` | Preguntar al usuario el nombre |
| Para Railway DB queries | `railway run --service <s> -- bash -c '... $VAR ...'` (sin imprimir $VAR) |

Si la operación requiere un secret inline (no hay alternativa reference-
based), **parar y decírselo al usuario**. No inventar workaround que
embebe el secret.

## Step 4: La allow-list de `~/.claude/settings.json` no es protección

Las deny rules de `settings.json` bloquean comandos exactos, no patterns.

```jsonc
{
  "deny": ["Bash(env)", "Bash(railway run env)"]
}
```

Esto bloquea `env` literal pero **no** bloquea:

- `bash -c 'env'` (el outer command es `bash`, el `env` está nested).
- `printenv` (comando distinto).
- `env | grep X` (depende del matcher; muchas veces pasa).
- Variantes con flags.

Tratar la allow-list como **safety net**, no como sustituto de la regla
de no leer secrets. La rule above es lo que protege contra los nested.

**Corollary crítico:** al aprobar un Bash command que **contiene un
secret inline** (`export AWS_SECRET=...`, `psql postgresql://...`), no
seleccionar **"Allow always"**. Eso persiste el secret en la
`settings.json` para siempre. Si ya pasó, editar el archivo a mano
para quitarlo, y rotar el secret.

## Step 5: Si un secret entró al contexto, rotarlo

Si por accidente un secret aparece en tu transcript, output de
tool result, archivo loguageado, o allow-list:

1. **Avisar al usuario inmediatamente** con qué secret y dónde apareció.
2. El usuario rota el secret en el sistema correspondiente (AWS console,
   Railway dashboard, GitHub settings, etc.).
3. Limpiar el contexto si es local (editar `settings.json`, borrar el
   archivo de log si aplica).
4. Documentar el incidente brevemente para no repetir el mismo failure
   mode.

No "tratar de olvidar" el secret esperando que el modelo lo descarte.
El transcript es persistente.

## Quick Reference

| Situación | Acción |
|---|---|
| Necesito el valor de un secret | Pedirle al usuario que lo aporte, no leerlo |
| Necesito el nombre de un env var | Pedirle al usuario o mirar `.env.example` |
| Necesito autenticarme a un servicio | Pedir al usuario que corra el login en su shell |
| Comando requiere password inline | Parar, decirle al usuario, buscar alternativa reference-based |
| Aparece "Allow always" para comando con secret | NO seleccionar — editar settings.json a mano |
| Un secret apareció en el transcript | Avisar al usuario + rotar el secret |
| Hook me sugiere leer `~/.aws/credentials` | NO leer — pedir al usuario |

## Common Mistakes

- **`cat ~/.aws/credentials` "solo para ver la estructura".** Contiene
  el secret. No leer.
- **`env | grep DB` "para ver si la var existe".** Si el value contiene
  un PEM o JSON multi-line, fragmentos se cuelan.
- **`railway run env` o `heroku config` para listar variables.**
  Imprime values. Los nombres salen de `.env.example` o del repo.
- **Leer `~/.config/gh/hosts.yml` para extraer el token.** El token sale
  intacto al contexto. Usar `gh auth status` para verificar auth.
- **Aprobar "Allow always" un comando con secret inline.** Persiste el
  secret en `settings.json` para siempre.
- **Combinar `railway run` con `env`/`printenv`/`set` en el subshell.**
  El subshell imprime values igual.
- **Asumir que las deny rules cubren bash -c '...'.** Solo matchean
  comandos exactos. Las variantes nested pasan.

## Red Flags

**Nunca:**
- Leer archivos de credenciales (`~/.aws/credentials`, `~/.netrc`,
  `~/.config/gh/hosts.yml`, `~/.docker/config.json`, etc.).
- Ejecutar `env`, `printenv`, `set` (con o sin filtros).
- Usar passwords/tokens/keys inline en comandos.
- Aprobar "Allow always" un comando que contiene un secret.
- Combinar `<cloud-cli> run --` con `env`/`printenv`/`set` en el subshell.
- Tratar de "encontrar" un secret en archivos del filesystem para
  usarlo en un comando.

**Siempre:**
- Asumir que el shell parent ya está autenticado. Si no, pedir al
  usuario que lo haga.
- Preferir tools reference-based (`aws s3 ls`, `gh api`, `psql -U`)
  sobre comandos con credenciales inline.
- Si un secret entra al contexto por accidente: avisar al usuario,
  rotar, limpiar.
- Pedir al usuario nombres de env vars; ellos te dan el nombre,
  tú lo referencias sin verlo.
