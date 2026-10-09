# Architecture

## Place inside the ERP

The ERP uses one database per module (Accounting, Inventory, etc.) with no integration between modules.
The assistant adds **one new database** (`rag_db`, image `pgvector/pgvector:pg16`) for the index and chat
history. The transactional databases' schemas are untouched, and the production Postgres image does not
need to change (switching an Alpine image to a Debian one on the same volume risks collation issues).

```text
app/accounting/asisten/
  page.tsx            chat page (server component): conversation history + chat
  kirim/route.ts      multipart POST (message + receipt photo ≤5 MB → OCR)
src/modules/assistant/
  agent.ts            LLM ↔ tools loop, history, action confirmation, UI view model
  llm.ts              OpenAI-compatible client
  prompt.ts           dynamic system prompt (name, role, permissions, date in WIB)
  tools-read.ts       read tools
  tools-write.ts      write tools (prepare/execute)
  retrieval.ts        hybrid search + RRF
  indexer.ts          index synchronization
  embeddings.ts       local embeddings (optional)
  actions.ts          Save / Cancel server actions
docs/panduan-asisten.md   ERP user guide → also indexed as knowledge
scripts/assistant-reindex.ts
```

## rag_db schema

| Table | Purpose |
|---|---|
| `AssistantDocument` | `source` (ACCOUNT/JOURNAL/ASSET/GUIDE), `sourceId`, `title`, `content`, `metadata`, `contentHash`, `embedding vector(384)`, `tsv tsvector` (*generated column*) |
| `AssistantSyncState` | per-source sync cursor: `(lastSyncedAt, lastId)` |
| `AssistantConversation` | a conversation owned by one username |
| `AssistantMessage` | user/assistant/tool messages, `toolCalls`, attachments, `kind` (OCR context, action result) |
| `AssistantAction` | proposed write: `args`, validated `payload`, `preview`, `status` PENDING → EXECUTING → EXECUTED/FAILED, or CANCELLED |

Indexes: GIN on `tsv`, trigram GIN on `title`, HNSW (cosine) on `embedding`.

## Retrieval

```text
query ─┬─ tokens → tsquery AND  (weight 1.3)  ┐
       ├─ tokens → tsquery OR   (weight 0.6)  │
       ├─ word_similarity(query, title)  0.8   ├─► RRF (k=60) ─► top-k
       └─ embed("query: …") <=> embedding 1.2 ┘
```

- Simple tokenization (letters/digits, ≥2 characters), which keeps tsquery safe from special characters.
- `simple` configuration (no stemming), because the data mixes Indonesian, English, and many codes.
- A `leafAccountsOnly` filter finds accounts that can actually be used in journal lines.
- Journal hits are re-checked against the transactional DB so deleted journals never show up.

## Index synchronization

| Trigger | Scope |
|---|---|
| Before answering a chat (throttled to 30 seconds) | up to 300 changed rows per source, cursor `(updatedAt, id)` |
| After a successful assistant action | the entity just written (immediately) |
| `npm run assistant:reindex` (daily cron) | all documents + remove orphans + fill missing embeddings |

Documents with an unchanged `contentHash` are skipped, so embeddings are computed only for what changed.

## Agent loop

1. Save the user message (plus an OCR context message if there is an attachment).
2. Sync the index (incremental).
3. Build messages: system prompt + up to 40 history messages (tool_calls without complete answers are dropped).
4. Call the LLM with tools **filtered by role** (Zod → JSON Schema).
5. For each tool call: validate arguments (Zod) → run. Validation/permission errors are returned to the
   LLM as text (not exceptions) so it can explain them to the user.
6. Write tool → save a PENDING `AssistantAction` and return the status `MENUNGGU_KONFIRMASI_USER`
   ("awaiting user confirmation").
7. At most 8 rounds; the final answer is saved.

Confirmation: `updateMany where status = PENDING → EXECUTING` (atomic, safe against double clicks) →
`execute()` re-checks permissions → ERP service → `[Asisten]` audit entry → index the entity → result message.
