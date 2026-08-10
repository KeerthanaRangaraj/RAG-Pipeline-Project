# MedRAG — Medical Product RAG Pipeline

A retrieval-augmented search app over a medical-product document corpus
(PDF, DOCX, XLSX, CSV, images), built to scale to ~500 files / 5GB and
deployed on Vercel.

## Architecture

```
Browser ──upload──> Vercel Blob (raw files)
                         │
                         ▼
                 /api/ingest (per file)
       parse → chunk → embed (Voyage) → upsert
                         │
                         ▼
              Supabase Postgres + pgvector
                (chunks, files, chat_messages)
                         │
        query embedding  │  vector search (match_chunks RPC)
                         ▼
                  /api/chat → Claude (streamed, cited answer)
```

- **Raw files** live in Vercel Blob, not in the database.
- **Vectors + metadata** live in Supabase (pgvector). One Postgres database
  covers file status, chunks, embeddings, and chat history — no separate
  vector DB service to manage.
- **Embeddings**: Voyage AI `voyage-3` (1024-dim). Swap in
  `voyage-multimodal-3` in `lib/embeddings.ts` if you want to embed product
  images directly instead of captioning them first.
- **Generation**: Claude, grounded strictly in retrieved chunks, streamed to
  the UI with inline citations and a source-relevance strip.
- **Images**: captioned by Claude vision on ingest (product name, model/lot
  numbers, warnings, legible label text) — the caption is what gets chunked
  and embedded, so a photo of a label becomes searchable text.

## Why ingestion is per-file, not one big batch job

Vercel serverless/edge functions have execution time ceilings (up to 300–800s
depending on plan). Parsing and embedding 5GB across 500 files can't happen
in a single request. This app ingests **one file per request**:

- The upload page queues files client-side with limited concurrency
  (`components/FileUploader.tsx`), good for tens of files interactively.
- For the full 500-file corpus, run `scripts/bulk-ingest.ts` locally/in CI —
  it walks a folder, uploads to Blob, and calls `/api/ingest` per file with
  retry and concurrency control, independent of any single function's time
  limit.

### Scaling ingestion further

If you need durability across restarts, automatic retries, and a dashboard
(recommended once you're running this on a schedule or ingesting
continuously), move the ingestion step from `scripts/bulk-ingest.ts` into an
[Inngest](https://www.inngest.com) function: same per-file steps
(download → parse → chunk → embed → upsert), but each step is checkpointed
and retried independently, and a stalled step doesn't restart the whole file.
Inngest deploys alongside this app on Vercel with no separate infra.

## Setup

1. **Supabase**: create a project, then run `supabase/schema.sql` in the SQL
   editor. This enables `pgvector` and creates `files`, `chunks`,
   `chat_messages`, and the `match_chunks` search function.
2. **Voyage AI**: create an API key at dash.voyageai.com.
3. **Anthropic**: create an API key at console.anthropic.com.
4. **Vercel Blob**: in your Vercel project settings, enable Blob storage —
   this populates `BLOB_READ_WRITE_TOKEN` automatically.
5. Copy `.env.example` to `.env.local` and fill in the four services above.

## Local development

```bash
npm install
npm run dev
```

Generate the starter sample dataset (35 synthetic medical-product files —
DOCX spec sheets, XLSX lot-tracking sheets, CSV inventory exports, and PNG
product labels across 5 categories):

```bash
python3 scripts/generate_sample_data.py
```

Upload `sample-data/` through the Library page at `/upload`, or run it
through the bulk script once deployed:

```bash
APP_URL=http://localhost:3000 npx tsx scripts/bulk-ingest.ts ./sample-data
```

## Deploying to Vercel

```bash
npm install -g vercel
vercel link
vercel env pull .env.local        # after setting env vars in the dashboard, or:
vercel env add SUPABASE_URL
vercel env add SUPABASE_SERVICE_ROLE_KEY
vercel env add VOYAGE_API_KEY
vercel env add ANTHROPIC_API_KEY
vercel --prod
```

`vercel.json` extends `/api/ingest` to 300s and `/api/chat` to 60s — both
require a Pro plan or higher for durations beyond 10s (Hobby plan default).

Once deployed, point the bulk script at production:

```bash
APP_URL=https://your-app.vercel.app npx tsx scripts/bulk-ingest.ts ./your-500-files
```

## Scaling to the real 500-file / 5GB corpus

- **File size limits**: `app/api/blob-upload/route.ts` caps uploads at 50MB
  each — raise `maximumSizeInBytes` if any single file is larger.
- **Cost shape**: embeddings (Voyage) are the cheapest line item; Claude
  vision captioning on images and chat generation are the larger ones.
  Budget roughly by (number of images × caption cost) + (chat volume ×
  generation cost) — text/table embedding cost is comparatively negligible
  at this corpus size.
- **Vector index**: `ivfflat` in `schema.sql` is tuned for tens of thousands
  of chunks (a 500-file/5GB corpus lands well within that). If you later grow
  10x+, revisit `lists` in the index or move to `hnsw` (supported by pgvector
  0.5+ on Supabase).
- **Categories**: the `category` field on `files`/`chunks` supports filtered
  search (`/api/chat` already accepts an optional `category`) — useful once
  the library spans many product lines.

## Notes

- Answers are grounded only in retrieved chunks and explicitly scoped as
  product/catalog information, not medical or treatment advice — see the
  system prompt in `lib/anthropic.ts`.
- `chat_messages` table is created but not yet wired into `/api/chat` —
  add a session id and insert user/assistant turns there if you want
  persistent chat history across page loads.
