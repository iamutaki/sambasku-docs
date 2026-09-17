# API Audit - Jejak Audit Mutasi Data

Mengikuti `api-base-stack.md`: Clean Architecture feature-based
(`modules/audit/`) - Section 3 (struktur folder), 9 (`@hono/zod-openapi`),
10 (testing), 11 (versioning `/api/v1/`), 13 (envelope & error),
15 (rate limiting), 21 (audit trail - fondasi modul ini).

Modul ini punya dua sisi:
- **Sisi tulis** (tanpa endpoint): `AuditLogRepository` di-inject ke use
  case modul lain - sudah berlaku untuk register (`user.create`),
  reset password (`user.password_change`), dan create word (`word.create`)
- **Sisi baca** (endpoint di bawah): untuk auditor - **hanya role
  `admin` dan `root`**

---

## Prompt

```text
Buatkan modul "audit" (pembaca jejak audit) untuk backend Kamus Digital
Sambas-Indonesia, mengikuti struktur clean architecture feature-based
yang sudah ditetapkan di api-base-stack.md. Konvensi audit trail ada
di Section 21 api-base-stack.md - modul ini adalah sisi pembacanya.

STACK:
- Hono (Node.js runtime) - route pakai OpenAPIHono (Section 9)
- Drizzle ORM + PostgreSQL (tabel audit_logs sudah ada di skema)
- Zod untuk validasi query/response (Section 9)

LOKASI MODUL: modules/audit/

STRUKTUR FILE:

modules/audit/
├── domain/
│   ├── entities/
│   │   └── audit-log.entity.ts
│   └── repositories/
│       └── audit-log.repository.ts       # interface - DIEKSPOR untuk
│                                         # use case modul lain (Section 4)
├── application/
│   └── use-cases/
│       └── list-audit-logs.use-case.ts
├── infrastructure/
│   └── audit-log.repository.impl.ts
└── presentation/
    └── v1/
        ├── audit.routes.ts
        ├── audit.controller.ts
        └── validators/
            └── audit-log.validator.ts

ENDPOINT (semua di bawah prefix /api/v1 - Section 11):

1. GET /api/v1/admin/audit-logs
   Middleware: authenticate + authorizeRole('admin', 'root')
   + rateLimit 500/menit (kategori admin, Section 15)

   Query (semua opsional, kombinasi AND):
   - user_id     ULID 26 char - jejak per pelaku
   - entity_type string       - 'user' | 'word' | dst
   - entity_id   ULID 26 char - riwayat satu entitas
   - from, to    ISO datetime - rentang created_at
   - limit       int (default 20, max 100) - jumlah item per halaman
   - cursor      ULID 26 char - id item terakhir halaman sebelumnya
                                 (opsional; kosong = halaman pertama)

   Use case: ListAuditLogsUseCase
   - Urut id DESC (terbaru dulu; ULID ≈ created_at time-sortable, Section 19)
   - Cursor-based pagination (Section 13): WHERE id < cursor, LIMIT limit+1
     untuk deteksi has_more
   - Response 200 - envelope + meta cursor-based (Section 13):
     { "success": true,
       "data": [ { "id": "01ARZ…", "user_id": "01ARZ…",
                   "action": "create", "entity_type": "word",
                   "entity_id": "01ARZ…",
                   "old_data": null,
                   "new_data": { "lemma": "makatn", "status": "published" },
                   "request_id": "a1b2c3…",
                   "created_at": "2026-09-10T01:00:00.000Z" } ],
       "meta": { "limit": 20,
                 "next_cursor": "01ARZ3NDEKTSV4RRFFQ69G5FAV" | null,
                 "has_more": true | false } }
     next_cursor = id item terakhir di data[] kalau has_more, selainnya null;
     client memakai ini di query param cursor untuk halaman berikutnya
     (pola "Muat lagi" / infinite scroll - TANPA total_items/total_pages).

   Response gagal - envelope standar:
   - 400 VALIDATION_ERROR (query tidak valid) + details
   - 401 UNAUTHORIZED / TOKEN_EXPIRED (dari authenticate)
   - 403 FORBIDDEN - role selain admin/root (editor, contributor,
     reviewer, publik)

KEAMANAN & CATATAN:
- KEPUTUSAN DESAIN: pembatasan role dilakukan di route (middleware),
  use case murni query tanpa tahu siapa pemanggilnya
- new_data/old_data sudah dijaga bersih di sisi tulis (Section 21:
  tanpa password/token) - sisi baca TIDAK perlu menyaring ulang, tapi
  JANGAN pernah menambahkan field sensitif ke payload audit
- Endpoint ini READ-ONLY - modul audit tidak menyediakan write endpoint
  apapun (penulisan hanya dari use case modul lain)
- Ekspor/dump CSV menyusul sebagai prompt terpisah kalau dibutuhkan

CARA DEFINISI ENDPOINT (WAJIB - Section 9):
- createRoute() + app.openapi() dengan tags ['Audit'], summary, dan
  schema Zod untuk query + response (sukses + error)
- Schema response HARUS envelope standar (Section 13)

TESTING (Section 10):
- Unit: ListAuditLogsUseCase - mapping filter + transform ke
  meta { limit, next_cursor, has_more }
- Integration: AuditLogRepositoryImpl - record + list (filter
  entity_type/user_id, cursor pagination 2 halaman tanpa overlap,
  has_more & next_cursor sesuai, urut terbaru dulu) + bukti kontrak
  best-effort (insert gagal → TIDAK melempar)
- E2E: admin 200 + meta cursor (limit/next_cursor/has_more) + halaman
  berikutnya via cursor, contributor 403, tanpa token 401,
  query invalid (cursor bukan ULID) 400

FORMAT RESPONSE - ENVELOPE STANDAR (Section 13 api-base-stack.md).
Error code yang dipakai semuanya sudah ada di ERROR_CODES.md.
```

---

## Catatan Implementasi

- Sisi tulis sudah aktif di modul lain lewat interface yang modul ini
  ekspor: register (`user.create`), reset password (`user.password_change`),
  create word (`word.create`) - semuanya membawa `request_id` dari
  `requestIdMiddleware`.
- `record()` best-effort per kontrak Section 21: gagal insert hanya
  di-log `error`, request utama tidak ikut gagal.
- Bruno: `http/audit/list-audit-logs.bru` (pakai `access_token` admin
  hasil login koleksi auth).

## Referensi Terkait

- `api-base-stack.md` - Section 21 (konvensi audit trail), Section 14
  (request_id), Section 15 (rate limit admin)
- `00-api-auth.md`, `01-api-tambah-kata.md` - modul penulis audit
- `docs/dbdiagram.dbml` - definisi tabel `audit_logs`
- `ERROR_CODES.md` - katalog error code
