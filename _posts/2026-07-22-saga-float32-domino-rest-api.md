---
layout: post
title: "A saga float32: os centavos que sumiam na Domino REST API"
date: 2026-07-22
categories: [domino, drapi, java, spring, data-integrity, debugging]
---

Um dia, um relatório fechou diferente do ERP. Não muito — centavos. Um valor que no sistema era 3.209.185,63 chegava do outro lado como 3.209.185,8. Nenhum erro no log, nenhum warning, HTTP 200 em tudo. Só centavos sumindo, e apenas em valores grandes.

Se você já caçou um bug desses, sabe que centavos em valores milionários são o pior tipo de sintoma: pequenos demais para gritar, grandes demais para ignorar num sistema financeiro. Este post é a história completa dessa caça — da suspeita ao culpado improvável — na Domino REST API (antigo Project KEEP), a camada REST que uso para modernizar um ERP Domino de 25 anos. É a saga que o [post sobre o servidor MCP]({% post_url 2026-07-22-mcp-server-domino-rest-api %}) prometeu contar.

## O banco não mente

Primeira hipótese de qualquer um: dado corrompido no banco. Fui olhar o documento no cliente Notes: **3.209.185,63**, correto até o último centavo. Exportei o documento em DXL, o formato XML de baixo nível do Domino, que mostra o valor como ele vive no disco: precisão cheia. O legado sempre gravou os números como ponto flutuante de 64 bits, e 25 anos de dados estavam íntegros.

Mas o JSON que a API devolvia trazia 3.209.185,8. O mesmo documento. O disco não mentia — a serialização sim. O valor era truncado no caminho entre o disco e a resposta HTTP, tanto na leitura por view quanto na busca direta do documento.

Uma armadilha dentro da armadilha: a ferramenta de depuração também mente. Se você pega o JSON da resposta e faz *parse* para inspecionar, o parser da sua linguagem converte o número para float de 64 bits e formata bonito — mascarando exatamente o que você quer ver. A investigação só andou quando passei a inspecionar o **texto cru** da resposta, com regex, antes de qualquer parse. Em bug de precisão numérica, olhe os bytes, não os objetos.

## Sete dígitos

A causa estava no schema. Na Domino REST API, cada campo declarado no schema tem um `format` — e o servidor serializa o valor **conforme o format declarado**, não conforme o que está no disco. Os campos monetários estavam declarados como `number` com `format: "float"`.

`float` significa ponto flutuante de 32 bits. E float de 32 bits tem cerca de **7 dígitos significativos**.

Faça a conta: 3.209.185,63 tem nove dígitos significativos. Não cabe. O serializador arredonda para o float de 32 bits mais próximo — 3.209.185,8 — e é isso que sai no JSON. Valores pequenos passam ilesos (cabem nos 7 dígitos); valores na casa das centenas de milhares para cima perdem os centavos. Por isso o sintoma era seletivo: só os valores grandes, só nas pontas.

E aqui a história dá a volta que a torna digna de post. Quem declarou `float` nesses campos? **Nós.** O gerador de schemas que eu usava para provisionar os scopes da API mapeava campo numérico do Notes para `float` por default. Não era um bug do produto — era o meu provisionador reintroduzindo a armadilha, silenciosamente, em cada schema novo que gerava. O culpado da cena do crime era o mordomo da minha própria casa.

## O quase-desastre: ler e gravar de volta

Até aqui, um bug de leitura. Irritante, mas os dados estavam seguros. O perigo real era outro, e percebê-lo a tempo foi questão de sorte e paranoia.

Pense no ciclo mais natural de uma aplicação: buscar o documento pela API, alterar um campo qualquer — o status, uma observação — e salvar de volta. O valor monetário veio **truncado** na leitura. Se o cliente grava o documento inteiro de volta, ele grava a truncação. O disco, que estava íntegro, passa a ter 3.209.185,8 *de verdade*. O bug de leitura vira corrupção permanente de dado financeiro, espalhada silenciosamente por cada documento que qualquer processo tocar.

Foi por pouco. E é essa a lição que eu sublinharia três vezes: em uma API schema-driven, **um erro de leitura nunca é só de leitura** enquanto existir qualquer caminho de read-then-write-back.

## O conserto: um flip de schema, sem restart

A correção, felizmente, é anticlimática — no bom sentido. O `format` é um atributo do schema, e o schema é editável em runtime via POST no endpoint de administração (`setup-v1`): ler o schema, trocar `float` por `double` nos campos numéricos, gravar de volta. Sem restart do servidor, reversível, e o servidor não "re-descobre" o format sozinho — o flip sobrevive. Na versão que uso, o efeito é imediato: a mesma consulta que devolvia 3.209.185,8 passa a devolver 3.209.185,63.

O que *não* foi anticlimático foi garantir que o conserto pegasse **tudo**. A varredura que escrevi para auditar e flipar os campos tinha três pontos cegos, e cada um me custou uma rodada extra:

- **Tempo** — schemas gerados *depois* da varredura nasciam com `float` de novo, porque o gerador continuava com o default errado. Consertar os schemas sem consertar o gerador é enxugar gelo.
- **Lugar** — scopes provisionados fora do caminho habitual escaparam da primeira passada.
- **Profundidade** — campos multivalor (arrays) declaram o format em `items: {format}`, um nível abaixo de onde a varredura olhava. Escalares consertados, arrays ainda truncando. Foi o terceiro eixo, descoberto semanas depois.

E um alerta operacional que quase virou pegadinha: **backups de schema guardam o passado**. Um dump antigo restaurado via POST reintroduz `float` em massa, desfazendo o conserto num comando. Backup de schema é código — trate como tal.

## O atalho que era um beco

Antes do flip, cogitei o atalho clássico: "dinheiro dá problema em float? Declaro como string e formato eu mesmo." Não faça isso. Na Domino REST API, valor numérico contra campo declarado `string` no schema é **dropado no write** — o documento salva, HTTP 200, e o campo simplesmente não está lá. O atalho troca centavos perdidos por valores inteiros perdidos.

A regra que ficou, e que hoje vale como diretiva no projeto: dinheiro é `number` com `format: "double"` no schema, `Double` no cliente Java, ponta a ponta. E diferença de centavo em comparação de valores é **bug para investigar, nunca tolerância para acomodar** — foi tolerância a "diferencinhas" que deixou esse bug invisível por tempo demais.

## Moral: o schema é um contrato de precisão

Três lições ficaram, em ordem crescente de generalidade:

1. **Em API schema-driven, o schema não é só o shape — é a precisão.** O servidor entrega o que o schema manda, não o que o disco tem. Um `format` errado corrompe dados sem tocar no banco.
2. **Audite seus geradores, não só seus dados.** O bug não estava em nenhum schema específico; estava na fábrica de schemas. Varreduras corrigem o estoque; só consertar o gerador estanca a fonte.
3. **Teste round-trip dos campos numéricos no início do projeto.** Escrever um valor de nove dígitos significativos, ler de volta, comparar byte a byte no JSON cru. Um teste de cinco linhas teria pago meses de caça.

Hoje essa saga inteira tem um epílogo curioso: ela virou verbete no compêndio que o [servidor MCP serve como ferramenta]({% post_url 2026-07-22-mcp-server-domino-rest-api %}) — foi justamente a história que uma sessão de IA sem contexto nenhum me recontou, por conta própria, quando testei a ferramenta. A lição custou caro uma vez; agora é infraestrutura.

---

A versão de referência desta história — Sintoma → Causa → Correção, junto com as demais armadilhas de schema e tipos — está no [schema-types.md](https://github.com/mmtcastro/domino-rest-api-guide/blob/main/schema-types.md) do repo público [domino-rest-api-guide](https://github.com/mmtcastro/domino-rest-api-guide).
