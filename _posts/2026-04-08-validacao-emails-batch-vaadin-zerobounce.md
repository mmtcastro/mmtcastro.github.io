---
layout: post
title: "Validacao de Emails em Lote: ZeroBounce + Virtual Threads + Vaadin Dialog"
date: 2026-04-08
categories: [domino, drapi, java, spring, vaadin, zerobounce, virtual-threads, email]
---

Contatos inativos com emails invalidos se acumulam no sistema ao longo dos anos. Alguns dominios como `voeazul.com.br` respondem corretamente que o email nao existe; outros como `klabin.com.br` sao catch-all e aceitam tudo. Construimos um dialog Vaadin que valida todos os emails de uma empresa ou grupo economico em lote via ZeroBounce, e permite deletar ou desativar contatos invalidos com um clique — respeitando regras de negocio.

## O problema

A TDec tem milhares de contatos cadastrados. Muitos sao de pessoas que mudaram de empresa, foram demitidas ou simplesmente nao existem mais. Validar um por um e impraticavel. Precisavamos de:

1. Validacao em batch de todos os contatos de uma empresa/grupo
2. Classificacao automatica: valido, invalido, incerto (catch-all)
3. Determinacao inteligente da acao: **deletar** (nunca fez negocios) ou **desativar** (tem historico comercial)
4. Interface para o usuario selecionar e confirmar as acoes

## ZeroBounce API

Usamos a API da ZeroBounce para validar cada email individualmente:

```java
public Zerobounce validarEmail(String email) {
    return webClient.get()
        .uri(uriBuilder -> uriBuilder.path("/validate")
            .queryParam("api_key", API_KEY)
            .queryParam("email", email).build())
        .retrieve().bodyToMono(Zerobounce.class)
        .onErrorResume(e -> Mono.empty()).block();
}
```

A resposta classifica em: `valid`, `invalid`, `do_not_mail`, ou `catch-all`. Tambem adicionamos consulta de creditos restantes:

```java
public Integer getCredits() {
    // GET /getcredits?api_key=...
    // Retorna {"Credits":"4750"}
}
```

O saldo aparece no header do dialog para o usuario saber quantas validacoes restam.

## Virtual Threads: o desafio da sessao Vaadin

A validacao de 40+ emails leva ~40 segundos. Para nao travar a UI, usamos `Thread.startVirtualThread()` com atualizacao da ProgressBar via `UI.access()`:

```java
Thread.startVirtualThread(() -> {
    for (ContatoValidation cv : validations) {
        Zerobounce result = zerobounceService.validarEmail(email);
        // classificar resultado...
        
        ui.access(() -> {
            progressBar.setValue(current);
            progressLabel.setText(current + " / " + total);
        });
    }
});
```

Mas ao tentar determinar se o contato pode ser deletado (consultando negocios e compras via DRAPI), encontramos o problema: **virtual threads nao tem acesso a VaadinSession**. O `getUserToken()` falhava com "Usuario nao autenticado".

### Tentativa 1: VaadinSession.setCurrent() — falhou

Propagar a sessao manualmente nao funciona porque `UI.getCurrent().getSession()` depende de thread-locals que nao se propagam para virtual threads.

### Tentativa 2: ui.access() — falhou

O callback de `ui.access()` tambem nao garante acesso completo a sessao quando chamado de uma virtual thread.

### Solucao: executeJs round-trip

A solucao foi usar `getElement().executeJs("return true").then(...)` — isso faz um round-trip ao browser e o callback executa em uma **thread HTTP real** com sessao Vaadin completa:

```java
ui.access(() -> {
    // Validacao terminou, agendar analise de permissoes
    getElement().executeJs("return true").then(Boolean.class, result -> {
        // AQUI a sessao Vaadin esta disponivel!
        for (ContatoValidation cv : validations) {
            if (cv.getStatus() == EmailValidationStatus.INVALID) {
                determineAllowedAction(cv); // usa getUserToken() internamente
            }
        }
    });
});
```

## Regras de negocio: deletar vs desativar

Reutilizamos o `ContatoService.getPodeDeletar()` que ja existia:

```java
private void determineAllowedAction(ContatoValidation cv) {
    List<String> motivos = contatoService.getPodeDeletar(cv.getContato());
    if (motivos == null || motivos.isEmpty()) {
        cv.setAllowedAction("Deletar");
    } else {
        cv.setAllowedAction("Desativar"); // tem negocios ou compras
    }
}
```

Internamente, `getPodeDeletar()` consulta:
- `negocioService.existsByCodigoContato()` — view `(NegociosContatos)` em sales.nsf
- `comprasService.existsByCodigoFornecedor()` — view `(ComprasFornecedor)` em compras.nsf

### Gotcha do DRAPI: documents=true

Descobrimos que a view `(NegociosContatos)` retornava `[]` em todas as consultas. Testando via curl direto no DRAPI, confirmamos: **sem o parametro `documents=true`, a view retorna vazia**. A correcao:

```java
.uri("/lists/{viewName}?dataSource=sales&key={key}&count=1&documents=true&mode=default",
    "(NegociosContatos)", codigoContato)
```

## Interface do dialog

O `EmailValidationDialog` mostra:
- **Header**: titulo com quantidade de contatos + creditos ZeroBounce
- **Toolbar**: botao "Validar Emails" + ProgressBar
- **Filtros**: Invalidos / Todos / Validos / Incertos
- **Grid**: checkbox, nome, email, empresa, status (badge colorido), fez negocios, acao permitida
- **Footer**: resumo dos selecionados + botao "Executar Acoes"

Badges coloridos: verde = valido, vermelho = invalido, amarelo = incerto (catch-all).

Contatos invalidos sao pre-selecionados. O usuario revisa e confirma. Contatos com historico comercial so podem ser desativados — a consistencia e garantida pelo mesmo servico usado na tela individual.

## Resultado

Com 4750 creditos ZeroBounce, podemos validar centenas de contatos por grupo economico. No teste com o grupo AZUL (41 contatos ativos, dominio `voeazul.com.br`), identificamos 18 emails invalidos em ~40 segundos. Os que tinham negocios foram corretamente marcados como "Desativar"; os demais como "Deletar".
