---
layout: post
title: "O LLM como query planner: Spring AI tools + MCP sobre uma base Domino"
date: 2026-05-12
categories: [domino, drapi, java, spring, spring-ai, mcp, ai, llm, agents, architecture]
---

Imagine perguntar *"quais livros foram emprestados mais de cinco vezes no último trimestre"* e receber a resposta de uma base de dados projetada antes da Web ser comum. Não é text-to-SQL. Não é RAG sobre dump de documentos. É um LLM escolhendo, entre dezenas de métodos Java anotados, qual chamar — e o método vai ler uma view nativa do Domino via REST. Esse post descreve a arquitetura que torna isso possível, por que ela é mais simples do que parece, e o que muda mentalmente quando o LLM passa a ser o seu query planner.

## O improvável que funciona

Domino foi projetado em 1989. Schemaless, multivalue, documentos hierárquicos, views como índices materializados. Tudo o que um query planner moderno *não* gosta. Spring AI e o protocolo MCP (Model Context Protocol) são de 2024–2025. Tudo o que um banco legado *não* sabe que existe.

A ponte tem três peças:

```
┌──────────────────────────┐
│ LLM (Claude / GPT / etc) │
└────────────┬─────────────┘
             │  tool calls (JSON-RPC)
             ▼
┌──────────────────────────┐
│ MCP Server (Spring Boot) │
└────────────┬─────────────┘
             │  Spring AI ToolCallbackProvider
             ▼
┌──────────────────────────┐
│ @Tool methods (Java)     │
└────────────┬─────────────┘
             │  WebClient
             ▼
┌──────────────────────────┐
│ Domino REST API (DRAPI)  │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│ HCL Domino — NSF + views │
└──────────────────────────┘
```

A parte que parece milagre — o LLM escolhendo a função certa, montando argumentos, interpretando a resposta — é Spring AI fazendo tool calling padronizado. A parte que parece impossível — uma base de 1989 falando JSON estruturado — é DRAPI: um servidor REST que a HCL distribui e que expõe views, formulários e operações CRUD sobre qualquer NSF. O que sobra pra construir é a camada do meio: os métodos `@Tool`.

## @Tool é onde mora a inteligência

O LLM não enxerga sua base. Ele enxerga apenas o que você descreve em prosa, na anotação `@Tool`. Cada método vira uma operação atômica que o modelo pode invocar:

```java
@Component
public class CatalogTools {

    private final DrapiClient drapi;

    @Tool(description = """
        Lista livros do catálogo filtrados por gênero.
        Use quando o usuário pedir livros de uma categoria específica
        (ex: "ficção científica", "biografia", "infantil").
        Retorna apenas título, autor e ano — campos completos via
        consultarLivroPorCodigo.
        """)
    public List<BookSummary> listarLivrosPorGenero(
            @ToolParam(description = "Gênero literário em minúsculas") String genero,
            @ToolParam(description = "Quantos resultados, máximo 50") int limite) {

        return drapi.queryViewEntries(
                "biblioteca", "(LivrosPorGenero)",
                genero, Math.min(limite, 50));
    }
}
```

Duas coisas importantes vivem aqui, e nenhuma é o corpo do método:

**A descrição do `@Tool` é prompt engineering.** O modelo decide quando usar essa função lendo essa string. "Use quando o usuário pedir..." direciona; "Retorna apenas X — campos completos via Y" educa o modelo a encadear chamadas em vez de carregar dados desnecessários. Frases descuidadas aqui causam alucinação ou escolha de tool errada. Frases precisas viram um *contrato narrativo* entre você e o modelo.

**A descrição do `@ToolParam` também é prompt.** "Gênero literário em minúsculas" não é validação — é instrução pra como o modelo normaliza a entrada do humano antes de chamar. Você escreve menos `if (input.isUpperCase())` no Java e mais prosa na anotação.

## O duplo modo: views ou documentos

DRAPI expõe duas formas de ler uma view, e a diferença é gritante em performance:

- `documents=false` — só as colunas que a view publicou (rápido)
- `documents=true` — carrega o documento inteiro de cada entry (lento, mas tudo lá)

Numa view de centenas de itens, o segundo modo pode ser ordens de magnitude mais caro. E o LLM não tem como saber qual usar — *a menos que você ensine*. O padrão que destrava é expor duas tools com descrições opinativas:

```java
@Tool(description = """
    **PREFIRA ESTA TOOL.** Lista entries de uma view, retornando
    APENAS as colunas que a view publica. Use em ~80% dos casos.
    Rápida.
    """)
public String consultarEntries(String view, String chave, int limite) { ... }

@Tool(description = """
    **USE COM PARCIMÔNIA.** Carrega o documento completo de cada
    entry. Mais lenta. Use APENAS quando precisar de um campo que
    NÃO está exposto na view (ex: corpo de texto, anexos).
    """)
public String consultarDocumentos(String view, String chave, int limite) { ... }
```

A negrita e o adjetivo são deliberados. Modelos seguem instruções com ênfase. Numa amostra de algumas centenas de chamadas reais, a tool "rápida" foi escolhida sozinha em quase todas as situações em que cabia — e a "lenta" só apareceu quando o pedido genuinamente exigia campo fora da view.

## Descoberta dinâmica via ToolCallbackProvider

Em um sistema pequeno você lista as tools à mão. Em um sistema que cresce, isso vira passivo. Spring AI resolve com um `ToolCallbackProvider` que varre o contexto Spring e expõe todo `@Bean` com métodos `@Tool`:

```java
@Configuration
public class ToolsConfig {

    @Bean
    public ToolCallbackProvider catalogTools(
            CatalogTools catalog,
            LoansTools loans,
            MembersTools members) {

        return MethodToolCallbackProvider.builder()
                .toolObjects(catalog, loans, members)
                .build();
    }
}
```

Adicionar uma nova área de domínio é criar uma classe `@Component`, anotar métodos com `@Tool`, e incluir no provider. Sem rota, sem DTO, sem decorator. O LLM "descobre" no próximo handshake.

## Por que MCP separado, e não tools embarcadas

Você pode embarcar Spring AI direto no app que faz chat — e em muitos casos isso é o suficiente. Mas montar a camada de tools como um **servidor MCP independente** (também Spring Boot, porta própria) compra três coisas:

1. **Reuso entre hosts.** Claude Desktop, Cursor, OpenAI Agents SDK, IDE plugins — todos falam MCP. Uma única superfície de tools serve todos.
2. **Desacoplamento de ciclo de vida.** O servidor de tools pode subir, descer, escalar independente do app que consome.
3. **Borda de segurança explícita.** O MCP server é o lugar único pra impor escopo, rate limit, auditoria. Sem isso espalhado por N hosts.

O wrapper MCP em si é curto — registra as `@Tool`s no padrão MCP, expõe via stdio ou HTTP, e fim. O peso intelectual continua nas tools.

## O que muda mentalmente

Três deslocamentos que esse padrão produz, e que valem nomear:

### 1. A descrição é a API

Em REST tradicional, o contrato é a URL + o JSON schema. O cliente humano lê documentação. No padrão `@Tool`, o cliente é um LLM e o contrato é prosa. Você não escreve OpenAPI; você escreve instruções condicionais ("use quando...", "prefira... a menos que..."). Isso é programação, só que a linguagem alvo é semântica natural. Tratar a descrição como código de primeira classe — review, versionamento, testes A/B com prompts — é o caminho.

### 2. O LLM é o novo query planner

Banco relacional moderno aceita SQL e roda um otimizador caro pra escolher plano. Nesse padrão, o "plano" é a sequência de tool calls que o LLM emite. Suas tools são os operadores físicos. As descrições são as estatísticas. O LLM raramente erra por falta de inteligência — erra por falta de informação no catálogo. Cuidar do catálogo é cuidar do otimizador.

### 3. Bancos legados viram cidadãos de primeira classe

A intuição de descartar bancos pré-Web por "modernização" assume que a interface é o problema. Não é — o motor de leitura de views indexadas do Domino é, em muitos casos, mais rápido que uma query naive contra um schema relacional moderno. O que faltava era um cliente que falasse com humanos. Spring AI + MCP entrega esse cliente sem tocar no motor.

## O que sobra de trabalho

Não é livre de fricção. Os pontos onde mais tempo se gasta:

- **Calibrar descrições.** Frases ambíguas causam tool errada. O ciclo é: rodar prompts representativos, ver o que o modelo escolheu, ajustar prosa. É um loop, não um one-shot.
- **Cuidar do escopo de leitura.** A primeira versão expõe só leitura. Tools de escrita exigem outra ordem de cuidado — confirmação humana, audit log, idempotência. Não embarque tudo de uma vez.
- **Cache e custo.** LLM faz tool calls que podem ser repetitivas. Cache em memória curta (segundos) na borda da DRAPI corta custo de chamadas duplicadas no mesmo turno de conversa.

Mas o esqueleto — as três camadas, a anotação, a descoberta dinâmica — é pequeno. Algumas centenas de linhas de Java pra ter uma base de décadas conversando em linguagem natural com modelos de fronteira. A parte fascinante é que o stack alvo não precisou se modernizar: é o stack ao redor que ficou inteligente o bastante pra aproveitá-lo como ele é.
