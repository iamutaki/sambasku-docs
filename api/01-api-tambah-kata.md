# API Word — Tambah Kata Baru (Admin)

Mengikuti `api-base-stack.md`: Clean Architecture feature-based (`modules/word/`)
— Section 3 (struktur folder), 9 (`@hono/zod-openapi` + Scalar), 10 (testing),
11 (versioning `/api/v1/`), 13 (envelope & error), 15 (rate limiting),
19 (ULID sebagai ID).

---

## Prompt

```text
Buatkan modul "word" (fitur Tambah Kata Baru) untuk backend Kamus Digital
Sambas-Indonesia, mengikuti struktur clean architecture feature-based
yang sudah ditetapkan di api-base-stack.md.

STACK:
- Hono (Node.js runtime) — route pakai OpenAPIHono dari @hono/zod-openapi (Section 9)
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
│   └── ports/                           # (kosong dulu — tambah kalau ada
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
        └── (versi lain ditambah saat ada v2 — Section 11)

TABEL DATABASE TERKAIT (didefinisikan terpusat di
shared/database/drizzle/schema/, sesuai docs/dbdiagram.dbml):

- languages, dialects, word_classes (baru — data referensi)
- words, meanings, meaning_translations, examples,
  word_categories, categories, lexical_relations, pronunciations (baru)
- contributions, contribution_reviews (baru — audit & workflow review)

PERUBAHAN SKHEMA (WAJIB — ikuti alur Section 7):
1. Tambahkan semua tabel di atas sebagai file *.schema.ts (ULID varchar(26)
   untuk semua PK/FK — Section 19; kolom `status` BARU di tabel words)
2. words.status: varchar(30) NOT NULL DEFAULT 'draft'
   — nilai: 'draft' | 'pending_review' | 'published'
   (update juga docs/dbdiagram.dbml agar tetap sumber desain)
3. Jalankan:
   pnpm drizzle-kit generate   → REVIEW SQL yang dihasilkan
   pnpm drizzle-kit migrate    → apply ke database
4. Commit schema + migration SQL dalam satu PR

ENDPOINT UTAMA (semua di bawah prefix /api/v1 — Section 11):

1. POST /api/v1/admin/words
   Middleware: authenticate + authorizeRole('admin', 'editor', 'contributor')
   + rateLimit 30 request/menit per user_id (Section 15)

   Body (semua *_id berupa ULID string — Section 19):
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
     "category_ids": ["01HXYZ…", "01HXYZ…"],
     "synonym_word_ids": [],
     "pronunciation": { "notation": "ipa", "value": "/makatn/" },
     "status": "draft"                    // atau "published"
   }

   Use case: CreateWordUseCase
   a. VALIDASI INPUT (create-word.validator.ts, Zod):
      - lemma wajib, tidak boleh kosong/whitespace saja
      - minimal 1 meaning, tiap meaning minimal 1 translation
      - language_id, word_class_id, target language_id harus ada di DB
        (hasil cek masuk details VALIDATION_ERROR per field terkait)
   b. CEK DUPLIKASI: lemma sama pada language_id (+ dialect_id jika diisi)
      → BUKAN error, kembalikan sebagai `warnings` di data (lihat response)
   c. TRANSAKSI: WordRepository.saveWithRelations(...) — SATU method di
      interface yang dijamin atomik; implementasi Drizzle membungkus
      SEMUA insert dalam satu db.transaction(). Urutan insert:
      words → meanings → meaning_translations → examples →
      word_categories → lexical_relations (jika ada synonym_word_ids) →
      pronunciations (jika ada) → contributions (user_id dari token,
      entity_type 'word', entity_id word_id, action 'create').
      Gagal salah satu → rollback semua.
      (Use case TIDAK tahu soal transaction — itu urusan infrastructure)
   d. STATUS DRAFT vs PUBLISHED:
      - "draft" → simpan apa adanya (words.status = 'draft')
      - "published" + role admin/editor → words.status = 'published'
        (langsung tayang)
      - "published" + role contributor → words.status = 'pending_review'
        + entry contribution_reviews (status 'pending')
   e. created_by = user_id dari token (c.get('user')) di SEMUA tabel
      yang punya kolom created_by
   f. Setelah sukses: log event bisnis 'word created' level info dengan
      request_id dari context (Section 14)

   Response sukses (201) — envelope standar Section 13:
   { "success": true,
     "data": { "word_id": "01ARZ3NDEKTSV4RRFFQ69G5FAV",
               "lemma": "makatn",
               "status": "draft",
               "created_at": "2026-09-09T10:00:00Z",
               "warnings": [
                 { "field": "lemma",
                   "message": "Lemma serupa sudah ada di bahasa ini" }
               ] } }        // warnings hanya ada kalau duplikat terdeteksi

   Response gagal — envelope standar Section 13 (tanpa field custom):
   - 400: { "success": false, "error_code": "VALIDATION_ERROR",
            "message": "Data yang dikirim tidak valid",
            "details": [ { "field": "lemma",
                           "message": "Kata tidak boleh kosong" },
                         { "field": "meanings",
                           "message": "Minimal harus ada 1 makna" } ] }
   - 401: error_code "UNAUTHORIZED" / "TOKEN_EXPIRED" (dari authenticate)
   - 403: error_code "FORBIDDEN" (dari authorizeRole)
   - 500: error_code "INTERNAL_ERROR" (dari error-handler global —
          JANGAN tulis handler 500 sendiri di controller)

ENDPOINT PENDUKUNG (dibutuhkan form admin — definisikan di modul
pemiliknya masing-masing, pola createRoute sama):

- GET /api/v1/languages            → modul language (baru)
    ?is_active=true — populate dropdown bahasa
- GET /api/v1/dialects?language_id=… → modul language
- GET /api/v1/word-classes         → modul word (data referensi,
    termasuk hierarki parent_id)      boleh file terpisah word-class.*)
- GET /api/v1/categories           → modul category (Section 3)
- GET /api/v1/words/search?q=…     → modul word (SearchWordsUseCase) —
    searchable dropdown pilih sinonim/antonim; cari hanya kata yang
    belum soft-deleted; response bentuk list + meta pagination (Section 13)

Semua endpoint pendukung: response envelope Section 13, GET publik
boleh tanpa authenticate, rate limit 100 request/menit per IP (Section 15).

CARA DEFINISI ENDPOINT (WAJIB — Section 9 api-base-stack.md):
- Route file pakai OpenAPIHono, tiap endpoint didefinisikan dengan
  createRoute() + app.openapi() — BUKAN app.post() biasa
- Tiap endpoint isi: tags (mis. ['Words', 'Admin']), summary, schema Zod
  untuk request body/query DAN response (sukses + error) di validators/
- Schema response HARUS bentuk envelope standar (Section 13)
- Dokumentasi otomatis muncul di GET /docs (Scalar) tanpa langkah tambahan

KEAMANAN & CATATAN:
- Semua query parameterized (otomatis aman via Drizzle — hindari sql`` raw
  dengan input user)
- Rate limiting WAJIB terpasang sejak endpoint dibuat (Section 15)
- Jangan pernah percaya created_by dari body — SELALU dari token
- Endpoint admin lain (update/soft-delete kata) menyusul di prompt
  terpisah dengan pola yang sama

FORMAT RESPONSE — ENVELOPE STANDAR (Section 13 api-base-stack.md):
- Sukses: { "success": true, "data": { ... } } (+ meta untuk list)
- Gagal: { "success": false, "error_code": "...", "message": "...",
          "details": null | [ { field, message } ] }
- Tidak ada pengecualian "response custom" per modul
- Error code yang dipakai fitur ini sudah ada di ERROR_CODES.md
  (VALIDATION_ERROR, UNAUTHORIZED, TOKEN_EXPIRED, FORBIDDEN,
  INTERNAL_ERROR) — tidak ada kode baru
```

---

## Catatan Implementasi

- Use case **tidak boleh** import Drizzle/transaction API — atomicity
  diekspresikan di kontrak repository (`saveWithRelations`), implementasi
  `db.transaction()` hidup di `word.repository.impl.ts` saja.
- Modul lain (language, category) menyusul dengan struktur yang sama;
  komunikasi antar modul lewat interface yang di-export, bukan import
  internal (Section 4).
- `main.ts` me-register: `app.route('/api/v1/admin/words', adminWordRoutes)`
  (dengan middleware authenticate + authorizeRole) dan
  `app.route('/api/v1/words', publicWordRoutes)` untuk search.
- Testing (Section 10): use case wajib unit test (mock repository —
  verifikasi urutan argumen `saveWithRelations`, logika status per role),
  repository impl wajib integration test (termasuk bukti rollback:
  insert gagal di tengah → TIDAK ada baris words tersisa), endpoint
  wajib minimal 1 e2e happy path + 1 gagal (validasi/authorization).

## Referensi Terkait

- `api-base-stack.md` — definisi stack & struktur folder lengkap
- repo `http/` — saat modul ini dikerjakan, buat folder `http/word/`
  dengan satu file `.bru` per endpoint + `tests` (Section 20);
  endpoint pendukung juga (mis. `http/language/list-languages.bru`)
- `docs/dbdiagram.dbml` — skema database (semua *_id kini ULID varchar 26)
- `00-api-auth.md` — modul auth (authenticate/authorizeRole dipakai di sini)
- `ERROR_CODES.md` — katalog error code
- `docs/admin/admin-tambah-kata.md` — UI admin (frontend) yang mengonsumsi
  endpoint ini
