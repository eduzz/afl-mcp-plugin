# afl-mcp-plugin

Plugin do **Claude Code** que adiciona o servidor MCP do **Agents for Life** (hub `/api/mcp/hub`) — dá acesso, dentro do Claude Code, aos agentes e às tools de leitura (`list_agents`, `chat_with_agent`, `jira_search`, `hubspot_search`, `notion_query`, `search_knowledge_base`, `query_database`).

Instalar o plugin entrega duas coisas de uma vez:
- **MCP server `afl`** (`.mcp.json`) — as tools do hub.
- **Skill `afl-hub`** (`skills/afl-hub/SKILL.md`) — ensina o Claude Code a **operar** essas tools em capacidade máxima (modelo service-agent, qual tool para cada caso, JQL válido, queries específicas no KB, multi-turn, tratamento de erro).

> Guia completo do hub: [`afl-hub-mcp-handbook.md`](https://github.com/eduzz/labzz_afl/blob/main/docs/afl-hub-mcp-handbook.md).

## Autenticação: OAuth no browser (sem colar token)

O plugin usa **OAuth**. Você **não** cria nem cola API key: na primeira vez que o Claude Code usar o servidor `afl`, ele abre o **browser** para você fazer login no AFL; o token fica salvo no keychain do sistema. O AFL faz **Dynamic Client Registration** automaticamente — nada a configurar.

## Instalar

```bash
# Via marketplace (recomendado):
/plugin marketplace add eduzz/afl-mcp-plugin
/plugin install afl-mcp@afl

# Teste local (a partir de um clone deste repo):
claude --plugin-dir .
```

Depois, no Claude Code: rode `/mcp` — o servidor `afl` aparece; na primeira chamada de tool o browser abre para o login. Verifique também `/skill list` (deve listar `afl-mcp:afl-hub`).

## Configuração incluída (`.mcp.json`)

```json
{
  "mcpServers": {
    "afl": {
      "type": "http",
      "url": "${AFL_API_BASE_URL:-https://app.agentsforlife.org}/api/mcp/hub"
    }
  }
}
```

- Sem header/token: o Claude Code detecta o `WWW-Authenticate` do hub, descobre o Authorization Server (`/.well-known/oauth-authorization-server`), registra-se via DCR e faz o fluxo `authorization_code` + PKCE no browser.
- `${AFL_API_BASE_URL}` — base URL; default produção. Sobrescreva para homolog: `export AFL_API_BASE_URL="https://app-sandbox.agentsforlife.org"`.

## Requisito de ambiente

O login OAuth exige o AFL com o **Authorization Server (Fase 2)** publicado no ambiente-alvo (prod por default). Se o ambiente ainda não tiver OAuth, use o modo de **token estático** manualmente (fora do plugin), criando uma API key em `/profile` → API Keys e:

```bash
claude mcp add --transport http afl \
  https://app.agentsforlife.org/api/mcp/hub \
  --header "Authorization: Bearer afl_live_SEU_TOKEN" --scope user
```

(Ver o handbook para detalhes.)
