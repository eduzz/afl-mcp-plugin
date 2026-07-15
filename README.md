# afl-mcp-plugin

Plugin do **Claude Code** que adiciona o servidor MCP do **Agents for Life** (hub `/api/mcp/hub`) — dá acesso, dentro do Claude Code, aos agentes e às tools de leitura (`list_agents`, `chat_with_agent`, `jira_search`, `hubspot_search`, `notion_query`, `search_knowledge_base`, `query_database`).

> O plugin só empacota a **configuração** (não o token). Cada usuário usa a **própria** API key (token por-usuário) — o token **nunca** é embutido no plugin. Guia completo do hub: [`afl-hub-mcp-handbook.md`](https://github.com/eduzz/labzz_afl/blob/main/docs/afl-hub-mcp-handbook.md).

Instalar o plugin entrega duas coisas de uma vez:
- **MCP server `afl`** (`.mcp.json`) — as tools do hub (`list_agents`, `chat_with_agent`, `jira_search`, etc.).
- **Skill `afl-hub`** (`skills/afl-hub/SKILL.md`) — ensina o Claude Code a **operar** essas tools em capacidade máxima (modelo service-agent, qual tool para cada caso, JQL válido, queries específicas no KB, multi-turn, tratamento de erro).

## Pré-requisito: sua API key

1. Acesse **`/profile`** no AFL → seção **API Keys** → **Nova chave**.
2. Selecione os escopos (`agents:chat` e/ou `tools:read`) e copie a chave (`afl_live_...`) — ela só aparece uma vez.
3. Exporte no ambiente (persista no `~/.zshrc` / `~/.bash_profile`):
   ```bash
   export AFL_TOKEN="afl_live_..."
   # Opcional (default = produção): apontar para homolog
   # export AFL_API_BASE_URL="https://app-sandbox.agentsforlife.org"
   ```

## Instalar

```bash
# Teste local (a partir da raiz do repo):
claude --plugin-dir ./afl-mcp-plugin

# Ou publique este diretório num marketplace/repo e instale por nome:
#   claude plugin install <org>/afl-mcp-plugin
```

Depois, no Claude Code, rode `/mcp` para confirmar que o servidor `afl` conectou.

## Configuração incluída (`.mcp.json`)

```json
{
  "mcpServers": {
    "afl": {
      "type": "http",
      "url": "${AFL_API_BASE_URL:-https://app.agentsforlife.org}/api/mcp/hub",
      "headers": { "Authorization": "Bearer ${AFL_TOKEN}" }
    }
  }
}
```

- `${AFL_TOKEN}` — sua API key (obrigatória).
- `${AFL_API_BASE_URL}` — base URL; default produção. Sobrescreva para homolog.

## Futuro (OAuth)

Quando o AFL expuser OAuth/OIDC (planejado), o `.mcp.json` troca o header por um bloco `oauth` e o login passa a ser via browser — **sem** criar/colar token. Este plugin será atualizado para essa versão.
