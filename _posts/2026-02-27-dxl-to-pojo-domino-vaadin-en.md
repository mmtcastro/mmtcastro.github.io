---
layout: post
title: "From DXL to POJO: transforming Domino Forms into Java objects for Vaadin"
date: 2026-02-27
categories: [domino, drapi, java, vaadin, dxl]
lang: en
---

When we started migrating our Notes applications to Spring Boot + Vaadin, the first question was: how do we transform the fields of a Domino Form into a Java class that works both for receiving data from DRAPI and for feeding Vaadin forms?

The answer involved understanding **DXL** (Domino XML Language), the **DRAPI schema**, and building an automatic POJO generator. This post explains the theory and practice of this process.

## The problem: two worlds, same data

In Domino, a Form defines fields with specific types (Text, Number, DateTime, RichText, Names, Readers, Authors). In the Java/Vaadin world, we need classes with typed fields, Jackson annotations for deserialization, and compatibility with Vaadin form components.

```
Domino Form (DXL)              Java POJO                    Vaadin Form
┌─────────────────┐     ┌──────────────────────┐     ┌─────────────────────┐
│ CustomerName     │     │ String customerName   │     │ TextField           │
│ (Text)          │ ──> │ @JsonProperty(...)    │ ──> │ binder.forField()   │
├─────────────────┤     ├──────────────────────┤     ├─────────────────────┤
│ InvoiceDate     │     │ LocalDate invoiceDate │     │ DatePicker          │
│ (DateTime)      │ ──> │ @JsonDeserialize(...) │ ──> │ binder.forField()   │
├─────────────────┤     ├──────────────────────┤     ├─────────────────────┤
│ Total           │     │ Double total          │     │ NumberField         │
│ (Number)        │ ──> │ @JsonProperty(...)    │ ──> │ binder.forField()   │
├─────────────────┤     ├──────────────────────┤     ├─────────────────────┤
│ Notes           │     │ RichText notes        │     │ RichTextEditor      │
│ (RichText)      │ ──> │ @JsonDeserialize(...) │ ──> │ binder.forField()   │
└─────────────────┘     └──────────────────────┘     └─────────────────────┘
```

## What is DXL

DXL (Domino XML Language) is the XML representation of Domino design elements. When you export a Form via ODP (On-Disk Project), it becomes a `.form` file in XML:

```xml
<?xml version='1.0' encoding='utf-8'?>
<note class='form' xmlns='http://www.lotus.com/dxl'>
  <noteinfo noteid='2ee' unid='80CB47ADF51F724B...'>
    <created><datetime>20190522T185229</datetime></created>
  </noteinfo>

  <!-- Text field -->
  <item name='CustomerName'><text/></item>

  <!-- Number field -->
  <item name='Total' summary='false'><number>0</number></item>

  <!-- Date field -->
  <item name='InvoiceDate'><datetime>20260101T000000</datetime></item>

  <!-- Form metadata -->
  <item name='$TITLE'><text>Invoice</text></item>
</note>
```

Each `<item>` represents a field with its implicit type: `<text>`, `<number>`, `<datetime>`, etc.

## The DRAPI schema: the bridge

Instead of parsing DXL directly, we use the DRAPI schema endpoint which returns the form fields in a structured JSON format:

```
GET /api/v1/scope/form/{formName}?dataSource={scope}
Authorization: Bearer {token}
```

Response:

```json
{
  "formName": "Invoice",
  "formModes": [{
    "modeName": "default",
    "fields": [
      {
        "name": "CustomerName",
        "externalName": "CustomerName",
        "type": "string",
        "format": null,
        "fieldAccess": "RW",
        "summaryField": true
      },
      {
        "name": "InvoiceDate",
        "externalName": "InvoiceDate",
        "type": "string",
        "format": "date-time",
        "fieldAccess": "RW"
      },
      {
        "name": "Total",
        "externalName": "Total",
        "type": "number",
        "format": "float",
        "fieldAccess": "RW"
      },
      {
        "name": "Notes",
        "externalName": "Notes",
        "type": "string",
        "format": "richtext",
        "fieldAccess": "RW"
      },
      {
        "name": "Tags",
        "externalName": "Tags",
        "type": "array",
        "items": { "type": "string" },
        "fieldAccess": "RW"
      },
      {
        "name": "Payments",
        "externalName": "Payments",
        "type": "array",
        "items": { "type": "number" },
        "fieldAccess": "RW"
      }
    ],
    "required": ["CustomerName", "InvoiceDate"]
  }]
}
```

This JSON gives us everything we need: field name, type, format, whether it's an array, and whether it's read-only.

## Type mapping: DXL → DRAPI Schema → Java

The mapping table we built:

| DRAPI Type | Format | Java Type | Deserializer |
|---|---|---|---|
| `string` | *(none)* | `String` | native |
| `string` | `date-time` | `ZonedDateTime` | `ZonedDateTimeDeserializer` |
| `string` | `date` | `LocalDate` | native |
| `string` | `richtext` | `RichText` (inner class) | `BodyDeserializer` |
| `number` | *(any)* | `Double` | native |
| `integer` | *(any)* | `Integer` | native |
| `boolean` | *(any)* | `Boolean` | native |
| `array` | items: `string` | `List<String>` | native |
| `array` | items: `number` | `List<Double>` | native |
| `array` | items: `date-time` | `List<ZonedDateTime>` | `ZonedDateTimeDeserializer` (content) |
| `array` | items: `richtext` | `List<RichText>` | `BodyDeserializer` (content) |

## The base class: AbstractModelDoc

All generated POJOs extend a base class that contains the fields common to every Domino document:

```java
@JsonIgnoreProperties(ignoreUnknown = true)
public abstract class AbstractModelDoc {

    // Domino metadata
    @JsonProperty("@meta")   private Meta meta;     // UNID, NoteID, etc.
    @JsonProperty("@unid")   private String unid;
    @JsonProperty("Form")    private String form;

    // Fields common to all forms
    @JsonProperty("Id")          private String id;          // Generated ID
    @JsonProperty("Codigo")      private String codigo;      // Business code
    @JsonProperty("Status")      private String status;
    @JsonProperty("Nome")        private String nome;
    @JsonProperty("Descricao")   private String descricao;
    @JsonProperty("Autor")       private String autor;
    @JsonProperty("Criacao")     private ZonedDateTime criacao;
    @JsonProperty("Valor")       private Double valor;

    // Access control
    @JsonProperty("Autores")     private TreeSet<String> autores;  // Authors
    @JsonProperty("Leitores")    private TreeSet<String> leitores; // Readers
    @JsonProperty("Revision")    private String revision;

    // Attachments
    @JsonProperty("$FILES")      private List<String> fileNames;

    // Inner type for RichText
    public static class RichText {
        private String type;       // "text/html"
        private String content;    // HTML or Base64
        private String encoding;   // "BASE64" or null
    }

    // Initialization methods
    public void init() {
        this.form = this.getClass().getSimpleName();
        this.id = generateNewModelId();
        this.criacao = ZonedDateTime.now();
    }
}
```

The POJO generator uses reflection to check which fields already exist in this base class and **does not duplicate them** in the child class. This prevents deserialization conflicts.

## The automatic POJO generator

The generator receives the JSON schema from DRAPI and produces Java source code:

```java
public class DominoPojoGenerator {

    public String generatePojoSource(String packageName, String className,
            FormSchemaResponse schema, int modeIndex,
            Class<?> baseModelClass) {

        List<FieldDef> fields = schema.formModes.get(modeIndex).fields;

        // 1. Discover fields already in AbstractModelDoc (via reflection)
        Set<String> baseNames = extractJsonNamesFromClassHierarchy(baseModelClass);

        // 2. Filter: generate only fields NOT in the base class
        List<FieldDef> fieldsToGenerate = fields.stream()
            .filter(f -> !baseNames.contains(pickPropName(f)))
            .toList();

        // 3. For each field, generate the mapping
        StringBuilder sb = new StringBuilder();
        sb.append("package ").append(packageName).append(";\n\n");
        // ... imports ...

        sb.append("@Getter @Setter\n");
        sb.append("@JsonIgnoreProperties(ignoreUnknown = true)\n");
        sb.append("public class ").append(className)
          .append(" extends AbstractModelDoc {\n\n");

        for (FieldDef f : fieldsToGenerate) {
            String prop = pickPropName(f);
            String javaType = resolveJavaType(f);
            String javaField = toSafeJavaField(prop);

            sb.append("    @JsonProperty(\"").append(prop).append("\")\n");
            sb.append("    @JsonAlias({\"").append(prop)
              .append("\", \"").append(lowerFirst(prop)).append("\"})\n");

            // Add deserializer if needed
            if ("ZonedDateTime".equals(javaType)) {
                sb.append("    @JsonDeserialize(using = ZonedDateTimeDeserializer.class)\n");
            }
            if ("AbstractModelDoc.RichText".equals(javaType)) {
                sb.append("    @JsonDeserialize(using = BodyDeserializer.class)\n");
            }

            sb.append("    private ").append(javaType).append(" ")
              .append(javaField).append(";\n\n");
        }

        sb.append("}\n");
        return sb.toString();
    }

    private String resolveJavaType(FieldDef f) {
        String type = f.type;    // "string", "number", "array", ...
        String format = f.format; // "date-time", "richtext", ...

        if ("richtext".equals(format))  return "AbstractModelDoc.RichText";
        if ("date-time".equals(format)) return "ZonedDateTime";
        if ("date".equals(format))      return "LocalDate";

        if ("array".equals(type)) {
            String innerType = f.items != null ? f.items.type : "string";
            String innerFormat = f.items != null ? f.items.format : "";

            if ("richtext".equals(innerFormat))  return "List<AbstractModelDoc.RichText>";
            if ("date-time".equals(innerFormat)) return "List<ZonedDateTime>";
            if ("number".equals(innerType))      return "List<Double>";
            return "List<String>";
        }

        if ("number".equals(type))  return "Double";
        if ("integer".equals(type)) return "Integer";
        if ("boolean".equals(type)) return "Boolean";
        return "String";
    }
}
```

## Example: generated POJO for the "Invoice" form

From the schema shown above, the generator produces:

```java
/* AUTO-GENERATED from DRAPI form schema */
package com.example.invoices.model;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import com.fasterxml.jackson.annotation.JsonProperty;
import com.fasterxml.jackson.annotation.JsonAlias;
import com.fasterxml.jackson.databind.annotation.JsonDeserialize;
import java.time.ZonedDateTime;
import java.util.List;
import lombok.Getter;
import lombok.Setter;

@Getter
@Setter
@JsonIgnoreProperties(ignoreUnknown = true)
public class Invoice extends AbstractModelDoc {

    public Invoice() {
        super();
    }

    /** string | RW */
    @JsonProperty("CustomerName")
    @JsonAlias({"CustomerName", "customerName"})
    private String customerName;

    /** string | date-time | RW */
    @JsonProperty("InvoiceDate")
    @JsonAlias({"InvoiceDate", "invoiceDate"})
    @JsonDeserialize(using = ZonedDateTimeDeserializer.class)
    private ZonedDateTime invoiceDate;

    /** number | float | RW */
    @JsonProperty("Total")
    @JsonAlias({"Total", "total"})
    private Double total;

    /** string | richtext | RW */
    @JsonProperty("Notes")
    @JsonAlias({"Notes", "notes"})
    @JsonDeserialize(using = BodyDeserializer.class)
    private AbstractModelDoc.RichText notes;

    /** array | RW */
    @JsonProperty("Tags")
    @JsonAlias({"Tags", "tags"})
    private List<String> tags;

    /** array | RW */
    @JsonProperty("Payments")
    @JsonAlias({"Payments", "payments"})
    private List<Double> payments;
}
```

Note that fields like `Id`, `Codigo`, `Status`, `Criacao`, `Form` do not appear — they are already in `AbstractModelDoc`.

## Why @JsonAlias with case variants?

DRAPI may return field names with different capitalization depending on context (view vs document, default mode vs custom mode). `@JsonAlias` ensures the mapping works in all cases:

```java
@JsonProperty("CustomerName")                    // Primary name
@JsonAlias({"CustomerName", "customerName"})     // Accepted variants
```

This prevents silent deserialization errors — the field simply remains `null` without any error.

## Custom deserializers: the details that matter

### Dates (ZonedDateTime)

Domino returns dates in variable formats. Our deserializer handles them all:

```java
public class ZonedDateTimeDeserializer extends JsonDeserializer<ZonedDateTime> {

    private static final ZoneId DEFAULT_ZONE = ZoneId.of("America/Sao_Paulo");

    @Override
    public ZonedDateTime deserialize(JsonParser p, DeserializationContext ctx) {
        String text = p.getText();
        try {
            // Primary format: ISO with offset (2026-01-10T12:00:00-03:00)
            return ZonedDateTime.parse(text, DateTimeFormatter.ISO_OFFSET_DATE_TIME);
        } catch (Exception e) {
            // Fallback: date only (2026-01-10) → convert to ZonedDateTime
            LocalDate date = LocalDate.parse(text, DateTimeFormatter.ISO_LOCAL_DATE);
            return date.atStartOfDay(DEFAULT_ZONE);
        }
    }
}
```

### RichText (Domino HTML)

RichText fields come as objects with type, encoding, and content:

```json
{
  "type": "text/html",
  "encoding": "BASE64",
  "content": "PGh0bWw+PGJvZHk+..."
}
```

The deserializer extracts and decodes the HTML for use in Vaadin components.

## From POJO to Vaadin form

With the POJO ready, mapping to a Vaadin form uses the `Binder`:

```java
public class InvoiceFormView extends VerticalLayout {

    private final Binder<Invoice> binder = new Binder<>(Invoice.class);

    private final TextField customerName = new TextField("Customer");
    private final DatePicker invoiceDate = new DatePicker("Date");
    private final NumberField total = new NumberField("Total");

    public InvoiceFormView() {
        binder.forField(customerName)
            .asRequired("Customer is required")
            .bind(Invoice::getCustomerName, Invoice::setCustomerName);

        binder.forField(invoiceDate)
            .bind(invoice -> invoice.getInvoiceDate() != null
                ? invoice.getInvoiceDate().toLocalDate() : null,
                (invoice, date) -> invoice.setInvoiceDate(
                    date != null ? date.atStartOfDay(ZoneId.systemDefault()) : null));

        binder.forField(total)
            .bind(Invoice::getTotal, Invoice::setTotal);

        add(customerName, invoiceDate, total);
    }

    public void setInvoice(Invoice invoice) {
        binder.setBean(invoice);
    }
}
```

The cycle closes: **DXL → DRAPI Schema → Java POJO → Vaadin Binder → Form on screen**.

## The complete flow

```
Domino Designer
    │
    ▼
Form definition (DXL)
    │
    ▼
DRAPI exposes schema: GET /scope/form/{formName}
    │
    ▼
DominoPojoGenerator.generatePojoSource()
    │
    ├── Parses the JSON schema
    ├── Filters fields already in AbstractModelDoc
    ├── Maps types: string/date-time → ZonedDateTime, etc.
    ├── Generates @JsonProperty + @JsonAlias
    ├── Adds @JsonDeserialize when needed
    │
    ▼
Generated POJO: Invoice.java extends AbstractModelDoc
    │
    ├── Used by Service: objectMapper.readValue(json, Invoice.class)
    ├── Used by Vaadin: binder.setBean(invoice)
    │
    ▼
Working Vaadin form connected to DRAPI
```

## Lessons learned

| Challenge | Solution |
|---|---|
| Duplicate fields between base and child | Reflection to discover AbstractModelDoc fields |
| Field names with variable case | @JsonAlias with variants (Original, lowerFirst) |
| Dates in inconsistent formats | Deserializer with fallback (ISO offset → ISO date) |
| RichText as complex object | Inner class RichText + BodyDeserializer |
| Fields with $ and @ in name | Sanitization to valid Java names |
| Arrays (multivalue fields) | Mapping to List\<T\> with correct item type |
| Generator produces incorrect code | @JsonIgnoreProperties(ignoreUnknown = true) as safety net |

---

With this pipeline, adding a new entity to the system takes minutes instead of hours: just point the generator at the form in DRAPI, review the generated POJO, and connect it to Vaadin. In the next post, we will explore the Domino ACL model and how to replicate it in Spring Security.
