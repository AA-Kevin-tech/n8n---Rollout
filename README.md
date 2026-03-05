# n8n RAG System for Rollout (Company Policy) Documentation

This project provides n8n workflows that watch a Google Drive folder for new or updated company policy documents, index them into a vector store, and answer employee questions via a webhook using retrieved context.

## What's in this repo

| Item | Purpose |
|------|--------|
| `n8n - Rollout/workflows/rollout-drive-watch.json` | Runs on a schedule; lists files in your policy folder that are new or modified since the last run; triggers the ingest workflow for each file. |
| `n8n - Rollout/workflows/rollout-ingest.json` | Triggered by the watch workflow (or manually for backfill): downloads each file, extracts text, chunks, embeds, and upserts into the vector store. |
| `n8n - Rollout/workflows/rollout-policy-ask-workflow.json` | Webhook for employee questions: retrieves relevant chunks from the vector store and returns a grounded answer (with optional Structured Governor). |
| `n8n - Rollout/docs/` | Internal notes (chunking, folder ID, etc.). |

## Prerequisites

1. **n8n** instance (cloud or self-hosted).
2. **Google Drive**  
   - A dedicated folder for Rollout policy docs (signage rules, feedouts, employee/manager expectations).  
   - [Google Drive OAuth2 credentials](https://docs.n8n.io/integrations/builtin/credentials/google/) in n8n (same as "Google Drive" or "Google API").
3. **OpenAI**  
   - [OpenAI API key](https://platform.openai.com/api-keys) for embeddings (and for the Q&A LLM).
4. **Vector store**  
   - **Qdrant** ([Qdrant Cloud](https://qdrant.tech/) or self-hosted) and [Qdrant credentials](https://docs.n8n.io/integrations/credentials/qdrant/) in n8n, **or**  
   - **Supabase** with pgvector and corresponding credentials in n8n.

## Setup

### 1. Credentials in n8n

- **Google Drive**: Create OAuth2 credentials (e.g. "Google Drive OAuth2") and complete the consent flow so n8n can list and download files from your policy folder.
- **OpenAI**: Create an API key credential in n8n.
- **Qdrant** (or **Supabase**): Create the credential and note the collection name (e.g. `rollout_policy`).

### 2. Create the vector store collection

- **Qdrant**: Create a collection (e.g. `rollout_policy`) with the same dimension as your embedding model (e.g. 1536 for `text-embedding-3-small`, 3072 for `text-embedding-3-large`). You can create it via Qdrant UI or let the first ingest run create it if the node supports auto-creation.
- **Supabase**: Create a table with a vector column and use the same collection/table name in the workflows.

### 3. Import the workflows

1. In n8n: **Workflows → Import from File** (or paste JSON).
2. Import in this order (so IDs and names match):
   - `n8n - Rollout/workflows/rollout-ingest.json`
   - `n8n - Rollout/workflows/rollout-drive-watch.json`
   - `n8n - Rollout/workflows/rollout-policy-ask-workflow.json`
3. In **rollout-drive-watch**:
   - Set the **Google Drive** node to your credential and set the **folder ID** (see [Finding your folder ID](#finding-your-folder-id)).
   - In the **Execute Workflow** node, select the **rollout-ingest** workflow so each new/updated file triggers ingestion.
4. In **rollout-ingest**:
   - Attach your **Google Drive**, **OpenAI**, and **Qdrant** (or **Supabase**) credentials to the relevant nodes.
   - Set the **vector store collection name** to match your collection (e.g. `rollout_policy`).
5. In **rollout-policy-ask-workflow**:
   - Attach **OpenAI** and **Qdrant** (or **Supabase**) credentials.
   - Set the same **collection name** as in the ingest workflow.

### 4. Activate

- Activate **rollout-drive-watch** so it runs on the schedule (e.g. every 30–60 minutes).
- Activate **rollout-policy-ask-workflow** so the webhook is live.

## Finding your folder ID

Open your policy folder in Google Drive in the browser. The URL looks like:

`https://drive.google.com/drive/folders/FOLDER_ID`

Use `FOLDER_ID` in the **rollout-drive-watch** Google Drive node (e.g. in the "Folder" or "Search" query).

## Webhook: asking questions

**Endpoint:** `POST /webhook/ask-rollout-policy` (or the path shown in your n8n Webhook node).

**Request body (JSON):**

```json
{
  "message": "What are the current signage rules for the break room?"
}
```

Alternatively you can send `user_message` or `text` instead of `message`.

**Response (200):**

```json
{
  "success": true,
  "answer": "...",
  "sources": [
    { "file_name": "Signage Policy 2024.pdf", "snippet": "..." }
  ]
}
```

On error you get `success: false` and an `error` field; status code 400 for bad request, 502 if the LLM or vector store fails.

## Optional: Structured Governor

To keep answers factual and low-drift (like the diet feedout flow), you can run the Structured Governor on the same server and add an HTTP Request node in **rollout-policy-ask-workflow** that calls `POST http://localhost:5050/govern` with the user message and RAG context, then return the governor's `resolution.answer`. See the Structured Gov README for running the governor and the diet-feedout-ask workflow for the payload shape.

## File format support

| Format | Supported | Notes |
|--------|-----------|--------|
| Google Docs | Yes | Export as HTML/plain text; template uses Markdown conversion where applicable. |
| PDF (text-based) | Yes | Use **Extract From File** on the downloaded binary. |
| DOCX | Yes | Use **Extract From File** (or a DOCX community node) on the downloaded binary. |
| Scanned PDFs | No (v1) | Would require OCR; add later if needed. |

## Reindex / backfill

To index all existing files in the folder once:

1. Run **rollout-drive-watch** once with the "Code" node temporarily changed so `lastRun` is set to a very old date (e.g. `'1970-01-01T00:00:00Z'`), so the Search returns all files.
2. Or create a one-off workflow that lists all files in the folder (no date filter) and executes **rollout-ingest** with that list.

## Troubleshooting

- **No files found:** Check folder ID and that the Google Drive credential has access to that folder. Ensure `lastRun` in the Code node is not in the future.
- **Ingest fails on a file:** Check file type (Google Docs vs binary). For Google Docs, ensure the node is set to export (e.g. HTML/text). For PDF/DOCX, ensure **Extract From File** receives the binary and the correct MIME type.
- **Empty or poor answers:** Confirm the vector store collection has data (run ingest first). Increase "top k" or limit in the Q&A workflow retrieval step. Check that chunk size and overlap in the ingest pipeline suit your policy doc length.

## License

Same as the rest of your n8n Rollout project.
