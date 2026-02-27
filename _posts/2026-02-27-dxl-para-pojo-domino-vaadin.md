---
layout: post
title: "De DXL a POJO: transformando Forms do Domino em objetos Java para o Vaadin"
date: 2026-02-27
categories: [domino, drapi, java, vaadin, dxl]
---

Quando comecamos a migrar nossas aplicacoes Notes para Spring Boot + Vaadin, a primeira pergunta foi: como transformar os campos de um Form do Domino em uma classe Java que funcione tanto para receber dados do DRAPI quanto para alimentar formularios do Vaadin?

A resposta passou por entender o **DXL** (Domino XML Language), o **schema do DRAPI**, e construir um gerador automatico de POJOs. Este post explica a teoria e a pratica desse processo.

## O problema: dois mundos, mesmos dados

No Domino, um Form define campos com tipos especificos (Text, Number, DateTime, RichText, Names, Readers, Authors). No mundo Java/Vaadin, precisamos de classes com campos tipados, anotacoes Jackson para desserializacao e compatibilidade com os componentes de formulario do Vaadin.

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

## O que e DXL

DXL (Domino XML Language) e a representacao XML dos elementos de design do Domino. Quando voce exporta um Form via ODP (On-Disk Project), ele vira um arquivo `.form` em XML:

```xml
<?xml version='1.0' encoding='utf-8'?>
<note class='form' xmlns='http://www.lotus.com/dxl'>
  <noteinfo noteid='2ee' unid='80CB47ADF51F724B...'>
    <created><datetime>20190522T185229</datetime></created>
  </noteinfo>

  <!-- Campo texto -->
  <item name='CustomerName'><text/></item>

  <!-- Campo numerico -->
  <item name='Total' summary='false'><number>0</number></item>

  <!-- Campo data -->
  <item name='InvoiceDate'><datetime>20260101T000000</datetime></item>

  <!-- Metadados do form -->
  <item name='$TITLE'><text>Invoice</text></item>
</note>
```

Cada `<item>` representa um campo com seu tipo implicito: `<text>`, `<number>`, `<datetime>`, etc.

## O schema do DRAPI: a ponte

Em vez de parsear DXL diretamente, usamos o endpoint de schema do DRAPI que retorna os campos do form em formato JSON estruturado:

```
GET /api/v1/scope/form/{formName}?dataSource={scope}
Authorization: Bearer {token}
```

Resposta:

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

Esse JSON nos da tudo o que precisamos: nome do campo, tipo, formato, se e array, se e somente leitura.

## Mapeamento de tipos: DXL → DRAPI Schema → Java

A tabela de mapeamento que construimos:

| Tipo DRAPI | Formato | Tipo Java | Deserializer |
|---|---|---|---|
| `string` | *(nenhum)* | `String` | nativo |
| `string` | `date-time` | `ZonedDateTime` | `ZonedDateTimeDeserializer` |
| `string` | `date` | `LocalDate` | nativo |
| `string` | `richtext` | `RichText` (inner class) | `BodyDeserializer` |
| `number` | *(qualquer)* | `Double` | nativo |
| `integer` | *(qualquer)* | `Integer` | nativo |
| `boolean` | *(qualquer)* | `Boolean` | nativo |
| `array` | items: `string` | `List<String>` | nativo |
| `array` | items: `number` | `List<Double>` | nativo |
| `array` | items: `date-time` | `List<ZonedDateTime>` | `ZonedDateTimeDeserializer` (content) |
| `array` | items: `richtext` | `List<RichText>` | `BodyDeserializer` (content) |

## A classe base: AbstractModelDoc

Todos os POJOs gerados estendem uma classe base que contem os campos comuns a todos os documentos Domino:

```java
@JsonIgnoreProperties(ignoreUnknown = true)
public abstract class AbstractModelDoc {

    // Metadados Domino
    @JsonProperty("@meta")   private Meta meta;     // UNID, NoteID, etc.
    @JsonProperty("@unid")   private String unid;
    @JsonProperty("Form")    private String form;

    // Campos comuns a todos os forms
    @JsonProperty("Id")          private String id;          // ID gerado
    @JsonProperty("Codigo")      private String codigo;      // Codigo de negocio
    @JsonProperty("Status")      private String status;
    @JsonProperty("Nome")        private String nome;
    @JsonProperty("Descricao")   private String descricao;
    @JsonProperty("Autor")       private String autor;
    @JsonProperty("Criacao")     private ZonedDateTime criacao;
    @JsonProperty("Valor")       private Double valor;

    // Controle de acesso
    @JsonProperty("Autores")     private TreeSet<String> autores;  // Authors
    @JsonProperty("Leitores")    private TreeSet<String> leitores; // Readers
    @JsonProperty("Revision")    private String revision;

    // Anexos
    @JsonProperty("$FILES")      private List<String> fileNames;

    // Tipo interno para RichText
    public static class RichText {
        private String type;       // "text/html"
        private String content;    // HTML ou Base64
        private String encoding;   // "BASE64" ou null
    }

    // Metodos de inicializacao
    public void init() {
        this.form = this.getClass().getSimpleName();
        this.id = generateNewModelId();
        this.criacao = ZonedDateTime.now();
    }
}
```

O gerador de POJOs consulta via reflexao quais campos ja existem nessa classe base e **nao os duplica** na classe filha. Isso evita conflitos de desserializacao.

## O gerador automatico de POJOs

O gerador recebe o schema JSON do DRAPI e produz o codigo-fonte Java:

```java
public class DominoPojoGenerator {

    public String generatePojoSource(String packageName, String className,
            FormSchemaResponse schema, int modeIndex,
            Class<?> baseModelClass) {

        List<FieldDef> fields = schema.formModes.get(modeIndex).fields;

        // 1. Descobrir campos ja existentes no AbstractModelDoc (via reflexao)
        Set<String> baseNames = extractJsonNamesFromClassHierarchy(baseModelClass);

        // 2. Filtrar: gerar apenas campos que NAO estao na classe base
        List<FieldDef> fieldsToGenerate = fields.stream()
            .filter(f -> !baseNames.contains(pickPropName(f)))
            .toList();

        // 3. Para cada campo, gerar o mapeamento
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

            // Adicionar deserializer se necessario
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

## Exemplo: POJO gerado para o form "Invoice"

A partir do schema mostrado acima, o gerador produz:

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

Note que campos como `Id`, `Codigo`, `Status`, `Criacao`, `Form` nao aparecem — eles ja estao no `AbstractModelDoc`.

## Por que @JsonAlias com variantes de case?

O DRAPI pode retornar nomes de campos com capitalizacao diferente dependendo do contexto (view vs documento, modo default vs modo custom). O `@JsonAlias` garante que o mapeamento funcione em todos os casos:

```java
@JsonProperty("CustomerName")                    // Nome primario
@JsonAlias({"CustomerName", "customerName"})     // Variantes aceitas
```

Isso evita erros silenciosos de desserializacao — o campo simplesmente fica `null` sem nenhum erro.

## Deserializadores customizados: os detalhes que importam

### Datas (ZonedDateTime)

O Domino retorna datas em formatos variaveis. Nosso deserializador trata todos:

```java
public class ZonedDateTimeDeserializer extends JsonDeserializer<ZonedDateTime> {

    private static final ZoneId DEFAULT_ZONE = ZoneId.of("America/Sao_Paulo");

    @Override
    public ZonedDateTime deserialize(JsonParser p, DeserializationContext ctx) {
        String text = p.getText();
        try {
            // Formato principal: ISO com offset (2026-01-10T12:00:00-03:00)
            return ZonedDateTime.parse(text, DateTimeFormatter.ISO_OFFSET_DATE_TIME);
        } catch (Exception e) {
            // Fallback: apenas data (2026-01-10) → converte para ZonedDateTime
            LocalDate date = LocalDate.parse(text, DateTimeFormatter.ISO_LOCAL_DATE);
            return date.atStartOfDay(DEFAULT_ZONE);
        }
    }
}
```

### RichText (HTML do Domino)

Campos RichText vem como objetos com tipo, encoding e conteudo:

```json
{
  "type": "text/html",
  "encoding": "BASE64",
  "content": "PGh0bWw+PGJvZHk+..."
}
```

O deserializador extrai e decodifica o HTML para uso em componentes Vaadin.

## Do POJO ao formulario Vaadin

Com o POJO pronto, o mapeamento para um formulario Vaadin usa o `Binder`:

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

O ciclo se fecha: **DXL → Schema DRAPI → POJO Java → Vaadin Binder → Formulario na tela**.

## O fluxo completo

```
Domino Designer
    │
    ▼
Form definition (DXL)
    │
    ▼
DRAPI expoe schema: GET /scope/form/{formName}
    │
    ▼
DominoPojoGenerator.generatePojoSource()
    │
    ├── Parseia o JSON do schema
    ├── Filtra campos ja existentes no AbstractModelDoc
    ├── Mapeia tipos: string/date-time → ZonedDateTime, etc.
    ├── Gera @JsonProperty + @JsonAlias
    ├── Adiciona @JsonDeserialize quando necessario
    │
    ▼
POJO gerado: Invoice.java extends AbstractModelDoc
    │
    ├── Usado pelo Service: objectMapper.readValue(json, Invoice.class)
    ├── Usado pelo Vaadin: binder.setBean(invoice)
    │
    ▼
Formulario Vaadin funcional conectado ao DRAPI
```

## Licoes aprendidas

| Desafio | Solucao |
|---|---|
| Campos duplicados entre base e filha | Reflexao para descobrir campos do AbstractModelDoc |
| Nomes de campos com case variavel | @JsonAlias com variantes (Original, lowerFirst) |
| Datas em formatos inconsistentes | Deserializador com fallback (ISO offset → ISO date) |
| RichText como objeto complexo | Inner class RichText + BodyDeserializer |
| Campos com $ e @ no nome | Sanitizacao para nomes Java validos |
| Arrays (multivalue fields) | Mapeamento para List\<T\> com tipo correto dos items |
| Gerador gera codigo incorreto | @JsonIgnoreProperties(ignoreUnknown = true) como rede de seguranca |

---

Com esse pipeline, adicionar uma nova entidade ao sistema leva minutos em vez de horas: basta apontar o gerador para o form no DRAPI, revisar o POJO gerado, e conectar ao Vaadin. No proximo post, vamos explorar o modelo de ACL do Domino e como reproduzi-lo no Spring Security.
