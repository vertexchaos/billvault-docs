# BillVault

BillVault turns receipts and invoices into structured, reviewable data.

## What it does

- **Takes documents in** from an upload, a watched folder, a Dropbox or OneDrive account, or a scheduled
  poll, as PDF, image or XML (UBL / ZUGFeRD).
- **Extracts the fields** with a fixed waterfall: invoice templates first, then an OCR service, then an AI
  service only when the earlier steps find nothing. Deterministic steps always run before any AI call.
- **Protects card data**: card-number-shaped values in extracted text are detected, checked and redacted
  before anything is stored, and the receipt is flagged for review.
- **Keeps a person in charge**: every extracted receipt goes to a review queue, with the reason it needs
  a look, and page operations (rotate, delete, reorder, split, merge) on the source documents.
- **Signs people in** with Google, Microsoft, LinkedIn or GitHub.

## Documentation

- [Architecture diagrams](diagrams/index.md): how documents flow through the system and how sign-in works.
- [API reference](reference/api.md): every endpoint and schema, generated from the running application.
