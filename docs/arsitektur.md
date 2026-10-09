# Arsitektur

## Letak di dalam ERP

ERP memakai satu database per modul (Accounting, Inventaris, dll.) tanpa integrasi antar modul.
Asisten menambah **satu database baru** (`rag_db`, image `pgvector/pgvector:pg16`) untuk index dan riwayat
chat. Database transaksi tidak diubah skemanya, dan tidak perlu mengganti image Postgres produksi
(mengganti image Alpine → Debian pada volume yang sama berisiko masalah collation).

```text
app/accounting/asisten/
  page.tsx            halaman chat (server component): riwayat percakapan + chat
  kirim/route.ts      POST multipart (pesan + foto nota ≤5 MB → OCR)
src/modules/assistant/
  agent.ts            loop LLM ↔ tools, riwayat, konfirmasi aksi, view model UI
  llm.ts              klien OpenAI-compatible
  prompt.ts           system prompt dinamis (nama, peran, hak akses, tanggal WIB)
  tools-read.ts       tools baca
  tools-write.ts      tools tulis (prepare/execute)
  retrieval.ts        pencarian hybrid + RRF
  indexer.ts          sinkronisasi index
  embeddings.ts       embedding lokal (opsional)
  actions.ts          server actions Simpan / Batal
docs/panduan-asisten.md   panduan pemakaian ERP → ikut di-index sebagai pengetahuan
scripts/assistant-reindex.ts
```

## Skema rag_db

| Tabel | Fungsi |
|---|---|
| `AssistantDocument` | `source` (ACCOUNT/JOURNAL/ASSET/GUIDE), `sourceId`, `title`, `content`, `metadata`, `contentHash`, `embedding vector(384)`, `tsv tsvector` (*generated column*) |
| `AssistantSyncState` | kursor sinkronisasi per sumber: `(lastSyncedAt, lastId)` |
| `AssistantConversation` | percakapan milik satu username |
| `AssistantMessage` | pesan user/assistant/tool, `toolCalls`, lampiran, `kind` (konteks OCR, hasil aksi) |
| `AssistantAction` | usulan tulis: `args`, `payload` tervalidasi, `preview`, `status` PENDING → EXECUTING → EXECUTED/FAILED, atau CANCELLED |

Index: GIN pada `tsv`, GIN trigram pada `title`, HNSW (cosine) pada `embedding`.

## Retrieval

```text
query ─┬─ token → tsquery AND  (bobot 1.3)  ┐
       ├─ token → tsquery OR   (bobot 0.6)  │
       ├─ word_similarity(query, title)  0.8 ├─► RRF (k=60) ─► top-k
       └─ embed("query: …") <=> embedding 1.2┘
```

- Tokenisasi sederhana (huruf/angka ≥2 karakter) — aman dari karakter khusus tsquery.
- Konfigurasi `simple` (tanpa stemming) karena isi data campuran Indonesia/Inggris dan banyak kode.
- Filter `leafAccountsOnly` untuk mencari akun yang boleh dipakai di jurnal.
- Hasil jurnal dicek ulang ke DB transaksi supaya jurnal yang sudah dihapus tidak muncul.

## Sinkronisasi index

| Pemicu | Cakupan |
|---|---|
| Sebelum menjawab chat (throttle 30 detik) | maks. 300 baris berubah per sumber, kursor `(updatedAt, id)` |
| Setelah aksi Asisten berhasil | entitas yang baru ditulis (langsung) |
| `npm run assistant:reindex` (cron harian) | semua dokumen + hapus yang yatim + isi embedding yang kosong |

Dokumen dengan `contentHash` sama dilewati sehingga embedding hanya dihitung untuk yang berubah.

## Loop agent

1. Simpan pesan user (+ pesan konteks OCR bila ada lampiran).
2. Sinkron index (incremental).
3. Susun pesan: system prompt + ≤40 pesan riwayat (tool_calls tanpa jawaban lengkap dibuang).
4. Panggil LLM dengan tools yang **disaring per peran** (Zod → JSON Schema).
5. Untuk tiap tool call: validasi argumen (Zod) → jalankan. Error validasi/hak akses dikembalikan
   sebagai teks ke LLM (bukan exception), supaya LLM bisa menjelaskan ke user.
6. Tool tulis → simpan `AssistantAction` PENDING, kembalikan status `MENUNGGU_KONFIRMASI_USER`.
7. Maks. 8 putaran; jawaban akhir disimpan.

Konfirmasi: `updateMany where status = PENDING → EXECUTING` (atomik, aman dari klik ganda) →
`execute()` cek ulang hak akses → service ERP → audit `[Asisten]` → index entitas → pesan hasil.
