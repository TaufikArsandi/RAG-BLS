# Keputusan Desain & Pelajaran

## 1. Hybrid: RAG untuk mencari, tool untuk menghitung

Data ERP sebagian besar terstruktur. RAG vektor murni buruk untuk "berapa saldo akun X per tanggal Y"
karena angka harus dihitung dari ribuan baris, bukan "ditemukan". Maka:

- **Mencari** (akun yang cocok, jurnal yang mirip, panduan) → retrieval hybrid.
- **Menghitung** (saldo, laporan) → memanggil fungsi laporan ERP yang sudah ada.

Akibatnya angka yang dijawab Asisten selalu sama dengan halaman ERP. Ini diuji: hasil tool buku besar
dibandingkan langsung dengan fungsi service untuk akun dengan mutasi terbanyak.

## 2. Tools memanggil service yang sudah ada

Alternatifnya adalah membiarkan LLM menulis SQL. Opsi itu ditolak:

- Aturan bisnis (akun induk tidak boleh dipakai, debit = kredit, periode tertutup, nomor jurnal
  otomatis, posting atomik) sudah ada di service. Menduplikasinya berarti dua sumber kebenaran.
- SQL bebas dari LLM sulit diamankan per peran.

## 3. Dua langkah untuk setiap tulis

`prepare` (validasi + pratinjau, tanpa efek samping) dipisah dari `execute` (setelah klik user).
Payload tervalidasi (id akun hasil resolusi kode, dsb.) disimpan di `AssistantAction`, jadi yang
dieksekusi persis yang dilihat user di kartu. Data tidak diambil ulang dari teks LLM.

Detail pengaman:
- Klaim aksi atomik (`updateMany ... where status = PENDING`) mencegah jurnal ganda saat klik ganda.
- Aksi milik user lain tidak bisa dikonfirmasi.
- Usulan kedaluwarsa setelah 24 jam.
- Lampiran nota hanya boleh URL yang memang diunggah di percakapan yang sama.
- Hapus/void jurnal sengaja **tidak** disediakan lewat chat.

## 4. Index di database terpisah

`rag_db` bisa dibuang dan dibangun ulang kapan saja. Riwayat chat tidak mencampuri DB transaksi,
dan image Postgres produksi tidak perlu diganti untuk pgvector.

## 5. Embedding lokal, opsional

`multilingual-e5-small` (384 dimensi) berjalan di proses Node. Keuntungannya: tanpa biaya per-token
dan teks ERP tidak dikirim ke layanan embedding. Kalau model gagal dimuat, pencarian otomatis jatuh
ke full-text + trigram tanpa error di sisi user. Mode ini juga dipakai untuk test dan CI.

## 6. Dokumen akun memuat jalur induk

COA punya banyak sub akun bernama identik di project/tim berbeda. Dokumen akun ditulis seperti:

```text
Akun 73045xxxxx BIAYA BBM KENDARAAN. Tipe: Beban/Biaya.
Induk: 7000000000 BEBAN USAHA › 7300000000 BEBAN USAHA <TIM> › 73045xxxxx KK OPERASIONAL <TIM> <KOTA>.
Akun detail (sub akun) — bisa dipakai di baris jurnal.
```

System prompt mewajibkan LLM bertanya ke user bila ada lebih dari satu kandidat yang masuk akal.

## 7. Bug yang ditemukan saat membangun (dan cara menemukannya)

| Bug | Gejala | Perbaikan | Cara ketahuan |
|---|---|---|---|
| Paginasi reindex mencampur urutan `updatedAt` & kursor `id` | 2.088 dari 21.014 jurnal ter-index | Mode paginasi eksplisit (urut `id`) | Hitung dokumen per sumber setelah reindex |
| Kursor incremental hanya `updatedAt` | Baris ber-timestamp sama (hasil import massal) bisa terlewat di batas batch | Kursor `(updatedAt, id)` | Review kode sebelum test |
| Full-text OR-prefix | "kas ho" → "kasbon" mendominasi | Jalur AND berbobot 1.3, OR 0.6 | Uji kueri manual |
| Riwayat tool-call parsial | API menolak percakapan bila proses mati di tengah putaran | Sanitasi riwayat + unit test | Review adversarial |
| Login menolak user Inventaris untuk `/accounting/asisten` | Halaman Asisten tidak terbuka untuk peran Inventaris | Pengecualian khusus path Asisten | E2E Playwright per peran |
| Kode akun dirender rata kanan | Kolom kode terlihat seperti nominal | Heuristik angka: kode polos ≥7 digit tetap rata kiri | Screenshot E2E |

## 8. Pengujian tanpa akses LLM

Selama pengembangan, API LLM tidak bisa dijangkau dari lingkungan build. Solusinya:

- **Test integrasi** memakai `ChatFn` tiruan berisi skrip tool call. Seluruh jalur di belakangnya
  (validasi Zod, service ERP, audit, index, konfirmasi) tetap nyata terhadap Postgres berisi data migrasi.
- **E2E browser** memakai server HTTP kecil yang meniru endpoint `/chat/completions`. Klien LLM
  produksi dipakai apa adanya dengan `LLM_BASE_URL` diarahkan ke server tiruan.

Yang **belum** teruji: kualitas jawaban model asli (pemilihan akun, gaya bahasa). Ini perlu diuji
dengan pertanyaan nyata tim setelah API key dipasang.
