---
layout: post
title: "Chat with Attachments and Images: Extending Vaadin Chat with DRAPI as Repository"
date: 2026-03-10
categories: [domino, drapi, java, spring, vaadin, chat, rich-text, signals]
lang: en
---

> **Updated on 2026-03-11.** The first version of this post described only messages to yourself. Now the chat supports 1:1 conversations between users, online presence, real-time notifications with **Vaadin Signals**, and read receipts. Features like group messages and full-text search are still under development.

When migrating the internal chat from HCL Domino Notes to a modern stack, the challenge wasn't just replicating text messages — it was preserving the complete experience: **rich text**, **file attachments**, and **images pasted directly into the conversation**. This post documents the architecture and design decisions for a corporate chat system using Vaadin 25, Spring Boot 4, and the Domino REST API (DRAPI) as the sole data repository.

## The legacy: Notes chat

The original Domino chat was simple but functional: messages between collaborators stored as Notes documents, with native support for rich text and attachments via MIME fields. In the migration, we needed to maintain this capability without introducing an additional database.

## Solution architecture

The key decision was to use **DRAPI as the single repository** — each message is a document in a dedicated NSF database for chat, accessed via REST API. This maintains compatibility with Notes (administrators can still view messages through the Notes client) and eliminates the need for a relational database for chat.

### Main components

```
ChatView (route /chat)
├── ContactListPanel (contact list with presence and badges)
├── ConversationPane (conversation panel)
│   ├── ChatMessageBubble (message rendering)
│   └── ChatMessageComposer (rich text editor + upload)
└── Services
    ├── ChatMessageService (CRUD and queries via DRAPI)
    └── ChatNotificationService (Signals for badge and toast)
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
- **Unread counts**: `getUnreadCounts()` returns a `conversationId -> count` map to feed badges
- **Mark as read**: `markConversationAsRead()` updates the `Lida` field on documents, executed in a virtual thread to avoid blocking the UI

```java
// Efficient query without MIME for listing
String uri = String.format("/query?dataSource=%s&mode=%s&fields=De,Para,...", scope, MODE_NOMIME);
```

## Vaadin Signals: reactive real-time notifications

This is the most innovative part of the implementation. **Vaadin Signals** is an **experimental feature in Vaadin 25** that introduces shared reactive state across different users' sessions — no custom WebSocket, message broker, or polling needed.

### What are Signals?

Signals are a reactive primitive inspired by Signals in frontend frameworks (like SolidJS and Angular), but applied in Vaadin's **full-stack** context. A Signal is an observable value container that:

1. **Is shared across sessions**: using `SignalFactory.IN_MEMORY_SHARED`, the same Signal is visible to all connected UIs
2. **Notifies automatically**: when the value changes, all components observing that Signal are updated via `@Push`
3. **Is typed**: `NumberSignal` for counters, `ValueSignal<T>` for generic values

### Enabling Signals

Being experimental in Vaadin 25, the feature flag must be activated:

```properties
# vaadin-featureflags.properties
com.vaadin.experimental.flowFullstackSignals=true
```

### How we use Signals in Chat

We created `ChatNotificationService` with two Signals per user:

```java
@Service
public class ChatNotificationService {

    /** Unread counter — feeds the badge in the side menu. */
    public NumberSignal getUnreadSignal(String shortName) {
        return SignalFactory.IN_MEMORY_SHARED.number("chat-unread-" + shortName);
    }

    /** Last notification — triggers a toast with message preview. */
    public ValueSignal<String> getLastMessageSignal(String shortName) {
        return SignalFactory.IN_MEMORY_SHARED.value("chat-lastmsg-" + shortName, "");
    }

    /** Called when a new message is persisted. */
    public void notifyNewMessage(String recipientShortName, String senderName, String preview) {
        getUnreadSignal(recipientShortName).incrementBy(1);
        getLastMessageSignal(recipientShortName)
            .value(senderName + "|" + preview + "|" + System.currentTimeMillis());
    }

    /** Clears the counter when the user opens the chat. */
    public void clearUnread(String shortName) {
        getUnreadSignal(shortName).value(0);
    }
}
```

### Reacting to Signals in MainLayout

The `MainLayout` (application shell, present on every page) binds the menu badge and toast notifications to the Signals using `ComponentEffect.effect()`:

```java
// Reactive badge in the side menu
ComponentEffect.effect(chatBadge, () -> {
    int count = unreadSignal.valueAsInt();
    if (count > 0) {
        chatBadge.setText(count > 99 ? "99+" : String.valueOf(count));
        chatBadge.getStyle().set("display", "inline-flex");
    } else {
        chatBadge.getStyle().set("display", "none");
    }
});

// Reactive toast — shows message preview when outside ChatView
ComponentEffect.effect(chatBadge, () -> {
    String lastMsg = lastMsgSignal.value();
    if (lastMsg == null || lastMsg.isBlank()) return;

    // Only shows toast if NOT currently in ChatView
    // Format: "senderName|preview|timestamp"
    // ... creates Notification with click handler to navigate to chat
});
```

### Why Signals instead of polling?

| Aspect | Polling | Signals |
|--------|---------|---------|
| Latency | 4-10 seconds | Instant (via `@Push`) |
| Server load | N requests/minute per user | Zero requests — reactive push |
| Complexity | Timer + UI.access() | Declarative (effect) |
| Shared state | Each UI queries independently | Single Signal, N observers |

Polling is still used to synchronize **message content** inside `ConversationPane` (where temporal precision matters and data comes from DRAPI). But for **notifications** (badge + toast), Signals completely eliminated polling.

### The complete flow

```
User A sends a message
  → ConversationPane.sendMessage()
    → chatMessageService.save(msg)           // persists to DRAPI
    → chatNotificationService.notifyNewMessage(userB, ...)
      → NumberSignal("chat-unread-userB").incrementBy(1)
      → ValueSignal("chat-lastmsg-userB").value("Marcelo|Hi!|1741...")

User B (on any page of the application)
  → MainLayout detects Signal change via @Push
    → ComponentEffect updates badge: "1"
    → ComponentEffect shows toast: "Marcelo: Hi!"
    → Click on toast → navigates to /chat
```

No polling code was needed for notifications. Vaadin handles everything via server push.

## Online presence

We use a `SessionRegistry` that tracks active HTTP sessions. `ChatView` updates presence every 5 seconds via poll:

```java
Set<String> onlineUsers = sessionRegistry.getActiveSessions().stream()
    .map(SessionInfo::username)
    .collect(Collectors.toSet());
contactListPanel.updatePresence(onlineUsers);
```

The contact list reorders automatically: online users first (with green indicator), offline below.

## Polling and pending messages

To synchronize message content (which comes from DRAPI), we use polling every 4 seconds:

```java
private static final int POLL_INTERVAL_MS = 4000;
```

An important detail: when the user sends a message, it appears **immediately** as "pending" in the UI, before being confirmed by DRAPI. When the next poll returns the saved message, the pending version is replaced by the real one. This eliminates the perception of latency.

### Smart message enrichment

Bulk DQL queries don't include the `Body` field (MIME) or `$FILES` — this would cause serialization errors on older documents and degrade performance. Instead, we do **individual fetch** only for the last 30 messages to load rich text and attachment names:

```java
private void enrichMessages(List<ChatMessage> messages) {
    int startIdx = Math.max(0, messages.size() - 30);
    List<ChatMessage> recentMessages = messages.subList(startIdx, messages.size());
    for (ChatMessage msg : recentMessages) {
        ChatMessage fullMsg = chatMessageService.loadMessageWithBody(unid);
        // ... enriches Body and fileNames
    }
}
```

## Read receipts with Virtual Threads

When the user opens a conversation, messages are marked as read **in the background using Java 21 virtual threads**. This avoids blocking the UI while DRAPI processes the updates:

```java
private void markAsReadAsync(String conversationId) {
    String token = chatMessageService.getUserToken(); // capture in UI thread
    Thread.startVirtualThread(() -> {
        chatMessageService.markConversationAsRead(conversationId, currentShortName, token);
    });
}
```

## ConversationId: the key to everything

Each conversation is identified by a deterministic `ConversationId` — the sorted concatenation of both usernames. This ensures both participants always reference the same conversation:

```java
String conversationId = Stream.of(user1, user2)
    .sorted()
    .collect(Collectors.joining("::"));
```

## CSS: details that make a difference

The chat CSS uses Vaadin's Lumo variables to maintain visual consistency:

- Own bubbles: `--lumo-primary-color-10pct` background, right-aligned
- Others' bubbles: `--lumo-contrast-5pct` background, left-aligned
- Asymmetric border-radius (4px on the sender's corner) to indicate direction
- Auto-scroll to the latest message

## Lessons learned

1. **DRAPI as a chat repository works** — latency is acceptable for corporate chat (it's not WhatsApp, it's a work tool)
2. **Rich text MIME is tricky** — DRAPI has different modes for reading with and without MIME; bulk queries with MIME can fail, so we use `nomime` for listing and individual fetch for rich text
3. **Pasted images via base64** work well in Vaadin RichTextEditor, but need to be extracted as separate attachments to avoid inflating the Body field
4. **Polling with pending messages** is a simple pattern that works surprisingly well — the user doesn't notice the 4-second delay
5. **Vaadin Signals are a game changer** for notifications — the badge and toast are purely reactive, no polling needed. `ComponentEffect.effect()` makes code declarative and easy to maintain
6. **Virtual threads for background I/O** — marking messages as read involves HTTP calls to DRAPI; virtual threads avoid blocking the UI thread
7. **Extracted components** (ChatMessageBubble, ChatMessageComposer, ContactListPanel) make maintenance easier and allow independent evolution

## Next steps

- ~~Push notifications via Vaadin `@Push` to replace polling~~ **Done** (via Signals)
- Group messages (multiple participants)
- Full-text search in conversations via DQL
- "Typing..." indicator via Signals (natural candidate)
- Signal persistence (currently in-memory; if the server restarts, badges reset to zero)

---

*This post is part of the series on [migrating HCL Domino to Java/Vaadin/DRAPI]({{ site.baseurl }}/).*
