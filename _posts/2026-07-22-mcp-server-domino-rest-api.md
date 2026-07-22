---
layout: post
title: "Um servidor MCP sobre um ERP Domino de 25 anos"
date: 2026-07-22
categories: [domino, drapi, java, spring, spring-ai, mcp, ai, architecture]
---

Tem um ERP rodando em Lotus Notes/Domino há 25 anos. Ele atravessou versões, servidores, gerações de tecnologia — e continua sendo o coração operacional de um negócio real. Na semana passada, um agente de IA perguntou a esse sistema, em linguagem natural, sobre clientes, pedidos e histórico de dados. E recebeu resposta. Nenhuma linha do legado foi alterada para isso acontecer. A ponte inteira foi construída do lado de fora, sobre a Domino REST API (antigo Project KEEP), com um servidor MCP em Spring Boot.

Este post conta como essa ponte foi feita, as três decisões de arquitetura que a sustentam — e o detalhe que, para mim, é o mais interessante de tudo: o momento em que o *conhecimento acumulado sobre a API* virou ele mesmo uma ferramenta.

## O contexto: migração gradual, banco vivo

Quem acompanha o blog sabe a história: estou migrando gradualmente um ERP Domino para Spring Boot + Vaadin. A palavra importante é *gradualmente*. O Domino não foi desligado nem congelado — ele continua sendo o banco de dados vivo da operação, e a aplicação nova conversa com ele através da Domino REST API. Módulos migram um a um; os dados ficam onde sempre estiveram.

Nesse cenário, apareceu uma necessidade que eu não tinha previsto no início: as sessões de IA que me ajudam no desenvolvimento precisavam *enxergar* os dados. Não adianta pedir para um agente escrever a camada de acesso de um módulo se ele não pode verificar quais campos um documento realmente tem, o que uma view devolve, ou se aquele campo numérico vem como string. Eu passava tempo demais copiando JSON de respostas REST para dentro do chat. O agente precisava de olhos próprios.

## A construção: um servidor MCP em Spring Boot

A resposta foi um servidor MCP (Model Context Protocol) construído com Spring Boot e Spring AI, expondo cerca de 60 ferramentas sobre a Domino REST API. Não vou listar as sessenta — elas caem em poucas categorias:

- **Busca por chave de negócio**: localizar um cliente, um pedido, um documento pelo código que o negócio usa;
- **Consulta a views**: ler os índices nativos do Domino com paginação;
- **Análises agregadas**: totais, rankings, evoluções no tempo;
- **Introspecção**: descobrir o schema de um formulário, listar as views disponíveis.

Um padrão de projeto que valeu muito a pena: as ferramentas vivem num **módulo compartilhado**, consumido por dois clientes distintos. O servidor MCP as expõe para agentes externos (Claude Code, por exemplo); o chat interno da aplicação Vaadin consome exatamente as mesmas classes via Spring AI. Escrevo uma ferramenta uma vez e ela aparece nos dois mundos — o assistente embutido no ERP e o agente de desenvolvimento na minha máquina usam o mesmo código, com as mesmas descrições.

O servidor é leve: sobe em cerca de 4,5 segundos e roda como um processo separado da aplicação principal (nos meus exemplos, na porta 8090). Um agente conecta via MCP, descobre as ferramentas e começa a perguntar.

## Três decisões que valem o post

### 1. Read-only por design

O servidor não escreve nada no Domino. Nenhuma ferramenta de criação, atualização ou exclusão — e isso é uma decisão de arquitetura, não uma limitação.

O raciocínio: uma ferramenta de escrita genérica é *conveniente demais*. Com várias sessões de IA rodando em paralelo — e é assim que trabalho hoje —, escrita genérica e conveniente é receita para clobber: duas sessões alterando o mesmo documento com visões defasadas uma da outra. Sobre um banco de produção com 25 anos de dados, esse risco não se negocia.

Quando é preciso escrever, a escrita acontece em **scripts versionados**: código que entra no git, passa por revisão, tem diff e pode ser reexecutado ou revertido de forma controlada. O agente pode *propor* o script; a execução é um ato deliberado e auditável. Leitura livre, escrita cerimoniosa. Essa asimetria tem se mostrado exatamente o equilíbrio certo.

### 2. Service account com cache de token

A autenticação na Domino REST API é por token JWT com expiração. A implementação ingênua — autenticar a cada chamada — funciona nos primeiros minutos e depois começa a falhar de formas estranhas. O motivo: o Domino tem um mecanismo de proteção anti-DDoS que funciona como um *tarpit*. Um cliente que martela o endpoint de autenticação passa a ser tratado como atacante, e as respostas ficam progressivamente mais lentas até virar bloqueio na prática. Com um agente de IA disparando dezenas de chamadas por minuto, você chega lá rápido.

A solução é um gerenciador de token com três comportamentos:

```
TokenManager:
  - autentica uma única vez com a service account
  - agenda renovação proativa pouco antes do exp do token
  - em caso de 401 inesperado, re-autentica e repete a chamada UMA vez
```

Autentica-se uma vez na subida, renova-se antes de expirar (sem esperar o 401), e o retry em 401 existe só como rede de segurança — limitado a uma tentativa, para nunca degenerar em loop de autenticação. Desde que esse gerenciador existe, o tarpit nunca mais apareceu.

### 3. Ferramentas de introspecção — e as armadilhas que elas evitam

As ferramentas mais usadas pelos agentes não são as de negócio: são as de introspecção. Consultar o schema de um formulário antes de escrever código que o consome elimina uma classe inteira de erros de adivinhação.

Duas regras aprendidas com cicatriz estão embutidas no design das ferramentas:

**Nunca usar UNID como identificador de negócio.** O UNID parece um ID perfeito — universal, único, está em todo documento. Mas ele pode mudar entre réplicas. Todas as ferramentas de lookup trabalham com chaves de negócio (o código do cliente, o número do pedido), que sobrevivem a qualquer réplica.

**Paginar por cursor, sempre.** Queries na Domino REST API **truncam silenciosamente**: você recebe uma página de resultados sem nenhum aviso de que havia mais. Uma ferramenta que devolve "os pedidos do trimestre" sem paginação por cursor devolve, na verdade, "alguns pedidos do trimestre" — e o agente não tem como saber. As ferramentas de consulta expõem o cursor e o agente aprende a iterar até o fim.

## O pulo do gato: conhecimento servido como ferramenta

Aqui está a parte que me fez escrever este post.

Depois de meses trabalhando sobre a Domino REST API, as lições estavam por toda parte — espalhadas em memórias de dezenas de sessões de IA, em múltiplos workspaces. Cada sessão nova começava do zero ou quase: sabia o que estava na memória daquele workspace, ignorava o resto. O conhecimento existia, mas não circulava.

A solução veio em duas etapas. Primeiro, um workflow de agentes destilou tudo: **8 leitores em paralelo** varreram as memórias existentes, uma etapa de **síntese** consolidou os achados, e um **crítico de completude** apontou o que tinha ficado de fora — realimentando o ciclo. O resultado foi um compêndio em markdown com **15 seções** e cerca de **560 fatos** sobre a API: autenticação, paginação, tipos de dados, armadilhas, padrões de consulta.

A segunda etapa é o pulo do gato: **o compêndio virou ele mesmo uma ferramenta MCP.** Três operações — consultar o índice, ler uma seção por tema, buscar por texto. O arquivo é lido do disco a cada chamada, o que significa que atualizar o conhecimento é editar um `.md`. Sem rebuild. Sem restart. Descobri algo novo hoje? Edito o arquivo e, na chamada seguinte, todos os agentes já sabem.

O efeito prático é difícil de exagerar: qualquer sessão de IA, em qualquer workspace, conectada ao servidor MCP, passa a "saber" tudo o que foi aprendido sobre a API em meses de projeto. A memória deixou de ser por-workspace e virou infraestrutura.

## A prova

Pouco depois de a ferramenta entrar no ar, fiz um teste honesto: abri uma sessão de IA completamente nova, sem nenhum contexto prévio, e perguntei sobre um bug sutil de precisão numérica que tinha nos custado caro meses antes (a saga completa merece — e vai ganhar — um post próprio).

A sessão consultou o compêndio, encontrou a seção certa e reconstruiu a história completa: o sintoma, a causa, a correção aplicada. E então fez algo que eu não pedi: chamou por conta própria a ferramenta de introspecção de schema para verificar, ao vivo, que o conserto continuava valendo no ambiente atual. Conhecimento destilado mais dados vivos, combinados por iniciativa do agente. Foi o momento em que a arquitetura se pagou.

## Reflexão: MCP como camada de leitura sobre o legado

O que mais me impressiona nessa história é o que **não** aconteceu: o ERP não mudou. Nenhum agente Notes novo, nenhum campo, nenhum deploy no servidor Domino. Toda a "legibilidade por agentes" foi adicionada por fora — a Domino REST API expõe os dados, o servidor MCP dá semântica e segurança, o compêndio dá contexto.

Suspeito que esse seja um padrão geral, para muito além do Domino: **MCP como camada de leitura sobre sistemas legados** é possivelmente o caminho de menor atrito para juntar IA e sistemas antigos. Não é preciso migrar primeiro, nem envelopar o legado em microsserviços, nem esperar o grande projeto de modernização terminar. Se existe uma API de leitura — e para a maioria dos legados existe, ou é viável — dá para dar olhos a um agente hoje. Read-only, com introspecção e com o conhecimento institucional servido como ferramenta, é o suficiente para o sistema de 25 anos começar a responder perguntas.

O legado não ficou moderno. Ficou *legível*. E isso, na prática, mudou tudo.

---

As lições de produção sobre a Domino REST API que alimentaram este post — os modos de falha, os drops silenciosos e os consertos, no formato Sintoma → Causa → Correção — estão publicadas no repo [domino-rest-api-guide](https://github.com/mmtcastro/domino-rest-api-guide).
