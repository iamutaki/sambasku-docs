# Admin UI - Edit Kata

Form admin untuk fitur "Edit Kata" (update kata yang sudah ada) - mengonsumsi
API yang dispesifikasikan di `docs/api/05-api-edit-kata.md`, dengan konvensi
backend di `docs/api/api-base-stack.md` (envelope response, error code, auth,
approval gate) dan konvensi frontend di `docs/admin/admin-base-stack.md`
(clean architecture 3 lapis, cursor list, antd v6, responsif Section 17).
Koleksi uji fungsional endpoint-nya ada di repo `http/`
(`http/word/get-admin-word.bru`, `http/word/update-word.bru`).

Status fondasi saat dokumen ini dibuat (JANGAN diduplikasi):

- Backend edit kata **SUDAH LENGKAP**: `GET /api/v1/admin/words/:id`
  (prefill semua status) + `PUT /api/v1/admin/words/:id` (full replace),
  use case `UpdateWordUseCase`, audit `update`, preserve `is_corrected` -
  lihat `docs/api/05-api-edit-kata.md`.
- Frontend admin **BELUM ADA**: route hanya `/words` (list) dan `/words/new`
  (create). Tombol "Detail" di `words-page.tsx:76` masih `message.info(...)`
  placeholder. Fitur ini menambahkan halaman edit `/words/:id/edit`.
- Form create SUDAH di-refactor jadi blok bersama
  `src/features/words/presentation/word-form-blocks.tsx`
  (`MeaningFields`, `RelatedWordItem`, `InlineWordEditor`, opsi builder) -
  dipakai ulang DI SINI tanpa duplikasi (aturan cross-feature di admin-base
  stack Section 2).

---

## Prompt

```text
Buatkan UI halaman admin "Edit Kata" untuk aplikasi Kamus Digital
Sambas-Indonesia. Ini form CRUD untuk content management, dipakai
verifikator (admin/editor/root/reviewer) memperbaiki satu entri kosakata
yang SUDAH ADA - pengganti pola lama "hapus lalu buat ulang".

KONTEKS:
Form ini 95% sama dengan "Tambah Kata Baru" (01-admin-tambah-kata.md) -
karena PUT /api/v1/admin/words/:id menerima body yang SAMA PERSIS dengan
POST create (full replace, bukan PATCH). Perbedaannya HANYA:

1. Kata di-load dulu dari GET /api/v1/admin/words/:id (prefill form dari
   data yang sudah tersimpan, SEMUA status: draft/pending_review/
   published/rejected).
2. Submit memakai PUT /api/v1/admin/words/:id dengan body LENGKAP hasil
   form (field yang tidak disentuh TETAP ikut dikirim - full replace).
3. "Kata baru (buat sekaligus)" untuk relasi/disposisi TIDAK TERSEDIA
   (Form B ditolak backend) - relasi hanya bisa ke kata yang SUDAH ada.
4. Route terpisah /words/:id/edit - buka dari tombol "Ubah" di daftar
   kata, atau tebakan id (deep link).

JANGAN buat ulang blok form - pakai word-form-blocks.tsx yang sama
persis dengan create-word-page.tsx (MeaningFields, RelatedWordItem,
InlineWordEditor, dsb).

STRUKTUR HALAMAN:

1. PageHeader + breadcrumb "Kata / Edit Kata" (tambah label segmen 'Edit
   Kata' di BREADCRUMB_LABELS console-layout.tsx).
2. GET /api/v1/admin/words/:id → detail SEMUA status. Selama loading:
   Skeleton. Gagal 404 WORD_NOT_FOUND → Alert merah + tombol "Kembali ke
   daftar" (kata dihapus orang lain) - jangan render form kosong.
3. Form ter-prefill:
   - DATA KATA DASAR: lemma, bahasa (default dari word.language_id, jangan
     dari pickDefaultLanguageIds), dialek, catatan, jenis entri
   - MAKNA/ARTI + contoh kalimat nested (children semua status ikut
     ter-prefill - child pending_review juga tampil, JANGAN disaring:
     prefill harus jujur, alur review menangani keputusan)
   - KATEGORI (multi-select dari word.category_ids)
   - RELASI KATA: Form A saja (link ke kata existing). "Bentuk = Kata baru
     (buat sekaligus)" di Nonaktifkan/disembunyikan - RelatedWordItem
     diberi prop allowInline=false → Segmented hanya opsi link
   - BENTUK TURUNAN + PENGUCAPAN + GAMBAR (jika ada di response)
4. AKSI FORM (reuse pola action bar Section 17 admin-base-stack):
   - "Batal" → kembali /words
   - "Simpan sebagai Draft" → status 'draft' (KATA published YANG DIEDIT
     JADI DRAFT = sengaja di-unpublish - kalau kata lama published,
     tampilkan Popconfirm konfirmasi: "Kata ini akan turun dari daftar
     publik (unpublish). Lanjut?")
   - "Simpan & Publikasikan" → status 'published'
   - Status AKHIR selalu dibaca dari data.status RESPONSE (bukan asumsi)
     - admin/editor/root/reviewer + published → langsung tayang +
       is_verified true (self-verified)
     - kata 'rejected' yang diedit + published → HIDUP LAGI (toast harus
       kata: "Kata yang ditolak diaktifkan kembali dan ditayangkan")
   - Toast sukses + warning duplikat (data.warnings, sama seperti create -
     banner kuning TIDAK memblokir)
   - RESPONSE punya updated_at, is_corrected → see catatan UI

INTEGRASI BACKEND (WAJIB - kontrak di 05-api-edit-kata.md):

1. AUTENTIKASI & ROLE
   - Halaman khusus verifikator: admin, editor, root, reviewer.
     CONTRIBUTOR TIDAK BOLEH (403 FORBIDDEN) - tombol "Ubah" di daftar
     kata di-hide untuk role contributor, dan navigasi manual ke route
     ini juga menampilkan 403 (jangan kosongkan halaman).
   - Akses token memory + auto refresh httpOnly cookie - mekanisme yang
     sudah ada (shared/api/client.ts), tidak ada tambahan.

2. PREFILL
   GET /api/v1/admin/words/:id  (role verifikator, rate limit 500/menit)
   Response: bentuk SAMA dengan GET /api/v1/words/:id publik + status,
   is_verified, is_corrected, created_at, updated_at, children semua
   status. → mapping ke CreateWordFormValues (lihat Reuse di bawah).

3. SUBMIT
   PUT /api/v1/admin/words/:id  (rate limit 30/menit; aturan sama create)
   Body: buildCreateWordBody(values, status) - FULL, bukan delta (field
   yang dihapus dari form = DIHAPUS dari kata; field tidak dikirim tetap
   dihapus). validation:
   - OPTIONAL yang dibiarkan kosong → omit (sama seperti create)
   - related_words TIDAK boleh ber-mode inline - kalau terlanjur ada di
     form, blok submit; kalau lolos, backend balas 400 VALIDATION_ERROR
     details [{ field: 'related_words', message: ... }] → map ke error
     inline form

4. ALUR STATUS (Section 22 approval gate - resolvePublication sisi server)
   - 'draft' → tidak tayang (published → draft = unpublish, konfirmasi UI)
   - 'published' + role verifikator → langsung tayang + is_verified true
   - 'pending_review'/'rejected' TIDAK bisa di-set user - tombol draft/
     published saja
   - is_corrected di-preserve backend (admin edit biasa tidak boleh
     me-reset flag "pernah dikoreksi verifikator") - UI TIDAK menampilkan
     field/toggle untuk is_corrected

5. HANDLE RESPONSE (envelope standar Section 13) - pola SAMA create:
   - Sukses: { success: true, data: { word_id, lemma, word_type, status,
     is_verified, is_corrected, updated_at, warnings? } } → toast + kembali
     /words
   - 400 VALIDATION_ERROR: details[] [{ field, message }] → PETAKAN via
     fieldToNamePath (create-word-utils.ts) → error inline; khusus field
     'related_words' pesan "Kreasi kata inline hanya lewat POST - buat
     dulu, lalu link"
   - 403 (contributor / role berubah) → toast/message + redirect list
   - 404 (kata hilang antara prefill & submit) → Alert + kembali list
   - 429 RATE_LIMITED → pesan + Retry-After
   - Audit action 'update' ditulis otomatis backend - TIDAK ada UI tambahan

REUSE kode yang SUDAH ADA (JANGAN duplikasi):

- src/features/words/presentation/word-form-blocks.tsx → seluruh blok
  form create dipakai apa adanya; tambahkan prop allowInline di
  RelatedWordItem (default true; halaman edit kirim false) + sembunyikan
  opsi inline di Segmented "Bentuk"
- src/features/words/application/create-word-utils.ts →
  buildCreateWordBody (submit PUT), fieldToNamePath (error mapping),
  urutan/render opsi
- src/features/words/application/use-reference-data.ts →
  languages/dialects/word-classes/categories (dropdown prefill)
- src/features/words/application/use-word-search.ts +
  src/features/words/presentation/word-search-select.tsx → relasi Form A
- src/features/words/domain/create-word.ts → CreateWordFormValues + label
  konstanta (satu sumber kebenaran, jangan salin tipe)
- src/features/words/application/use-create-word.ts → POLA mutation hook
  (useMutation + invalidate ['words']); buat useUpdateWord mengikuti
- src/features/contributions/application/correct-contribution.ts →
  FRONTEND: wordEntityToFormValues menunjukkan pola mapping detail→form
  values (jangan ikutkan/dipinjam - buat mapper khusus di fitur words
  karena sumber datanya WordDetail, bukan contribution WordEntityView;
  pembaliknya buildCreateWordBody sudah dipakai di dua-duanya)

FILE BARU/DIUBAH (ikutan konvensi penamaan Section 5 admin-base-stack):

src/features/words/
├── domain/
│   ├── word-detail.ts                    # BARU: WordDetail (response GET
│   │                                     #   /admin/words/:id) + tipe child
│   └── word.ts                           # (mungkin) konstanta dibagi
├── application/
│   ├── word-detail-mappers.ts            # BARU: wordDetailToFormValues()
│   │                                     #   WordDetail → CreateWordFormValues
│   ├── use-word-detail.ts                # BARU: query ['words','detail',id]
│   │                                     #   enabled: !!id, staleTime 30s
│   └── use-update-word.ts                # BARU: PUT mutation, invalidate
│                                         #   ['words'] + ['words','detail',id]
├── infrastructure/
│   └── word-api.ts                       # TAMBAH: getAdminWordDetailRequest(id)
│                                         #   + updateWordRequest(id, body)
└── presentation/
    ├── word-form-blocks.tsx              # UBAH: RelatedWordItem prop
    │                                     #   allowInline?: boolean
    ├── edit-word-page.tsx                # BARU: halaman edit (/words/:id/edit)
    └── words-page.tsx                    # UBAH: kolom aksi + tombol "Ubah"
                                         #   (hidden untuk contributor) →
                                         #   navigate /words/:id/edit

src/app/router.tsx                        # UBAH: route /words/:id/edit di bawah
                                          #   console-layout + BREADCRUMB_LABELS

Pola data (contoh, sesuaikan kontrak 05-api-edit-kata.md):

GET /api/v1/admin/words/{id}  → 200
{
  "success": true,
  "data": {
    "id": "01HXYZ…", "lemma": "makatn", "word_type": "word",
    "language_id": "01HXYZ…", "dialect_id": null, "notes": null,
    "status": "published", "is_verified": true, "is_corrected": false,
    "created_at": "…", "updated_at": "…",
    "meanings": [{ "id": "…", "word_class_id": "…", "definition": "…",
      "order_index": 1, "translations": [{ "language_id": "…",
        "translation_text": "makan", "translation_type": "direct" }],
      "examples": [{ "source_sentence": "…", "target_sentence": "…",
        "source_language_id": "…", "target_language_id": "…",
        "source_type": "native_speaker" }], "appearsIn": [] }],
    "pronunciations": [], "images": [], "variants": [],
    "category_ids": [], "related_words": [], "synonym_ids": []
  }
}

PUT /api/v1/admin/words/{id}  → 200 (full replace; body = bentuk create)
{
  "language_id": "…", "lemma": "makatn (revisi)", "word_type": "word",
  "dialect_id": null, "notes": "Catatan revisi",
  "meanings": [ /* …sama seperti create… */ ],
  "category_ids": [], "related_words": [], "variants": [],
  "pronunciation": null, "images": [], "status": "published"
}

CARA DEFINISI UI (WAJIB - admin-base-stack):

- Route baru didaftarkan SEKALIGUS di router.tsx DAN (kalau segmen beda)
  BREADCRUMB_LABELS di console-layout.tsx - jangan lupa "Edit Kata".
- Kolom aksi words-page pakai meta: { fixed: 'right' } (Section 13);
  tombol "Ubah" type="link" di samping "Detail" → navigate ke
  /words/:id/edit dengan id dari row kata.
- Action bar tombol WAJIB pola responsif Section 17 (Grid.useBreakpoint,
  vertical + block di HP, Space wrap) - sama persis create-word-page.
- Loading state: Skeleton antd saat prefill; jangan flashing form kosong.
- Error submit: ApiError.fieldErrors() → form.setFields (pola create).

TESTING (Section 15 admin-base-stack - hanya logika murni):

- Unit word-detail-mappers.test.ts:
  * wordDetailToFormValues mencakup semua child (meanings/pronunciations/
    images/examples/variants) + category_ids + related_words Form A
  * base case: language_id terisi (bukan pick default), status/id tidak
    bocor ke form values
  * round-trip: wordDetailToFormValues → buildCreateWordBody → bentuk
    body PUT sesuai kontrak 05 doc (lemma, meanings[0].definition, dst)
- Unit create-word-utils tambahan: allowInline=false di RelatedWordItem
  SUDAH dihapus opsi inline (testing pure helper output Segmented options
  bila bisa dipisahlan)
- Command verifikasi: pnpm typecheck && pnpm lint && pnpm test && pnpm build

BRUNO (sudah ada, tinggal dipakai manual - repo http/word/):
- get-admin-word.bru   # GET prefill semua status
- update-word.bru      # PUT full replace + asserts respons
Urutan uji manual: buat kata → prefill GET → ubah lemma via PUT →
GET /words/search memantulkan perubahan → 403 sebagai contributor.

PERILAKU UX & EDGE CASE:

- Prefill TETAP terbuka untuk draft/pending_review/rejected (verifikator
  mengedit langsung - BUKAN lewat alur correct). Alur correct di antrean
  review (03 doc) membuat inti baru; edit CARA LANGSUNG mengubah entri.
  Keduanya menulis baris audit action berbeda ('correct' vs 'update').
- Kata published yang diedit → simpan draft = unpublish (Popconfirm).
- Kata rejected yang diedit → published = hidup lagi (toast jujur).
- Duplikat lemma: TIDAK memblokir - warnings di response, tampilkan
  banner kuning setelah sukses (backend sudah mengecualikan diri sendiri,
  jadi edit tanpa ganti lemma tidak akan memicu warning).
- Child pending_review ikut tampil di prefill (prefill jujur) - jangan
  filter manual; keputusan pending child tetap lewat antrean review.
- Kata yang kena soft-delete di tengah proses → 404 → Alert + keluar.
```

---

## Catatan Implementasi

- `WordDetail` (domain) TIDAK boleh mengimpor React/antd/axios - murni
  tipe; mapper (`wordDetailToFormValues`) fungsi murni yang bisa di-test
  tanpa browser (Section 15).
- Jangan menyalin tipe/konstanta dari `create-word.ts` - import langsung;
  `buildCreateWordBody` dipakai create, edit, DAN koreksi (contribution)
  sekaligus - tiga konsumen, satu fungsi.
- `roadmap` soft-delete (`DELETE /api/v1/admin/words/:id`) menyusul di
  prompt terpisah - tombol "Hapus" TIDAK masuk fitur ini.
- Rating/validasi `related_words` Form B: dibuat impossible di UI
  (allowInline=false) DI TAMBAH pesan error inline kalau backend tetap
  menolak (defense-in-depth, bukan cuma UX).

## Referensi Terkait

- `docs/api/05-api-edit-kata.md` - kontrak edit kata (GET prefill + PUT
  full replace) yang dikonsumsi halaman ini
- `docs/api/01-api-tambah-kata.md` - kontrak induk bentuk body create
  (edit menerima body yang sama)
- `docs/admin/01-admin-tambah-kata.md` - halaman create; edit = saudara
  kembarnya dengan delta prefill + PUT + tanpa Form B
- `docs/api/04-api-sinonim-inline.md` - Form B: hanya di create, bukan edit
- `docs/admin/admin-base-stack.md` - stack, clean architecture 3 lapis,
  router/menu, action bar responsif, testing, error handling
- `repo http/` - `http/word/get-admin-word.bru`, `http/word/update-word.bru`
- `src/features/words/` - create-word-page.tsx (referensi form),
  word-form-blocks.tsx (blok bersama), create-word-utils.ts (body/error
  path), use-create-word.ts (pola mutation)