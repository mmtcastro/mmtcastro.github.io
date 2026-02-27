---
layout: post
title: "Dashboard no Vaadin com CSS puro: KPI cards, donut charts e bar charts sem bibliotecas externas"
date: 2026-02-27
categories: [domino, drapi, java, vaadin, dashboard]
---

Quando precisamos de um cockpit com KPIs e graficos no Vaadin, a primeira opcao seria uma biblioteca como Chart.js ou Recharts. Mas descobrimos que CSS moderno — com `conic-gradient`, `linear-gradient` e transicoes — resolve 90% dos casos de dashboard sem nenhuma dependencia externa. Este post mostra como construimos um cockpit completo no Vaadin com CSS puro.

## O resultado: um cockpit de faturas

O dashboard exibe:
- **6 KPI cards** com icones e gradientes
- **Donut charts** com `conic-gradient()` para distribuicao por status e tipo
- **Bar charts** com barras animadas para evolucao mensal
- **Seletor de periodo** (Mes, Trimestre, Ano) com DatePickers

Tudo em componentes Vaadin Java-side — zero JavaScript.

## KPI Cards com linear-gradient

```java
private Div createKpiCard(String title, String value,
        VaadinIcon iconType, String color) {

    Div card = new Div();
    card.getStyle()
        .set("padding", "var(--lumo-space-l)")
        .set("border-radius", "12px")
        .set("background",
            "linear-gradient(135deg, "
            + color + "15, " + color + "05)")
        .set("border", "1px solid " + color + "30")
        .set("min-width", "160px")
        .set("flex", "1 1 160px")
        .set("box-shadow",
            "0 4px 6px -1px rgba(0, 0, 0, 0.1)");

    // Container do icone com fundo colorido
    Icon icon = iconType.create();
    icon.getStyle()
        .set("color", color)
        .set("width", "28px")
        .set("height", "28px");

    Div iconContainer = new Div(icon);
    iconContainer.getStyle()
        .set("background", color + "20")
        .set("border-radius", "8px")
        .set("padding", "var(--lumo-space-s)")
        .set("display", "inline-flex")
        .set("margin-bottom", "var(--lumo-space-s)");

    // Valor principal
    H3 valueLabel = new H3(value);
    valueLabel.getStyle()
        .set("margin", "0")
        .set("font-size", "1.25rem")
        .set("font-weight", "700");

    // Titulo
    Span titleLabel = new Span(title);
    titleLabel.getStyle()
        .set("color", "var(--lumo-secondary-text-color)")
        .set("font-size", "0.8rem");

    VerticalLayout content = new VerticalLayout(
        iconContainer, valueLabel, titleLabel);
    content.setPadding(false);
    content.setSpacing(false);

    card.add(content);
    return card;
}
```

### Uso: 6 cards em layout flexbox

```java
HorizontalLayout kpiLayout = new HorizontalLayout();
kpiLayout.setWidthFull();
kpiLayout.getStyle()
    .set("flex-wrap", "wrap")
    .set("gap", "var(--lumo-space-m)");

kpiLayout.add(
    createKpiCard("Total de Faturas",
        String.valueOf(totalFaturas),
        VaadinIcon.FILE_TEXT_O, "#4f46e5"),
    createKpiCard("Valor Total",
        CURRENCY_FORMAT.format(valorTotal),
        VaadinIcon.MONEY, "#22c55e"),
    createKpiCard("Faturas este Mes",
        String.valueOf(faturasMesAtual),
        VaadinIcon.CALENDAR, "#a855f7"),
    createKpiCard("Valor este Mes",
        CURRENCY_FORMAT.format(valorMesAtual),
        VaadinIcon.CHART, "#10b981"),
    createKpiCard("Situacao OK",
        String.valueOf(faturasOk),
        VaadinIcon.CHECK_CIRCLE, "#06b6d4"),
    createKpiCard("Vendas",
        String.valueOf(faturasVenda),
        VaadinIcon.CART, "#f97316")
);
```

O truque esta nos sufixos hexadecimais de opacidade: `color + "15"` = 8% opacidade para o background, `color + "30"` = 19% para a borda. Isso cria o efeito de card "tinted" sem precisar de RGBA.

## Donut Charts com conic-gradient

CSS `conic-gradient()` cria graficos de pizza nativamente. Para transformar em donut, adicionamos um circulo branco no centro:

### Construindo o gradiente

```java
private String buildConicGradient(
        Map<String, Long> counts) {

    long total = counts.values().stream()
        .mapToLong(Long::longValue).sum();

    if (total == 0) {
        return "conic-gradient("
            + "var(--lumo-contrast-20pct) 0 100%)";
    }

    double start = 0;
    StringBuilder gradient = new StringBuilder(
        "conic-gradient(");
    int colorIndex = 0;

    for (Map.Entry<String, Long> entry
            : counts.entrySet()) {
        double percent =
            (entry.getValue() * 100.0) / total;
        double end = start + percent;

        gradient.append(getPaletteColor(colorIndex))
            .append(" ").append(start)
            .append("% ").append(end).append("%");

        start = end;
        colorIndex++;
        if (colorIndex < counts.size()) {
            gradient.append(", ");
        }
    }
    gradient.append(")");
    return gradient.toString();
}
```

### Resultado CSS gerado

```css
conic-gradient(
  #4f46e5 0% 33.33%,   /* Status A: 33.33% */
  #22c55e 33.33% 66.66%, /* Status B: 33.33% */
  #f97316 66.66% 100%    /* Status C: 33.34% */
)
```

### O componente donut

```java
// Circulo externo com gradiente
Div chartOuter = new Div();
chartOuter.getStyle()
    .set("width", "120px")
    .set("height", "120px")
    .set("border-radius", "50%")
    .set("background", buildConicGradient(counts))
    .set("position", "relative")
    .set("box-shadow",
        "0 2px 8px rgba(0, 0, 0, 0.1)");

// Circulo interno (buraco do donut)
Div chartInner = new Div();
chartInner.getStyle()
    .set("width", "60px")
    .set("height", "60px")
    .set("border-radius", "50%")
    .set("background", "var(--lumo-base-color)")
    .set("position", "absolute")
    .set("top", "50%")
    .set("left", "50%")
    .set("transform", "translate(-50%, -50%)")
    .set("display", "flex")
    .set("align-items", "center")
    .set("justify-content", "center")
    .set("flex-direction", "column");

// Total no centro do donut
Span totalValue = new Span(String.valueOf(total));
totalValue.getStyle()
    .set("font-weight", "700")
    .set("font-size", "1rem")
    .set("color", "var(--lumo-primary-color)");
Span totalLabel = new Span("total");
totalLabel.getStyle()
    .set("font-size", "0.6rem")
    .set("color", "var(--lumo-secondary-text-color)");

chartInner.add(totalValue, totalLabel);
chartOuter.add(chartInner);
```

## Bar Charts com transicoes CSS

As barras horizontais usam width proporcional ao valor maximo e transicao animada:

```java
private HorizontalLayout createBarRow(String label,
        Long value, long max, int colorIndex) {

    Span labelSpan = new Span(label);
    labelSpan.getStyle()
        .set("min-width", "60px")
        .set("font-size", "0.8rem")
        .set("color", "var(--lumo-secondary-text-color)");

    // Container da barra (fundo cinza)
    Div barContainer = new Div();
    barContainer.getStyle()
        .set("flex-grow", "1")
        .set("background", "var(--lumo-contrast-10pct)")
        .set("border-radius", "6px")
        .set("height", "20px")
        .set("overflow", "hidden");

    // Barra colorida (proporcional ao valor)
    Div bar = new Div();
    double percent = max == 0 ? 0
        : (value * 100.0 / max);
    String color = getPaletteColor(colorIndex);

    bar.getStyle()
        .set("width", percent + "%")
        .set("height", "20px")
        .set("border-radius", "6px")
        .set("background",
            "linear-gradient(90deg, "
            + color + ", " + color + "cc)")
        .set("transition", "width 0.3s ease");

    barContainer.add(bar);

    // Valor numerico
    Span valueSpan = new Span(String.valueOf(value));
    valueSpan.getStyle()
        .set("min-width", "35px")
        .set("text-align", "right")
        .set("font-weight", "600")
        .set("color", color);

    HorizontalLayout row = new HorizontalLayout(
        labelSpan, barContainer, valueSpan);
    row.setWidthFull();
    row.setAlignItems(Alignment.CENTER);
    return row;
}
```

A propriedade `transition: width 0.3s ease` faz a barra animar suavemente quando os dados mudam.

## Seletor de periodo

```java
private HorizontalLayout createPeriodToolbar() {
    LocalDate now = LocalDate.now();

    Button mesBtn = new Button("Mes");
    mesBtn.addClickListener(e -> {
        startDatePicker.setValue(now.withDayOfMonth(1));
        endDatePicker.setValue(
            now.withDayOfMonth(now.lengthOfMonth()));
        loadData();
    });

    Button trimestreBtn = new Button("Trimestre");
    trimestreBtn.addClickListener(e -> {
        int q = (now.getMonthValue() - 1) / 3;
        LocalDate qStart = now.withMonth(q * 3 + 1)
            .withDayOfMonth(1);
        LocalDate qEnd = qStart.plusMonths(2)
            .withDayOfMonth(
                qStart.plusMonths(2).lengthOfMonth());
        startDatePicker.setValue(qStart);
        endDatePicker.setValue(qEnd);
        loadData();
    });

    Button anoBtn = new Button("Ano");
    anoBtn.addClickListener(e -> {
        startDatePicker.setValue(
            now.withMonth(1).withDayOfMonth(1));
        endDatePicker.setValue(
            now.withMonth(12).withDayOfMonth(31));
        loadData();
    });

    return new HorizontalLayout(mesBtn, trimestreBtn,
        anoBtn, startDatePicker, endDatePicker);
}
```

## Agregacao de dados

```java
// Contagem por mes
private Map<String, Long> countByMonth(
        List<Invoice> invoices) {
    Map<YearMonth, Long> counts = new LinkedHashMap<>();
    for (Invoice e : invoices) {
        if (e.getDate() == null) continue;
        YearMonth ym = YearMonth.from(e.getDate());
        counts.merge(ym, 1L, Long::sum);
    }
    DateTimeFormatter fmt =
        DateTimeFormatter.ofPattern("MM/yyyy");
    return counts.entrySet().stream()
        .sorted(Map.Entry.comparingByKey())
        .collect(Collectors.toMap(
            e -> fmt.format(e.getKey()),
            Map.Entry::getValue,
            (a, b) -> a, LinkedHashMap::new));
}

// Soma de valores por chave
private Map<String, Double> sumValueBy(
        List<Invoice> invoices,
        Function<Invoice, String> keyFn) {
    Map<String, Double> values = new LinkedHashMap<>();
    for (Invoice e : invoices) {
        String key = keyFn.apply(e);
        String normalized = (key == null || key.isBlank())
            ? "N/A" : key;
        values.merge(normalized, getValue(e), Double::sum);
    }
    return values.entrySet().stream()
        .sorted(Map.Entry.<String, Double>comparingByValue()
            .reversed())
        .collect(Collectors.toMap(
            Map.Entry::getKey, Map.Entry::getValue,
            (a, b) -> a, LinkedHashMap::new));
}
```

## Paleta de cores

```java
private String getPaletteColor(int index) {
    String[] palette = {
        "#4f46e5", "#22c55e", "#f97316", "#06b6d4",
        "#e11d48", "#a855f7", "#84cc16", "#64748b",
        "#facc15", "#0ea5e9"
    };
    return palette[index % palette.length];
}
```

## Licoes aprendidas

| Desafio | Solucao |
|---|---|
| Biblioteca de graficos adiciona peso | CSS conic-gradient() e linear-gradient() resolvem |
| Donut chart sem canvas/SVG | Dois circulos: externo (gradiente) + interno (branco) |
| Barras precisam animar | `transition: width 0.3s ease` no CSS |
| Cards precisam parecer modernos | linear-gradient com opacidade + box-shadow |
| Cores consistentes entre graficos | Paleta fixa de 10 cores com indice modular |
| Performance com muitos dados | Carregar por ano, filtrar client-side por periodo |

---

Com CSS moderno, o Vaadin permite construir dashboards profissionais sem JavaScript e sem dependencias externas. Os componentes sao 100% Java-side, renderizam rapido e se adaptam ao tema Lumo automaticamente.
