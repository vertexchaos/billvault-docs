# Ingestion pipeline

How a document gets from an upload, a directory, a scheduled poll or a cloud-storage account to a persisted Receipt. Names are taken from app/api/routes.py, app/tasks.py, app/services/ingestor.py and app/services/extractor.py.

```mermaid
%%{init: {'theme':'dark','themeVariables':{'background':'#111111','primaryColor':'#1c4468','primaryTextColor':'#f1f3f5','primaryBorderColor':'#ced4da','lineColor':'#ced4da','secondaryColor':'#1f5133','tertiaryColor':'#2f343a','edgeLabelBackground':'#111111','fontFamily':'Cascadia Code, Cascadia Mono, monospace'}}}%%
flowchart TB
  n_dirapi["<b>POST /api/ingest/directory</b><br/>is_allowed_ingest_root() else 422; ImportBatch QUEUED"]:::blue
  n_upload["<b>POST /api/documents/upload</b><br/>413 over max_upload_size_bytes; save_bytes(); ImportBatch QUEUED"]:::blue
  n_poll["<b>poll_ingest_directories_task</b><br/>arq cron every 15 min; skips files whose hash is already known"]:::gray
  n_cloud["<b>poll_cloud_storage_accounts_task</b><br/>arq cron every 15 min; Dropbox and OneDrive accounts"]:::gray
  n_dirtask["<b>run_ingest_directory_task</b><br/>discover_supported_files(); sets files_discovered"]:::green
  n_filetask["<b>run_ingest_file_task</b><br/>max_tries=3; Retry(defer=30) on a transient gateway error"]:::green
  n_ingest["<b>ingest_file()</b><br/>_create_document(); .xml to extract_xml(), else extract()"]:::green
  n_xml["<b>extract_xml()</b><br/>UBL / ZUGFeRD with lxml"]:::green
  n_t1["<b>1. invoice2data templates</b><br/>_try_invoice2data() on a PDF"]:::green
  n_t2["<b>2. OCR gateway</b><br/>_try_unlimited_ocr(); also the only path for JPEG, PNG and TIFF"]:::green
  n_t3["<b>3. AI gateway</b><br/>_try_ai_gateway(): PDF text sent to an LLM"]:::green
  n_persist["<b>_persist()</b><br/>duplicate skip; card redaction (Luhn); review flags; Receipt + LineItems"]:::orange
  n_counters["<b>increment_batch_counter()</b><br/>atomic UPDATE...RETURNING: receipts_created or duplicates_skipped"]:::orange
  n_done["<b>ImportBatch COMPLETED</b><br/>upload: on success; directory: when files_processed == files_discovered"]:::orange
  n_dirapi -->|"enqueue"| n_dirtask
  n_upload -->|"enqueue"| n_filetask
  n_poll --> n_filetask
  n_cloud --> n_filetask
  n_dirtask -->|"one job per file"| n_filetask
  n_filetask --> n_ingest
  n_ingest -->|".xml"| n_xml
  n_ingest -->|"PDF"| n_t1
  n_ingest -->|"image bytes"| n_t2
  n_t1 -->|"no template match"| n_t2
  n_t2 -->|"no OCR result"| n_t3
  n_xml --> n_persist
  n_t1 --> n_persist
  n_t2 --> n_persist
  n_t3 --> n_persist
  n_persist --> n_counters
  n_counters --> n_done
  classDef blue fill:#1c4468,stroke:#ced4da,color:#f1f3f5,stroke-width:1px
  classDef green fill:#1f5133,stroke:#ced4da,color:#f1f3f5,stroke-width:1px
  classDef gray fill:#2f343a,stroke:#ced4da,color:#f1f3f5,stroke-width:1px
  classDef orange fill:#63401a,stroke:#ced4da,color:#f1f3f5,stroke-width:1px
```

Download: [SVG](ingestion-pipeline.svg) · [Excalidraw](ingestion-pipeline.excalidraw) · [Mermaid source](ingestion-pipeline.mmd) · [Interactive HTML](ingestion-pipeline.html) (docs site)

| Colour | Meaning |
|---|---|
| blue | Entry point or request handler |
| green | Worker task or pipeline step |
| gray | Scheduled trigger |
| orange | Persistence and batch state |

| Node | Detail |
|---|---|
| `POST /api/ingest/directory` | is_allowed_ingest_root() else 422; ImportBatch QUEUED |
| `POST /api/documents/upload` | 413 over max_upload_size_bytes; save_bytes(); ImportBatch QUEUED |
| `poll_ingest_directories_task` | arq cron every 15 min; skips files whose hash is already known |
| `poll_cloud_storage_accounts_task` | arq cron every 15 min; Dropbox and OneDrive accounts |
| `run_ingest_directory_task` | discover_supported_files(); sets files_discovered |
| `run_ingest_file_task` | max_tries=3; Retry(defer=30) on a transient gateway error |
| `ingest_file()` | _create_document(); .xml to extract_xml(), else extract() |
| `extract_xml()` | UBL / ZUGFeRD with lxml |
| `1. invoice2data templates` | _try_invoice2data() on a PDF |
| `2. OCR gateway` | _try_unlimited_ocr(); also the only path for JPEG, PNG and TIFF |
| `3. AI gateway` | _try_ai_gateway(): PDF text sent to an LLM |
| `_persist()` | duplicate skip; card redaction (Luhn); review flags; Receipt + LineItems |
| `increment_batch_counter()` | atomic UPDATE...RETURNING: receipts_created or duplicates_skipped |
| `ImportBatch COMPLETED` | upload: on success; directory: when files_processed == files_discovered |

**Waterfall order**

- XML first, then invoice2data templates, then the OCR gateway, then the AI gateway
- Deterministic paths always win before any AI call

**Failure handling**

- A transient gateway error retries the job after 30 s, up to 3 tries
- In a directory batch one bad file is counted in receipts_failed and does not stop the others

[All diagrams](index.md)
