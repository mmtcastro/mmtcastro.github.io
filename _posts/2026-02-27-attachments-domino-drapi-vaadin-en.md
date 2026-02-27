---
layout: post
title: "Domino Attachments in Vaadin: download, upload and deletion with deferred save via DRAPI"
date: 2026-02-27
categories: [domino, drapi, java, vaadin, attachments]
lang: en
---

In Domino, attachments live inside RichText fields of documents. DRAPI exposes dedicated endpoints to list, download, upload and delete attachments. But there is a UX challenge: we want the user to be able to add and remove files freely, and only commit the changes when saving the form. If they cancel, nothing changes on the server.

This post shows how we implemented this **deferred save** model for attachments in Vaadin.

## DRAPI endpoints for attachments

| Operation | Method | Endpoint |
|---|---|---|
| List names | GET | `/attachmentnames/{unid}?dataSource={scope}` |
| Download file | GET | `/attachments/{unid}/{fileName}?dataSource={scope}` |
| Upload file | POST | `/attachments/{unid}?fieldName={field}&dataSource={scope}` |
| Delete file | DELETE | `/attachments/{unid}/{fileName}?dataSource={scope}` |

**Important**: upload and delete require the document UNID. This means the document must be saved first to have a UNID — and that is exactly why attachment processing is **deferred**.

## The model: control fields in AbstractModelDoc

The base model maintains four lists to control the attachment lifecycle:

```java
public abstract class AbstractModelDoc {

    // List of file names (comes from DRAPI via $FILES)
    @JsonProperty("$FILES")
    protected List<String> fileNames;

    // Files in memory waiting for upload
    @JsonIgnore
    protected List<UploadedFile> uploads = new ArrayList<>();

    // Already downloaded files (local cache)
    @JsonIgnore
    protected List<UploadedFile> anexos = new ArrayList<>();

    // File names marked for deletion
    @JsonIgnore
    protected List<String> anexosParaExcluir = new ArrayList<>();

    // RichText field name in Domino (default: "Anexos")
    @JsonIgnore
    protected String attachmentFieldName = "Anexos";

    // Inner class for in-memory files
    public static class UploadedFile {
        private String fileName;
        private byte[] fileData;

        public UploadedFile(String fileName, byte[] fileData) {
            this.fileName = fileName;
            this.fileData = fileData;
        }
    }

    // Helper: add file to upload queue
    public void adicionarAnexo(UploadedFile file) {
        if (uploads == null) uploads = new ArrayList<>();
        uploads.add(file);
    }

    // Helper: check if there are pending changes
    public boolean hasAnexosChanges() {
        boolean hasUploads = uploads != null && !uploads.isEmpty();
        boolean hasDeletes = anexosParaExcluir != null
                && !anexosParaExcluir.isEmpty();
        return hasUploads || hasDeletes;
    }
}
```

The `uploads` and `anexosParaExcluir` lists are marked with `@JsonIgnore` — they are not included in the DRAPI JSON. They are internal control between the Vaadin component and the service.

## The Vaadin component: Anexos

The `Anexos` component displays the file list and allows download, upload and deletion:

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

        buildHeader();     // Icon + title "Attachments"
        buildFileList();   // File list
        if (!readOnly) {
            buildUploadComponent();  // Upload button
        }
    }
```

### Displaying the file list

The list shows two types of files:
1. **Saved files** (already on the server) — with download link
2. **Pending uploads** (only in memory) — with "(pending)" indicator

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

        // 1. Saved files — with download and delete button
        String unid = getDocumentUnid();
        if (!fileNames.isEmpty() && unid != null) {
            for (String fileName : fileNames) {
                FileResponse response = service.getAnexo(unid, fileName);
                if (response == null || response.getFileData() == null)
                    continue;

                // Download link via StreamResource
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

        // 2. Pending uploads — with visual indicator
        for (UploadedFile file : uploads) {
            Span pending = new Span(file.getFileName() + " (pending)");
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

### Download: Vaadin StreamResource

The download uses Vaadin's `StreamResource` to serve the file directly from the byte array:

```java
StreamResource resource = new StreamResource(fileName,
    () -> new ByteArrayInputStream(response.getFileData()));
resource.setContentType(response.getMediaType());

Anchor link = new Anchor(resource, fileName);
link.getElement().setAttribute("download", true);
```

The `download` attribute forces the browser to download the file instead of trying to open it.

### Upload: Vaadin Upload with handler

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

The file is read into memory as `byte[]` and added to the model's pending uploads list. **Nothing is sent to the server yet.**

### Deletion: marking for deferred deletion

```java
    private void deleteAnexo(String fileName) {
        // Mark for deletion on next save
        model.getAnexosParaExcluir().add(fileName);

        // Remove from local lists (UI)
        model.getAnexos().removeIf(
            f -> f.getFileName().equalsIgnoreCase(fileName));
        model.getUploads().removeIf(
            f -> f.getFileName().equalsIgnoreCase(fileName));
        model.getFileNames().removeIf(
            f -> f.equalsIgnoreCase(fileName));

        refreshFileList();
    }
```

The file **is not deleted from the server** — only marked in the `anexosParaExcluir` list. If the user cancels, the list is discarded and nothing changes in Domino.

## The deferred save: processarAnexos()

The actual processing only happens **after** the document is saved successfully:

### In the view (AbstractViewDoc):

```java
protected void save() {
    // 1. Save the document (without attachments)
    SaveResponse response = getService().save(model);

    if (response.isSuccess()) {
        // 2. If there are attachment changes, process now
        if (useAnexos() && model.hasAnexosChanges()
                && response.getMeta() != null) {
            String unid = response.getMeta().getUnid();
            boolean ok = anexos.processarAposSave(unid);
            if (!ok) {
                Notification.show("Failed to process attachments.");
            }
        }

        // 3. Update the interface
        if (useAnexos()) refreshAnexosSection();
    }
}
```

### In the service (AbstractService):

```java
public boolean processarAnexos(T model, String unid) {
    boolean ok = true;

    // 1. Delete marked files
    for (String fileName : model.getAnexosParaExcluir()) {
        FileResponse resp = deleteAnexo(unid, fileName);
        if (!resp.isDeleteSuccess()) ok = false;
    }
    model.getAnexosParaExcluir().clear();

    // 2. Fetch updated file list from server
    FileResponse namesResp = getAttachmentNames(unid);
    List<String> existing = namesResp.getFileNames();

    // 3. Upload new files (only if they don't exist on server)
    for (UploadedFile file : model.getUploads()) {
        if (!existing.contains(file.getFileName())) {
            uploadAnexo(unid, model.getAttachmentFieldName(),
                file.getFileName(),
                new ByteArrayInputStream(file.getFileData()));
            existing.add(file.getFileName());
        }
    }
    model.getUploads().clear();

    // 4. Update file names in the model
    model.setFileNames(existing);

    return ok;
}
```

### The DRAPI methods in the service:

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

// Delete
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

## The cancel flow

The key point of deferred save is cancellation. When the user clicks cancel:

```java
protected void cancel() {
    // Reload the original document from DRAPI
    Response<T> response = getService().findByUnid(model.getUnid());
    model = response.getModel();
    binder.setBean(model);

    // Rebuild the attachments section (original state)
    refreshAnexosSection();
}
```

By reloading the model from DRAPI, all pending lists (`uploads`, `anexosParaExcluir`) are discarded. The original files reappear in the UI. No operations were performed on the server.

## The complete cycle

```
Initial state: document with 2 attachments (report.pdf, photo.jpg)

1. User opens the document
   └─ GET /attachmentnames/{unid}
      → fileNames: ["report.pdf", "photo.jpg"]

2. User deletes "photo.jpg"
   └─ anexosParaExcluir: ["photo.jpg"]
      fileNames: ["report.pdf"]          ← removed from UI
      (server: photo.jpg STILL exists)

3. User uploads "contract.pdf"
   └─ uploads: [UploadedFile("contract.pdf", bytes)]
      UI shows: "contract.pdf (pending)"
      (server: contract.pdf DOES NOT exist yet)

4a. User clicks SAVE:
    ├─ PUT /document/{unid}          ← saves the document
    ├─ DELETE /attachments/{unid}/photo.jpg  ← deletes marked
    ├─ GET /attachmentnames/{unid}   ← updated list
    └─ POST /attachments/{unid}      ← uploads contract.pdf
       body: multipart/form-data
    Result: ["report.pdf", "contract.pdf"]

4b. User clicks CANCEL:
    ├─ GET /document/{unid}          ← reloads document
    └─ Pending lists discarded
    Result: ["report.pdf", "photo.jpg"] (unchanged)
```

## Why deferred save?

| Approach | Problem |
|---|---|
| Immediate upload | User cannot cancel — file is already on the server |
| Immediate delete | User cannot undo — file is already removed |
| Save everything together | DRAPI endpoint does not accept document + attachments in the same call |

Deferred save solves all these problems:
- **Upload** stays in memory until save
- **Delete** stays marked until save
- **Cancel** discards everything without touching the server
- **Sequence**: document first, attachments second (UNID guaranteed)

## Lessons learned

| Challenge | Solution |
|---|---|
| DRAPI requires UNID for upload | Save document first, attachments second |
| User needs to be able to cancel | In-memory pending lists (@JsonIgnore) |
| File names with spaces and accents | URLEncoder.encode() with replace of "+" to "%20" |
| Large files blow up memory | Upload via temp file (UploadHandler.toTempFile) |
| Duplicate uploads | Check existing list before sending |
| Delete of pending upload | Remove from uploads[] and not from anexosParaExcluir[] |

---

With this model, the user has the experience of adding and removing attachments freely — just like in any modern application — and the synchronization with Domino happens transparently and safely only at save time.
