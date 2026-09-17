# API Contribution - Kontribusi User & Verifikasi Admin (Approval Gate)

Mengikuti `api-base-stack.md`: Clean Architecture feature-based
(`modules/contribution/` + endpoint media di `modules/word/`) - Section 3
(struktur folder), 9 (`@hono/zod-openapi` + Scalar), 10 (testing), 11
(versioning `/api/v1/`), 13 (envelope & error), 15 (rate limiting), 19
(ULID sebagai ID), 21 (audit trail), **22 (approval gate - model
publikasi & verifikasi konten)**.

Dokumen ini kontrak untuk:
1. **Antrean review** - verifikator (admin/root/reviewer) melihat kontribusi
   yang menunggu, lalu **approve** / **reject** (alasan wajib) / **correct**
   (koreksi langsung oleh verifikator)
2. **Kontribusi media mandiri** - user menambah **gambar**, **pronounce**
   (pelafalan), dan **sample** (contoh kalimat) pada kata yang sudah tayang
3. **Search miss** - pencarian yang TIDAK menemukan hasil tercatat sebagai
   peluang kontribusi: muncul di panel admin + beranda user lain
   ("sedang dicari, belum ada artinya")

---

## Prompt

```text
Buatkan modul "contribution" (antrean review kontribusi) plus tiga endpoint
kontribusi media di modul word, untuk backend Kamus Digital
Sambas-Indonesia, mengikuti struktur clean architecture feature-based
yang sudah ditetapkan di api-base-stack.md.

STACK:
- Hono (Node.js runtime) - route pakai OpenAPIHono dari @hono/zod-openapi (Section 9)
- Drizzle ORM + PostgreSQL
- Zod untuk validasi request/response + generate OpenAPI spec (Section 9)
- Migration: Drizzle Kit (drizzle-kit generate / migrate)

LOKASI MODUL:
- modules/contribution/ - antrean review (baru)
- modules/search-miss/ - pencarian kosong → peluang kontribusi (baru)
- modules/word/presentation/v1/word-media.routes.ts - endpoint kontribusi
  media (menempel di modul word karena entity-nya milik word)

STRUKTUR FILE YANG PERLU DIBUAT:

modules/contribution/
├── domain/
│   ├── entities/
│   │   └── contribution.entity.ts        # Contribution, ContributionReview,
│   │                                     # ContributionStatus, EntityType
│   └── repositories/
│       └── contribution.repository.ts    # interface (contract)
├── application/
│   └── use-cases/
│       ├── list-contributions.use-case.ts
│       ├── get-contribution-detail.use-case.ts
│       ├── review-contribution.use-case.ts      # approve + reject
│       └── correct-contribution.use-case.ts
├── infrastructure/
│   └── contribution.repository.impl.ts   # implementasi Drizzle + transaction
├── presentation/
│   └── v1/
│       ├── contribution.routes.ts        # OpenAPIHono + createRoute (Section 9)
│       ├── contribution.controller.ts
│       └── validators/
│           └── contribution.validator.ts # Zod schema request + response
└── __tests__/                            # Section 10
    ├── unit/        (4 use case)
    ├── integration/ (contribution.repository.impl)
    └── e2e/v1/      (contribution.e2e)

modules/word/ (TAMBAHAN):
├── application/
│   ├── utils/resolve-publication.ts      # helper dibagikan (dipakai
│   │                                     # create-word + 3 use case media)
│   └── use-cases/
│       ├── add-pronunciation.use-case.ts
│       ├── add-word-image.use-case.ts
│       └── add-example.use-case.ts
└── presentation/v1/
    ├── validators/word-media.validator.ts
    └── word-media.routes.ts

modules/search-miss/ (BARU - pencarian kosong → peluang kontribusi):
├── domain/entities/search-miss.entity.ts
├── domain/repositories/search-miss.repository.ts
├── application/use-cases/
│   ├── list-search-misses.use-case.ts    # beranda + panel admin
│   └── dismiss-search-miss.use-case.ts
├── infrastructure/search-miss.repository.impl.ts
└── presentation/v1/
    ├── search-miss.routes.ts             # publik + admin (2 factory)
    ├── search-miss.controller.ts
    └── validators/search-miss.validator.ts

TABEL DATABASE TERKAIT (perubahan terpusat di
shared/database/drizzle/schema/, sesuai docs/dbdiagram.dbml):

PERUBAHAN SKHEMA (WAJIB - ikuti alur Section 7):
1. words: TAMBAH is_corrected boolean NOT NULL DEFAULT false;
   status varchar(30) NOT NULL DEFAULT 'draft' dengan nilai
   'draft' | 'pending_review' | 'published' | 'rejected'
   ('pending_review'/'rejected' hanya di-set sistem - Section 22)
2. pronunciations, word_images, examples: TAMBAH tiga kolom yang sama
   di tiap tabel - status varchar(30) NOT NULL DEFAULT 'published',
   is_verified boolean NOT NULL DEFAULT false,
   is_corrected boolean NOT NULL DEFAULT false
   (data lama otomatis 'published' via DEFAULT - tidak mengubah tampilan)
3. contributions: TAMBAH status varchar(30) NOT NULL DEFAULT 'pending'
   dengan nilai 'pending' | 'approved' | 'rejected' | 'corrected',
   plus index pada kolom itu (query antrean filter status)
4. BARU tabel search_misses:
   id varchar(26) PK, term varchar(255) NOT NULL (normalized lower/trim),
   direction varchar(20) NOT NULL DEFAULT 'lemma' (lemma | translation),
   hit_count int NOT NULL DEFAULT 1, last_searched_at timestamp NOT NULL,
   created_at/updated_at/deleted_at/deleted_by (soft delete Section 7),
   UNIQUE (term, direction). Fulfilment TIDAK disimpan kolom - derived
   saat dibaca (JOIN words published dengan lower(lemma) = term)
5. Jalankan:
   pnpm drizzle-kit generate --name=contribution-review-workflow
   → REVIEW SQL yang dihasilkan
   → APPEND satu statement backfill SEBELUM migrate:
     UPDATE "contributions" SET "status" = 'approved';
     (baris kontribusi lama JANGAN membanjiri antrean sebagai pending)
   pnpm drizzle-kit migrate
6. Commit schema + migration SQL dalam satu PR

ATURAN INTI (Section 22 - approval gate):

resolvePublication(requested, role) - SATU helper dipakai semua endpoint
submit (create-word + 3 media):
- requested 'draft' (hanya word) → { status: 'draft', isVerified: false }
- role admin/editor/root/reviewer → { status: 'published', isVerified: true }
  (self-verified - mereka tim verifikator)
- role contributor (dan role lain) → { status: 'pending_review',
  isVerified: false } - TIDAK tayang, masuk antrean

Setiap insert konten (word & media) juga menyisipkan baris `contributions`
dengan status turunan: entity 'pending_review' → contributions.status
'pending'; selain itu → 'approved'. Baris `contribution_reviews` TIDAK
dibuat saat submit - dibuat saat verifikator mengambil keputusan.

BACA PUBLIK (WAJIB): GET /api/v1/words/:id dan search hanya menyertakan
kata status='published' (sudah ada) DAN anak-anaknya (pronunciations,
images, examples) yang status='published'. Anak pending_review/rejected
hanya terlihat lewat detail antrean review.

======================================================================
ANTREAN REVIEW - 5 ENDPOINT (semua prefix /api/v1/admin/contributions)
======================================================================
Middleware SEMUA endpoint di bawah:
authenticate + authorizeRole('admin', 'root', 'reviewer')
+ rateLimit 500 request/60 detik (Section 15 - tier admin)

1. GET /api/v1/admin/contributions
   Query: ?status=pending|approved|rejected|corrected (opsional)
          &entity_type=word|pronunciation|word_image|example (opsional)
          &action=create|update (opsional)
          &limit=20 (1-100) &cursor=<ULID>  - cursor pagination Section 13
   Urutan: ORDER BY id DESC (terbaru dulu), has_more via fetch limit+1.
   Item: { "id", "user_id", "contributor_username", "entity_type",
           "entity_id", "action", "status", "created_at" }
   (contributor_username dari JOIN ke users - untuk tampilan antrean)

2. GET /api/v1/admin/contributions/:id
   Response data: { "contribution": {…item di atas},
                   "review": null | { "reviewer_id", "status",
                                      "comment", "created_at" },
                   "entity": <payload utuh per entity_type> }
   - entity_type 'word' → detail kata lengkap TANPA filter status
     (anak-anak ikut semua status - untuk layar review)
   - 'pronunciation' → { id, word_id, word_lemma, dialect_id, notation,
     value, audio_url, speaker_name, notes, status, is_verified, is_corrected }
   - 'word_image' → { id, word_id, word_lemma, provider, provider_file_id,
     url, alt_text, is_primary, status, is_verified, is_corrected }
   - 'example' → { id, word_id, word_lemma, meaning_id, source_sentence,
     target_sentence, source_type, notes, status, is_verified, is_corrected }
   :id tidak ditemukan / soft-deleted → 404 CONTRIBUTION_NOT_FOUND

3. POST /api/v1/admin/contributions/:id/approve
   Body: { "comment": "…" }   // opsional
   Use case: ReviewContributionUseCase (decision 'approve')
   a. Kontribusi tidak ditemukan → 404 CONTRIBUTION_NOT_FOUND
   b. contributions.status BUKAN 'pending' →
      409 CONTRIBUTION_ALREADY_REVIEWED
   c. SATU TRANSAKSI (repository review()):
      - entity: status='published', is_verified=true; words juga
        verified_by + verified_at
      - contributions.status='approved'
      - INSERT contribution_reviews { reviewer_id (dari token),
        status='approved', comment }
   d. Audit trail (Section 21): action 'approve', entity_type sesuai
      kontribusi, new_data { status, is_verified: true }
   Response 200: { "success": true, "data": { "contribution_id",
     "entity_type", "entity_id", "status": "approved" } }

4. POST /api/v1/admin/contributions/:id/reject
   Body: { "comment": "alasan wajib" }   // WAJIB - kosong → 400
         VALIDATION_ERROR details [{field: "comment"}]
   Use case: ReviewContributionUseCase (decision 'reject')
   - Efek sama seperti approve TAPI entity status='rejected'
     (terminal - kontributor kirim ulang sebagai kontribusi baru),
     contributions.status='rejected', review row status='rejected'
   - Audit: action 'reject', new_data menyertakan comment (alasan)
   - Response 200: { …, "status": "rejected" }

5. POST /api/v1/admin/contributions/:id/correct
   Body: discriminated union pada "entity_type" (WAJIB cocok dengan
   entity_type kontribsi yang dituju - beda → 400 VALIDATION_ERROR):
   - { "entity_type": "word", …payload lengkap sama seperti POST
     admin/words TANPA field status… }        // replace semantics
   - { "entity_type": "pronunciation", "notation": "ipa", "value": "…",
       "dialect_id": null, "audio_url": null, "speaker_name": "…",
       "notes": null }                        // field opsional boleh dihapus
   - { "entity_type": "word_image", "url": "…", "provider_file_id": "…",
       "alt_text": "…", "is_primary": false }
   - { "entity_type": "example", "source_sentence": "…",
       "target_sentence": "…", "target_language_id": null,
       "source_type": null, "notes": null }
   Use case: CorrectContributionUseCase
   a. Pre-check sama seperti approve (404 / 409 / entity_type cocok)
   b. AMBIL SNAPSHOT entity pra-koreksi → audit old_data (WAJIB -
      jejak apa yang diubah verifikator)
   c. Terapkan koreksi + publish + verified:
      - word → WordRepository.updateWithRelations(entityId, {...payload,
        status 'published', isVerified true, isCorrected true}, reviewerId)
      - anak → patch barisnya SET ...field, status='published',
        is_verified=true, is_corrected=true (DI DALAM transaksi review())
   d. contributions.status='corrected' + review row status='corrected'
      (comment opsional di body)
   e. Audit: action 'correct', old_data snapshot, new_data ringkas
   Response 200: { …, "status": "corrected",
     "is_corrected": true }

   CATATAN: correct pada entity word = dua tulis (update entity, lalu
   transaksi review) - bukan satu transaksi. Window kecil, risiko
   diterima; correct pada anak sepenuhnya atomik di review().
   (ponytail: dokumentasikan; naik ke satu tx kalau jadi masalah nyata)

======================================================================
KONTRIBUSI MEDIA - 3 ENDPOINT (menempel pada kata existing)
======================================================================
Middleware SEMUA endpoint di bawah:
authenticate + authorizeRole('admin', 'editor', 'contributor',
'root', 'reviewer') + rateLimit 30 request/menit per user_id
(Section 15 - tier "sudah login, tulis data", key word-media:<user_id>)

6. POST /api/v1/words/:wordId/pronunciations
   Body: { "dialect_id": "01HXYZ…" | null,   // opsional
           "notation": "ipa",                // default 'ipa'
           "value": "/makatn/",
           "audio_url": null,                // opsional
           "speaker_name": null,             // opsional
           "notes": null }                   // opsional
   - wordId tidak ditemukan / soft-deleted → 404 WORD_NOT_FOUND
   - Hasil by role (resolvePublication) - contributor: 201
     { "id", "word_id", "status": "pending_review",
       "is_verified": false, "is_corrected": false, ...field }
     role verifier: 201 { "status": "published", "is_verified": true, ... }
   - Duplikat pelafalan (unique word_id+dialect+notation+value) →
     400 VALIDATION_ERROR field 'value' (mapping kode 23505, pola yang
     sama seperti create-word)
   - Baris contributions: entityType 'pronunciation', entityId=id baru,
     action 'create', status turunan

7. POST /api/v1/words/:wordId/images
   Body: { "url": "https://ik.imagekit.io/…/makan.jpg",
           "provider_file_id": "file_abc123",
           "alt_text": "Ilustrasi orang makan",
           "is_primary": false }
   - Gambar hasil DIRECT UPLOAD client via upload-token (pola modul
     image, 01-api-tambah-kata.md) - backend tidak memvalidasi kepemilikan
     file, cukup url valid + provider_file_id terisi
   - wordId tidak ditemukan → 404 WORD_NOT_FOUND
   - Response & aturan sama seperti endpoint 6 (entityType 'word_image')

8. POST /api/v1/meanings/:meaningId/examples
   Body: { "source_language_id": "01HXYZ…",
           "source_sentence": "Kami udah makatn tadi.",
           "target_language_id": "01HXYZ…",   // opsional
           "target_sentence": "Kami sudah makan tadi.",
           "source_type": "native_speaker",   // opsional
           "notes": null }
   - meaningId tidak ditemukan → 404 MEANING_NOT_FOUND (kode BARU -
     daftarkan di ERROR_CODES.md di PR yang sama)
   - source_language_id harus ada di DB → VALIDATION_ERROR per field
   - Response & aturan sama seperti endpoint 6 (entityType 'example')

======================================================================
KONTRIBUSI ANONIM (TANPA LOGIN)
======================================================================
Pengunjung tanpa akun boleh mengusulkan kata. Semua kontribusi anonim
diatribusikan ke USER SISTEM Anonim (username `anonim`,
anonim@iamutaki.com, ULID stabil 01ANONIM... via shared/constants/anonim,
role contributor, password acak permanen sehingga TIDAK bisa login).
Endpoint publik = reuse use case create-word yang sama dengan actor
Anonim, sehingga resolvePublication otomatis mengarahkan ke
pending_review (tidak pernah langsung tayang). Tanpa field status di
body - dipaksa 'published' (= kirim untuk review; draft anonim tidak
bermakna karena tidak bisa dilanjutkan siapa pun).

12. POST /api/v1/contributions/words   (PUBLIK - tanpa auth)
    Middleware: rate limit 5 request/JAM per IP (tier "publik tulis"
    Section 15 - ketat karena penyalahgunaan anonim tidak bisa diblok
    per akun; di Workers key memakai cf-connecting-ip)
    Body: sama seperti POST /admin/words TANPA field status
    Response 201: bentuk create-word (status selalu pending_review,
    is_verified false). Kontribusi muncul di antrean dengan
    contributor_username 'anonim'.
    400/429: validasi / rate limit.

Untuk mencegah spam lanjutan (di luar rate limit): approval tetap
manual, dan verifikator bisa melihat pola dari IP? - TIDAK: alamat IP
tidak disimpan (privasi); identitas spam hanya terlihat dari pola
volume user Anonim di antrean.

======================================================================
SEARCH MISS - PENCARIAN KOSONG JADI PELUANG KONTRIBUSI
======================================================================
Alur: user A mencari "Kalintiak" (Sambas→Indonesia) → 0 hasil →
istilah TERCATAT otomatis (upsert hit_count, normalized lower/trim,
satu baris per term+direction). Miss muncul di BERANDA user lain
("sedang dicari, belum ada artinya - kontribusikan!") supaya
kontributor mengisi; panel admin melihat semuanya + bisa dismiss.
FULFILMENT DERIVED: begitu ada kata published dengan lemma = term
(case-insensitive), miss otomatis hilang dari beranda dan ber-flag
is_fulfilled di panel - tanpa hook/kolom sinkron.

Pencatatan dilakukan SearchWordsUseCase (modul word) lewat interface
SearchMissRepository - best-effort (kegagalan pencatatan TIDAK boleh
menggagalkan response search). Query < 2 karakter tidak dicatat.

9. GET /api/v1/search-misses   (PUBLIK - beranda)
   Middleware: rate limit 100 req/menit per IP (Section 15, tier baca)
   Query: ?direction=lemma|translation (opsional) &limit=10 (1-50)
   Urutan: hit_count DESC (paling dicari) - top-N halaman tunggal,
   meta.next_cursor selalu null (ponytail: compound-cursor kalau nanti
   perlu paging; order hit_count tidak unik untuk cursor sederhana)
   Hanya miss yang BELUM terjawab (NOT EXISTS kata published).
   Item: { "id", "term", "direction", "hit_count", "last_searched_at",
           "is_fulfilled": false, "created_at" }

10. GET /api/v1/admin/search-misses   (panel admin)
    Middleware: authenticate + authorizeRole('admin','root') + 500/60s
    Query: ?direction=&fulfilled=&limit=20 (1-100) &cursor=<ULID>
    - cursor pagination Section 13 (ORDER BY id DESC)
    Semua miss (terjawab/belum) dengan flag is_fulfilled.

11. POST /api/v1/admin/search-misses/:id/dismiss
    Middleware: sama seperti #10
    Soft delete (Section 7) - miss hilang dari beranda & panel, jejak
    tetap ada. Audit action 'delete', entityType 'search_miss'.
    :id tidak ditemukan → 404 SEARCH_MISS_NOT_FOUND
    Response 200: { "success": true, "data": null }

======================================================================
KONTRAK LINTAS (WAJIB UNTUK SEMUA ENDPOINT DI ATAS)
======================================================================

- Semua response envelope standar Section 13; list pakai meta cursor
  (limit, next_cursor, has_more - TANPA total)
- Route WAJIB createRoute() + app.openapi() (Section 9) - summary,
  tags ['Contributions','Admin'] / ['Words'], schema Zod request DAN
  response (sukses + error)
- Enum status di schema RESPONSE wajib 4 nilai ('draft'|'pending_review'|
  'published'|'rejected') supaya OpenAPI spec tidak basi
- Error code BARU (daftarkan di ERROR_CODES.md di PR yang sama):
  CONTRIBUTION_NOT_FOUND (404), CONTRIBUTION_ALREADY_REVIEWED (409),
  MEANING_NOT_FOUND (404), SEARCH_MISS_NOT_FOUND (404)
- Audit trail (Section 21): aksi 'approve'|'reject'|'correct' lewat
  AuditLogRepository; correct WAJIB membawa old_data snapshot;
  request_id dari context di SEMUA entri audit
- Race double-review: cek contributions.status DI DALAM transaksi -
  dua verifikator klik bersamaan → satu sukses, satu 409 (bukan 500)
- Testing (Section 10): unit per use case (mock repo - verifikasi
  keputusan status per role, comment wajib, urutan pemanggilan),
  integration untuk repository (transaksi review per entity_type,
  double-review 409, filter publik anak), e2e per endpoint minimal
  1 happy + 1 gagal (403/404/409)
- Deliverable termasuk koleksi Bruno: folder http/contribution/ berisi
  .bru per endpoint antrean + http/word/add-*.bru untuk kontribusi
  media + http/search-miss/ + http/auth/login-contributor.bru -
  Section 20, satu PR sama
- Sample response JSON: docs/json/ (folder contribution/ + search-miss/
  + word/) - tiga sumber (OpenAPI, Bruno docs, docs/json) jangan saling
  tertinggal
```

---

## Catatan Implementasi

- Modul contribution TIDAK punya tabel sendiri - dia mengelola
  `contributions` + `contribution_reviews` (schema terpusat, Section 3)
  dan meng-UPDATE entity di tabel modul lain (words, pronunciations,
  word_images, examples) lewat repository sendiri. Komunikasi ke modul
  word tetap lewat interface yang di-export (`WordRepository`) untuk
  updateWithRelations dan detail kata - jaga batas modul (Section 4).
- `review()` di `ContributionRepository` adalah SATU method transaksional:
  load kontribsi (deleted_at IS NULL) → tolak 404/409 → update entity
  per (entity_type × decision) → set contributions.status → INSERT
  contribution_reviews. Use case hanya memutuskan decision + audit.
- Grandfathering: kata published lama TETAP published (jangan
  menayangkan ulang kamus yang sudah hidup); baris contributions lama
  di-backfill 'approved' di migration; admin bisa verify/unverify
  kasus per kasus.
- `rejected` itu terminal untuk fase ini - alur "kontributor revisi &
  resubmit" menyusul di prompt terpisah (butuh endpoint update milik
  contributor + relasi kontribsi ↔ kontribsi baru).
- `main.ts`/`app.ts` me-register:
  `app.route('/api/v1/admin/contributions', createContributionRoutes({...}))`,
  media routes di `/api/v1/words` + `/api/v1/meanings` - media routes
  DI-MOUNT SEBELUM public word routes (public punya rate limit IP global;
  30/menit per-user tetap jadi batas efektif).

## Referensi Terkait

- `api-base-stack.md` - Section 22 (approval gate) jadi acuan model;
  Section 21 (audit), 13 (envelope), 15 (rate limit), 20 (Bruno)
- `01-api-tambah-kata.md` - modul word (create-word memakai
  resolvePublication yang sama; verify/unverify pasca-publikasi)
- `docs/dbdiagram.dbml` - skema: words, pronunciations, word_images,
  examples, contributions, contribution_reviews
- `ERROR_CODES.md` - katalog error code (+3 kode baru)
- repo `http/` - folder `contribution/` + `search-miss/` +
  `word/add-*.bru` + `auth/login-contributor.bru` (Section 20)
- `docs/json/` - sample response per endpoint (folder `contribution/`
  dan `search-miss/`)
- `docs/admin/admin-tambah-kata.md` - UI admin (toast contributor jujur
  menyebut status akhir; halaman antrean review menyusul di prompt
  admin terpisah)
