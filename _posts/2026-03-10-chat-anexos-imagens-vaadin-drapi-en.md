---
layout: post
title: "Chat with Attachments and Images: Extending Vaadin Chat with DRAPI as Repository"
date: 2026-03-10
categories: [domino, drapi, java, spring, vaadin, chat, rich-text]
lang: en
---

When migrating the internal chat from HCL Domino Notes to a modern stack, the challenge wasn't just replicating text messages — it was preserving the complete experience: **rich text**, **file attachments**, and **images pasted directly into the conversation**. This post documents how we built a corporate chat system using Vaadin 25, Spring Boot 4, and the Domino REST API (DRAPI) as the sole data repository.

## The legacy: Notes chat

The original Domino chat was simple but functional: messages between collaborators stored as Notes documents, with native support for rich text and attachments via MIME fields. In the migration, we needed to maintain this capability without introducing an additional database.

## Solution architecture

The key decision was to use **DRAPI as the single repository** — each message is a document in the `chat.nsf` database, accessed via REST API. This maintains compatibility with Notes (administrators can still view messages through the Notes client) and eliminates the need for a relational database for chat.

### Main components

```
ChatView (route /chat)
├── ContactListPanel (contact list with search)
└── ConversationPane (conversation panel)
    ├── ChatMessageBubble (message rendering)
    └── ChatMessageComposer (rich text editor + upload)
```

## ChatMessageComposer: editor with superpowers

The composition component combines three capabilities:

### 1. Compact Rich Text Editor

We use Vaadin's `RichTextEditor` configured with a 200px max height — compact enough not to dominate the screen, but with full formatting (bold, italic, lists, links).

```java
editor = new RichTextEditor();
editor.setWidthFull();
editor.setMaxHeight("200px");
```

### 2. File upload

Vaadin's `Upload` component accepts multiple files. Each file is converted to `Base64` and stored as an `UploadedFile` — the same structure that `AbstractService` uses to persist attachments in DRAPI:

```java
Upload upload = new Upload(UploadHandler.inMemory(
    (InputStream is, UploadMetadata meta) -> {
        byte[] bytes = is.readAllBytes();
        String base64 = Base64.getEncoder().encodeToString(bytes);
        pendingUploads.add(new AbstractModelDoc.UploadedFile(
            meta.fileName(), meta.contentType(), base64));
    }
));
```

### 3. Image paste (Ctrl+V)

The most natural feature: the user does `Ctrl+V` with a screenshot and the image appears inline in the message. Vaadin's `RichTextEditor` already supports this natively — pasted images are inserted as `data:image/png;base64,...` in the HTML. When saving, we extract these images from the HTML and convert them into separate attachments in the DRAPI document.

## ChatMessageBubble: smart rendering

Each message is rendered as a bubble with:

- **Rich text** from the `Body` field (MIME) when available
- **Fallback** to plain text from the `Mensagem` field
- **Attachment list** with download icon and DRAPI link

```java
public class ChatMessageBubble extends Div {
    public ChatMessageBubble(ChatMessage msg, boolean isOwn, ChatMessageService service) {
        addClassName(isOwn ? "chat-bubble-own" : "chat-bubble-other");
        // Renders rich text body or plain text fallback
        // Lists attachments with StreamResource for download
    }
}
```

Attachments are downloaded on demand via `StreamResource` that does a GET on DRAPI — the binary never stays in server memory.

## ChatMessageService: the DRAPI layer

The service extends `AbstractService<ChatMessage>` and inherits the entire CRUD infrastructure, including:

- **Save with attachments**: `AbstractService.save()` already serializes `UploadedFile` as MIME parts in the document
- **DQL queries**: searches messages by `ConversationId` using Domino Query Language
- **`nomime` mode**: for bulk queries (conversation list), we use a mode without rich text to avoid MIME serialization errors and improve performance

```java
// Efficient query without MIME for listing
String uri = String.format("/query?dataSource=%s&mode=%s&fields=De,Para,...", scope, MODE_NOMIME);
```

## Polling and pending messages

To simulate real-time without WebSocket, we use polling every 4 seconds with `UI.access()`:

```java
private static final int POLL_INTERVAL_MS = 4000;
```

An important detail: when the user sends a message, it appears **immediately** as "pending" in the UI, before being confirmed by DRAPI. When the next poll returns the saved message, the pending version is replaced by the real one. This eliminates the perception of latency.

## ConversationId: the key to everything

Each conversation is identified by a deterministic `ConversationId` — the sorted concatenation of both usernames. This ensures both participants always reference the same conversation:

```java
String conversationId = Stream.of(user1, user2)
    .sorted()
    .collect(Collectors.joining("_"));
```

## CSS: details that make a difference

The chat CSS uses Vaadin's Lumo variables to maintain visual consistency:

- Own bubbles: `--lumo-primary-color-10pct` background, right-aligned
- Others' bubbles: `--lumo-contrast-5pct` background, left-aligned
- Asymmetric border-radius (4px on the sender's corner) to indicate direction
- Auto-scroll to the latest message

## Lessons learned

1. **DRAPI as a chat repository works** — latency is acceptable for corporate chat (it's not WhatsApp, it's a work tool)
2. **Rich text MIME is tricky** — DRAPI has different modes for reading with and without MIME; bulk queries with MIME can fail
3. **Pasted images via base64** work well in Vaadin RichTextEditor, but need to be extracted as separate attachments to avoid inflating the Body field
4. **Polling with pending messages** is a simple pattern that works surprisingly well — the user doesn't notice the 4-second delay
5. **Extracted components** (ChatMessageBubble, ChatMessageComposer) make maintenance and testing easier

## Next steps

- Push notifications via Vaadin `@Push` to replace polling
- Group messages (multiple participants)
- Full-text search in conversations via DQL
- "Typing..." indicator via presence

---

*This post is part of the series on [migrating HCL Domino to Java/Vaadin/DRAPI]({{ site.baseurl }}/).*
