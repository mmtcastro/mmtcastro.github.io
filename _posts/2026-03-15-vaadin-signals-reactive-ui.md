---
layout: post
title: "Vaadin 25.1 Signals: UI Reativa Server-Side sem JavaScript"
date: 2026-03-15
categories: [domino, drapi, java, spring, vaadin, signals, reactive]
---

O Vaadin 25.1 introduziu **Signals** — primitivos reativos server-side que funcionam como os hooks do React, mas executam inteiramente no servidor Java. Na nossa migracao do HCL Domino para Java/Vaadin, adotamos Signals para substituir flags booleanos e listeners manuais, simplificando drasticamente o controle de visibilidade e estado dos formularios.

## O problema: formularios complexos com estado condicional

No cadastro de contatos, o formulario segue um fluxo de **progressive disclosure** — campos aparecem conforme o usuario avanca:

1. Seleciona o **Grupo Economico** → aparece o ComboBox de Empresa
2. Seleciona a **Empresa** → aparece o campo de Email
3. Valida o **Email** (via ZeroBounce) → aparecem todos os campos extras (nome, telefone, cargo, etc.)

Antes de Signals, isso era controlado com flags booleanos e chamadas manuais a `setVisible()`:

```java
// ANTES: flag + listener manual
private boolean grupoSelecionado = false;

grupoEconomicoComboBox.addValueChangeListener(e -> {
    grupoSelecionado = e.getValue() != null;
    empresaComboBox.setVisible(grupoSelecionado && isNovo);
    // ... mais 10 linhas atualizando outros campos
});
```

Cada mudanca de estado exigia propagar manualmente para todos os componentes afetados. Com 5+ flags interdependentes, o codigo ficava fragil e propenso a bugs.

## A solucao: ValueSignal + bindVisible

Com Vaadin Signals, declaramos o estado uma vez e os componentes **reagem automaticamente**:

```java
import com.vaadin.flow.signals.Signal;
import com.vaadin.flow.signals.local.ValueSignal;

// Signals reativos para progressive disclosure
private final ValueSignal<Boolean> grupoSelecionadoSignal = new ValueSignal<>(false);
private final ValueSignal<Boolean> empresaSelecionadaSignal = new ValueSignal<>(false);
private final ValueSignal<Boolean> emailValidadoSignal = new ValueSignal<>(false);
```

Os bindings ficam declarativos — uma linha por componente:

```java
// Step 1→2: Grupo selecionado → mostra empresa
empresaComboBox.bindVisible(() -> novoSignal.get() && grupoSelecionadoSignal.get());

// Step 2→3: Empresa selecionada → mostra email
codigoField.bindVisible(() -> empresaSelecionadaSignal.get() || !novoSignal.get());
codigoField.bindReadOnly(() -> !novoSignal.get());

// Step 3→extras: Email validado → mostra todos os campos
Signal<Boolean> extraVisible = () -> emailValidadoSignal.get() || !novoSignal.get();
bindAllVisible(extraVisible, tipoContato, primeiroNome, sobrenome, telefone, celular, cargo);
```

E o trigger e simples — um `set(true)` no momento certo:

```java
grupoEconomicoComboBox.addValueChangeListener(e -> {
    grupoSelecionadoSignal.set(e.getValue() != null && !e.getValue().isBlank());
});
```

## bindAllVisible: helper para aplicar Signal a multiplos componentes

Criamos um metodo utilitario no `AbstractViewDoc` para aplicar o mesmo Signal a varios componentes:

```java
protected void bindAllVisible(Signal<Boolean> signal, Component... components) {
    for (Component c : components) {
        c.bindVisible(signal::get);
    }
}
```

## Signals no AbstractViewDoc: readOnly e novo

Os Signals `readOnlySignal` e `novoSignal` sao definidos na classe base e controlam o comportamento de todos os formularios:

```java
// Na classe base AbstractViewDoc
protected final ValueSignal<Boolean> readOnlySignal = new ValueSignal<>(true);
protected final ValueSignal<Boolean> novoSignal = new ValueSignal<>(false);
```

Qualquer campo que deve ser somente-leitura se liga ao signal:

```java
bindAllReadOnly(readOnlySignal::get, nome, email, telefone, cargo);
```

Quando o usuario clica "Editar", basta `readOnlySignal.set(false)` e todos os campos reagem.

## Composicao de Signals

O poder real aparece na composicao. Um Signal pode depender de outros:

```java
// Campos extras visiveis se: email validado OU documento existente
Signal<Boolean> extraVisible = () -> emailValidadoSignal.get() || !novoSignal.get();
```

Isso e uma **expressao derivada** — recalcula automaticamente quando qualquer dependencia muda. Sem listeners, sem flags intermediarios, sem bugs de sincronizacao.

## Resultado

A adocao de Signals no ContatoView reduziu ~40 linhas de listeners manuais para ~10 linhas declarativas. O codigo ficou mais legivel, mais seguro (sem estados inconsistentes) e mais facil de manter. O padrao se espalhou para EmpresaView, GrupoEconomicoView e NegocioView com o mesmo beneficio.

Signals sao o equivalente server-side do que React hooks fizeram para o frontend — tornam o estado reativo a norma, nao a excecao.
