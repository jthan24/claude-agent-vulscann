# Credenciales

| Servicio | Tipo de credencial | Para qué |
|---|---|---|
| GitHub MCP (`https://api.githubcopilot.com/mcp/x/all`) | `static_bearer` (fine-grained PAT) | listar ramas y PRs, crear ramas `security/*`, subir archivos, abrir PRs |

Permisos mínimos del fine-grained PAT (Settings → Developer settings → Fine-grained tokens):

- Resource owner: la organización dueña de `jthan24/spring-petclinic`.
- Repository access: **Only select repositories** → `spring-petclinic`.
- Repository permissions: **Contents: Read and write**, **Pull requests: Read and write**, Metadata: Read (automático).
- Expiración: la más corta que acepte tu ciclo (p. ej. 90 días); anótala para rotarla.

El repo se monta sin token (es público), así que no hay que pasar ningún secreto al Deployment.

Carga (el secreto lo pegas tú; nunca pasa por el chat ni por un archivo):

```sh
VAULT_ID=$(jq -r '.resources["./agents/petclinic-vuln-patcher/vault.yaml"].id' claude-lock.json)
read -rs CMA_TOKEN && export CMA_TOKEN
jq -n -f /dev/stdin <<'JQ' | ant beta:vaults:credentials create --vault-id "$VAULT_ID"
{display_name: "github", auth: {type: "static_bearer", mcp_server_url: "https://api.githubcopilot.com/mcp/x/all", token: (env.CMA_TOKEN // error("CMA_TOKEN is not set"))}}
JQ
unset CMA_TOKEN
```
