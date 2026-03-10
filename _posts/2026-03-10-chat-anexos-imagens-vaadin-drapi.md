---
layout: post
title: "Chat com Anexos e Imagens: Estendendo o Vaadin Chat com DRAPI como Repositorio"
date: 2026-03-10
categories: [domino, drapi, java, spring, vaadin, chat, rich-text]
---

Quando migramos o chat interno do HCL Domino Notes para uma stack moderna, o desafio nao era apenas replicar mensagens de texto — era preservar a experiencia completa: **rich text**, **anexos de arquivos** e **imagens coladas diretamente na conversa**. Este post documenta como construimos um sistema de chat corporativo usando Vaadin 25, Spring Boot 4 e o Domino REST API (DRAPI) como unico repositorio de dados.

## O legado: chat do Notes

O chat original do Domino era simples mas funcional: mensagens entre colaboradores armazenadas como documentos Notes, com suporte nativo a rich text e anexos via campo MIME. Na migracao, precisavamos manter essa capacidade sem introduzir um banco de dados adicional.

## Arquitetura da solucao

A decisao-chave foi usar o **DRAPI como repositorio unico** — cada mensagem e um documento no banco `chat.nsf`, acessado via REST API. Isso mantem compatibilidade com o Notes (administradores ainda podem ver mensagens pelo cliente Notes) e elimina a necessidade de um banco relacional para o chat.

### Componentes principais

```
ChatView (rota /chat)
├── ContactListPanel (lista de contatos com busca)
└── ConversationPane (painel de conversa)
    ├── ChatMessageBubble (renderizacao de mensagens)
    └── ChatMessageComposer (editor rich text + upload)
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

```java
// Query eficiente sem MIME para listagem
String uri = String.format("/query?dataSource=%s&mode=%s&fields=De,Para,...", scope, MODE_NOMIME);
```

## Polling e mensagens pendentes

Para simular tempo real sem WebSocket, usamos polling a cada 4 segundos com `UI.access()`:

```java
private static final int POLL_INTERVAL_MS = 4000;
```

Um detalhe importante: quando o usuario envia uma mensagem, ela aparece **imediatamente** como "pendente" no UI, antes mesmo de ser confirmada pelo DRAPI. Quando o proximo poll retorna a mensagem salva, a versao pendente e substituida pela versao real. Isso elimina a sensacao de latencia.

## ConversationId: a chave de tudo

Cada conversa e identificada por um `ConversationId` determinisico — a concatenacao ordenada dos dois usernames. Isso garante que ambos os participantes sempre referenciam a mesma conversa:

```java
String conversationId = Stream.of(user1, user2)
    .sorted()
    .collect(Collectors.joining("_"));
```

## CSS: detalhes que fazem diferenca

O CSS do chat usa variaveis Lumo do Vaadin para manter consistencia visual:

- Bolhas proprias: fundo `--lumo-primary-color-10pct`, alinhadas a direita
- Bolhas dos outros: fundo `--lumo-contrast-5pct`, alinhadas a esquerda
- Border-radius assimetrico (4px no canto do remetente) para indicar direcao
- Scroll automatico para a ultima mensagem

## Licoes aprendidas

1. **DRAPI como repositorio de chat funciona** — a latencia e aceitavel para chat corporativo (nao e WhatsApp, e ferramenta de trabalho)
2. **Rich text MIME e complicado** — o DRAPI tem modos diferentes para leitura com e sem MIME; queries bulk com MIME podem falhar
3. **Imagens coladas via base64** funcionam bem no Vaadin RichTextEditor, mas precisam ser extraidas como anexos separados para nao inflar o campo Body
4. **Polling com mensagens pendentes** e um padrao simples que funciona surpreendentemente bem — o usuario nao percebe os 4 segundos de delay
5. **Componentes extraidos** (ChatMessageBubble, ChatMessageComposer) facilitam manutencao e testes

## Proximos passos

- Notificacoes push via Vaadin `@Push` para substituir polling
- Mensagens de grupo (multiplos participantes)
- Busca full-text nas conversas via DQL
- Indicador de "digitando..." via presenca

---

*Este post faz parte da serie sobre a [migracao de HCL Domino para Java/Vaadin/DRAPI]({{ site.baseurl }}/).*
