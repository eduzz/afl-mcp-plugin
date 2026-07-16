# AFL para o Claude Code

Este plugin conecta o **Agents for Life (AFL)** ao **Claude Code**. Depois de instalar, o Claude Code consegue conversar com os seus agentes do AFL e consultar os dados ligados a eles (Jira, HubSpot, Notion, base de conhecimento e bancos de dados) — tudo direto do seu terminal ou editor, sem sair para o navegador.

> **Nunca usou o Claude Code?** É a ferramenta de linha de comando da Anthropic (`claude`). Instale-a primeiro seguindo o guia oficial em https://claude.com/claude-code e volte aqui.

## O que você ganha ao instalar

O plugin instala duas coisas de uma vez:

- **Acesso aos seus agentes e dados do AFL** — um conjunto de ferramentas que o Claude Code passa a poder usar: listar seus agentes, conversar com eles, buscar na base de conhecimento e consultar Jira / HubSpot / Notion / bancos ligados ao agente.
- **Um guia embutido** que ensina o Claude Code a usar essas ferramentas do jeito certo — você não precisa decorar comando nenhum, é só pedir em linguagem natural.

Exemplos do que dá para pedir depois de instalar:

- *"Liste meus agentes do AFL."*
- *"Pergunte ao meu agente financeiro qual foi o faturamento do último mês."*
- *"Busque no Jira as tarefas em andamento do projeto X."*
- *"Procure na base de conhecimento do meu agente de suporte a política de reembolso."*

## Antes de começar

Você precisa de:

1. **Claude Code instalado** (o comando `claude` funcionando no terminal).
2. **Uma conta no AFL** — a mesma que você usa em https://app.agentsforlife.org.

Só isso. Você **não** precisa criar nem colar nenhuma senha ou token: o login é feito pelo navegador (veja abaixo).

## Instalação (2 comandos)

Abra o Claude Code e digite:

```
/plugin marketplace add eduzz/afl-mcp-plugin
/plugin install afl-mcp@afl
```

Pronto. Na primeira vez que você pedir algo que use o AFL, o Claude Code vai **abrir o seu navegador** para você fazer login na sua conta do AFL. Depois disso, o acesso fica salvo com segurança na sua máquina e você não precisa logar de novo.

## Como saber se deu certo

No Claude Code, rode:

```
/mcp
```

Você deve ver o servidor **`afl`** na lista. Da primeira vez que você usar uma ferramenta dele, o navegador abre para o login — depois aparece como conectado.

Para conferir o guia embutido, rode:

```
/skill list
```

Deve aparecer `afl-mcp:afl-hub`.

## Como usar no dia a dia

Não há comando especial: basta conversar com o Claude Code em português. Quando você mencionar seus agentes, o AFL, ou pedir para buscar algo que está no AFL, ele usa as ferramentas automaticamente. Por exemplo:

> **Você:** Resuma os últimos chamados abertos no HubSpot pelo meu agente de suporte.
>
> **Claude Code:** *(lista seus agentes, escolhe o de suporte, busca no HubSpot e resume)*

Se algo não funcionar, o Claude Code costuma dizer exatamente o que falta (por exemplo, que você precisa escolher qual agente usar).

## Perguntas frequentes

**Preciso criar uma chave/token?**
Não. O login é pelo navegador, na sua conta do AFL. É só clicar.

**O plugin pode alterar meus dados?**
Não. Hoje ele é **somente leitura** — consulta e conversa, mas não cria nem edita nada (não abre chamados, não envia e-mails, etc.).

**Quais dados ele acessa?**
Apenas os agentes que são seus (ou da sua organização) e as fontes de dados ligadas a esses agentes. Nada além do que você já vê no AFL.

**A janela de login não abriu / expirou.**
Rode `/mcp` no Claude Code e acione o servidor `afl` de novo — ele reabre o navegador para você logar.

## Alternativa avançada: conectar com uma chave de API

O modo recomendado é o login pelo navegador acima. Se você preferir usar uma chave fixa (por exemplo, em um ambiente sem navegador), gere uma chave no AFL em **Perfil → API Keys** e rode:

```
claude mcp add --transport http afl \
  https://app.agentsforlife.org/api/mcp/hub \
  --header "Authorization: Bearer SUA_CHAVE" --scope user
```

Trate essa chave como uma senha — não a compartilhe nem a versione.

## Precisa de mais detalhes?

O guia completo (todas as ferramentas, exemplos avançados e resolução de problemas) está em
[**afl-hub-mcp-handbook.md**](https://github.com/eduzz/labzz_afl/blob/main/docs/afl-hub-mcp-handbook.md).
