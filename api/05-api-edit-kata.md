# API Word - Edit Kata (Admin)

Mengikuti `api-base-stack.md`: Section 3 (struktur folder), 9
(`@hono/zod-openapi` + Scalar), 10 (testing), 11 (versioning `/api/v1/`),
13 (envelope & error), 15 (rate limiting), 19 (ULID), 21 (audit trail),
22 (approval gate). Kontrak body, validasi, dan model publikasi
**mengikuti `01-api-tambah-kata.md`** - dokumen ini hanya mendefinisikan
DELTA terhadap create.

Status fondasi saat dokumen ini dibuat (JANGAN diduplikasi):
- `WordRepository.updateWithRelations(id, word, actorId)` SUDAH ADA -
  interface + implementasi Drizzle (replace semantics, SATU transaksi,
  baris `contributions` action `update` otomatis, FK 23503 → ValidationError).
  Saat ini dipakai alur *correct* (`03-api-kontribusi-verifikasi.md`).
- Yang BELUM: `UpdateWordUseCase`, validator, endpoint PUT, endpoint
  detail admin (prefill form), perubahan `findDuplicate`, tests, Bruno.

---

## Prompt

```text
Buatkan fitur "Edit Kata" (update kata existing) untuk modul word,
mengikuti persis pola & alur CreateWordUseCase di 01-api-tambah-kata.md.

LOKASI MODUL: modules/word/ (sudah ada - LENGKAPI, jangan buat modul baru)

STRUKTUR FILE YANG PERLU DIBUAT/DIUBAH:

modules/word/
├── application/
│   ├── use-cases/
│   │   ├── update-word.use-case.ts          # BARU
│   │   └── get-word-by-id.use-case.ts       # UBAH: param opsional
│   │                                        #   { includeAllStatuses?: boolean }
│   │                                        #   (prefill admin - tanpa use case baru)
│   └── dto/
│       └── update-word.dto.ts               # BARU (turunan CreateWordDto)
├── domain/repositories/word.repository.ts   # UBAH: findDuplicate + param ke-3 opsional
├── infrastructure/word.repository.impl.ts   # UBAH: findDuplicate exclude id
├── presentation/v1/
│   ├── word.routes.ts                       # UBAH: +2 route di createAdminWordRoutes
│   ├── word.controller.ts                   # UBAH: +update, +adminDetail
│   └── validators/update-word.validator.ts  # BARU
└── __tests__/
    ├── unit/update-word.use-case.test.ts            # BARU
    ├── integration/word.repository.impl.test.ts     # TAMBAH kasus update
    └── e2e/v1/update-word.e2e.test.ts               # BARU

PERUBAHAN REPOSITORY (SATU-SATUNYA - backward compatible):
- findDuplicate(languageId, lemma, excludeWordId?: string)
  → cek duplikat lemma case-insensitive pada language_id sama, belum
  soft-deleted, DAN (bila diisi) id != excludeWordId
  → tanpa excludeWordId perilaku identik dengan sekarang (create tak berubah)

TIDAK ADA perubahan skema database / migration (semua tabel sudah ada).
TIDAK ADA error code baru (WORD_NOT_FOUND, VALIDATION_ERROR, UNAUTHORIZED,
TOKEN_EXPIRED, FORBIDDEN, RATE_LIMITED, INTERNAL_ERROR sudah ada di
ERROR_CODES.md).

ENDPOINT:

1. GET /api/v1/admin/words/:id
   (prefill form edit admin - publik GET /api/v1/words/:id hanya menampilkan
   status 'published', sedangkan form edit harus bisa membuka draft/
   pending_review/rejected juga)

   Middleware: authenticate + authorizeRole('admin', 'editor', 'root',
   'reviewer') + rate limit 500/menit (kategori admin, Section 15)

   Use case: GetWordByIdUseCase.execute(id, { includeAllStatuses: true })
   (method yang sama dengan publik, param baru opsional)

   Response 200: bentuk SAMA dengan GET /api/v1/words/:id (01 doc), plus:
   - status SELALU terisi (draft/pending_review/published/rejected)
   - is_verified, is_corrected
   - created_at, updated_at
   - children (meanings[], pronunciations[], images[], examples[]) SEMUA
     status ikut terisi (prefill jujur - child pending_review juga tampak;
     behavior includeAllStatuses yang sudah ada di entity)
   Tidak ditemukan / soft-deleted → 404 WORD_NOT_FOUND

2. PUT /api/v1/admin/words/:id
   (edit kata - FULL REPLACE, bukan PATCH; lihat "Catatan Implementasi")

   Middleware: authenticate + authorizeRole('admin', 'editor', 'root',
   'reviewer') + rateLimit 30 request/menit per user_id (Section 15,
   kategori sama dengan POST create)

   Body: BENTUK SAMA PERSIS dengan POST /api/v1/admin/words (01 doc) -
   language_id, dialect_id, lemma, notes, meanings[] (word_class_id,
   definition, order_index, translations[], examples[]), word_type,
   category_ids, related_words[], variants[], pronunciation, images[],
   status ('draft' | 'published') - dengan DUA pengecualian:
   a. related_words HANYA Form A ({ word_id, relation_type }) - link ke
      kata yang SUDAH ada. Form B (kreasi sinonim inline) DITOLAK dengan
      VALIDATION_ERROR details field related_words: "Kreasi kata inline
      hanya lewat POST /admin/words - buat dulu, lalu link".
      (repository hanya punya saveWithInlineRelations untuk CREATE;
      jangan buat updateWithInlineRelations di iterasi ini)
   b. field yang tidak dikirim tetap dihapus (full replace) - UI wajib
      mengirim ulang seluruh form hasil prefill GET di atas

   Use case: UpdateWordUseCase - urutan WAJIB sama dengan create:

   a. MUAT KATA LAMA: wordRepo.findDetailById(id, { includeAllStatuses: true })
      → null → NotFoundError('WORD_NOT_FOUND', 'Kata dengan id tersebut
      tidak ditemukan') (termasuk soft-deleted)
      Snapshot ini dipakai untuk: audit old_data + preserve is_corrected.
      SEMUA status boleh diedit (draft/pending_review/published/rejected).
   b. ATURAN SILANG (sama seperti create): word_type 'word' tidak boleh
      punya related_words relation_type 'has_component'
   c. TOLAK FORM B (lihat body di atas)
   d. VALIDASI REFERENSI: wordRepo.findMissingReferences(...) - bentuk
      panggilan sama dengan create (bagian inline kosong); hasil missing
      → details VALIDATION_ERROR per field terkait
   e. CEK DUPLIKAT: findDuplicate(language_id, lemma, excludeWordId: id)
      - WAJIB exclude diri sendiri, kalau tidak setiap edit selalu
        "duplikat" dengan dirinya sendiri
      → tetap WARNING di response, bukan error (sama seperti create)
   f. MODEL PUBLIKASI: resolvePublication(status, actor.role) - helper
      yang sama dengan create (Section 22):
      - status 'draft' → words.status 'draft' (kata published yang diedit
        jadi draft = sengaja di-unpublish; UI toast harus jujur)
      - status 'published' + admin/editor/root/reviewer → langsung
        'published', is_verified true (self-verified)
      - 'pending_review'/'rejected' TIDAK pernah di-set dari request user
      - catatan: kata 'rejected' yang diedit verifier dengan status
        'published' HIDUP LAGI (rejected memang terminal hanya untuk
        jalur kontribusi contributor)
   g. PRESERVE is_corrected: teruskan isCorrected dari snapshot langkah (a)
      ke WordToSave - JANGAN biarkan default false meng-reset flag
      "pernah dikoreksi verifikator" saat admin melakukan edit biasa
   h. TRANSAKSI: wordRepo.updateWithRelations(id, wordToSave, user_id
      dari token) → null (kalah race soft-delete) → 404 WORD_NOT_FOUND.
      Replace children + baris contributions action 'update' otomatis
      (sudah ditangani impl); contribution_reviews TIDAK dibuat
      (keputusan review hanya lewat antrean - 03 doc)
   i. AUDIT (Section 21): action 'update', entity_type 'word',
      old_data { lemma, word_type, language_id, status, is_verified,
      meanings_count } dari snapshot (a), new_data bentuk sama dengan
      create (01 doc), request_id dari context
   j. Log event bisnis 'word updated' level info + request_id (Section 14)

   Response sukses (200) - envelope standar Section 13:
   { "success": true,
     "data": { "word_id": "01HXYZ…",
               "lemma": "makatn",
               "word_type": "word",
               "status": "published",
               "is_verified": true,
               "is_corrected": false,
               "updated_at": "2026-09-18T10:00:00Z",
               "warnings": [ { "field": "lemma",
                               "message": "Lemma serupa sudah ada di bahasa ini" } ] } }
   // warnings hanya ada kalau duplikat (selain dirinya) terdeteksi

   Response gagal: envelope standar (400/401/403/404/429/500) - pola
   persis create; controller TIDAK menulis handler 500 sendiri

KEPUTUSAN SEMANTIK (disengaja, beda dari create):
- Role editor boleh edit (bagian tim verifikator - self-verified), tapi
  contributor TIDAK BOLEH edit entri existing: perubahan contributor
  atas entri orang lain adalah kontribusi → jalurnya antrean review
  (03 doc), bukan endpoint ini. Kalau kelak mau dibuka, itu keputusan
  produk terpisah (dampaknya: kata published turun ke pending_review
  = unpublish sampai disetujui).
- PUT full-replace dipilih karena kontrak repository updateWithRelations
  memang replace children (hapus semua → insert ulang dalam satu
  transaksi). PATCH per-field justru menambah kompleksitas diff tanpa
  manfaat - data kamus kecil, UI sudah memegang seluruh form.

CARA DEFINISI ENDPOINT (WAJIB - Section 9): createRoute() + app.openapi()
di word.routes.ts (file/route-group yang sama dengan create - registrasi
app.route('/api/v1/admin/words', …) di app.ts tidak berubah), schema Zod
request+response di validators/update-word.validator.ts, tags
['Words', 'Admin'], summary jelas. Dokumentasi otomatis di /docs.

KEAMANAN & CATATAN:
- updated_by SELALU dari token (c.get('user')), tidak pernah dari body
- created_by kolom tidak disentuh update (jejak pembuat tetap)
- Rate limit terpasang sejak endpoint dibuat (Section 15)
- Race FK & duplikat: penanganan sama seperti create (check-then-act di
  luar transaksi; impl sudah memetakan 23503 → ValidationError)
- Deliverable termasuk Bruno: http/word/get-admin-word.bru +
  http/word/update-word.bru (Section 20) dengan blok tests (status,
  envelope, field kunci; PUT mengubah lemma lalu GET memastikan berubah)

TESTING (Section 10):
- Unit update-word.use-case.test.ts (mock repository):
  * 404 saat findDetailById null; urutan & argumen updateWithRelations
  * findDuplicate dipanggil dengan excludeWordId = id kata
  * Form B → ValidationError; has_component + word_type 'word' → error
  * resolvePublication per role; is_corrected lama diteruskan (bukan false)
  * audit 'update' memuat old_data + new_data
- Integration word.repository.impl.test.ts (tambahan):
  * update menyimpan children baru & menghapus children lama
  * findDuplicate(…, excludeWordId) tidak match kata itu sendiri
  * bukti rollback: update gagal di tengah (FK invalid) → data LAMA
    tidak berubah sama sekali (children lama tetap utuh)
- E2E update-word.e2e.test.ts: happy path (login admin → GET prefill →
  PUT → GET publik memantulkan perubahan) + gagal (404 id acak,
  403 contributor) + GET admin detail menampilkan draft
```

---

## Catatan Implementasi

- Use case TIDAK boleh import Drizzle/transaction API - atomicity sudah
  diekspresikan kontrak `updateWithRelations` (impl `db.transaction()`
  hidup di `word.repository.impl.ts`, dipakai bersama alur correct).
- `UpdateWordDto` = `CreateWordDto` dengan `relatedWords` dipersempit
  Form A saja (`{ wordId, relationType }`) - definisikan sebagai tipe
  turunan, jangan salin ulang field (satu sumber kebenaran dengan
  validator create).
- Validator `update-word.validator.ts` me-render ulang schema body create
  (reuse/extend schema dari `create-word.validator.ts` jika memungkinkan)
  + params `id` ULID 26 char (Section 19).
- Controller mengambil `requestId` dari `c.get('requestId')` dan
  meneruskannya lewat `Actor` - sama seperti create.
- `GET /api/v1/admin/words/:id` tidak mengubah kontrak publik
  `GET /api/v1/words/:id` (tetap hanya published - Section 22).
- Soft-delete kata menyusul di prompt terpisah (endpoint
  `DELETE /api/v1/admin/words/:id`; method `softDelete` sudah tersedia
  di impl) - JANGAN dicampur ke PR ini.

## Referensi Terkait

- `01-api-tambah-kata.md` - kontrak induk: bentuk body, validasi referensi,
  duplikat-warning, response envelope, alur CreateWordUseCase yang
  digenapi oleh edit ini
- `api-base-stack.md` - stack, envelope, audit, approval gate, rate limit
- `03-api-kontribusi-verifikasi.md` - alur correct yang juga memakai
  `updateWithRelations` (memakai `isCorrected: true`); antrean review
  untuk perubahan contributor atas entri existing
- `04-api-sinonim-inline.md` - Form B inline (hanya di create, bukan edit)
- `docs/dbdiagram.dbml` - skema database (tidak berubah)
- `ERROR_CODES.md` - katalog error code (tidak ada kode baru)
- repo `http/` - `http/word/get-admin-word.bru`, `http/word/update-word.bru`
- `docs/admin/` - UI admin edit kata (frontend) yang mengonsumsi endpoint ini
