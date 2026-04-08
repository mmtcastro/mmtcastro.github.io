---
layout: post
title: "Batch Email Validation: ZeroBounce + Virtual Threads + Vaadin Dialog"
date: 2026-04-08
categories: [domino, drapi, java, spring, vaadin, zerobounce, virtual-threads, email]
lang: en
---

Inactive contacts with invalid emails accumulate in the system over the years. Some domains like `voeazul.com.br` correctly report that an email doesn't exist; others like `klabin.com.br` are catch-all and accept everything. We built a Vaadin dialog that validates all emails from a company or economic group in batch via ZeroBounce, and allows deleting or deactivating invalid contacts with a single click — respecting business rules.

## The problem

TDec has thousands of registered contacts. Many belong to people who changed companies, were let go, or simply no longer exist. Validating one by one is impractical. We needed:

1. Batch validation of all contacts in a company/group
2. Automatic classification: valid, invalid, uncertain (catch-all)
3. Smart action determination: **delete** (never did business) or **deactivate** (has commercial history)
4. Interface for the user to select and confirm actions

## ZeroBounce API

We use ZeroBounce's API to validate each email individually:

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

The response classifies as: `valid`, `invalid`, `do_not_mail`, or `catch-all`. We also added a credits balance query:

```java
public Integer getCredits() {
    // GET /getcredits?api_key=...
    // Returns {"Credits":"4750"}
}
```

The balance appears in the dialog header so the user knows how many validations remain.

## Virtual Threads: the Vaadin session challenge

Validating 40+ emails takes ~40 seconds. To avoid freezing the UI, we use `Thread.startVirtualThread()` with ProgressBar updates via `UI.access()`:

```java
Thread.startVirtualThread(() -> {
    for (ContatoValidation cv : validations) {
        Zerobounce result = zerobounceService.validarEmail(email);
        // classify result...
        
        ui.access(() -> {
            progressBar.setValue(current);
            progressLabel.setText(current + " / " + total);
        });
    }
});
```

But when trying to determine if a contact can be deleted (querying deals and purchases via DRAPI), we hit a wall: **virtual threads don't have access to VaadinSession**. The `getUserToken()` failed with "User not authenticated".

### Attempt 1: VaadinSession.setCurrent() — failed

Manually propagating the session doesn't work because `UI.getCurrent().getSession()` depends on thread-locals that don't propagate to virtual threads.

### Attempt 2: ui.access() — failed

The `ui.access()` callback also doesn't guarantee full session access when called from a virtual thread.

### Solution: executeJs round-trip

The solution was to use `getElement().executeJs("return true").then(...)` — this does a round-trip to the browser and the callback executes on a **real HTTP thread** with full Vaadin session:

```java
ui.access(() -> {
    // Validation finished, schedule permission analysis
    getElement().executeJs("return true").then(Boolean.class, result -> {
        // HERE the Vaadin session IS available!
        for (ContatoValidation cv : validations) {
            if (cv.getStatus() == EmailValidationStatus.INVALID) {
                determineAllowedAction(cv); // uses getUserToken() internally
            }
        }
    });
});
```

## Business rules: delete vs deactivate

We reused the existing `ContatoService.getPodeDeletar()`:

```java
private void determineAllowedAction(ContatoValidation cv) {
    List<String> motivos = contatoService.getPodeDeletar(cv.getContato());
    if (motivos == null || motivos.isEmpty()) {
        cv.setAllowedAction("Deletar"); // Delete
    } else {
        cv.setAllowedAction("Desativar"); // Deactivate - has deals or purchases
    }
}
```

Internally, `getPodeDeletar()` queries:
- `negocioService.existsByCodigoContato()` — view `(NegociosContatos)` in sales.nsf
- `comprasService.existsByCodigoFornecedor()` — view `(ComprasFornecedor)` in compras.nsf

### DRAPI gotcha: documents=true

We discovered that the `(NegociosContatos)` view returned `[]` for all queries. Testing via curl directly against DRAPI confirmed: **without the `documents=true` parameter, the view returns empty**. The fix:

```java
.uri("/lists/{viewName}?dataSource=sales&key={key}&count=1&documents=true&mode=default",
    "(NegociosContatos)", codigoContato)
```

## Dialog interface

The `EmailValidationDialog` shows:
- **Header**: title with contact count + ZeroBounce credits
- **Toolbar**: "Validate Emails" button + ProgressBar
- **Filters**: Invalid / All / Valid / Uncertain
- **Grid**: checkbox, name, email, company, status (color-coded badge), has deals, allowed action
- **Footer**: selection summary + "Execute Actions" button

Color-coded badges: green = valid, red = invalid, yellow = uncertain (catch-all).

Invalid contacts are pre-selected. The user reviews and confirms. Contacts with commercial history can only be deactivated — consistency is guaranteed by the same service used in the individual contact screen.

## Result

With 4,750 ZeroBounce credits, we can validate hundreds of contacts per economic group. In the test with the AZUL group (41 active contacts, domain `voeazul.com.br`), we identified 18 invalid emails in ~40 seconds. Those with deals were correctly marked as "Deactivate"; the rest as "Delete".
