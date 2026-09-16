# Admin UI — Tambah Kata Baru

Form admin untuk fitur "Tambah Kata" — mengonsumsi API yang dispesifikasikan
di `docs/api/01-api-tambah-kata.md`, dengan konvensi backend di
`docs/api/api-base-stack.md` (envelope response, error code, auth).
Koleksi uji fungsional endpoint-nya ada di repo `http/`.
Alur verifikasi kontribusi (antrean review approve/reject/correct)
didefinisikan di `docs/api/03-api-kontribusi-verifikasi.md` — halaman
UI antreannya menyusul di prompt admin terpisah.

---

## Prompt

```text
Buatkan UI halaman admin "Tambah Kata Baru" untuk aplikasi Kamus Digital
Sambas-Indonesia. Ini adalah form CRUD untuk content management, dipakai
admin/kontributor menambahkan satu entri kosakata lengkap dalam satu
submit.

KONTEKS:
Aplikasi kamus mirip SpanishDictionary.com, tapi untuk bahasa Sambas
(bahasa daerah Kalimantan Barat) ke Indonesia. Skema database sudah
mendukung: kata dasar, dialek, kelas kata (hierarkis), makna/definisi,
terjemahan makna, contoh kalimat, kategori/glosarium, sinonim,
pengucapan, dan sumber data.

STRUKTUR FORM (bagi jadi beberapa section/step):

1. DATA KATA DASAR

   - Kata Sambas / Lemma (text input, wajib)
   - Bahasa (dropdown, default: "Sambas", disabled kalau cuma 1 bahasa)
   - Dialek (dropdown/searchable, opsional — misal "Umum", "Sambas Kota",
     "Sambas Pesisir")
   - Catatan tambahan (textarea, opsional)
   - Jenis entri (dropdown: Kata / Idiom / Peribahasa / Ungkapan,
     default Kata — menentukan label/badge & mengaktifkan relasi
     "kata pembentuk" untuk entri frasa)
2. MAKNA / ARTI (bisa lebih dari satu, tombol "+ Tambah Makna")
   Untuk setiap makna:

   - Kelas kata (dropdown searchable: Nomina, Verba, Adjektiva, dst —
     tampilkan hierarki kalau ada, misal "Verba > Verba Transitif")
   - Definisi konseptual (textarea, wajib)
   - Terjemahan ke Indonesia (text input, wajib) — ini yang tampil
     sebagai arti utama
   - Tipe terjemahan (dropdown: Langsung/Deskriptif/Idiomatik, default
     "Langsung")
   - Urutan tampil (angka, auto-increment tapi bisa diubah manual)
3. CONTOH KALIMAT (nested di dalam tiap makna, tombol "+ Tambah Contoh")

   - Kalimat dalam bahasa Sambas (textarea, wajib)
   - Terjemahan kalimat ke Indonesia (textarea, wajib)
   - Sumber contoh (dropdown: Penutur Asli/Buku/Korpus/Wawancara/Lainnya)
4. KATEGORI / GLOSARIUM (multi-select searchable, dengan opsi
   "+ buat kategori baru" langsung dari dropdown)

   - Contoh tag yang sudah ada: Kekerabatan, Alam, Makanan, dst
5. RELASI KATA (opsional, collapsible section)

   - Setiap item = kata (searchable) + TIPE relasi (dropdown):
     Sinonim / Antonim / Kata pembentuk (khusus entri frasa:
     idiom/peribahasa/ungkapan) / Turunan dari (derived_from)
   - Validasi frontend: "Kata pembentuk" disembunyikan/disabled
     ketika jenis entri = Kata (backend juga menolak — 400)
6. BENTUK TURUNAN (opsional, collapsible section)

   - Daftar bentuk surface milik entri ini (mis. "memakan" milik
     "makan"): teks bentuk + jenis (fleksi/turunan/alternatif/
     pengulangan) + afiks terstruktur (tipe: awalan/akhiran/afiks
     ganda/pengulangan + nilai mis. "me-")
7. PENGUCAPAN (opsional, collapsible section)

   - Notasi IPA (text input)
   - Audio upload (placeholder untuk fitur nanti, tampilkan sebagai
     "Coming Soon" atau upload file biasa)

AKSI FORM:

- Tombol "Simpan sebagai Draft" → status 'draft'
- Tombol "Simpan & Publikasikan" → status 'published'
  (PERHATIAN: hasil AKHIR tergantung role — lihat alur status di
  INTEGRASI BACKEND; untuk contributor tombol ini menghasilkan
  'pending_review', tampilkan itu di toast konfirmasinya)
- Validasi: field wajib ditandai, tampilkan error inline, jangan biarkan
  submit kalau makna/lemma kosong

PERILAKU UX:

- Bagian "Makna" dan "Contoh Kalimat" bisa ditambah/dihapus dinamis
  (accordion/card list dengan tombol tambah & hapus)
- Autosave draft setiap beberapa detik (opsional, sebutkan sebagai nice-
  to-have)
- Setelah submit sukses, tampilkan toast konfirmasi + redirect ke daftar
  kata atau halaman detail kata yang baru dibuat (id = ULID string)

GAYA VISUAL:

- Bersih, mirip dashboard admin modern (referensi: Notion database view
  atau form builder seperti Airtable)
- Gunakan card/section terpisah per bagian data biar tidak terasa
  seperti satu form panjang yang membingungkan
- Responsive, prioritaskan desktop tapi tetap bisa dipakai di tablet

INTEGRASI BACKEND (WAJIB — kontrak di 01-api-tambah-kata.md):

1. AUTENTIKASI (00-api-auth.md)
   - Halaman untuk role: admin, editor, contributor, root, reviewer
   - access_token disimpan di memory (BUKAN localStorage), dikirim sebagai
     header Authorization: Bearer <token>
   - refresh_token otomatis lewat httpOnly cookie — 401 TOKEN_EXPIRED →
     panggil POST /api/v1/auth/refresh → retry request; gagal → redirect
     ke halaman login

2. SUBMIT FORM
   POST /api/v1/admin/words  (rate limit 30 req/menit per user)
   Body: language_id, dialect_id?, lemma, notes?, meanings[] (word_class_id,
   definition, order_index, translations[], examples[]), category_ids[],
   synonym_word_ids[], pronunciation?, status
   — semua *_id adalah ULID string pilihan dari dropdown, BUKAN input bebas

3. ALUR STATUS (words.status — Section 22 approval gate)
   - 'draft' → tombol Simpan Draft (tidak tayang, tidak masuk antrean)
   - 'published' + role admin/editor/root/reviewer → langsung tayang,
     is_verified true (self-verified)
   - 'published' + role contributor → backend simpan 'pending_review'
     (masuk antrean review — TIDAK tayang sampai disetujui verifikator)
     — toast harus jujur menyebut status akhir dari response (bukan
     asumsi), karena data.status adalah sumber kebenaran
   - Keputusan antrean (approve/reject dengan alasan/correct dengan
     koreksi verifikator) dikonsumsi halaman antrean review terpisah —
     kontrak API-nya di 03-api-kontribusi-verifikasi.md

4. HANDLE RESPONSE (envelope standar Section 13)
   - Sukses: { success: true, data: { word_id, lemma, word_type, status,
     is_verified, created_at, warnings? } } → toast + redirect
   - data.warnings (duplikat lemma serupa) → tampilkan sebagai warning
     banner/toast kuning SETELAH sukses — bukan blokir
   - 400 VALIDATION_ERROR: details[] = [{ field, message }] → PETAKAN
     field ke error inline di form (field bertitik nested mis.
     "meanings.0.definition" → error di makna ke-0)
   - 401/403: sesuai alur auth; 403 FORBIDDEN berarti role tidak
     diizinkan
   - 429 RATE_LIMITED: tampilkan pesan + hitung ulang dari header
     Retry-After
   - 500 INTERNAL_ERROR: pesan generik, jangan tampilkan detail teknis

   INFO TAMBAH: halaman detail kata (web) menampilkan bagian
   "muncul dalam" (appears_in) — peribahasa/idiom yang memakai kata
   itu sebagai komponen; data berasal dari GET /words/:id field
   appears_in (relasi invers, otomatis).

5. DATA DROPDOWN (semua GET publik; reference data kecil tanpa limit,
   search endpoint memakai cursor-based pagination Section 13)
   - GET /api/v1/languages → dropdown Bahasa (flat list, data kecil)
   - GET /api/v1/dialects?language_id=… → dropdown Dialek (refresh saat
     bahasa berubah, flat list)
   - GET /api/v1/word-classes → dropdown Kelas Kata (tampilkan hierarki
     parent, flat list)
   - GET /api/v1/categories → multi-select Kategori (flat list)
   - GET /api/v1/words/search?q=…&limit=20&cursor=… (debounce 300ms)
     → pilih Sinonim/Antonim; response envelope list + meta cursor:
       meta: { limit, next_cursor: ULID|null, has_more: bool }
     UX: tampilkan lemma + bahasa di opsi; jika has_more=true tampilkan
     tombol "Muat lagi" di bawah daftar opsi yang onclick memanggil
     ulang endpoint dengan cursor=next_cursor dan menggabungkan hasil
     (append bukan replace). Tidak ada pagination page numbers.
```
