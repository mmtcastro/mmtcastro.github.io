---
layout: post
title: "Multivalue fields do Domino como objetos Java: de arrays paralelos a grids editaveis no Vaadin"
date: 2026-02-27
categories: [domino, drapi, java, vaadin, multivalue]
---

No Domino, um campo multivalue armazena multiplos valores em um unico campo. E muito comum usar varios campos multivalue em paralelo para representar uma "tabela" dentro de um documento — por exemplo, tres campos `productName`, `productQty` e `productPrice`, cada um com N valores, onde o indice i de cada campo forma a "linha" i de uma tabela logica de produtos.

Quando migramos para Java/Vaadin, precisamos transformar esses arrays paralelos em **listas de objetos** que facao sentido no mundo orientado a objetos — e que possam ser exibidos em grids editaveis do Vaadin. E depois, na hora de salvar, converter tudo de volta para arrays paralelos para manter a compatibilidade com o Form do Domino.

## O problema: arrays paralelos vs objetos

No Domino, um documento com produtos fica assim internamente:

```
Document
├── productName:  ["Notebook", "Mouse", "Monitor"]
├── productQty:   [2, 10, 3]
├── productPrice: [4500.00, 89.90, 1200.00]
└── productUnit:  ["un", "un", "un"]
```

No DRAPI, o JSON reflete exatamente essa estrutura:

```json
{
  "@unid": "ABC123...",
  "Codigo": "ORDER-001",
  "productName":  ["Notebook", "Mouse", "Monitor"],
  "productQty":   [2, 10, 3],
  "productPrice": [4500.00, 89.90, 1200.00],
  "productUnit":  ["un", "un", "un"]
}
```

No Java, o que queremos e:

```java
List<Product> products = [
    Product { name: "Notebook", qty: 2, price: 4500.00, unit: "un" },
    Product { name: "Mouse",    qty: 10, price: 89.90,  unit: "un" },
    Product { name: "Monitor",  qty: 3, price: 1200.00, unit: "un" }
]
```

E no Vaadin, queremos um grid editavel:

```
┌──────────┬─────┬──────────┬────────┬─────────┐
│ Name     │ Qty │ Price    │ Unit   │ Actions │
├──────────┼─────┼──────────┼────────┼─────────┤
│ Notebook │  2  │ 4.500,00 │ un     │ ✏ 🗑    │
│ Mouse    │ 10  │    89,90 │ un     │ ✏ 🗑    │
│ Monitor  │  3  │ 1.200,00 │ un     │ ✏ 🗑    │
└──────────┴─────┴──────────┴────────┴─────────┘
                                      [+ Adicionar]
```

## A solucao: AbstractModelDocMultivalue e AbstractModelListaMultivalue

Criamos duas classes abstratas que formam a base do padrao:

### AbstractModelDocMultivalue — o item individual

Cada "linha" da tabela logica e representada por um objeto que estende esta classe:

```java
@Getter @Setter
public class AbstractModelDocMultivalue extends AbstractModel
        implements Comparable<AbstractModelDocMultivalue> {

    protected String idMulti; // ID unico para controle de add/remove

    public AbstractModelDocMultivalue() {
        idMulti = UUID.randomUUID().toString().substring(0, 5);
    }

    // Retorna apenas os campos validos (exclui idMulti, serialVersionUID, etc.)
    public ArrayList<Field> getValidFields() {
        ArrayList<String> removeFields = new ArrayList<>();
        removeFields.add("idMulti");
        removeFields.add("serialVersionUID");

        Field[] declaredFields = this.getClass().getDeclaredFields();
        ArrayList<Field> fields = new ArrayList<>();
        for (Field field : declaredFields) {
            if (!field.isSynthetic() && !removeFields.contains(field.getName())) {
                fields.add(field);
            }
        }
        return fields;
    }

    @Override
    public int compareTo(AbstractModelDocMultivalue other) {
        return 0; // subclasses podem implementar ordenacao customizada
    }
}
```

O `idMulti` e um UUID curto (5 caracteres) que identifica cada item do lado do cliente. Isso permite que o Vaadin rastreie adicoes e remocoes sem depender de indices de array.

### AbstractModelListaMultivalue — o container da lista

A lista de itens e encapsulada em um container generico:

```java
@Getter @Setter
public class AbstractModelListaMultivalue<E extends AbstractModelDocMultivalue>
        extends AbstractModel implements Iterable<E> {

    protected List<E> lista = new ArrayList<>();

    // Operacoes de colecao
    public void addBottom(E model) { lista.add(model); sort(); }
    public void addTop(E model)    { lista.add(0, model); sort(); }

    // Busca e remocao por idMulti
    public E get(String id) {
        for (E doc : lista) {
            if (id.equals(doc.getIdMulti())) return doc;
        }
        return null;
    }

    public void remove(E doc) {
        for (int i = 0; i < lista.size(); i++) {
            if (doc.getIdMulti().equals(lista.get(i).getIdMulti())) {
                lista.remove(i);
                sort();
                return;
            }
        }
    }

    public void sort() { Collections.sort(lista); }

    @Override
    public Iterator<E> iterator() { return lista.iterator(); }

    // Converte para Map<campo, List<valores>> — util para persistencia
    public Map<String, List<?>> convertToMap() {
        Map<String, List<?>> ret = new HashMap<>();
        for (E model : lista) {
            for (Field campo : model.getAllModelFields()) {
                campo.setAccessible(true);
                Method getter = model.getGetMethod(campo.getName());
                if (getter != null) {
                    List<Object> bucket = (List<Object>) ret
                        .computeIfAbsent(campo.getName(), k -> new ArrayList<>());
                    bucket.add(getter.invoke(model));
                }
            }
        }
        return ret;
    }
}
```

## Inner classes: o padrao mais comum

O caso mais frequente e definir a estrutura multivalue como **inner classes** dentro do modelo pai. Isso e ideal quando a estrutura pertence exclusivamente a aquele documento:

```java
@Getter @Setter
public class Order extends AbstractModelDoc {

    // Campos simples do documento
    @JsonProperty("Codigo")
    private String codigo;

    @JsonProperty("Customer")
    private String customer;

    // Campo multivalue — marcado com @JsonIgnore
    @JsonIgnore
    private Products products;

    // Lazy initialization
    public Products getProducts() {
        if (products == null) products = new Products();
        return products;
    }

    public void setProducts(Products p) {
        this.products = (p != null ? p : new Products());
    }

    // --- Inner classes multivalue ---

    @Getter @Setter
    public static class Product extends AbstractModelDocMultivalue {
        private String name;
        private Integer qty;
        private Double price;
        private String unit;
    }

    @Getter @Setter
    public static class Products extends AbstractModelListaMultivalue<Product> {}
}
```

**Pontos importantes:**

- O campo `products` e marcado com `@JsonIgnore` — ele nao vem diretamente no JSON do DRAPI
- Os dados reais vem nos arrays paralelos: `productName`, `productQty`, `productPrice`, `productUnit`
- A convencao de nomes e: **prefixo** (nome da inner class em minusculo) + **NomeDoCampo** (capitalizado)
- O wrapper (`Products`) e a forma plural do item (`Product`)

### Convencao de nomes

```
Inner class: Product       →  prefixo: "product"
Wrapper:     Products      →  campo:   "products"

Campos no Domino/DRAPI:
  product + Name  = "productName"
  product + Qty   = "productQty"
  product + Price = "productPrice"
  product + Unit  = "productUnit"
```

## Exemplo com multiplas listas: documento financeiro

Um documento pode ter varias estruturas multivalue independentes:

```java
@Getter @Setter
@JsonIgnoreProperties(ignoreUnknown = true)
public class Invoice extends AbstractModelDoc {

    @JsonProperty("Codigo")
    private String codigo;

    @JsonProperty("ValorTotal")
    private Double valorTotal;

    // Lista 1: parcelas de pagamento
    @JsonIgnore
    private Installments installments = new Installments();

    // Lista 2: itens da nota
    @JsonIgnore
    private LineItems lineItems = new LineItems();

    // Lista 3: historico de conciliacao
    @JsonIgnore
    private Reconciliations reconciliations = new Reconciliations();

    // --- Parcelas ---
    @Getter @Setter
    public static class Installment extends AbstractModelDocMultivalue {
        @JsonDeserialize(using = ZonedDateTimeDeserializer.class)
        private ZonedDateTime dueDate;
        private Double amount;
        private String account;
        private String description;
    }

    @Getter @Setter
    public static class Installments extends AbstractModelListaMultivalue<Installment> {}

    // --- Itens ---
    @Getter @Setter
    public static class LineItem extends AbstractModelDocMultivalue {
        private String productName;
        private Integer quantity;
        private Double unitPrice;
    }

    @Getter @Setter
    public static class LineItems extends AbstractModelListaMultivalue<LineItem> {}

    // --- Conciliacoes ---
    @Getter @Setter
    public static class Reconciliation extends AbstractModelDocMultivalue {
        @JsonDeserialize(using = ZonedDateTimeDeserializer.class)
        private ZonedDateTime date;
        private Double value;
        private String bankAccount;
    }

    @Getter @Setter
    public static class Reconciliations
            extends AbstractModelListaMultivalue<Reconciliation> {}
}
```

Cada lista e totalmente independente e mapeada para seu proprio conjunto de arrays paralelos no Domino.

## Classes independentes: reuso entre modelos

Quando a mesma estrutura multivalue aparece em varios documentos diferentes, faz sentido criar **classes independentes** em vez de inner classes. Por exemplo, se tanto `Order` quanto `Invoice` quanto `Quote` tem uma lista de contatos:

```java
// Arquivo: Contact.java (classe independente)
@Getter @Setter
public class Contact extends AbstractModelDocMultivalue {
    private String name;
    private String phone;
    private String email;
}

// Arquivo: Contacts.java (wrapper independente)
@Getter @Setter
public class Contacts extends AbstractModelListaMultivalue<Contact> {}
```

Uso em multiplos modelos:

```java
public class Order extends AbstractModelDoc {
    @JsonIgnore
    private Contacts contacts = new Contacts();
    // ...
}

public class Invoice extends AbstractModelDoc {
    @JsonIgnore
    private Contacts contacts = new Contacts();
    // ...
}

public class Quote extends AbstractModelDoc {
    @JsonIgnore
    private Contacts contacts = new Contacts();
    // ...
}
```

A vantagem e clara: uma unica definicao de `Contact`/`Contacts` reutilizada em tres modelos. Qualquer ajuste na estrutura (adicionar um campo, mudar o tipo) se propaga automaticamente.

## Como o load do DRAPI funciona: populaMultivalues()

O momento critico: quando o JSON do DRAPI chega, precisamos transformar arrays paralelos em objetos. O metodo `populaMultivalues()` faz isso via reflexao:

```java
private void populaMultivalues(Object model, JsonNode root) {
    Class<?> clazz = model.getClass();

    // 1. Encontrar inner classes que estendem AbstractModelDocMultivalue
    for (Class<?> inner : clazz.getDeclaredClasses()) {
        if (!AbstractModelDocMultivalue.class.isAssignableFrom(inner))
            continue;

        String prefix = inner.getSimpleName().toLowerCase(); // "product"
        Field[] innerFields = inner.getDeclaredFields();

        // 2. Ler colunas do JSON (arrays paralelos)
        Map<Field, List<Object>> columns = new LinkedHashMap<>();
        int size = 0;

        for (Field f : innerFields) {
            String mvKey = prefix + capitalize(f.getName()); // "productName"
            JsonNode arr = getIgnoreCase(root, mvKey);

            List<Object> values = new ArrayList<>();
            if (arr != null && arr.isArray()) {
                for (JsonNode n : arr) {
                    values.add(objectMapper.convertValue(n, f.getType()));
                }
            }
            columns.put(f, values);
            size = Math.max(size, values.size());
        }

        if (size == 0) continue; // nenhum array encontrado

        // 3. Montar linhas (transposta: colunas → objetos)
        List<Object> rows = new ArrayList<>(size);
        for (int i = 0; i < size; i++) {
            Object instance = inner.getDeclaredConstructor().newInstance();
            for (Map.Entry<Field, List<Object>> e : columns.entrySet()) {
                Field f = e.getKey();
                List<Object> vals = e.getValue();
                Object val = (i < vals.size()) ? vals.get(i) : null;
                f.setAccessible(true);
                f.set(instance, val);
            }
            rows.add(instance);
        }

        // 4. Injetar no wrapper (ex: "products")
        String wrapperName = addPlural(prefix); // "products"
        Field wrapperField = getFieldByNameDeep(clazz, wrapperName);
        if (wrapperField == null) continue;

        wrapperField.setAccessible(true);
        Object wrapper = wrapperField.get(model);
        if (wrapper == null) {
            wrapper = wrapperField.getType().getDeclaredConstructor().newInstance();
            wrapperField.set(model, wrapper);
        }

        // Substitui a lista dentro do wrapper
        Field listaField = getFieldByNameDeep(wrapper.getClass(), "lista");
        listaField.setAccessible(true);
        listaField.set(wrapper, rows);
    }
}
```

### O algoritmo visual

```
JSON do DRAPI (arrays paralelos):
  productName:  ["Notebook", "Mouse",  "Monitor"]
  productQty:   [2,          10,        3        ]
  productPrice: [4500.00,    89.90,     1200.00  ]

         │ transposta (colunas → linhas)
         ▼

Lista de objetos Java:
  [0] Product { name:"Notebook", qty:2,  price:4500.00 }
  [1] Product { name:"Mouse",    qty:10, price:89.90   }
  [2] Product { name:"Monitor",  qty:3,  price:1200.00 }
```

### Identificacao automatica de multivalue fields

O sistema identifica campos multivalue automaticamente via reflexao:

1. **Varre as inner classes** do modelo pai
2. **Filtra** as que estendem `AbstractModelDocMultivalue`
3. **Constroi nomes** usando a convencao `{prefix}{FieldName}`
4. **Busca no JSON** os arrays correspondentes (case-insensitive)

Nao e necessario nenhuma anotacao ou configuracao manual — basta seguir a convencao de nomes.

## Como o save funciona: flattenForDomino()

Na hora de salvar, o processo inverso transforma os objetos de volta em arrays paralelos:

```java
protected ObjectNode flattenForDomino(T model) {
    ObjectNode root = objectMapper.valueToTree(model);

    Class<?> clazz = model.getClass();
    for (Class<?> inner : clazz.getDeclaredClasses()) {
        if (!AbstractModelDocMultivalue.class.isAssignableFrom(inner))
            continue;

        String prefix = inner.getSimpleName().toLowerCase();
        String wrapperName = addPlural(prefix);
        Field wrapperField = getFieldByNameDeep(clazz, wrapperName);
        if (wrapperField == null) continue;

        wrapperField.setAccessible(true);
        Object wrapper = wrapperField.get(model);
        if (!(wrapper instanceof AbstractModelListaMultivalue<?> w)) continue;

        var rows = w.getLista();
        if (rows == null) continue;

        // Para cada campo da inner class, criar um array paralelo
        for (Field f : inner.getDeclaredFields()) {
            f.setAccessible(true);
            String arrayName = prefix + capitalize(f.getName());

            ArrayNode arrayValues = root.putArray(arrayName);
            boolean hasRealValue = false;

            for (Object row : rows) {
                if (row == null) continue;
                Object val = f.get(row);

                if (val == null) {
                    // Strings viram "" (aceito pelo Domino)
                    if (f.getType().equals(String.class)) {
                        arrayValues.add("");
                        hasRealValue = true;
                    }
                } else {
                    arrayValues.add(objectMapper.valueToTree(val));
                    hasRealValue = true;
                }
            }

            if (!hasRealValue) root.remove(arrayName);
        }

        // CRITICO: remover o wrapper do JSON final
        // Apenas os arrays paralelos devem ir para o Domino
        root.remove(wrapperName);
    }
    return root;
}
```

### O processo inverso

```
Lista de objetos Java:
  [0] Product { name:"Notebook", qty:2,  price:4500.00 }
  [1] Product { name:"Mouse",    qty:10, price:89.90   }
  [2] Product { name:"Monitor",  qty:3,  price:1200.00 }

         │ flattenForDomino() (linhas → colunas)
         ▼

JSON para o DRAPI (arrays paralelos):
{
  "productName":  ["Notebook", "Mouse",  "Monitor"],
  "productQty":   [2,          10,        3        ],
  "productPrice": [4500.00,    89.90,     1200.00  ]
}
```

O wrapper `products` e removido do JSON final — o Domino so entende arrays paralelos.

## No Vaadin: MultivalueGrid com edicao inline

Para exibir e editar os multivalue no Vaadin, criamos o componente `MultivalueGrid`:

```java
public class MultivalueGrid<T extends AbstractModelDocMultivalue>
        extends Composite<VerticalLayout> {

    private final AbstractModelListaMultivalue<T> source;
    private final List<T> data;  // referencia viva para source.lista
    private final Grid<T> grid;
    private final Binder<T> binder;
    private final Editor<T> editor;

    public MultivalueGrid(Class<T> beanType,
            AbstractModelListaMultivalue<T> source) {
        this.source = Objects.requireNonNull(source);

        // Referencia viva — alteracoes no grid refletem no modelo
        List<T> live = source.getLista();
        if (live == null) {
            live = new ArrayList<>();
            source.setLista(live);
        }
        this.data = live;

        this.grid = new Grid<>(beanType, false);
        grid.setItems(data);

        this.binder = new Binder<>(beanType);
        this.editor = grid.getEditor();
        this.editor.setBinder(binder);
        this.editor.setBuffered(true);

        // Listeners do editor
        editor.addSaveListener(e -> syncToMultivalueFields());
    }

    // Adicionar e editar
    public void addItem(T item) {
        data.add(item);
        refresh();
        editor.editItem(item);
    }

    // Remover
    public boolean deleteItem(T item) {
        if (editor.isOpen() && Objects.equals(editor.getItem(), item)) {
            editor.cancel();
        }
        boolean removed = data.remove(item);
        refresh();
        return removed;
    }
}
```

### Uso na view: definindo colunas com fluent API

```java
@Override
protected void configureBinder(Binder<Order> binder) {
    // Campos simples do formulario...

    // Grid multivalue
    var products = model.getProducts();
    productGrid = new MultivalueGrid<>(Order.Product.class, products)
        .withColumns(config -> {
            config.addTextFieldColumn("Name",
                Order.Product::getName,
                Order.Product::setName);

            config.addDoubleFieldColumn("Qty",
                Order.Product::getQty,
                Order.Product::setQty);

            config.addDoubleFieldColumn("Price",
                Order.Product::getPrice,
                Order.Product::setPrice);

            config.addComboBoxColumn("Unit",
                Order.Product::getUnit,
                Order.Product::setUnit,
                List.of("un", "kg", "m", "box"));
        })
        .enableAddButton("Add", () -> {
            Order.Product p = new Order.Product();
            p.setUnit("un");
            p.setQty(1);
            p.setPrice(0.0);
            return p;
        })
        .addActionColumn()
        .setReadOnly(isReadOnly);

    productGrid.setLabel("Products");
    productGrid.setItems(products.getLista());

    form.setColspan(productGrid, 2);
    form.add(productGrid);
}
```

O `MultivalueGrid` oferece:
- **Colunas tipadas** (texto, numero, data, combobox)
- **Edicao inline** com buffered editor
- **Botao +** no header para adicionar itens
- **Botoes de acao** por linha (editar, salvar, cancelar, excluir)
- **Modo read-only** que esconde todos os controles de edicao

## O ciclo completo

```
Domino Form
  productName:  ["Notebook", "Mouse"]
  productQty:   [2, 10]
  productPrice: [4500.00, 89.90]
        │
        ▼  DRAPI GET /document/{unid}
JSON com arrays paralelos
        │
        ▼  populaMultivalues() — reflexao
List<Product>
  [0] { name:"Notebook", qty:2,  price:4500.00 }
  [1] { name:"Mouse",    qty:10, price:89.90   }
        │
        ▼  MultivalueGrid — referencia viva
Grid editavel no Vaadin
  ┌──────────┬─────┬──────────┬─────────┐
  │ Notebook │  2  │ 4.500,00 │ ✏ 🗑    │
  │ Mouse    │ 10  │    89,90 │ ✏ 🗑    │
  └──────────┴─────┴──────────┴─────────┘
  Usuario edita, adiciona "Monitor"...
        │
        ▼  flattenForDomino() — reflexao inversa
JSON com arrays paralelos (atualizado)
  productName:  ["Notebook", "Mouse", "Monitor"]
  productQty:   [2, 10, 3]
  productPrice: [4500.00, 89.90, 1200.00]
        │
        ▼  DRAPI PUT /document/{unid}
Domino Form (atualizado, compativel com Notes client)
```

## Inner class vs classe independente: quando usar cada uma

| Criterio | Inner class | Classe independente |
|---|---|---|
| Uso em um unico modelo | Ideal | Desnecessario |
| Uso em multiplos modelos | Duplicacao | Ideal — uma definicao, N usos |
| Escopo | Vinculada ao modelo pai | Global / compartilhada |
| Convencao de nomes | Automatica (prefixo = classe pai) | Precisa seguir a mesma convencao |
| Exemplo | `Order.Product` | `Contact` (usado em Order, Invoice, Quote) |

## Licoes aprendidas

| Desafio | Solucao |
|---|---|
| Arrays paralelos no Domino vs objetos no Java | Transposta via reflexao (colunas ↔ linhas) |
| Manter sincronizacao bidirecional | `populaMultivalues()` no load, `flattenForDomino()` no save |
| Rastrear add/remove no grid | `idMulti` (UUID curto) para cada item |
| Campos null em arrays de tamanhos diferentes | Preencher com `""` para Strings, ignorar para outros tipos |
| Grid precisa de referencia viva | `MultivalueGrid` usa `source.getLista()` diretamente |
| Compatibilidade com Notes client | `flattenForDomino()` remove wrapper e gera apenas arrays |
| Reusar estrutura em varios modelos | Classes independentes (nao inner) |
| Identificar campos multivalue automaticamente | Reflexao + convencao de nomes ({prefix}{Field}) |

---

Com esse padrao, campos multivalue do Domino — um conceito sem equivalente direto em bancos relacionais — se tornam listas de objetos Java de primeira classe. Voce manipula, edita no grid, valida campo a campo, e salva de volta mantendo total compatibilidade com o Form do Notes. O Domino continua funcionando normalmente — ele nem sabe que os dados foram editados por uma aplicacao Vaadin.
