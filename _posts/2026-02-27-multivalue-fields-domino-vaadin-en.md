---
layout: post
title: "Domino multivalue fields as Java objects: from parallel arrays to editable Vaadin grids"
date: 2026-02-27
categories: [domino, drapi, java, vaadin, multivalue]
lang: en
---

In Domino, a multivalue field stores multiple values in a single field. It is very common to use several parallel multivalue fields to represent a "table" within a document — for example, three fields `productName`, `productQty`, and `productPrice`, each with N values, where index i of each field forms "row" i of a logical product table.

When migrating to Java/Vaadin, we need to transform these parallel arrays into **lists of objects** that make sense in the object-oriented world — and that can be displayed in editable Vaadin grids. And when saving, convert everything back to parallel arrays to maintain compatibility with the Domino Form.

## The problem: parallel arrays vs objects

In Domino, a document with products looks like this internally:

```
Document
├── productName:  ["Notebook", "Mouse", "Monitor"]
├── productQty:   [2, 10, 3]
├── productPrice: [4500.00, 89.90, 1200.00]
└── productUnit:  ["un", "un", "un"]
```

In DRAPI, the JSON reflects exactly this structure:

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

In Java, what we want is:

```java
List<Product> products = [
    Product { name: "Notebook", qty: 2, price: 4500.00, unit: "un" },
    Product { name: "Mouse",    qty: 10, price: 89.90,  unit: "un" },
    Product { name: "Monitor",  qty: 3, price: 1200.00, unit: "un" }
]
```

And in Vaadin, we want an editable grid:

```
┌──────────┬─────┬──────────┬────────┬─────────┐
│ Name     │ Qty │ Price    │ Unit   │ Actions │
├──────────┼─────┼──────────┼────────┼─────────┤
│ Notebook │  2  │ 4,500.00 │ un     │ ✏ 🗑    │
│ Mouse    │ 10  │    89.90 │ un     │ ✏ 🗑    │
│ Monitor  │  3  │ 1,200.00 │ un     │ ✏ 🗑    │
└──────────┴─────┴──────────┴────────┴─────────┘
                                      [+ Add]
```

## The solution: AbstractModelDocMultivalue and AbstractModelListaMultivalue

We created two abstract classes that form the foundation of the pattern:

### AbstractModelDocMultivalue — the individual item

Each "row" of the logical table is represented by an object extending this class:

```java
@Getter @Setter
public class AbstractModelDocMultivalue extends AbstractModel
        implements Comparable<AbstractModelDocMultivalue> {

    protected String idMulti; // Unique ID for add/remove control

    public AbstractModelDocMultivalue() {
        idMulti = UUID.randomUUID().toString().substring(0, 5);
    }

    // Returns only valid fields (excludes idMulti, serialVersionUID, etc.)
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
        return 0; // subclasses can implement custom sorting
    }
}
```

The `idMulti` is a short UUID (5 characters) that identifies each item on the client side. This allows Vaadin to track additions and removals without depending on array indices.

### AbstractModelListaMultivalue — the list container

The list of items is encapsulated in a generic container:

```java
@Getter @Setter
public class AbstractModelListaMultivalue<E extends AbstractModelDocMultivalue>
        extends AbstractModel implements Iterable<E> {

    protected List<E> lista = new ArrayList<>();

    // Collection operations
    public void addBottom(E model) { lista.add(model); sort(); }
    public void addTop(E model)    { lista.add(0, model); sort(); }

    // Search and removal by idMulti
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

    // Converts to Map<field, List<values>> — useful for persistence
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

## Inner classes: the most common pattern

The most frequent case is defining the multivalue structure as **inner classes** within the parent model. This is ideal when the structure belongs exclusively to that document:

```java
@Getter @Setter
public class Order extends AbstractModelDoc {

    // Simple document fields
    @JsonProperty("Codigo")
    private String codigo;

    @JsonProperty("Customer")
    private String customer;

    // Multivalue field — marked with @JsonIgnore
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

    // --- Multivalue inner classes ---

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

**Key points:**

- The `products` field is marked with `@JsonIgnore` — it does not come directly from the DRAPI JSON
- The actual data comes in parallel arrays: `productName`, `productQty`, `productPrice`, `productUnit`
- The naming convention is: **prefix** (inner class name in lowercase) + **FieldName** (capitalized)
- The wrapper (`Products`) is the plural form of the item (`Product`)

### Naming convention

```
Inner class: Product       →  prefix: "product"
Wrapper:     Products      →  field:  "products"

Fields in Domino/DRAPI:
  product + Name  = "productName"
  product + Qty   = "productQty"
  product + Price = "productPrice"
  product + Unit  = "productUnit"
```

## Example with multiple lists: financial document

A document can have several independent multivalue structures:

```java
@Getter @Setter
@JsonIgnoreProperties(ignoreUnknown = true)
public class Invoice extends AbstractModelDoc {

    @JsonProperty("Codigo")
    private String codigo;

    @JsonProperty("TotalAmount")
    private Double totalAmount;

    // List 1: payment installments
    @JsonIgnore
    private Installments installments = new Installments();

    // List 2: invoice line items
    @JsonIgnore
    private LineItems lineItems = new LineItems();

    // List 3: reconciliation history
    @JsonIgnore
    private Reconciliations reconciliations = new Reconciliations();

    // --- Installments ---
    @Getter @Setter
    public static class Installment extends AbstractModelDocMultivalue {
        @JsonDeserialize(using = ZonedDateTimeDeserializer.class)
        private ZonedDateTime dueDate;
        private Double amount;
        private String account;
        private String description;
    }

    @Getter @Setter
    public static class Installments
            extends AbstractModelListaMultivalue<Installment> {}

    // --- Line items ---
    @Getter @Setter
    public static class LineItem extends AbstractModelDocMultivalue {
        private String productName;
        private Integer quantity;
        private Double unitPrice;
    }

    @Getter @Setter
    public static class LineItems extends AbstractModelListaMultivalue<LineItem> {}

    // --- Reconciliations ---
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

Each list is fully independent and mapped to its own set of parallel arrays in Domino.

## Standalone classes: reuse across models

When the same multivalue structure appears in several different documents, it makes sense to create **standalone classes** instead of inner classes. For example, if `Order`, `Invoice`, and `Quote` all have a contact list:

```java
// File: Contact.java (standalone class)
@Getter @Setter
public class Contact extends AbstractModelDocMultivalue {
    private String name;
    private String phone;
    private String email;
}

// File: Contacts.java (standalone wrapper)
@Getter @Setter
public class Contacts extends AbstractModelListaMultivalue<Contact> {}
```

Usage in multiple models:

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

The advantage is clear: a single definition of `Contact`/`Contacts` reused across three models. Any adjustment to the structure (adding a field, changing a type) automatically propagates.

## How the DRAPI load works: populaMultivalues()

The critical moment: when JSON arrives from DRAPI, we need to transform parallel arrays into objects. The `populaMultivalues()` method does this via reflection:

```java
private void populaMultivalues(Object model, JsonNode root) {
    Class<?> clazz = model.getClass();

    // 1. Find inner classes extending AbstractModelDocMultivalue
    for (Class<?> inner : clazz.getDeclaredClasses()) {
        if (!AbstractModelDocMultivalue.class.isAssignableFrom(inner))
            continue;

        String prefix = inner.getSimpleName().toLowerCase(); // "product"
        Field[] innerFields = inner.getDeclaredFields();

        // 2. Read columns from JSON (parallel arrays)
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

        if (size == 0) continue; // no arrays found

        // 3. Build rows (transpose: columns → objects)
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

        // 4. Inject into wrapper (e.g., "products")
        String wrapperName = addPlural(prefix); // "products"
        Field wrapperField = getFieldByNameDeep(clazz, wrapperName);
        if (wrapperField == null) continue;

        wrapperField.setAccessible(true);
        Object wrapper = wrapperField.get(model);
        if (wrapper == null) {
            wrapper = wrapperField.getType().getDeclaredConstructor().newInstance();
            wrapperField.set(model, wrapper);
        }

        // Replace the list inside the wrapper
        Field listaField = getFieldByNameDeep(wrapper.getClass(), "lista");
        listaField.setAccessible(true);
        listaField.set(wrapper, rows);
    }
}
```

### The visual algorithm

```
DRAPI JSON (parallel arrays):
  productName:  ["Notebook", "Mouse",  "Monitor"]
  productQty:   [2,          10,        3        ]
  productPrice: [4500.00,    89.90,     1200.00  ]

         │ transpose (columns → rows)
         ▼

Java object list:
  [0] Product { name:"Notebook", qty:2,  price:4500.00 }
  [1] Product { name:"Mouse",    qty:10, price:89.90   }
  [2] Product { name:"Monitor",  qty:3,  price:1200.00 }
```

### Automatic identification of multivalue fields

The system identifies multivalue fields automatically via reflection:

1. **Scans inner classes** of the parent model
2. **Filters** those extending `AbstractModelDocMultivalue`
3. **Builds field names** using the convention `{prefix}{FieldName}`
4. **Searches the JSON** for corresponding arrays (case-insensitive)

No annotations or manual configuration required — just follow the naming convention.

## How the save works: flattenForDomino()

When saving, the reverse process transforms objects back into parallel arrays:

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

        // For each inner class field, create a parallel array
        for (Field f : inner.getDeclaredFields()) {
            f.setAccessible(true);
            String arrayName = prefix + capitalize(f.getName());

            ArrayNode arrayValues = root.putArray(arrayName);
            boolean hasRealValue = false;

            for (Object row : rows) {
                if (row == null) continue;
                Object val = f.get(row);

                if (val == null) {
                    // Strings become "" (accepted by Domino)
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

        // CRITICAL: remove wrapper from the final JSON
        // Only parallel arrays should be sent to Domino
        root.remove(wrapperName);
    }
    return root;
}
```

### The reverse process

```
Java object list:
  [0] Product { name:"Notebook", qty:2,  price:4500.00 }
  [1] Product { name:"Mouse",    qty:10, price:89.90   }
  [2] Product { name:"Monitor",  qty:3,  price:1200.00 }

         │ flattenForDomino() (rows → columns)
         ▼

JSON for DRAPI (parallel arrays):
{
  "productName":  ["Notebook", "Mouse",  "Monitor"],
  "productQty":   [2,          10,        3        ],
  "productPrice": [4500.00,    89.90,     1200.00  ]
}
```

The `products` wrapper is removed from the final JSON — Domino only understands parallel arrays.

## In Vaadin: MultivalueGrid with inline editing

To display and edit multivalue fields in Vaadin, we created the `MultivalueGrid` component:

```java
public class MultivalueGrid<T extends AbstractModelDocMultivalue>
        extends Composite<VerticalLayout> {

    private final AbstractModelListaMultivalue<T> source;
    private final List<T> data;  // live reference to source.lista
    private final Grid<T> grid;
    private final Binder<T> binder;
    private final Editor<T> editor;

    public MultivalueGrid(Class<T> beanType,
            AbstractModelListaMultivalue<T> source) {
        this.source = Objects.requireNonNull(source);

        // Live reference — grid changes reflect in the model
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

        // Editor listeners
        editor.addSaveListener(e -> syncToMultivalueFields());
    }

    // Add and edit
    public void addItem(T item) {
        data.add(item);
        refresh();
        editor.editItem(item);
    }

    // Remove
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

### Usage in the view: defining columns with fluent API

```java
@Override
protected void configureBinder(Binder<Order> binder) {
    // Simple form fields...

    // Multivalue grid
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

The `MultivalueGrid` provides:
- **Typed columns** (text, number, date, combobox)
- **Inline editing** with buffered editor
- **+ button** in the header for adding items
- **Action buttons** per row (edit, save, cancel, delete)
- **Read-only mode** that hides all editing controls

## The complete cycle

```
Domino Form
  productName:  ["Notebook", "Mouse"]
  productQty:   [2, 10]
  productPrice: [4500.00, 89.90]
        │
        ▼  DRAPI GET /document/{unid}
JSON with parallel arrays
        │
        ▼  populaMultivalues() — reflection
List<Product>
  [0] { name:"Notebook", qty:2,  price:4500.00 }
  [1] { name:"Mouse",    qty:10, price:89.90   }
        │
        ▼  MultivalueGrid — live reference
Editable Vaadin grid
  ┌──────────┬─────┬──────────┬─────────┐
  │ Notebook │  2  │ 4,500.00 │ ✏ 🗑    │
  │ Mouse    │ 10  │    89.90 │ ✏ 🗑    │
  └──────────┴─────┴──────────┴─────────┘
  User edits, adds "Monitor"...
        │
        ▼  flattenForDomino() — reverse reflection
JSON with parallel arrays (updated)
  productName:  ["Notebook", "Mouse", "Monitor"]
  productQty:   [2, 10, 3]
  productPrice: [4500.00, 89.90, 1200.00]
        │
        ▼  DRAPI PUT /document/{unid}
Domino Form (updated, compatible with Notes client)
```

## Inner class vs standalone class: when to use each

| Criteria | Inner class | Standalone class |
|---|---|---|
| Used in a single model | Ideal | Unnecessary |
| Used in multiple models | Duplication | Ideal — one definition, N uses |
| Scope | Bound to the parent model | Global / shared |
| Naming convention | Automatic (prefix = parent class) | Must follow the same convention |
| Example | `Order.Product` | `Contact` (used in Order, Invoice, Quote) |

## Lessons learned

| Challenge | Solution |
|---|---|
| Parallel arrays in Domino vs objects in Java | Transpose via reflection (columns ↔ rows) |
| Maintaining bidirectional synchronization | `populaMultivalues()` on load, `flattenForDomino()` on save |
| Tracking add/remove in grid | `idMulti` (short UUID) for each item |
| Null fields in arrays of different sizes | Fill with `""` for Strings, skip for other types |
| Grid needs a live reference | `MultivalueGrid` uses `source.getLista()` directly |
| Compatibility with Notes client | `flattenForDomino()` removes wrapper and generates only arrays |
| Reusing structure across models | Standalone classes (not inner) |
| Identifying multivalue fields automatically | Reflection + naming convention ({prefix}{Field}) |

---

With this pattern, Domino multivalue fields — a concept with no direct equivalent in relational databases — become first-class Java object lists. You manipulate them, edit them in grids, validate field by field, and save back while maintaining full compatibility with the Notes Form. Domino continues working normally — it does not even know the data was edited by a Vaadin application.
