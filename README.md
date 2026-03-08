## Droplets

Droplets is a small RAG (Retrieval-Augmented Generation) playground built with the Next.js App Router.  
You can upload documents, have them chunked and embedded into Supabase, and then ask natural‑language questions over your personal corpus using OpenAI.

---

## Features

- **Document upload & processing**
  - Upload `PDF`, `DOCX`, or `TXT` files.
  - Server extracts raw text (via `pdf2json` and `mammoth` or simple UTF‑8 parsing).
  - Text is split into overlapping chunks using LangChain’s `RecursiveCharacterTextSplitter`.
  - Each chunk is embedded with `text-embedding-3-small` and stored in Supabase.

- **RAG search**
  - `/api/search` creates an embedding for your query.
  - Supabase RPC `match_documents` performs vector similarity search over stored chunks.
  - Top matches are passed as context to `gpt-4o-mini`, which generates the final answer.

- **Document management UI**
  - `Documents` page lists uploaded documents (name, type, size, chunk count, upload date).
  - Preview PDFs inline, or view extracted text for any document.
  - Download original files and delete documents (removes both DB rows and stored file).

- **Modern UI**
  - Next.js App Router with React 19.
  - Tailwind CSS 4 for styling.
  - Simple top navigation (`Search` / `Documents`), dark‑mode‑friendly styles.

---

## Tech Stack

- **Frontend**: Next.js `16.1.6` (App Router), React `19`, TypeScript
- **Styling**: Tailwind CSS `^4`
- **Backend / APIs**:
  - Next.js Route Handlers under `app/api`
  - Supabase (Postgres + Storage)
  - OpenAI (chat + embeddings)
- **RAG tooling**:
  - `@supabase/supabase-js`
  - `langchain` + `@langchain/textsplitters`
  - `pdf2json`, `mammoth`

---

## High‑Level Architecture

- **Supabase Postgres**
  - Table `documents` that stores:
    - `content` – text chunk
    - `metadata` – JSON (file name, type, size, upload date, document UUID, etc.)
    - `embedding` – vector for similarity search
  - RPC function `match_documents` that takes a query embedding and returns the most similar chunks.

- **Supabase Storage**
  - Bucket holding original uploaded files (PDF/DOCX/TXT).
  - Files are stored under a generated `documentId` + extension (e.g. `uuid.pdf`).

- **Next.js API Routes**
  - `/api/upload` – upload & ingest a new document.
  - `/api/documents` – list documents, fetch a document (and its full text), stream file contents, or delete a document.
  - `/api/search` – perform RAG query over uploaded content.

- **UI**
  - `app/page.tsx` – search experience over ingested documents.
  - `app/documents/page.tsx` – document list, preview, download, and delete.
  - `UploadModal` – handles file selection and calls `/api/upload`.
  - `PDFViewerModal` – previews PDFs in an iframe or shows plain extracted text.

---

## Prerequisites

- **Node.js**: 18+ (20+ recommended)
- **Package manager**: `pnpm` (project was created with `pnpm`, but `npm`/`yarn` work too)
- **Supabase project** with:
  - Postgres database
  - Storage bucket for documents
  - `pgvector` (or equivalent vector type) enabled
- **OpenAI API key** with access to:
  - `text-embedding-3-small`
  - `gpt-4o-mini`

---

## Environment Variables

Copy `env.sample` to `.env.local` and fill in the values:

```bash
cp env.sample .env.local
```

Required variables:

| Name                                  | Description                                                                                 |
| ------------------------------------- | ------------------------------------------------------------------------------------------- |
| `NEXT_PUBLIC_SUPABASE_URL`           | Supabase project URL                                                                        |
| `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_DEFAULT_KEY` | Supabase anon (public) key – used client & server side where appropriate          |
| `SUPABASE_SERVICE_ROLE_KEY`          | Supabase service role key – **server‑side only**, used for privileged storage operations   |
| `OPENAI_API_KEY`                     | OpenAI API key                                                                              |

Ensure `.env.local` is **never committed** to version control.

---

## Supabase Setup

### 1. Enable vector extension

In the Supabase SQL editor:

```sql
create extension if not exists vector;
```

### 2. Create `documents` table

This matches how the app writes data in `app/api/upload/route.ts` and reads it in `app/api/documents/route.ts` and `/api/search`:

```sql
create table if not exists documents (
  id bigserial primary key,
  content text,
  metadata jsonb,
  embedding vector(1536) -- must match OpenAI embedding dimension
);
```

### 3. RPC: `match_documents`

Create a function that performs vector similarity search over `documents.embedding`. One example implementation:

```sql
create or replace function match_documents(
  query_embedding vector(1536),
  match_threshold float,
  match_count int
)
returns table (
  content text,
  metadata jsonb,
  similarity float
)
language plpgsql
as $$
begin
  return query
  select
    d.content,
    d.metadata,
    1 - (d.embedding <#> query_embedding) as similarity
  from documents d
  where 1 - (d.embedding <#> query_embedding) >= match_threshold
  order by d.embedding <#> query_embedding
  limit match_count;
end;
$$;
```

The code in `/api/search` calls:

- `supabase.rpc("match_documents", { query_embedding, match_threshold, match_count })`

so the function name and parameter names must match.

### 4. Storage bucket

Create a storage bucket to hold uploaded files, for example `documents`.

The upload route (`app/api/upload/route.ts`) and document API (`app/api/documents/route.ts`) reference storage buckets; if you change the bucket name, update those files accordingly so uploads, previews, and downloads all point at the same bucket.

### 5. Security / RLS

- The upload route uses the **service role key** for storage writes to avoid RLS issues.
- The documents table might use RLS depending on your project; if you enable RLS, ensure your policies allow:
  - Inserts from the server‑side upload route.
  - Reads from the search and documents APIs appropriate to your security model.

---

## Running the App Locally

Install dependencies:

```bash
pnpm install
# or
npm install
```

Ensure `.env.local` is configured (see above), then start the dev server:

```bash
pnpm dev
# or
npm run dev
```

By default the app runs at `http://localhost:3000`.

---

## Usage

### 1. Upload documents

1. Navigate to the **Documents** tab (`/documents`).
2. Click **Upload Document**.
3. Select a `PDF`, `DOCX`, or `TXT` file.
4. Wait for the upload & processing to complete:
   - The file is saved to Supabase Storage.
   - Text is extracted and split into chunks.
   - Embeddings are generated and stored in the `documents` table.

You should then see the document appear in the table with chunk count and metadata.

### 2. Explore documents

From the **Documents** page:

- **Preview PDFs** – opens a modal with an iframe preview when possible.
- **View extracted text** – in the same modal, switch to the **Content** tab (or auto‑selected for non‑PDFs).
- **Download** – download the original file (where available).
- **Delete** – removes all chunks from Postgres and the file from Storage (if present).

### 3. Ask questions (RAG search)

1. Navigate to the **Search** tab (`/`).
2. Type a question in the text area (e.g. *“What does the company travel policy say about reimbursements?”*).
3. Click **Search** or press **Cmd/Ctrl + Enter**.
4. The app will:
   - Generate an embedding for your query.
   - Use Supabase `match_documents` to find similar chunks.
   - Call OpenAI `gpt-4o-mini` with the retrieved context and your question.
5. The answer is shown along with a list of **Sources**, each including:
   - `metadata.source` / `metadata.file_name`
   - A preview snippet of the matching chunk content.

---

## API Overview

### `/api/upload` – POST (multipart/form-data)

- **Body**: form data with a single `file` field.
- **Behavior**:
  - Stores file in Supabase Storage.
  - Extracts text and chunks it.
  - Generates embeddings and inserts rows into `documents` table with `metadata` and `embedding`.
- **Response** (success):

```json
{
  "success": true,
  "documentId": "uuid",
  "fileName": "example.pdf",
  "chunks": 42,
  "textLength": 12345,
  "fileUrl": "https://..."
}
```

### `/api/documents` – GET

- **Without query params**: returns a deduplicated list of documents (using `metadata.document_id`).
- **With `?id=<documentId>`**: returns full text and metadata for that document.
- **With `?id=<documentId>&file=true&view=true`**: streams the original file, optionally inline for PDFs.

### `/api/documents` – DELETE

- **With `?id=<documentId>`**:
  - Deletes all chunks for the given `document_id` from the `documents` table.
  - Attempts to remove the corresponding file from Supabase Storage.

### `/api/search` – POST (JSON)

- **Body**:

```json
{ "query": "your question here" }
```

- **Response**:

```json
{
  "answer": "LLM-generated answer...",
  "sources": [
    {
      "content": "chunk text...",
      "metadata": {
        "source": "file.pdf",
        "document_id": "uuid",
        "file_name": "file.pdf",
        "chunk_index": 0,
        "total_chunks": 42,
        "file_url": "https://...",
        "file_path": "uuid.pdf",
        "upload_date": "..."
      },
      "similarity": 0.87
    }
  ]
}
```

---

## Development Notes

- The app uses `"use client"` components for the main pages and modals to simplify client‑side state management.
- Embeddings are stored as JSON strings in the `embedding` column in this codebase; if you instead use a native `vector` type, you can adapt the insert and RPC logic accordingly.
- Error messages from Supabase Storage that mention RLS are surfaced from `/api/upload` to help debug permission issues.

---

## Deployment

You can deploy this project to any Next.js‑compatible platform (e.g. Vercel, a custom Node server, or similar). Ensure that:

- Environment variables from `.env.local` are configured in the hosting provider.
- The Supabase project (DB table, RPC function, and Storage bucket) is available from the deployed environment.
- The Node runtime version matches or exceeds your local development version.

For Vercel deployments, select the **Next.js** framework preset and copy your environment variables into the project settings.

