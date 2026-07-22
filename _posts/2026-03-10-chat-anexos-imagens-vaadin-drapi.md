---
layout: post
title: "Chat com Anexos e Imagens: Estendendo o Vaadin Chat com DRAPI como Repositorio"
date: 2026-03-10
categories: [domino, drapi, java, spring, vaadin, chat, rich-text, signals]
---

> **Atualizado em 11/03/2026.** A primeira versao deste post descrevia apenas mensagens para si mesmo. Agora o chat ja suporta conversas 1:1 entre usuarios, presenca online, notificacoes em tempo real com **Vaadin Signals** e marcacao de lidas. Funcionalidades como mensagens de grupo e busca full-text ainda estao em desenvolvimento.

Quando migramos o chat interno do HCL Domino Notes para uma stack moderna, o desafio nao era apenas replicar mensagens de texto — era preservar a experiencia completa: **rich text**, **anexos de arquivos** e **imagens coladas diretamente na conversa**. Este post documenta a arquitetura e as decisoes de design de um sistema de chat corporativo usando Vaadin 25, Spring Boot 4 e o Domino REST API (DRAPI) como unico repositorio de dados.

## O legado: chat do Notes

O chat original do Domino era simples mas funcional: mensagens entre colaboradores armazenadas como documentos Notes, com suporte nativo a rich text e anexos via campo MIME. Na migracao, precisavamos manter essa capacidade sem introduzir um banco de dados adicional.

## Arquitetura da solucao

A decisao-chave foi usar o **DRAPI como repositorio unico** — cada mensagem e um documento em uma base NSF dedicada ao chat, acessada via REST API. Isso mantem compatibilidade com o Notes (administradores ainda podem ver mensagens pelo cliente Notes) e elimina a necessidade de um banco relacional para o chat.

### Componentes principais

```
ChatView (rota /chat)
├── ContactListPanel (lista de contatos com presenca e badges)
├── ConversationPane (painel de conversa)
│   ├── ChatMessageBubble (renderizacao de mensagens)
│   └── ChatMessageComposer (editor rich text + upload)
└── Servicos
    ├── ChatMessageService (CRUD e queries via DRAPI)
    └── ChatNotificationService (Signals para badge e toast)
```

## ChatMessageComposer: editor com superpoderes

O componente de composicao combina tres capacidades:

### 1. Rich Text Editor compacto

Usamos o `RichTextEditor` do Vaadin configurado com altura maxima de 200px — compacto o suficiente para nao dominar a tela, mas com formatacao completa (negrito, italico, listas, links).

```java
editor = new RichTextEditor();
editor.setWidthFull();
editor.setMaxHeight("200px");
```

### 2. Upload de arquivos

O componente `Upload` do Vaadin aceita multiplos arquivos. Cada arquivo e convertido para `Base64` e armazenado como `UploadedFile` — a mesma estrutura que o `AbstractService` usa para persistir anexos no DRAPI:

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

### 3. Colar imagens (paste)

O recurso mais natural: o usuario faz `Ctrl+V` de uma captura de tela e a imagem aparece inline na mensagem. O `RichTextEditor` do Vaadin ja suporta isso nativamente — imagens coladas sao inseridas como `data:image/png;base64,...` no HTML. Na hora de salvar, extraimos essas imagens do HTML e as convertemos em anexos separados no documento DRAPI.

## ChatMessageBubble: renderizacao inteligente

Cada mensagem e renderizada como uma bolha com:

- **Rich text** do campo `Body` (MIME) quando disponivel
- **Fallback** para texto puro do campo `Mensagem`
- **Lista de anexos** com icone de download e link para o DRAPI

```java
public class ChatMessageBubble extends Div {
    public ChatMessageBubble(ChatMessage msg, boolean isOwn, ChatMessageService service) {
        addClassName(isOwn ? "chat-bubble-own" : "chat-bubble-other");
        // Renderiza body rich text ou fallback texto puro
        // Lista anexos com StreamResource para download
    }
}
```

Os anexos sao baixados sob demanda via `StreamResource` que faz GET no DRAPI — o binario nunca fica em memoria no servidor.

## ChatMessageService: a camada DRAPI

O servico estende `AbstractService<ChatMessage>` e herda toda a infraestrutura de CRUD, incluindo:

- **Save com anexos**: o `AbstractService.save()` ja serializa `UploadedFile` como MIME parts no documento
- **Queries com DQL**: busca mensagens por `ConversationId` usando Domino Query Language
- **Modo `nomime`**: para queries bulk (lista de conversas), usamos um mode sem rich text para evitar erros de serializacao MIME e melhorar performance
- **Contagem de nao lidas**: `getUnreadCounts()` retorna um mapa `conversationId -> quantidade` para alimentar badges
- **Marcar como lida**: `markConversationAsRead()` atualiza o campo `Lida` nos documentos, executado em virtual thread para nao bloquear a UI

```java
// Query eficiente sem MIME para listagem
String uri = String.format("/query?dataSource=%s&mode=%s&fields=De,Para,...", scope, MODE_NOMIME);
```

## Vaadin Signals: notificacoes reativas em tempo real

Esta e a parte mais inovadora da implementacao. **Vaadin Signals** e um recurso **experimental do Vaadin 25** que introduz estado compartilhado reativo entre sessoes de diferentes usuarios — sem necessidade de WebSocket customizado, message broker ou polling.

### O que sao Signals?

Signals sao uma primitiva reativa inspirada nos Signals de frameworks frontend (como SolidJS e Angular), mas aplicada no contexto **full-stack** do Vaadin. Um Signal e um container de valor observavel que:

1. **E compartilhado entre sessoes**: usando `SignalFactory.IN_MEMORY_SHARED`, o mesmo Signal e visivel por todos os UIs conectados
2. **Notifica automaticamente**: quando o valor muda, todos os componentes que observam aquele Signal sao atualizados via `@Push`
3. **E tipado**: `NumberSignal` para contadores, `ValueSignal<T>` para valores genericos

### Habilitando Signals

Por ser experimental no Vaadin 25, e necessario ativar a feature flag:

```properties
# vaadin-featureflags.properties
com.vaadin.experimental.flowFullstackSignals=true
```

### Como usamos no Chat

Criamos o `ChatNotificationService` com dois Signals por usuario:

```java
@Service
public class ChatNotificationService {

    /** Contador de nao lidas — alimenta o badge no menu lateral. */
    public NumberSignal getUnreadSignal(String shortName) {
        return SignalFactory.IN_MEMORY_SHARED.number("chat-unread-" + shortName);
    }

    /** Ultima notificacao — dispara toast com preview da mensagem. */
    public ValueSignal<String> getLastMessageSignal(String shortName) {
        return SignalFactory.IN_MEMORY_SHARED.value("chat-lastmsg-" + shortName, "");
    }

    /** Chamado quando uma nova mensagem e persistida. */
    public void notifyNewMessage(String recipientShortName, String senderName, String preview) {
        getUnreadSignal(recipientShortName).incrementBy(1);
        getLastMessageSignal(recipientShortName)
            .value(senderName + "|" + preview + "|" + System.currentTimeMillis());
    }

    /** Zera o contador quando o usuario abre o chat. */
    public void clearUnread(String shortName) {
        getUnreadSignal(shortName).value(0);
    }
}
```

### Reagindo aos Signals no MainLayout

O `MainLayout` (shell da aplicacao, presente em todas as paginas) vincula o badge do menu e os toasts de notificacao aos Signals usando `ComponentEffect.effect()`:

```java
// Badge reativo no menu lateral
ComponentEffect.effect(chatBadge, () -> {
    int count = unreadSignal.valueAsInt();
    if (count > 0) {
        chatBadge.setText(count > 99 ? "99+" : String.valueOf(count));
        chatBadge.getStyle().set("display", "inline-flex");
    } else {
        chatBadge.getStyle().set("display", "none");
    }
});

// Toast reativo — mostra preview da mensagem quando fora do ChatView
ComponentEffect.effect(chatBadge, () -> {
    String lastMsg = lastMsgSignal.value();
    if (lastMsg == null || lastMsg.isBlank()) return;

    // Só mostra toast se NÃO estiver no ChatView
    // Formato: "senderName|preview|timestamp"
    // ... cria Notification com click handler para navegar ao chat
});
```

### Por que Signals e nao polling?

| Aspecto | Polling | Signals |
|---------|---------|---------|
| Latencia | 4-10 segundos | Instantaneo (via `@Push`) |
| Carga no servidor | N requests/minuto por usuario | Zero requests — push reativo |
| Complexidade | Timer + UI.access() | Declarativo (effect) |
| Estado compartilhado | Cada UI consulta independente | Signal unico, N observers |

O polling continua sendo usado para sincronizar o **conteudo das mensagens** dentro do `ConversationPane` (onde a precisao temporal importa e os dados vem do DRAPI). Mas para **notificacoes** (badge + toast), os Signals eliminaram completamente o polling.

### O fluxo completo

```
Usuario A envia mensagem
  → ConversationPane.sendMessage()
    → chatMessageService.save(msg)           // persiste no DRAPI
    → chatNotificationService.notifyNewMessage(userB, ...)
      → NumberSignal("chat-unread-userB").incrementBy(1)
      → ValueSignal("chat-lastmsg-userB").value("Marcelo|Oi!|1741...")

Usuario B (em qualquer pagina da aplicacao)
  → MainLayout detecta mudanca nos Signals via @Push
    → ComponentEffect atualiza badge: "1"
    → ComponentEffect exibe toast: "Marcelo: Oi!"
    → Click no toast → navega para /chat
```

Nenhuma linha de codigo de polling foi necessaria para as notificacoes. O Vaadin cuida de tudo via server push.

## Presenca online

Usamos um `SessionRegistry` que rastreia sessoes HTTP ativas. O `ChatView` atualiza a presenca a cada 5 segundos via poll:

```java
Set<String> onlineUsers = sessionRegistry.getActiveSessions().stream()
    .map(SessionInfo::username)
    .collect(Collectors.toSet());
contactListPanel.updatePresence(onlineUsers);
```

A lista de contatos reordena automaticamente: usuarios online primeiro (com indicador verde), offline abaixo.

## Polling e mensagens pendentes

Para sincronizar o conteudo das mensagens (que vem do DRAPI), usamos polling a cada 4 segundos:

```java
private static final int POLL_INTERVAL_MS = 4000;
```

Um detalhe importante: quando o usuario envia uma mensagem, ela aparece **imediatamente** como "pendente" no UI, antes mesmo de ser confirmada pelo DRAPI. Quando o proximo poll retorna a mensagem salva, a versao pendente e substituida pela versao real. Isso elimina a sensacao de latencia.

### Enriquecimento inteligente de mensagens

As queries bulk via DQL nao incluem o campo `Body` (MIME) nem `$FILES` — isso causaria erros de serializacao em documentos antigos e degradaria a performance. Em vez disso, fazemos **fetch individual** apenas das ultimas 30 mensagens para carregar rich text e nomes de anexos:

```java
private void enrichMessages(List<ChatMessage> messages) {
    int startIdx = Math.max(0, messages.size() - 30);
    List<ChatMessage> recentMessages = messages.subList(startIdx, messages.size());
    for (ChatMessage msg : recentMessages) {
        ChatMessage fullMsg = chatMessageService.loadMessageWithBody(unid);
        // ... enriquece Body e fileNames
    }
}
```

## Marcacao de lidas com Virtual Threads

Quando o usuario abre uma conversa, as mensagens sao marcadas como lidas em **background usando virtual threads** do Java 21. Isso evita bloquear a UI enquanto o DRAPI processa as atualizacoes:

```java
private void markAsReadAsync(String conversationId) {
    String token = chatMessageService.getUserToken(); // captura na UI thread
    Thread.startVirtualThread(() -> {
        chatMessageService.markConversationAsRead(conversationId, currentShortName, token);
    });
}
```

## ConversationId: a chave de tudo

Cada conversa e identificada por um `ConversationId` deterministico — a concatenacao ordenada dos dois usernames. Isso garante que ambos os participantes sempre referenciam a mesma conversa:

```java
String conversationId = Stream.of(user1, user2)
    .sorted()
    .collect(Collectors.joining("::"));
```

## CSS: detalhes que fazem diferenca

O CSS do chat usa variaveis Lumo do Vaadin para manter consistencia visual:

- Bolhas proprias: fundo `--lumo-primary-color-10pct`, alinhadas a direita
- Bolhas dos outros: fundo `--lumo-contrast-5pct`, alinhadas a esquerda
- Border-radius assimetrico (4px no canto do remetente) para indicar direcao
- Scroll automatico para a ultima mensagem

## Licoes aprendidas

1. **DRAPI como repositorio de chat funciona** — a latencia e aceitavel para chat corporativo (nao e WhatsApp, e ferramenta de trabalho)
2. **Rich text MIME e complicado** — o DRAPI tem modos diferentes para leitura com e sem MIME; queries bulk com MIME podem falhar, entao usamos `nomime` para listagem e fetch individual para rich text
3. **Imagens coladas via base64** funcionam bem no Vaadin RichTextEditor, mas precisam ser extraidas como anexos separados para nao inflar o campo Body
4. **Polling com mensagens pendentes** e um padrao simples que funciona surpreendentemente bem — o usuario nao percebe os 4 segundos de delay
5. **Vaadin Signals mudam o jogo** para notificacoes — o badge e o toast sao puramente reativos, sem polling. O `ComponentEffect.effect()` torna o codigo declarativo e facil de manter
6. **Virtual threads para I/O em background** — marcar mensagens como lidas envolve chamadas HTTP ao DRAPI; usar virtual threads evita bloquear a UI thread
7. **Componentes extraidos** (ChatMessageBubble, ChatMessageComposer, ContactListPanel) facilitam manutencao e permitem evolucao independente

## Proximos passos

- ~~Notificacoes push via Vaadin `@Push` para substituir polling~~ **Feito** (via Signals)
- Mensagens de grupo (multiplos participantes)
- Busca full-text nas conversas via DQL
- Indicador de "digitando..." via Signals (candidato natural)
- Persistencia de Signals (atualmente in-memory; se o servidor reiniciar, badges voltam a zero)

---

*Este post faz parte da serie sobre a [migracao de HCL Domino para Java/Vaadin/DRAPI]({{ site.baseurl }}/).*
