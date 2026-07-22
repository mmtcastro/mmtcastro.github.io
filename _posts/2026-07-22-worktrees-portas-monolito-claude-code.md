---
layout: post
title: "Vinte worktrees, vinte portas, um monólito: desenvolvimento paralelo com Claude Code"
date: 2026-07-22
categories: [claude-code, java, spring, vaadin, git, maven, ai, workflow]
---

Numa tarde de trabalho comum, aqui é assim: uma sessão de Claude Code avança na migração do módulo de vendas. Outra mexe no financeiro. Uma terceira refatora componentes compartilhados, e uma quarta fecha uma feature de RH. Todas no **mesmo monólito** Java/Vaadin — cada uma compilando, cada uma com a aplicação inteira rodando e logada no meu browser, nenhuma pisando na outra.

Não tem microserviço nessa história. Não tem container por sessão, nem ferramenta exótica. Tem um repositório monolítico, `git worktree` — um recurso que existe desde 2015 — e três decisões de isolamento que custaram algumas colisões dolorosas até ficarem de pé. Este post inaugura a seção sobre Claude Code do blog contando como esse arranjo funciona.

## O gargalo mudou de lugar

Quem acompanha o blog sabe: estou migrando um ERP Domino de 25 anos para Spring Boot + Vaadin, módulo a módulo — cadastro, vendas, financeiro, compras, cada um com suas dependências. O repositório é um só, e a aplicação é uma só; os módulos são pacotes de um monólito, não serviços.

Trabalhando sozinho, módulo por vez, tudo bem. Mas agentes de IA mudaram a economia do desenvolvimento: escrever código deixou de ser o gargalo. Uma sessão de Claude Code toca uma frente inteira de migração com pouca supervisão — então por que não várias frentes ao mesmo tempo? A resposta honesta da primeira tentativa: porque elas colidem. Duas sessões no mesmo diretório disputam a working tree. Dois builds simultâneos corrompem o cache do Maven. Duas instâncias da aplicação brigam pela mesma porta, e duas abas logadas se derrubam mutuamente a sessão. O gargalo tinha mudado de lugar: não era mais escrever código — era **isolamento**.

## A base: um worktree por frente

A fundação é `git worktree`: o mesmo repositório, materializado em vários diretórios-irmãos, cada um numa branch própria.

```
git/
├── platform/                    ← master
├── platform-sales/              ← branch sales-migration
├── platform-finance/            ← branch finance-migration
├── platform-registry/           ← branch registry-migration
└── ...                          (~20 worktrees ativos hoje)
```

Cada frente de trabalho — a migração de um módulo, uma feature transversal, um endurecimento de core — vive no seu worktree, e cada worktree tem **uma sessão de Claude Code dedicada**, com o contexto daquele módulo e nada mais. Hoje são cerca de vinte. Worktree resolve o primeiro conflito (cada sessão tem sua working tree e sua branch), mas só ele não resolve nada do resto: os vinte diretórios contêm o *mesmo* código, o mesmo `application.properties`, o mesmo `pom.xml`. Subir dois deles "cru" colide na mesma porta. O que faz o arranjo funcionar são as duas camadas de identidade por cima.

## Isolamento de runtime: porta, cookie e um `.env` que nunca entra no git

A primeira camada dá a cada worktree uma **identidade de execução**. A convenção: cada frente sobe a aplicação numa porta própria da faixa 3nnn — vendas em 3000, financeiro em 3001, cadastro em 3002, e assim por diante. Decorável, previsível, e a colisão sumiu.

Mas a porta era a parte óbvia. A dor que eu não tinha previsto: abrir o módulo de vendas numa aba e o financeiro na outra, logar no segundo e... ser deslogado do primeiro. Mesmo host, mesmo nome de cookie de sessão — o browser só guarda um. A correção é dar a cada worktree **um cookie de sessão com nome próprio**: `JSESSIONID_APP_SALES`, `JSESSIONID_APP_FINANCE`. Com isso, vinte aplicações logadas convivem no mesmo browser, cada uma com sua sessão.

E onde mora essa identidade? Num `.env` **gitignored** na raiz de cada worktree:

```
PORT=3000
WORKSPACE_LABEL=sales
SESSION_COOKIE_NAME=JSESSIONID_APP_SALES
```

Um loader de dotenv carrega o arquivo no boot, e o `application.properties` — idêntico em todos os worktrees — só tem placeholders com defaults: `server.port=${PORT:8081}`, `server.servlet.session.cookie.name=${SESSION_COOKIE_NAME:JSESSIONID}`. O código versionado não sabe que worktrees existem; a identidade é local, por diretório, e **nunca entra no git** — o que também significa que nenhum merge, por pior que seja, consegue trocar a identidade de um workspace.

## Isolamento de build: um `.m2` por worktree

A segunda camada foi a lição mais cara. Vinte worktrees compartilhando o `~/.m2` global significa builds simultâneos gravando no mesmo cache de artefatos — e o reator de um monólito multi-módulo instala os módulos internos no repositório local. Sessão A roda `mvn install` enquanto a sessão B compila: B resolve um artefato interno meio-escrito, e o erro que aparece não diz nada disso. Pior: a aplicação *rodando* da sessão C pode estar servindo classes de um jar que acabou de ser sobrescrito.

A correção cabe numa linha, no `.mvn/maven.config` de cada worktree:

```
-Dmaven.repo.local=../.m2repo
```

O caminho **relativo** é o truque: cada worktree resolve para o *seu próprio* diretório de repositório Maven. Builds simultâneos deixam de se enxergar. Custa disco — vinte cópias de dependências — e disco é a coisa mais barata desse arranjo inteiro.

A essa camada pertence também uma regra de higiene aprendida no susto: ao derrubar uma aplicação, **matar pelo PID da porta específica**, nunca por filtro amplo de linha de comando ("mata tudo que parece spring-boot"). Um filtro amplo derruba os vinte workspaces de uma vez — incluindo os das sessões que estavam no meio de um teste.

## A consolidação: "vamos sincronizar tudo"

Vinte frentes andando em paralelo criam a pergunta inevitável: como isso volta a ser *um* sistema? O deploy é monolítico — empacota uma branch, uma app. Mergear o master em cada frente não resolve: isso leva a base comum para cada uma, mas nenhuma passa a conter o trabalho das outras. É preciso um ponto com **tudo**.

A cadência que se estabeleceu: as frentes trabalham independentes e, periodicamente, com todos os worktrees "quietos" — cada um num ponto buildável e coerente, tudo commitado —, eu digo "vamos sincronizar tudo" e um slash command custom do Claude Code executa o fluxo:

1. **Branch de integração** a partir do master. O master não é tocado até o final — tudo reversível.
2. **Merge `--no-ff` de cada frente, uma a uma.** O conflito clássico é sempre o mesmo, e virou rotina: a classe de navegação do menu principal, que toda frente edita para adicionar suas entradas. A resolução é mecânica — a *união* das entradas dos dois lados — mas fica no fluxo como parada explícita, porque de vez em quando o conflito não é mecânico: é uma decisão de produto disfarçada de diff.
3. **Build do reator inteiro.** Mesmo pulando a *execução* dos testes, eles **compilam** — e é aí que aparecem as quebras de API entre frentes: a assinatura que a frente A mudou e a frente B ainda consome. O build integrado é o primeiro momento em que os vinte mundos se encontram.
4. **O master adota por `--ff-only`** — o master só anda para a frente, nunca recebe merge com conflito — e o deploy sai dele.
5. **Sync de volta:** cada worktree faz fast-forward para o master. Como o master agora contém cada branch, todas são fast-forward-elegíveis — e as vinte frentes, o master e a produção terminam **no mesmo commit**. O ciclo recomeça do zero, com todo mundo igual.

O fluxo roda com autonomia, mas com **duas paradas intencionais** combinadas: confirmação humana antes do deploy de produção, e parada obrigatória se um conflito deixar de ser mecânico. Não é limitação da ferramenta — é o contrato. Autonomia no percurso, decisão humana nos pontos irreversíveis.

## O monólito não precisou virar microserviços

A reflexão que fica: quando os agentes de IA derrubaram o custo de escrever código, o gargalo do desenvolvimento migrou para isolamento e integração — e a resposta não exigiu nenhuma ferramenta nova. `git worktree` é de 2015. Dotenv é trivial. Uma convenção de portas é uma tabela num arquivo. Disciplina de merge é disciplina. O monólito continuou monólito; ele só ganhou **identidade por workspace** — porta, cookie, cache de build — que é o que o desenvolvimento paralelo de verdade exige.

E esse paralelismo explica uma decisão de um post anterior: foi por ter vinte sessões de IA simultâneas que o [servidor MCP sobre o ERP]({% post_url 2026-07-22-mcp-server-domino-rest-api %}) nasceu **read-only por design** — leitura livre para todas as sessões, escrita só por script versionado e revisável. As duas arquiteturas são o mesmo princípio em camadas diferentes: paralelismo generoso na leitura, cerimônia na escrita. Até agora, é o arranjo que deixa vinte frentes andarem rápido sem que nenhuma quebre o que as outras estão fazendo.
