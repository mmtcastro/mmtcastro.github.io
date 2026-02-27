---
layout: post
title: "RichText do Domino no Vaadin: de BASE64 no DRAPI ao editor visual"
date: 2026-02-27
categories: [domino, drapi, java, vaadin, richtext]
---

Campos RichText no Domino sao especiais — armazenam HTML formatado, imagens embutidas, tabelas e ate anexos. Quando voce busca um documento via DRAPI, o campo RichText chega como um objeto JSON com tipo, encoding e conteudo. Este post mostra como transportamos esse conteudo ate um editor visual no Vaadin, preservando formatacao e imagens.

## Como o DRAPI retorna RichText

Quando voce faz `GET /api/v1/document/{unid}`, campos RichText vem como objetos:

```json
{
  "@unid": "ABC123...",
  "Codigo": "DOC-001",
  "Title": "My Document",
  "Body": {
    "type": "text/html",
    "encoding": "BASE64",
    "headers": "Content-Type: text/html; charset=UTF-8",
    "content": "PGh0bWw+PGJvZHk+PHA+SGVsbG8gV29ybGQ8L3A+PC9ib2R5PjwvaHRtbD4="
  }
}
```

O campo `content` e HTML codificado em BASE64. Decodificando:

```
PGh0bWw+PGJvZHk+PHA+SGVsbG8gV29ybGQ8L3A+PC9ib2R5PjwvaHRtbD4=
                         ↓ Base64 decode
<html><body><p>Hello World</p></body></html>
```

Mas nem sempre o encoding e BASE64 — em alguns casos o DRAPI retorna o HTML como texto puro (`PLAIN`), ou ate como uma string simples sem o wrapper de objeto. Precisamos tratar todos esses casos.

## O POJO: a inner class RichText

No `AbstractModelDoc`, definimos uma inner class para representar o formato do DRAPI:

```java
public abstract class AbstractModelDoc {

    public static class RichText {
        private String type;       // "text/html"
        private String encoding;   // "BASE64" ou "PLAIN"
        private String headers;    // "Content-Type: text/html; charset=UTF-8"
        private String content;    // HTML em BASE64 ou texto puro

        public RichText() {
            this.type = "text/html";
            this.encoding = "BASE64";
            this.headers = "Content-Type: text/html; charset=UTF-8";
        }

        // Helper: define conteudo como texto, codificando em BASE64
        public void setPlainTextContent(String plainText) {
            this.content = Base64.getEncoder()
                .encodeToString(plainText.getBytes(StandardCharsets.UTF_8));
        }

        // Helper: define conteudo como HTML puro (sem encoding)
        public void setHtmlContent(String html) {
            this.content = html;
            this.encoding = "PLAIN";
        }

        // Garante que o formato esta em text/html + BASE64
        public void ensureHtmlBase64Encoding() {
            if (!"text/html".equalsIgnoreCase(this.type))
                this.type = "text/html";
            if (!"BASE64".equalsIgnoreCase(this.encoding))
                this.encoding = "BASE64";
        }
    }
}
```

No modelo concreto, o campo e anotado com o deserializador customizado:

```java
@Getter @Setter
public class Company extends AbstractModelDoc {

    @JsonProperty("Observacoes")
    @JsonAlias({"observacoes", "Observacoes"})
    @JsonDeserialize(using = BodyDeserializer.class)
    private RichText observacoes;

    public RichText getObservacoes() {
        return this.observacoes != null ? this.observacoes : new RichText();
    }
}
```

## O BodyDeserializer: tratando todos os formatos

O DRAPI pode retornar RichText de tres formas diferentes. O deserializador trata todas:

```java
public class BodyDeserializer extends JsonDeserializer<RichText> {

    @Override
    public RichText deserialize(JsonParser parser, DeserializationContext ctx)
            throws IOException {

        JsonNode node = parser.getCodec().readTree(parser);

        // Caso 1: null ou string vazia → RichText vazio
        if (node == null || node.isNull()
                || (node.isTextual() && node.asText().trim().isEmpty())) {
            return new RichText();
        }

        // Caso 2: objeto JSON completo (formato padrao do DRAPI)
        if (node.isObject()) {
            RichText rt = new RichText();

            rt.setType(node.has("type") && !node.get("type").isNull()
                ? node.get("type").asText() : "text/html");

            rt.setEncoding(node.has("encoding") && !node.get("encoding").isNull()
                ? node.get("encoding").asText() : "BASE64");

            rt.setHeaders(node.has("headers") && !node.get("headers").isNull()
                ? node.get("headers").asText()
                : "Content-Type: text/html; charset=UTF-8");

            rt.setContent(node.has("content") && !node.get("content").isNull()
                ? node.get("content").asText() : "");

            return rt;
        }

        // Caso 3: string pura (HTML sem wrapper)
        if (node.isTextual()) {
            RichText rt = new RichText();
            rt.setContent(node.asText());
            rt.setEncoding("PLAIN");
            return rt;
        }

        return new RichText();
    }
}
```

## O componente Vaadin: RichTextView

O componente que criamos implementa `HasValue<RichText>` para ser compativel com o Binder do Vaadin, e opera em dois modos:

### Modo leitura: HTML renderizado em um Div

```java
public class RichTextView extends VerticalLayout
        implements HasValue<ValueChangeEvent<RichText>, RichText> {

    private final RichTextEditor editor;  // Modo edicao
    private final Div readOnlyView;       // Modo leitura
    private boolean readOnly = false;
    private RichText value;

    public RichTextView() {
        // Editor para modo edicao
        editor = new RichTextEditor();
        editor.setWidthFull();
        editor.setHeight("800px");

        // Div para modo leitura (cresce livremente)
        readOnlyView = new Div();
        readOnlyView.setWidthFull();
        readOnlyView.getStyle()
            .set("padding", "var(--lumo-space-m)")
            .set("border", "1px solid var(--lumo-contrast-20pct)")
            .set("border-radius", "var(--lumo-border-radius-m)")
            .set("min-height", "200px")
            .set("overflow", "visible");
    }
```

### Atualizacao do display conforme o modo

```java
    private void updateDisplay() {
        removeAll();

        if (readOnly) {
            // Decodifica o conteudo e injeta no Div
            String html = getHtmlContent(value);
            // CSS para imagens responsivas
            String styled = "<style>"
                + "img { max-width: 100% !important; height: auto !important; }"
                + "</style>" + html;
            readOnlyView.getElement().setProperty("innerHTML", styled);
            add(readOnlyView);
        } else {
            // Carrega o HTML no RichTextEditor
            String html = getHtmlContent(value);
            editor.setValue(html);
            add(editor);
        }
    }
```

### Decodificacao: de RichText para HTML

```java
    private String getHtmlContent(RichText richText) {
        if (richText == null) return "";

        String raw = "";
        if ("BASE64".equalsIgnoreCase(richText.getEncoding())
                && richText.getContent() != null) {
            try {
                byte[] decoded = Base64.getDecoder()
                    .decode(richText.getContent());
                raw = new String(decoded, StandardCharsets.UTF_8);
            } catch (Exception e) {
                raw = richText.getContent(); // fallback
            }
        } else if ("PLAIN".equalsIgnoreCase(richText.getEncoding())) {
            raw = richText.getContent() != null ? richText.getContent() : "";
        }

        return stripMimeHeaders(raw);
    }
```

### Removendo headers MIME

O conteudo decodificado pode vir com headers MIME no inicio (ex: `Content-Type: text/html; charset=UTF-8`). O metodo `stripMimeHeaders()` encontra a primeira tag HTML real e descarta tudo antes:

```java
    private String stripMimeHeaders(String raw) {
        if (raw == null || raw.isEmpty()) return raw;

        String lower = raw.toLowerCase();
        String[] htmlTags = {"<!doctype", "<html", "<body", "<div", "<p"};

        int firstTag = -1;
        for (String tag : htmlTags) {
            int idx = lower.indexOf(tag);
            if (idx >= 0 && (firstTag < 0 || idx < firstTag)) {
                firstTag = idx;
            }
        }

        if (firstTag >= 0) return raw.substring(firstTag);

        // Fallback: procura ultimo Content-Type:
        int lastCt = lower.lastIndexOf("content-type:");
        if (lastCt >= 0) {
            return raw.substring(lastCt + "Content-Type:".length()).trim();
        }

        return raw;
    }
```

### Codificacao: de HTML para RichText (ao salvar)

Quando o usuario edita e salva, o HTML do editor e convertido de volta para o formato DRAPI:

```java
    public RichText getValue() {
        if (readOnly) {
            // Em leitura, retorna o valor original
            if (value == null || value.getContent() == null
                    || value.getContent().trim().isEmpty()) {
                return null; // evita enviar content vazio para o Domino
            }
            return value;
        }

        // Em edicao, pega o HTML do editor
        String html = editor.getValue();
        if (html == null || html.trim().isEmpty()) {
            return null; // campo vazio = nao enviar
        }

        RichText rt = new RichText();
        String base64 = Base64.getEncoder()
            .encodeToString(html.getBytes(StandardCharsets.UTF_8));
        rt.setContent(base64);
        rt.setEncoding("BASE64");
        rt.setType("text/html");
        rt.setHeaders("Content-Type: text/html; charset=UTF-8");
        return rt;
    }
```

**Detalhe critico**: se o conteudo estiver vazio, retornamos `null` em vez de um RichText com content vazio. Isso evita um `NullPointerException` no DRAPI ao tentar decodificar BASE64 de um content null.

## Protecao no save: removeEmptyRichTextFields()

Antes de enviar o JSON para o DRAPI, removemos campos RichText vazios do payload:

```java
private void removeEmptyRichTextFields(ObjectNode root) {
    List<String> toRemove = new ArrayList<>();

    Iterator<String> names = root.fieldNames();
    while (names.hasNext()) {
        String name = names.next();
        JsonNode value = root.get(name);

        // Verifica se parece um RichText (tem type + encoding)
        if (value.isObject()) {
            JsonNode typeNode = value.get("type");
            JsonNode encodingNode = value.get("encoding");
            JsonNode contentNode = value.get("content");

            if (typeNode != null && encodingNode != null) {
                boolean empty = contentNode == null || contentNode.isNull()
                    || (contentNode.isTextual()
                        && contentNode.asText().trim().isEmpty());
                if (empty) toRemove.add(name);
            }
        }
    }

    toRemove.forEach(root::remove);
}
```

## O converter para Binder: RichTextToMimeConverter

Para integrar com o Binder do Vaadin, temos um converter bidirecional:

```java
public class RichTextToMimeConverter implements Converter<String, RichText> {

    @Override
    public Result<RichText> convertToModel(String html, ValueContext ctx) {
        RichText rt = new RichText();
        if (html != null && !html.trim().isEmpty()) {
            String base64 = Base64.getEncoder()
                .encodeToString(html.getBytes(StandardCharsets.UTF_8));
            rt.setContent(base64);
            rt.setEncoding("BASE64");
            rt.setType("text/html");
        } else {
            rt.setContent("");
            rt.setEncoding("PLAIN");
        }
        return Result.ok(rt);
    }

    @Override
    public String convertToPresentation(RichText rt, ValueContext ctx) {
        if (rt == null || rt.getContent() == null || rt.getContent().isEmpty())
            return "";

        if ("BASE64".equalsIgnoreCase(rt.getEncoding())) {
            try {
                byte[] decoded = Base64.getDecoder().decode(rt.getContent());
                return stripMimeHeaders(
                    new String(decoded, StandardCharsets.UTF_8));
            } catch (Exception e) {
                return rt.getContent();
            }
        } else if ("PLAIN".equalsIgnoreCase(rt.getEncoding())) {
            return stripMimeHeaders(rt.getContent());
        }
        return "";
    }
}
```

## O fluxo completo

```
Domino Form (campo RichText)
  │
  ▼  DRAPI GET /document/{unid}
JSON:
  "Body": {
    "type": "text/html",
    "encoding": "BASE64",
    "content": "PGh0bWw+..."     ← HTML em BASE64
  }
  │
  ▼  BodyDeserializer
RichText POJO:
  type="text/html", encoding="BASE64", content="PGh0bWw+..."
  │
  ▼  RichTextView.setValue(richText)
  │
  ├─ Modo leitura:
  │   Base64.decode(content) → HTML → stripMimeHeaders() → Div.innerHTML
  │
  └─ Modo edicao:
      Base64.decode(content) → HTML → RichTextEditor.setValue(html)
                                        │
                                        ▼  usuario edita...
                                  editor.getValue() → HTML
                                        │
                                        ▼  RichTextView.getValue()
                                  Base64.encode(html) → RichText POJO
                                        │
                                        ▼  removeEmptyRichTextFields()
                                  JSON para DRAPI:
                                  "Body": {
                                    "type": "text/html",
                                    "encoding": "BASE64",
                                    "content": "PG5ld0h0bWw+..."
                                  }
                                        │
                                        ▼  DRAPI PUT /document/{unid}
                                  Domino Form (RichText atualizado)
```

## Licoes aprendidas

| Desafio | Solucao |
|---|---|
| DRAPI retorna formatos diferentes (objeto, string, null) | BodyDeserializer com tres casos |
| Content pode ter headers MIME misturados | stripMimeHeaders() busca primeira tag HTML |
| Imagens grandes estouravam o layout | CSS inline `max-width: 100%` no Div e no editor |
| Content vazio causa NullPointerException no DRAPI | getValue() retorna null se vazio |
| Campos RichText vazios no payload causam erro | removeEmptyRichTextFields() antes do envio |
| Binder do Vaadin espera String, modelo usa RichText | RichTextToMimeConverter bidirecional |
| RichTextEditor corta conteudo longo | overflow-y: auto + altura flexivel |

---

Com esse pipeline, o conteudo RichText do Domino — com toda sua formatacao, imagens e tabelas — transita de ponta a ponta entre o DRAPI e o Vaadin sem perda de dados. O usuario edita num editor visual moderno, e o conteudo volta para o Domino no formato que ele espera.
