# petclinic-vuln-patcher

Cada lunes audita `jthan24/spring-petclinic` (dependencias, imágenes y código) y abre un pull request
por cada CVE o hallazgo CRITICAL/HIGH nuevo, con un test que falla antes del fix y pasa después.
Deja `reporte.md` con todos los hallazgos. Tu equipo revisa y mergea; el agente nunca mergea.

- **Costo medido por corrida:** pendiente (primera prueba en dry run).

## Ficha

```
Modelo        claude-sonnet-5-5 · effort medium                       model.yaml
              por qué: es una demo (un fix roto no cuesta caro); Sonnet migra
              dependencias y escribe tests a una fracción del costo de Opus
Disparador    "0 3 * * 1" America/Bogota = todos los lunes a las 3:00 a. m.   schedule.yaml
              por qué: atrapa cada semana los CVEs nuevos
Entrega       mensaje (user.message), sin rúbrica                      kickoff.md
              por qué: el "listo" vive en el Goal del prompt; rúbrica después
              de 1-2 corridas reales
Tope de costo USD 15 por corrida (amount "1500") · máx. 10 PRs nuevos  schedule.yaml
              = tokens del modelo + USD 0,08/hora en running (sin búsquedas web)
System prompt system_prompt.md
Tools         agent_toolset_20260401: bash, read, write, edit, glob, grep · auto   tools.yaml
              apagadas: web_fetch, web_search (CVEs vía api.osv.dev con curl)
MCP           github  https://api.githubcopilot.com/mcp/x/all          mcp.yaml
              lectura (always_allow): get_file_contents, list_branches,
                list_pull_requests, search_pull_requests
              escritura (always_allow): create_branch, push_files, create_pull_request
              todo lo demás apagado (sin merge, update, close, delete)
Skills        security-audit (cloudflare/security-audit-skill, MIT) → SAST    skills.yaml
              java-vuln-remediation (propia) → umbral, ramas, test-first, PR
Herramientas  no usa
  propias
Subagentes    ninguno: un solo repo
Environment   cloud · red limited · MCP sí · gestores de paquetes sí   environment.yaml
              hosts: repo.maven.apache.org, repo1.maven.org, api.osv.dev,
              github.com, services.gradle.org, plugins.gradle.org,
              plugins-artifacts.gradle.org
Memoria       no usa: estado = ramas security/<ID> en GitHub (abiertas o cerradas)
Archivos      no usa
Repos         no montado: el agente clona el repo público con git al empezar
Credenciales  vault petclinic-vuln-patcher: GitHub MCP (static_bearer, fine-grained
              PAT solo ese repo: Contents R/W, Pull requests R/W)   credentials.md
Compuerta     solo: audita, prueba, crea ramas security/* y abre PRs
              persona: revisa y mergea cada PR
Fuera         web abierta, merge, memoria, outcome, Opus, esperar el CI de GitHub
```

## Archivos

| Archivo | Qué es |
|---|---|
| `model.yaml` | Nombre, descripción, modelo y effort |
| `system_prompt.md` | El system prompt, en Simplified Technical English |
| `tools.yaml` | Tools del sandbox y permisos |
| `mcp.yaml` | GitHub MCP y la lista blanca de sus tools |
| `skills.yaml` | Skills: `../../skills/security-audit`, `../../skills/java-vuln-remediation` |
| `environment.yaml` | El contenedor: red y hosts |
| `vault.yaml` + `credentials.md` | El vault (nunca un secreto) y cómo cargar el PAT |
| `schedule.yaml` + `kickoff.md` | El Deployment: cuándo, con qué mensaje, qué repo monta |
| `tests/dry-run.json` | Prueba sin PRs reales para `scripts/probar.py` |
| `agent.md`, `deployment-lunes.yaml` | Generados por `scripts/ensamblar.py`. No los edites a mano |

## Cambiar algo

```sh
uv run --script scripts/ensamblar.py agents/petclinic-vuln-patcher
F=(agents/petclinic-vuln-patcher/agent.md agents/petclinic-vuln-patcher/environment.yaml agents/petclinic-vuln-patcher/vault.yaml agents/petclinic-vuln-patcher/deployment-lunes.yaml skills/security-audit skills/java-vuln-remediation)
ant apply --dry-run -v "${F[@]}"
ant apply --yes "${F[@]}"     # solo después de leer el plan
```

## Probar y depurar

```sh
uv run --script scripts/probar.py agents/petclinic-vuln-patcher/tests/dry-run.json
uv run --script scripts/inspeccionar.py <session_id>
uv run --script scripts/trazas.py <session_id>
```

## IDs

Se llenan después de `ant apply` (también están en `claude-lock.json`).

| Recurso | ID |
|---|---|
| agent.md | `agent_01RxUMEYpPMahTxUi3umhyAK` (v2) |
| environment.yaml | `env_01BfZvhvesK1KdSMo9QfJf83` |
| vault.yaml | `vlt_011CgMMTHhKDYfy7PqYBpcZr` |
| deployment-lunes.yaml | `depl_01YFjoaGjEsr4tjLtRnQmD9x` (pausado) |
| skill java-vuln-remediation | `skill_01CemaYHe7VFgRJNcb6pvu7e` |
| skill security-audit | `skill_01BrKQvdJr7qHrneGwyapQny` |
