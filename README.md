# AFL para o Claude Code

Este plugin conecta o **Agents for Life (AFL)** ao **Claude Code**. Depois de instalar, o Claude Code consegue conversar com os seus agentes do AFL, consultar os dados ligados a eles (Jira, HubSpot, Notion, base de conhecimento e bancos de dados), **agir por eles** (criar/editar registros, com confirmação nas ações destrutivas) e ainda **disparar squads e automações** da sua organização — tudo direto do seu terminal ou editor, sem sair para o navegador.

> **Nunca usou o Claude Code?** É a ferramenta de linha de comando da Anthropic (`claude`). Instale-a primeiro seguindo o guia oficial em https://claude.com/claude-code e volte aqui.

## O que você ganha ao instalar

O plugin instala duas coisas de uma vez:

- **Acesso aos seus agentes e dados do AFL** — um conjunto amplo de ferramentas que o Claude Code passa a poder usar:
  - **Leitura:** listar seus agentes, conversar com eles, buscar na base de conhecimento e consultar Jira / HubSpot / Notion / bancos ligados ao agente.
  - **Escrita (novo):** ~40 ferramentas de ação (criar/editar registros nas integrações do agente). Ações destrutivas pedem uma confirmação explícita antes de executar.
  - **Squads e automações (novo):** disparar squads e automações da sua organização e acompanhar o resultado.
  - **Tarefas longas (novo):** rodar uma tarefa em segundo plano e buscar o resultado depois.
  - **Descoberta de skills (novo):** listar e ler as skills do sistema e as que estão habilitadas em cada agente, e usar prompts prontos (como `use_skill`) para acionar uma skill específica pelo seu agente.
- **Três guias embutidos** que ensinam o Claude Code a usar tudo isso do jeito certo — você não precisa decorar comando nenhum, é só pedir em linguagem natural:
  - **`afl-hub`** — *como chamar*: as ferramentas, o que cada retorno significa e as armadilhas que fazem a primeira tentativa falhar.
  - **`afl-design`** — *o que criar*: quando a resposta é um agente, uma skill, uma fonte de dados, um documento, um squad ou um app web; em que ordem montar; e como não dar mais acesso do que o necessário. Útil quando você está estruturando uma área ou um processo no AFL pela primeira vez, ou quando uma escrita falha e o conserto é de estrutura, não de parâmetro.
  - **`afl-migrate`** — *como trazer o que já existe*: portar para o AFL uma esteira que hoje roda em outro lugar (OpenSquad, CrewAI, LangGraph, um repositório de prompts, um runbook). O que muda de forma na travessia, quais disciplinas da origem não transferem, e as armadilhas que só aparecem quando você roda a primeira vez.

Exemplos do que dá para pedir depois de instalar:

- *"Liste meus agentes do AFL."*
- *"Pergunte ao meu agente financeiro qual foi o faturamento do último mês."*
- *"Busque no Jira as tarefas em andamento do projeto X."*
- *"Procure na base de conhecimento do meu agente de suporte a política de reembolso."*
- *"Crie um card no Jira do projeto X pelo meu agente."* (ação de escrita — o Claude confirma antes)
- *"Dispare o squad de onboarding da minha organização e me avise o resultado."*

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

Para conferir os guias embutidos, rode:

```
/skill list
```

Devem aparecer `afl-mcp:afl-hub`, `afl-mcp:afl-design` e `afl-mcp:afl-migrate`.

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
Pode, se você pedir. Além de ler e conversar, ele já tem **ferramentas de escrita** (criar/editar registros nas integrações do agente), **squads** e **automações**. Ações destrutivas exigem uma **confirmação explícita** antes de executar, e cada fonte de dados só aceita escrita se estiver habilitada para isso no AFL. O que cada login libera depende dos **escopos** concedidos (leitura, escrita, squads, automações).

**Quais dados ele acessa?**
Apenas os agentes que são seus (ou da sua organização) e as fontes de dados ligadas a esses agentes. Nada além do que você já vê e pode fazer no AFL.

**O que são as "skills" que ele descobre?**
São capacidades extras que um agente pode ter habilitadas. O plugin consegue listar as skills do sistema e as ligadas a cada agente e, se você pedir, usar um prompt pronto para o agente aplicar uma delas — tudo dentro do que aquele agente já pode fazer.

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
