---
layout: post
title: "Domino RichText in Vaadin: from BASE64 in DRAPI to visual editor"
date: 2026-02-27
categories: [domino, drapi, java, vaadin, richtext]
lang: en
---

RichText fields in Domino are special — they store formatted HTML, embedded images, tables, and even attachments. When you fetch a document via DRAPI, the RichText field arrives as a JSON object with type, encoding, and content. This post shows how we transport that content all the way to a visual editor in Vaadin, preserving formatting and images.

## How DRAPI returns RichText

When you call `GET /api/v1/document/{unid}`, RichText fields come as objects:

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

The `content` field is HTML encoded in BASE64. Decoding it:

```
PGh0bWw+PGJvZHk+PHA+SGVsbG8gV29ybGQ8L3A+PC9ib2R5PjwvaHRtbD4=
                         ↓ Base64 decode
<html><body><p>Hello World</p></body></html>
```

But the encoding is not always BASE64 — in some cases DRAPI returns HTML as plain text (`PLAIN`), or even as a simple string without the object wrapper. We need to handle all these cases.

## The POJO: the RichText inner class

In `AbstractModelDoc`, we define an inner class to represent the DRAPI format:

```java
public abstract class AbstractModelDoc {

    public static class RichText {
        private String type;       // "text/html"
        private String encoding;   // "BASE64" or "PLAIN"
        private String headers;    // "Content-Type: text/html; charset=UTF-8"
        private String content;    // HTML in BASE64 or plain text

        public RichText() {
            this.type = "text/html";
            this.encoding = "BASE64";
            this.headers = "Content-Type: text/html; charset=UTF-8";
        }

        // Helper: set content as text, encoding to BASE64
        public void setPlainTextContent(String plainText) {
            this.content = Base64.getEncoder()
                .encodeToString(plainText.getBytes(StandardCharsets.UTF_8));
        }

        // Helper: set content as raw HTML (no encoding)
        public void setHtmlContent(String html) {
            this.content = html;
            this.encoding = "PLAIN";
        }

        // Ensure format is text/html + BASE64
        public void ensureHtmlBase64Encoding() {
            if (!"text/html".equalsIgnoreCase(this.type))
                this.type = "text/html";
            if (!"BASE64".equalsIgnoreCase(this.encoding))
                this.encoding = "BASE64";
        }
    }
}
```

In the concrete model, the field is annotated with the custom deserializer:

```java
@Getter @Setter
public class Company extends AbstractModelDoc {

    @JsonProperty("Observations")
    @JsonAlias({"observations", "Observations"})
    @JsonDeserialize(using = BodyDeserializer.class)
    private RichText observations;

    public RichText getObservations() {
        return this.observations != null ? this.observations : new RichText();
    }
}
```

## The BodyDeserializer: handling all formats

DRAPI can return RichText in three different ways. The deserializer handles all of them:

```java
public class BodyDeserializer extends JsonDeserializer<RichText> {

    @Override
    public RichText deserialize(JsonParser parser, DeserializationContext ctx)
            throws IOException {

        JsonNode node = parser.getCodec().readTree(parser);

        // Case 1: null or empty string → empty RichText
        if (node == null || node.isNull()
                || (node.isTextual() && node.asText().trim().isEmpty())) {
            return new RichText();
        }

        // Case 2: complete JSON object (standard DRAPI format)
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

        // Case 3: plain string (HTML without wrapper)
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

## The Vaadin component: RichTextView

The component we created implements `HasValue<RichText>` for Binder compatibility, and operates in two modes:

### Read mode: HTML rendered in a Div

```java
public class RichTextView extends VerticalLayout
        implements HasValue<ValueChangeEvent<RichText>, RichText> {

    private final RichTextEditor editor;  // Edit mode
    private final Div readOnlyView;       // Read mode
    private boolean readOnly = false;
    private RichText value;

    public RichTextView() {
        // Editor for edit mode
        editor = new RichTextEditor();
        editor.setWidthFull();
        editor.setHeight("800px");

        // Div for read mode (grows freely)
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

### Display update based on mode

```java
    private void updateDisplay() {
        removeAll();

        if (readOnly) {
            String html = getHtmlContent(value);
            String styled = "<style>"
                + "img { max-width: 100% !important; height: auto !important; }"
                + "</style>" + html;
            readOnlyView.getElement().setProperty("innerHTML", styled);
            add(readOnlyView);
        } else {
            String html = getHtmlContent(value);
            editor.setValue(html);
            add(editor);
        }
    }
```

### Decoding: from RichText to HTML

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
                raw = richText.getContent();
            }
        } else if ("PLAIN".equalsIgnoreCase(richText.getEncoding())) {
            raw = richText.getContent() != null ? richText.getContent() : "";
        }

        return stripMimeHeaders(raw);
    }
```

### Stripping MIME headers

Decoded content may come with MIME headers at the beginning. The `stripMimeHeaders()` method finds the first real HTML tag and discards everything before it:

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

        int lastCt = lower.lastIndexOf("content-type:");
        if (lastCt >= 0) {
            return raw.substring(lastCt + "Content-Type:".length()).trim();
        }

        return raw;
    }
```

### Encoding: from HTML to RichText (on save)

When the user edits and saves, HTML from the editor is converted back to DRAPI format:

```java
    public RichText getValue() {
        if (readOnly) {
            if (value == null || value.getContent() == null
                    || value.getContent().trim().isEmpty()) {
                return null;
            }
            return value;
        }

        String html = editor.getValue();
        if (html == null || html.trim().isEmpty()) {
            return null; // empty field = do not send
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

**Critical detail**: if content is empty, we return `null` instead of a RichText with empty content. This prevents a `NullPointerException` in DRAPI when trying to decode BASE64 from a null content.

## Save protection: removeEmptyRichTextFields()

Before sending JSON to DRAPI, we remove empty RichText fields from the payload:

```java
private void removeEmptyRichTextFields(ObjectNode root) {
    List<String> toRemove = new ArrayList<>();

    Iterator<String> names = root.fieldNames();
    while (names.hasNext()) {
        String name = names.next();
        JsonNode value = root.get(name);

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

## The Binder converter: RichTextToMimeConverter

For Vaadin Binder integration, we have a bidirectional converter:

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

## The complete flow

```
Domino Form (RichText field)
  │
  ▼  DRAPI GET /document/{unid}
JSON:
  "Body": {
    "type": "text/html",
    "encoding": "BASE64",
    "content": "PGh0bWw+..."     ← HTML in BASE64
  }
  │
  ▼  BodyDeserializer
RichText POJO:
  type="text/html", encoding="BASE64", content="PGh0bWw+..."
  │
  ▼  RichTextView.setValue(richText)
  │
  ├─ Read mode:
  │   Base64.decode(content) → HTML → stripMimeHeaders() → Div.innerHTML
  │
  └─ Edit mode:
      Base64.decode(content) → HTML → RichTextEditor.setValue(html)
                                        │
                                        ▼  user edits...
                                  editor.getValue() → HTML
                                        │
                                        ▼  RichTextView.getValue()
                                  Base64.encode(html) → RichText POJO
                                        │
                                        ▼  removeEmptyRichTextFields()
                                  JSON for DRAPI:
                                  "Body": {
                                    "type": "text/html",
                                    "encoding": "BASE64",
                                    "content": "PG5ld0h0bWw+..."
                                  }
                                        │
                                        ▼  DRAPI PUT /document/{unid}
                                  Domino Form (RichText updated)
```

## Lessons learned

| Challenge | Solution |
|---|---|
| DRAPI returns different formats (object, string, null) | BodyDeserializer with three cases |
| Content may have mixed MIME headers | stripMimeHeaders() finds first HTML tag |
| Large images broke the layout | Inline CSS `max-width: 100%` on Div and editor |
| Empty content causes NullPointerException in DRAPI | getValue() returns null if empty |
| Empty RichText fields in payload cause errors | removeEmptyRichTextFields() before sending |
| Vaadin Binder expects String, model uses RichText | Bidirectional RichTextToMimeConverter |
| RichTextEditor cuts long content | overflow-y: auto + flexible height |

---

With this pipeline, Domino RichText content — with all its formatting, images, and tables — flows end-to-end between DRAPI and Vaadin without data loss. The user edits in a modern visual editor, and the content returns to Domino in the format it expects.
