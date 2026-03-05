# Rollout RAG – internal notes

## Folder ID

In **rollout-drive-watch.json**, open the "List new/updated files" (Google Drive) node and replace `YOUR_POLICY_FOLDER_ID` in the query string with your actual Google Drive folder ID.

To find it: open the policy folder in the browser and copy the ID from the URL:
`https://drive.google.com/drive/folders/FOLDER_ID`

## Chunking

The ingest workflow uses the **Default Data Loader** with "Simple" text splitting (Recursive Character Text Splitter, ~1000 chars chunk size, 200 overlap by default). To change chunk size or overlap, edit the "Chunk and load" node in **rollout-ingest.json** and switch to a custom text splitter if your n8n version supports it.

For policy docs, 500–1000 character chunks with 100–200 overlap usually work well so sentences are not split in the middle.

## Vector store collection

Use the same collection name (`rollout_policy` by default) in:

- **rollout-ingest.json** – "Insert into Vector Store" node
- **rollout-policy-ask-workflow.json** – "Retrieve from Vector Store" node

Collection dimension must match the embedding model (e.g. 1536 for `text-embedding-3-small`). Create the collection in Qdrant (or Supabase) with that dimension before the first ingest.

## Connecting Embeddings to Vector Store

In both the ingest and the Q&A workflows, the **OpenAI Embeddings** node must be connected to the **Vector Store** node (as its embedding input). After import, if retrieval or insert fails, check in the n8n editor that the Embeddings node is connected to the Vector Store.

## Execute Workflow (watch → ingest)

In **rollout-drive-watch.json**, the "Run Ingest for file" node must point to the **Rollout Ingest** workflow (by name or ID). After importing both workflows, set "Workflow" in that node to the ingest workflow so each new/updated file triggers one run of the ingest with that file as input.
