---
layout: post
title: "Anexos do Domino no Vaadin: download, upload e exclusao com save diferido via DRAPI"
date: 2026-02-27
categories: [domino, drapi, java, vaadin, attachments]
---

No Domino, anexos (attachments) vivem dentro de campos RichText dos documentos. O DRAPI expoe endpoints dedicados para listar, baixar, fazer upload e excluir anexos. Mas existe um desafio de UX: queremos que o usuario possa adicionar e remover arquivos livremente, e so efetivar as mudancas quando salvar o formulario. Se ele cancelar, nada muda no servidor.

Este post mostra como implementamos esse modelo de **save diferido** para anexos no Vaadin.

## Os endpoints do DRAPI para anexos

| Operacao | Metodo | Endpoint |
|---|---|---|
| Listar nomes | GET | `/attachmentnames/{unid}?dataSource={scope}` |
| Baixar arquivo | GET | `/attachments/{unid}/{fileName}?dataSource={scope}` |
| Fazer upload | POST | `/attachments/{unid}?fieldName={field}&dataSource={scope}` |
| Excluir arquivo | DELETE | `/attachments/{unid}/{fileName}?dataSource={scope}` |

**Importante**: upload e exclusao exigem o UNID do documento. Isso significa que o documento precisa ser salvo primeiro para ter um UNID — e e exatamente por isso que o processamento de anexos e **diferido**.

## O modelo: campos de controle no AbstractModelDoc

O modelo base mantem quatro listas para controlar o ciclo de vida dos anexos:

```java
public abstract class AbstractModelDoc {

    // Lista de nomes de arquivos (vem do DRAPI via $FILES)
    @JsonProperty("$FILES")
    protected List<String> fileNames;

    // Arquivos em memoria aguardando upload
    @JsonIgnore
    protected List<UploadedFile> uploads = new ArrayList<>();

    // Arquivos ja baixados (cache local)
    @JsonIgnore
    protected List<UploadedFile> anexos = new ArrayList<>();

    // Nomes de arquivos marcados para exclusao
    @JsonIgnore
    protected List<String> anexosParaExcluir = new ArrayList<>();

    // Nome do campo RichText no Domino (default: "Anexos")
    @JsonIgnore
    protected String attachmentFieldName = "Anexos";

    // Inner class para arquivos em memoria
    public static class UploadedFile {
        private String fileName;
        private byte[] fileData;

        public UploadedFile(String fileName, byte[] fileData) {
            this.fileName = fileName;
            this.fileData = fileData;
        }
    }

    // Helper: adiciona arquivo a fila de upload
    public void adicionarAnexo(UploadedFile file) {
        if (uploads == null) uploads = new ArrayList<>();
        uploads.add(file);
    }

    // Helper: verifica se ha mudancas pendentes
    public boolean hasAnexosChanges() {
        boolean hasUploads = uploads != null && !uploads.isEmpty();
        boolean hasDeletes = anexosParaExcluir != null
                && !anexosParaExcluir.isEmpty();
        return hasUploads || hasDeletes;
    }
}
```

As listas `uploads` e `anexosParaExcluir` sao marcadas com `@JsonIgnore` — elas nao vao para o JSON do DRAPI. Sao controle interno entre o componente Vaadin e o service.

## O componente Vaadin: Anexos

O componente `Anexos` exibe a lista de arquivos, permite download, upload e exclusao:

```java
public class Anexos<T extends AbstractModelDoc> extends VerticalLayout {

    private final T model;
    private final AbstractService<T> service;
    private boolean readOnly;
    private VerticalLayout fileListLayout;

    public Anexos(T model, AbstractService<T> service, boolean readOnly) {
        this.model = model;
        this.service = service;
        this.readOnly = readOnly;

        buildHeader();     // Icone + titulo "Anexos"
        buildFileList();   // Lista de arquivos
        if (!readOnly) {
            buildUploadComponent();  // Botao de upload
        }
    }
```

### Exibindo a lista de arquivos

A lista exibe dois tipos de arquivo:
1. **Arquivos salvos** (ja no servidor) — com link de download
2. **Uploads pendentes** (so em memoria) — com indicacao "(pendente)"

```java
    private void buildFileList() {
        fileListLayout = new VerticalLayout();

        List<String> fileNames = model.getFileNames();
        List<UploadedFile> uploads = model.getUploads();

        if (fileNames.isEmpty() && uploads.isEmpty()) {
            fileListLayout.add(new Span("No files attached."));
            add(fileListLayout);
            return;
        }

        // 1. Arquivos salvos — com download e botao excluir
        String unid = getDocumentUnid();
        if (!fileNames.isEmpty() && unid != null) {
            for (String fileName : fileNames) {
                FileResponse response = service.getAnexo(unid, fileName);
                if (response == null || response.getFileData() == null)
                    continue;

                // Link de download via StreamResource
                StreamResource resource = new StreamResource(fileName,
                    () -> new ByteArrayInputStream(response.getFileData()));
                resource.setContentType(response.getMediaType());

                Anchor link = new Anchor(resource,
                    fileName + " (" + (response.getFileData().length / 1024)
                    + " KB)");
                link.getElement().setAttribute("download", true);

                HorizontalLayout row = new HorizontalLayout(link);
                if (!readOnly) {
                    Button deleteBtn = new Button(VaadinIcon.TRASH.create(),
                        e -> deleteAnexo(fileName));
                    row.add(deleteBtn);
                }
                fileListLayout.add(row);
            }
        }

        // 2. Uploads pendentes — com indicacao visual
        for (UploadedFile file : uploads) {
            Span pending = new Span(file.getFileName() + " (pendente)");
            HorizontalLayout row = new HorizontalLayout(pending);
            if (!readOnly) {
                Button deleteBtn = new Button(VaadinIcon.TRASH.create(),
                    e -> deleteAnexo(file.getFileName()));
                row.add(deleteBtn);
            }
            fileListLayout.add(row);
        }

        add(fileListLayout);
    }
```

### Download: StreamResource do Vaadin

O download usa o `StreamResource` do Vaadin para servir o arquivo diretamente do byte array:

```java
StreamResource resource = new StreamResource(fileName,
    () -> new ByteArrayInputStream(response.getFileData()));
resource.setContentType(response.getMediaType());

Anchor link = new Anchor(resource, fileName);
link.getElement().setAttribute("download", true);
```

O atributo `download` forca o navegador a baixar o arquivo em vez de tentar abri-lo.

### Upload: Vaadin Upload com handler

```java
    private void buildUploadComponent() {
        BiConsumer<UploadMetadata, File> handler = (metadata, file) -> {
            try (InputStream input = new FileInputStream(file)) {
                byte[] content = input.readAllBytes();
                model.adicionarAnexo(
                    new UploadedFile(metadata.fileName(), content));
                Notification.show("File attached: " + metadata.fileName());
                refreshFileList();
            } catch (IOException e) {
                Notification.show("Error: " + e.getMessage());
            }
        };

        Upload upload = new Upload(
            UploadHandler.toTempFile(handler::accept));
        upload.setMaxFiles(10);
        upload.setMaxFileSize(50 * 1024 * 1024); // 50 MB
        add(upload);
    }
```

O arquivo e lido em memoria como `byte[]` e adicionado a lista de uploads pendentes do modelo. **Nada e enviado ao servidor ainda.**

### Exclusao: marcacao para exclusao diferida

```java
    private void deleteAnexo(String fileName) {
        // Marcar para exclusao no proximo save
        model.getAnexosParaExcluir().add(fileName);

        // Remover das listas locais (UI)
        model.getAnexos().removeIf(
            f -> f.getFileName().equalsIgnoreCase(fileName));
        model.getUploads().removeIf(
            f -> f.getFileName().equalsIgnoreCase(fileName));
        model.getFileNames().removeIf(
            f -> f.equalsIgnoreCase(fileName));

        refreshFileList();
    }
```

O arquivo **nao e excluido do servidor** — apenas marcado na lista `anexosParaExcluir`. Se o usuario cancelar, a lista e descartada e nada muda no Domino.

## O save diferido: processarAnexos()

O processamento real so acontece **depois** que o documento foi salvo com sucesso:

### Na view (AbstractViewDoc):

```java
protected void save() {
    // 1. Salvar o documento (sem anexos)
    SaveResponse response = getService().save(model);

    if (response.isSuccess()) {
        // 2. Se tem mudancas de anexos, processar agora
        if (useAnexos() && model.hasAnexosChanges()
                && response.getMeta() != null) {
            String unid = response.getMeta().getUnid();
            boolean ok = anexos.processarAposSave(unid);
            if (!ok) {
                Notification.show("Failed to process attachments.");
            }
        }

        // 3. Atualizar a interface
        if (useAnexos()) refreshAnexosSection();
    }
}
```

### No service (AbstractService):

```java
public boolean processarAnexos(T model, String unid) {
    boolean ok = true;

    // 1. Excluir arquivos marcados
    for (String fileName : model.getAnexosParaExcluir()) {
        FileResponse resp = deleteAnexo(unid, fileName);
        if (!resp.isDeleteSuccess()) ok = false;
    }
    model.getAnexosParaExcluir().clear();

    // 2. Buscar lista atualizada de arquivos no servidor
    FileResponse namesResp = getAttachmentNames(unid);
    List<String> existing = namesResp.getFileNames();

    // 3. Upload de arquivos novos (apenas se nao existem no servidor)
    for (UploadedFile file : model.getUploads()) {
        if (!existing.contains(file.getFileName())) {
            uploadAnexo(unid, model.getAttachmentFieldName(),
                file.getFileName(),
                new ByteArrayInputStream(file.getFileData()));
            existing.add(file.getFileName());
        }
    }
    model.getUploads().clear();

    // 4. Atualizar a lista de nomes no modelo
    model.setFileNames(existing);

    return ok;
}
```

### Os metodos DRAPI no service:

```java
// Download
public FileResponse getAnexo(String unid, String fileName) {
    String encoded = URLEncoder.encode(fileName, UTF_8).replace("+", "%20");
    return webClient.get()
        .uri("/attachments/{unid}/{fileName}?dataSource={scope}",
            unid, encoded, scope)
        .header("Authorization", "Bearer " + getUserToken())
        .retrieve()
        .toEntity(byte[].class)
        .map(entity -> new FileResponse(
            entity.getBody(),
            entity.getHeaders().getContentType().toString(),
            fileName))
        .block();
}

// Upload
public FileResponse uploadAnexo(String unid, String fieldName,
        String fileName, InputStream data) {
    return webClient.post()
        .uri("/attachments/{unid}?fieldName={field}&dataSource={scope}",
            unid, fieldName, scope)
        .header("Authorization", "Bearer " + getUserToken())
        .contentType(MediaType.MULTIPART_FORM_DATA)
        .body(BodyInserters.fromMultipartData("file",
            new ByteArrayResource(data.readAllBytes()) {
                @Override
                public String getFilename() { return fileName; }
            }))
        .retrieve()
        .bodyToMono(String.class)
        .map(body -> new FileResponse(true))
        .block();
}

// Exclusao
public FileResponse deleteAnexo(String unid, String fileName) {
    String encoded = URLEncoder.encode(fileName, UTF_8).replace("+", "%20");
    return webClient.delete()
        .uri("/attachments/{unid}/{fileName}?dataSource={scope}",
            unid, encoded, scope)
        .header("Authorization", "Bearer " + getUserToken())
        .retrieve()
        .toEntity(String.class)
        .map(entity -> new FileResponse(true, entity.getStatusCode().value()))
        .block();
}
```

## O fluxo de cancelamento

O ponto chave do save diferido e o cancelamento. Quando o usuario clica em cancelar:

```java
protected void cancel() {
    // Recarrega o documento original do DRAPI
    Response<T> response = getService().findByUnid(model.getUnid());
    model = response.getModel();
    binder.setBean(model);

    // Reconstroi a secao de anexos (estado original)
    refreshAnexosSection();
}
```

Ao recarregar o modelo do DRAPI, todas as listas pendentes (`uploads`, `anexosParaExcluir`) sao descartadas. Os arquivos originais reaparecem na interface. Nenhuma operacao foi feita no servidor.

## O ciclo completo

```
Estado inicial: documento com 2 anexos (report.pdf, photo.jpg)

1. Usuario abre o documento
   └─ GET /attachmentnames/{unid}
      → fileNames: ["report.pdf", "photo.jpg"]

2. Usuario exclui "photo.jpg"
   └─ anexosParaExcluir: ["photo.jpg"]
      fileNames: ["report.pdf"]          ← removido da UI
      (servidor: photo.jpg AINDA existe)

3. Usuario faz upload de "contract.pdf"
   └─ uploads: [UploadedFile("contract.pdf", bytes)]
      UI mostra: "contract.pdf (pendente)"
      (servidor: contract.pdf NAO existe ainda)

4a. Usuario clica SALVAR:
    ├─ PUT /document/{unid}          ← salva o documento
    ├─ DELETE /attachments/{unid}/photo.jpg  ← exclui marcados
    ├─ GET /attachmentnames/{unid}   ← lista atualizada
    └─ POST /attachments/{unid}      ← upload do contract.pdf
       body: multipart/form-data
    Resultado: ["report.pdf", "contract.pdf"]

4b. Usuario clica CANCELAR:
    ├─ GET /document/{unid}          ← recarrega documento
    └─ Listas pendentes descartadas
    Resultado: ["report.pdf", "photo.jpg"] (inalterado)
```

## Por que save diferido?

| Abordagem | Problema |
|---|---|
| Upload imediato | Usuario nao pode cancelar — arquivo ja esta no servidor |
| Exclusao imediata | Usuario nao pode desfazer — arquivo ja foi removido |
| Salvar tudo junto | Endpoint do DRAPI nao aceita documento + anexos na mesma chamada |

O save diferido resolve todos esses problemas:
- **Upload** fica em memoria ate o save
- **Exclusao** fica marcada ate o save
- **Cancelamento** descarta tudo sem tocar no servidor
- **Sequencia**: documento primeiro, anexos depois (UNID garantido)

## Licoes aprendidas

| Desafio | Solucao |
|---|---|
| DRAPI requer UNID para upload | Salvar documento primeiro, anexos depois |
| Usuario precisa poder cancelar | Listas de pendencia em memoria (@JsonIgnore) |
| Nomes de arquivo com espacos e acentos | URLEncoder.encode() com replace de "+" por "%20" |
| Arquivo grande estourar memoria | Upload via temp file (UploadHandler.toTempFile) |
| Duplicidade no upload | Verificar lista existente antes de enviar |
| Exclusao de upload pendente | Remover de uploads[] e nao de anexosParaExcluir[] |

---

Com esse modelo, o usuario tem a experiencia de adicionar e remover anexos livremente — como faria em qualquer aplicacao moderna — e a sincronizacao com o Domino acontece de forma transparente e segura apenas no momento do save.
