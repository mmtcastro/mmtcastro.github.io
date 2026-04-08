---
layout: post
title: "Vaadin 25.1 Signals: Server-Side Reactive UI Without JavaScript"
date: 2026-03-15
categories: [domino, drapi, java, spring, vaadin, signals, reactive]
lang: en
---

Vaadin 25.1 introduced **Signals** — server-side reactive state primitives that work like React hooks but run entirely on the Java server. In our HCL Domino to Java/Vaadin migration, we adopted Signals to replace boolean flags and manual listeners, dramatically simplifying form state and visibility control.

## The problem: complex forms with conditional state

Our contact registration form follows a **progressive disclosure** flow — fields appear as the user progresses:

1. Select **Economic Group** → Company ComboBox appears
2. Select **Company** → Email field appears
3. Validate **Email** (via ZeroBounce) → all extra fields appear (name, phone, role, etc.)

Before Signals, this was controlled with boolean flags and manual `setVisible()` calls:

```java
// BEFORE: flag + manual listener
private boolean grupoSelecionado = false;

grupoEconomicoComboBox.addValueChangeListener(e -> {
    grupoSelecionado = e.getValue() != null;
    empresaComboBox.setVisible(grupoSelecionado && isNovo);
    // ... 10 more lines updating other fields
});
```

Each state change required manual propagation to all affected components. With 5+ interdependent flags, the code was fragile and bug-prone.

## The solution: ValueSignal + bindVisible

With Vaadin Signals, we declare state once and components **react automatically**:

```java
import com.vaadin.flow.signals.Signal;
import com.vaadin.flow.signals.local.ValueSignal;

// Reactive signals for progressive disclosure
private final ValueSignal<Boolean> grupoSelecionadoSignal = new ValueSignal<>(false);
private final ValueSignal<Boolean> empresaSelecionadaSignal = new ValueSignal<>(false);
private final ValueSignal<Boolean> emailValidadoSignal = new ValueSignal<>(false);
```

Bindings become declarative — one line per component:

```java
// Step 1→2: Group selected → show company
empresaComboBox.bindVisible(() -> novoSignal.get() && grupoSelecionadoSignal.get());

// Step 2→3: Company selected → show email
codigoField.bindVisible(() -> empresaSelecionadaSignal.get() || !novoSignal.get());
codigoField.bindReadOnly(() -> !novoSignal.get());

// Step 3→extras: Email validated → show all fields
Signal<Boolean> extraVisible = () -> emailValidadoSignal.get() || !novoSignal.get();
bindAllVisible(extraVisible, tipoContato, firstName, lastName, phone, cell, role);
```

The trigger is simple — a `set(true)` at the right moment:

```java
grupoEconomicoComboBox.addValueChangeListener(e -> {
    grupoSelecionadoSignal.set(e.getValue() != null && !e.getValue().isBlank());
});
```

## bindAllVisible: helper for applying Signal to multiple components

We created a utility method in `AbstractViewDoc` to apply the same Signal to multiple components:

```java
protected void bindAllVisible(Signal<Boolean> signal, Component... components) {
    for (Component c : components) {
        c.bindVisible(signal::get);
    }
}
```

## Signals in AbstractViewDoc: readOnly and new document

The `readOnlySignal` and `novoSignal` are defined in the base class and control behavior across all forms:

```java
// In base class AbstractViewDoc
protected final ValueSignal<Boolean> readOnlySignal = new ValueSignal<>(true);
protected final ValueSignal<Boolean> novoSignal = new ValueSignal<>(false);
```

Any field that should be read-only binds to the signal:

```java
bindAllReadOnly(readOnlySignal::get, name, email, phone, role);
```

When the user clicks "Edit", a single `readOnlySignal.set(false)` makes all fields react.

## Signal composition

The real power comes from composition. A Signal can depend on others:

```java
// Extra fields visible if: email validated OR existing document
Signal<Boolean> extraVisible = () -> emailValidadoSignal.get() || !novoSignal.get();
```

This is a **derived expression** — it recalculates automatically when any dependency changes. No listeners, no intermediate flags, no synchronization bugs.

## Result

Adopting Signals in ContatoView reduced ~40 lines of manual listeners to ~10 declarative lines. The code became more readable, safer (no inconsistent states), and easier to maintain. The pattern spread to EmpresaView, GrupoEconomicoView, and NegocioView with the same benefit.

Signals are the server-side equivalent of what React hooks did for the frontend — they make reactive state the norm, not the exception.
