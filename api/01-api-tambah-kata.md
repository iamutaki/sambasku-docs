# API Word - Tambah Kata Baru (Admin)

Mengikuti `api-base-stack.md`: Clean Architecture feature-based (`modules/word/`)
- Section 3 (struktur folder), 9 (`@hono/zod-openapi` + Scalar), 10 (testing),
11 (versioning `/api/v1/`), 13 (envelope & error), 15 (rate limiting),
19 (ULID sebagai ID).

---

## Prompt

```text
Buatkan modul "word" (fitur Tambah Kata Baru) untuk backend Kamus Digital
Sambas-Indonesia, mengikuti struktur clean architecture feature-based
yang sudah ditetapkan di api-base-stack.md.

STACK:
- Hono (Node.js runtime) - route pakai OpenAPIHono dari @hono/zod-openapi (Section 9)
- Drizzle ORM + PostgreSQL
- Zod untuk validasi request/response + generate OpenAPI spec (Section 9)
- Migration: Drizzle Kit (drizzle-kit generate / migrate)

LOKASI MODUL: modules/word/

STRUKTUR FILE YANG PERLU DIBUAT:

modules/word/
├── domain/
│   ├── entities/
│   │   ├── word.entity.ts
│   │   └── meaning.entity.ts
│   └── repositories/
│       └── word.repository.ts           # interface (contract)
├── application/
│   ├── use-cases/
│   │   ├── create-word.use-case.ts
│   │   ├── get-word-by-id.use-case.ts
│   │   └── search-words.use-case.ts
│   ├── dto/
│   │   └── create-word.dto.ts
│   └── ports/                           # (kosong dulu - tambah kalau ada
│       └── .gitkeep                     #   service eksternal, pola *.port.ts)
├── infrastructure/
│   └── word.repository.impl.ts          # implementasi Drizzle + transaction
├── presentation/
│   └── v1/                              # Section 11: API versioning
│       ├── word.routes.ts               # OpenAPIHono + createRoute (Section 9)
│       ├── word.controller.ts
│       └── validators/
│           └── create-word.validator.ts # Zod schema request + response
└── __tests__/                           # Section 10
    ├── unit/
    │   └── create-word.use-case.test.ts
    ├── integration/
    │   └── word.repository.impl.test.ts
    └── e2e/
        ├── v1/
        │   └── create-word.e2e.test.ts
        └── (versi lain ditambah saat ada v2 - Section 11)

TABEL DATABASE TERKAIT (didefinisikan terpusat di
shared/database/drizzle/schema/, sesuai docs/dbdiagram.dbml):

- languages, dialects, word_classes (baru - data referensi)
- words (+kolom word_type), meanings, meaning_translations, examples,
  word_categories, categories, lexical_relations (+index target),
  pronunciations, word_images (gambar contoh, referensi provider),
  word_variants (bentuk surface + afiks terstruktur)
- contributions, contribution_reviews (baru - audit & workflow review)

PERUBAHAN SKHEMA (WAJIB - ikuti alur Section 7):
1. Tambahkan semua tabel di atas sebagai file *.schema.ts (ULID varchar(26)
   untuk semua PK/FK - Section 19; kolom `status` BARU di tabel words)
2. words.status: varchar(30) NOT NULL DEFAULT 'draft'
   - nilai: 'draft' | 'pending_review' | 'published' | 'rejected'
   ('pending_review'/'rejected' hanya di-set sistem - approval gate
   Section 22; request user tetap 'draft' | 'published')
   (update juga docs/dbdiagram.dbml agar tetap sumber desain)
3. Jalankan:
   pnpm drizzle-kit generate   → REVIEW SQL yang dihasilkan
   pnpm drizzle-kit migrate    → apply ke database
4. Commit schema + migration SQL dalam satu PR

ENDPOINT UTAMA (semua di bawah prefix /api/v1 - Section 11):

1. POST /api/v1/admin/words
   Middleware: authenticate + authorizeRole('admin', 'editor', 'contributor')
   + rateLimit 30 request/menit per user_id (Section 15)

   Body (semua *_id berupa ULID string - Section 19):
   {
     "language_id": "01HXYZ…",
     "dialect_id": "01HXYZ…",            // opsional
     "lemma": "makatn",
     "notes": "Contoh catatan tambahan",
     "meanings": [
       {
         "word_class_id": "01HXYZ…",
         "definition": "Aktivitas memasukkan makanan ke mulut",
         "order_index": 1,
         "translations": [
           { "language_id": "01HXYZ…", "translation_text": "makan",
             "translation_type": "direct" }
         ],
         "examples": [
           { "source_language_id": "01HXYZ…",
             "source_sentence": "Kami udah makatn tadi.",
             "target_language_id": "01HXYZ…",
             "target_sentence": "Kami sudah makan tadi.",
             "source_type": "native_speaker" }
         ]
       }
     ],
     "word_type": "word",                  // word | idiom | peribahasa | ungkapan
     "category_ids": ["01HXYZ…", "01HXYZ…"],
"related_words": [
        { "word_id": "01HXYZ…", "relation_type": "synonym" },   // Form A: link ke kata yang SUDAH ada
        { "relation_type": "synonym",                           // Form B: kreasi sinonim BARU inline -
          "word": { "lemma": "ngamakn",                        //   inherit definisi parent secara default
                    "inherit_meanings": true,
                    "meaning_overrides": [],
                    "status": "draft" } }                       // kontrak lengkap: 04-api-sinonim-inline.md
      ],
     "variants": [                          // bentuk surface, opsional
       { "form": "memakan", "variant_type": "derivation",
         "affix_type": "prefix", "affix_value": "me-" }
     ],
     "pronunciation": { "notation": "ipa", "value": "/makatn/" },
     "images": [
       { "url": "https://ik.imagekit.io/…/makan.jpg",   // hasil direct-upload
         "provider_file_id": "file_abc123",
         "alt_text": "Ilustrasi orang makan",
         "is_primary": true }             // maksimal SATU primary
     ],
     "status": "draft"                    // atau "published"
   }

   Use case: CreateWordUseCase
   a. VALIDASI INPUT (create-word.validator.ts, Zod):
      - lemma wajib, tidak boleh kosong/whitespace saja
      - minimal 1 meaning, tiap meaning minimal 1 translation
      - language_id, word_class_id, target language_id harus ada di DB
        (hasil cek masuk details VALIDATION_ERROR per field terkait)
   b. CEK DUPLIKASI: lemma sama (case-insensitive) pada language_id yang
      sama dan belum soft-deleted
      → BUKAN error, kembalikan sebagai `warnings` di data (lihat response)
      (dialect_id tidak jadi kunci duplikat - tabel words tidak punya
      kolom dialek; dialect_id tersimpan di pronunciations.dialect_id)
   c. TRANSAKSI: WordRepository.saveWithRelations(...) - SATU method di
      interface yang dijamin atomik; implementasi Drizzle membungkus
      SEMUA insert dalam satu db.transaction(). Urutan insert:
      words → meanings → meaning_translations → examples →
      word_categories → lexical_relations (jika ada related_words,
      relation_type dari request - bukan hanya synonym) →
      word_variants (jika ada, bentuk surface + afiks terstruktur) →
      pronunciations (jika ada) → word_images (jika ada, referensi
      provider eksternal - kolom provider + provider_file_id) →
      contributions (user_id dari token,
      entity_type 'word', entity_id word_id, action 'create').
      Gagal salah satu → rollback semua.
      (Use case TIDAK tahu soal transaction - itu urusan infrastructure)
   d. MODEL PUBLIKASI (Section 22 - approval gate):
      - "draft" → simpan apa adanya (words.status = 'draft', tidak tayang,
        tidak masuk antrean)
      - "published" + role admin/editor/root/reviewer → LANGSUNG tayang
        (status 'published', is_verified true - self-verified)
      - "published" + role contributor → status 'pending_review',
        is_verified false - TIDAK tayang, masuk antrean review
        (baris contributions dengan status 'pending')
      - Baris contribution_reviews TIDAK dibuat saat submit - dibuat
        saat verifikator mengambil keputusan
        (approve/reject/correct - lihat 03-api-kontribusi-verifikasi.md)
   e. created_by = user_id dari token (c.get('user')) di SEMUA tabel
      yang punya kolom created_by
   f. Audit trail (Section 21): lewat AuditLogRepository (interface
      modul audit) - action 'create', entity_type 'word',
      new_data { lemma, language_id, status, meanings_count },
      request_id dari context
   g. Setelah sukses: log event bisnis 'word created' level info dengan
      request_id dari context (Section 14)

   Response sukses (201) - envelope standar Section 13:
   { "success": true,
     "data": { "word_id": "01ARZ3NDEKTSV4RRFFQ69G5FAV",
               "lemma": "makatn",
               "word_type": "word",
               "status": "published",
               "is_verified": true,
               "created_at": "2026-09-09T10:00:00Z",
               "warnings": [
                 { "field": "lemma",
                   "message": "Lemma serupa sudah ada di bahasa ini" }
               ] } }        // warnings hanya ada kalau duplikat terdeteksi
   // Varian per role (Section 22 - approval gate):
   // - admin/editor/root/reviewer → status "published", is_verified true
   // - contributor → status "pending_review", is_verified false
   //   (tidak tayang - masuk antrean review; toast UI harus jujur
   //   menyebut status akhir dari response)

   Response gagal - envelope standar Section 13 (tanpa field custom):
   - 400: { "success": false, "error_code": "VALIDATION_ERROR",
            "message": "Data yang dikirim tidak valid",
            "details": [ { "field": "lemma",
                           "message": "Kata tidak boleh kosong" },
                         { "field": "meanings",
                           "message": "Minimal harus ada 1 makna" } ] }
   - 401: error_code "UNAUTHORIZED" / "TOKEN_EXPIRED" (dari authenticate)
   - 403: error_code "FORBIDDEN" (dari authorizeRole)
   - 500: error_code "INTERNAL_ERROR" (dari error-handler global -
          JANGAN tulis handler 500 sendiri di controller)

   KOSAKATA RELASI (related_words.relation_type) - semua di tabel
   lexical_relations, arah source = entri ini, target = entri lain:

   | relation_type  | Makna                                   | Sah untuk     |
   |----------------|-----------------------------------------|---------------|
   | synonym        | makna mirip antar entri setara          | semua word_type |
   | antonym        | lawan makna                              | semua word_type |
   | has_component  | frasa → kata pembentuknya (miyang rabong → miyang) | hanya idiom/peribahasa/ungkapan |
   | derived_from   | entri turunan morfologis (memakan → makan) | semua word_type |

   Aturan: invers TIDAK disimpan (derived dari query - appears_in di
   detail); duplikat word_id dalam satu request ditolak; relasi masuk
   tampil sebagai appears_in di detail kata komponen.
   Untuk relation_type "synonym", related_words juga menerima **Form B**:
   membuat entri sinonim BARU secara inline yang default-nya mewarisi
   definisi/makna induk (bisa di-override per makna). Kontrak lengkap di
   `04-api-sinonim-inline.md` - bentuk lain tetap harus lewat Form A
   (link kata yang sudah ada).

   BENTUK SURFACE (variants[]) - tabel word_variants:
   - form + variant_type (inflection|derivation|alternative|reduplication)
   - affix_type (prefix|suffix|circumfix|reduplication) + affix_value
     (mis. 'me-') - afiks terstruktur, bisa di-query/filter
   - affix_value WAJIB bila affix_type diisi; dialect_id opsional
   - Garis pemisah: punya makna sendiri → entri words + related_words
     (derived_from); hanya bentuk alternatif → word_variants

   CONTOH LENGKAP "miyang rabong":
   - entri "miyang" (k.keterangan: gatal) + entri "rabong" (k.benda:
     rebung) - dua entri mandiri
   - entri "miyang rabong" word_type 'peribahasa', 2 makna figuratif
     (translation_type 'idiomatic'), related_words has_component ×2
   - detail "miyang rabong" menampilkan related_words; detail "miyang"
     menampilkan appears_in: miyang rabong

2. GET /api/v1/words/:id
   (publik - dipakai halaman detail kata + redirect setelah admin submit)

   Use case: GetWordByIdUseCase
   - :id = ULID string
   - Hanya tampilkan kata yang statusnya 'published' DAN belum
     soft-deleted (draft/pending_review/rejected tidak tayang;
     pending_review/rejected hanya terlihat lewat antrean review -
     03-api-kontribusi-verifikasi.md)
   - Response 200: { "success": true, "data": { kata lengkap: lemma,
     language_id, word_type, meanings[] (word_class TERSEMAT sebagai
     objek {id, code, name, parent_id} - k.benda/k.kerja/k.sifat
     langsung terbaca tanpa request kedua; definition,
     translations[], examples[]), categories[], pronunciations[],
     images[] (url, alt_text, is_primary), related_words[]
     (word_id, lemma, relation_type), appears_in[] (relasi masuk -
     mis. komponen "muncul dalam" peribahasa), variants[] (form,
     variant_type, affix_type, affix_value, dialect_id, notes),
     status } }
   - Tidak ditemukan → NotFoundError('WORD_NOT_FOUND',
     'Kata dengan id tersebut tidak ditemukan') - kode sudah ada di
     ERROR_CODES.md, tidak ada kode baru

3. POST /api/v1/admin/words/:id/verify
   (verifikator: admin, root, reviewer - Section 22)

   Use case: VerifyWordUseCase
   - Set words.is_verified = true + verified_by (dari token) +
     verified_at (now)
   - Kata tidak ditemukan / soft-deleted → 404 WORD_NOT_FOUND
   - Audit trail: action 'verify', new_data { is_verified: true }
   - Response 200: { "success": true, "data": null }

4. POST /api/v1/admin/words/:id/unverify
   (cabut verifikasi - verifikator juga)

   Use case: VerifyWordUseCase (verified: false)
   - Set is_verified = false (verified_by/at ikut ter-update - jejak
     siapa yang mencabut)
   - Audit: action 'unverify'
   - Response 200: { "success": true, "data": null }

ENDPOINT PENDUKUNG (dibutuhkan form admin - definisikan di modul
pemiliknya masing-masing, pola createRoute sama):

- GET /api/v1/languages            → modul language (baru)
    ?is_active=true - populate dropdown bahasa
- GET /api/v1/dialects?language_id=… → modul language
- GET /api/v1/word-classes         → modul word (data referensi,
    termasuk hierarki parent_id)      boleh file terpisah word-class.*)
- GET /api/v1/categories           → modul category (Section 3)
- GET /api/v1/admin/images/upload-token → modul image (baru) -
    kredensial DIRECT UPLOAD ke provider gambar: client minta token ke
    backend (authenticate + role admin/editor/contributor), lalu upload
    file LANGSUNG ke CDN provider (backend tidak pernah melewati byte
    gambar), dapat url + file_id, lalu kirim sebagai images[] di atas.
    Provider saat ini: ImageKit via ImageStoragePort (wrapper
    provider-agnostic - ganti provider = ganti satu file impl, Section 8;
    tanpa konfigurasi env IMAGEKIT_* → 503 IMAGE_UPLOAD_UNAVAILABLE)
- GET /api/v1/words/search?q=…&limit=20&cursor=…
    → modul word (SearchWordsUseCase) - searchable dropdown pilih
    sinonim/antonim; cari hanya kata yang belum soft-deleted.
    DUA ARAH via query search_in (default 'lemma'):
    - search_in=lemma        → Sambas→Indonesia: cocokkan words.lemma
    - word_type=peribahasa   → filter jenis entri (word|idiom|
      peribahasa|ungkapan) - label UI/badge + daftar khusus
    - search_in=translation  → Indonesia→Sambas (reverse): cocokkan
      meaning_translations.translation_text, hasil = kata Sambas-nya;
      opsional translation_language_id untuk membatasi bahasa sumber.
      Item hasil punya field tambahan matched_translation (teks
      terjemahan yang cocok - untuk ditampilkan "makan → makatn").
    Response bentuk list + meta cursor-based (Section 13):
    meta: { limit: 20, next_cursor: "<ULID>|null", has_more: bool }
    - cursor = ULID id item terakhir, untuk halaman berikutnya;
    client pakai pola "Muat lagi" (bukan page numbers).
    Catatan performa: ilike '%q%' tidak memakai index B-tree - begitu
    data membesar, tambahkan index pg_trgm (gin_trgm_ops) pada
    words.lemma dan meaning_translations.translation_text lewat
    migration terpisah.

Semua endpoint pendukung: response envelope Section 13, GET publik
boleh tanpa authenticate, rate limit 100 request/menit per IP (Section 15).

CARA DEFINISI ENDPOINT (WAJIB - Section 9 api-base-stack.md):
- Route file pakai OpenAPIHono, tiap endpoint didefinisikan dengan
  createRoute() + app.openapi() - BUKAN app.post() biasa
- Tiap endpoint isi: tags (mis. ['Words', 'Admin']), summary, schema Zod
  untuk request body/query DAN response (sukses + error) di validators/
- Schema response HARUS bentuk envelope standar (Section 13)
- Dokumentasi otomatis muncul di GET /docs (Scalar) tanpa langkah tambahan

KEAMANAN & CATATAN:
- Semua query parameterized (otomatis aman via Drizzle - hindari sql`` raw
  dengan input user)
- Rate limiting WAJIB terpasang sejak endpoint dibuat (Section 15)
- Jangan pernah percaya created_by dari body - SELALU dari token
- Gambar: backend TIDAK memvalidasi kepemilikan provider_file_id (file
  publik hasil upload client) - cukup url valid + provider_file_id
  terisi; batas 10 gambar/kata, maksimal satu is_primary
- Race FK: pengecekan "id ada di DB" dilakukan SEBELUM transaksi; kalau
  kalah race (id dihapus di tengah), constraint FK meledak DI DALAM
  transaksi - implementasi repository wajib menangkap kode error
  PostgreSQL 23503 (foreign_key_violation) dan melemparnya sebagai
  ValidationError dengan field terkait, JANGAN dibiarkan jadi 500
- Race cek duplikat lemma (check-then-act) bisa lolos dua submit
  konkuren - AMAN by design karena duplikat memang hanya warning
  (bukan uniqueness constraint); dedupe final lewat workflow review
- Deliverable termasuk koleksi Bruno: folder http/word/ berisi .bru
  per endpoint (POST admin/words, GET :id, search) + endpoint
  pendukung di modul masing-masing - Section 20, satu PR yang sama
- Endpoint admin lain (update/soft-delete kata) menyusul di prompt
  terpisah dengan pola yang sama
- Antrean review (approve/reject/correct) + kontribusi media mandiri
  (gambar/pronounce/contoh pada kata existing): prompt terpisah -
  `03-api-kontribusi-verifikasi.md`

FORMAT RESPONSE - ENVELOPE STANDAR (Section 13 api-base-stack.md):
- Sukses: { "success": true, "data": { ... } } (+ meta untuk list)
- Gagal: { "success": false, "error_code": "...", "message": "...",
          "details": null | [ { field, message } ] }
- Tidak ada pengecualian "response custom" per modul
- Error code yang dipakai fitur ini sudah ada di ERROR_CODES.md
  (VALIDATION_ERROR, UNAUTHORIZED, TOKEN_EXPIRED, FORBIDDEN,
  INTERNAL_ERROR) - tidak ada kode baru
```

---

## Catatan Implementasi

- Use case **tidak boleh** import Drizzle/transaction API - atomicity
  diekspresikan di kontrak repository (`saveWithRelations`), implementasi
  `db.transaction()` hidup di `word.repository.impl.ts` saja.
- Modul lain (language, category) menyusul dengan struktur yang sama;
  komunikasi antar modul lewat interface yang di-export, bukan import
  internal (Section 4).
- `main.ts` me-register: `app.route('/api/v1/admin/words', adminWordRoutes)`
  (dengan middleware authenticate + authorizeRole) dan
  `app.route('/api/v1/words', publicWordRoutes)` untuk search.
- Testing (Section 10): use case wajib unit test (mock repository -
  verifikasi urutan argumen `saveWithRelations`, logika status per role),
  repository impl wajib integration test (termasuk bukti rollback:
  insert gagal di tengah → TIDAK ada baris words tersisa), endpoint
  wajib minimal 1 e2e happy path + 1 gagal (validasi/authorization).

## Referensi Terkait

- `api-base-stack.md` - definisi stack & struktur folder lengkap
- `03-api-kontribusi-verifikasi.md` - antrean review approve/reject/
  correct + kontribusi media (gambar, pronounce, contoh kalimat)
- repo `http/` - saat modul ini dikerjakan, buat folder `http/word/`
  dengan satu file `.bru` per endpoint + `tests` (Section 20);
  endpoint pendukung juga (mis. `http/language/list-languages.bru`)
- `docs/dbdiagram.dbml` - skema database (semua *_id kini ULID varchar 26)
- `00-api-auth.md` - modul auth (authenticate/authorizeRole dipakai di sini)
- `02-api-audit-logs.md` - sisi pembaca audit; create word menulis
  entri audit word.create (Section 21)
- `ERROR_CODES.md` - katalog error code
- `docs/admin/admin-tambah-kata.md` - UI admin (frontend) yang mengonsumsi
  endpoint ini
