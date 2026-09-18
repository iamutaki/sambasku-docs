# API Comment - Komentar Lemma & Moderasi Admin (Approval Gate)

Mengikuti `api-base-stack.md`: Section 3 (struktur folder), 7 (perubahan
skema), 9 (`@hono/zod-openapi` + Scalar), 10 (testing), 11 (versioning
`/api/v1/`), 13 (envelope & error), 15 (rate limiting), 19 (ULID), 21
(audit trail), 22 (approval gate). Model moderasi = approval gate
Section 22: komentar baru `pending_review`, tampil publik HANYA setelah
disetujui admin/root/reviewer.

Status fondasi saat dokumen ini dibuat (JANGAN diduplikasi):
- SUDAH ADA: modul vote + endpoint counts (08), pola antrean review
  admin (03 - contribution), soft-delete, audit, username via LEFT JOIN
  (pola audit_logs).
- Yang BELUM: tabel `comments`, modul comment, endpoint, tests, Bruno,
  docs/json, tombol UI admin & area komentar mobile.

KEPUTUSAN PRODUK (2026-09-18):
- Komentar HANYA oleh user login (semua role termasuk contributor).
- PRE-MODERATION: komentar langsung `pending_review`, tampil publik
  hanya setelah approve. Admin mengontrol penuh apa yang layak tampil.
- Komentar terikat ke LEMMA (word), bukan ke makna/contoh - kolom
  `word_id` FK langsung, TIDAK polymorphic.
- Vote boleh menarget komentar (target_type 'comment') - diskusi juga
  elemen yang layak dinilai.

---

## Prompt

```text
Buatkan modul comment (komentar pada lemma + antrean moderasi admin)
untuk backend Kamus Digital Sambas-Indonesia, mengikuti pola modul
contribution (antrean review) dan word (soft-delete).

LOKASI MODUL: modules/comment/ (BARU - modul lengkap baru)

STRUKTUR FILE YANG PERLU DIBUAT/DIUBAH:

modules/comment/
├── domain/
│   ├── entities/comment.entity.ts              # BARU
│   └── repositories/comment.repository.ts      # BARU - interface
├── application/
│   └── use-cases/
│       ├── create-comment.use-case.ts          # BARU
│       ├── list-word-comments.use-case.ts      # BARU
│       ├── delete-comment.use-case.ts          # BARU
│       ├── list-admin-comments.use-case.ts     # BARU
│       └── review-comment.use-case.ts          # BARU (approve|reject)
├── infrastructure/
│   └── comment.repository.impl.ts              # BARU
└── presentation/v1/
    ├── comment.routes.ts                       # BARU (publik + user)
    ├── admin-comment.routes.ts                 # BARU (moderasi)
    ├── comment.controller.ts                   # BARU
    └── validators/comment.validator.ts         # BARU

modules/vote/
├── domain/repositories/vote.repository.ts      # UBAH: +'comment' di
│                        # cek targetExists (case baru di switch)
└── presentation/v1/validators/vote.validator.ts # UBAH: enum +regex
                         # target_type menerima 'comment'

shared/database/drizzle/schema/comments.schema.ts # BARU
shared/database/drizzle/schema/index.ts           # UBAH: +export
app.ts                                            # UBAH: wiring + mount

PERUBAHAN SKHEMA (WAJIB - ikuti alur Section 7):
- Tabel BARU `comments`:
    id varchar(26) PK (ULID, Section 19)
    word_id varchar(26) NOT NULL ref words.id   -- terikat lemma
    user_id varchar(26) NOT NULL ref users.id   -- penulis
    body text NOT NULL                          -- 1..1000 char (trim)
    status varchar(30) NOT NULL default 'pending_review'
        -- kosakata Section 22: pending_review | published | rejected
    reviewed_by varchar(26) ref users.id        -- terisi saat moderasi
    reviewed_at timestamp
    created_at/updated_at timestamp, deleted_at/deleted_by (konvensi)
- INDEX (word_id, status, id) - list per kata; INDEX (status, id) -
  antrean moderasi.
- Alur: pnpm drizzle-kit generate --name=comments → REVIEW SQL →
  pnpm drizzle-kit migrate. TANPA backfill (tabel baru).
- Update docs/dbdiagram.dbml di PR yang sama.
- TIDAK menulis baris contributions (antrean 03 tetap khusus entitas
  kamus; komentar punya antrean sendiri - lihat KEPUTUSAN SEMANTIK).

REPOSITORY - interface CommentRepository (domain):
- create(data): Promise<Comment>
- listByWord(wordId, { limit, cursor }): Promise<CursorPage<Comment>>
  status 'published' + isNull(deleted_at), urut id DESC (terbaru dulu),
  LEFT JOIN users → username: string | null (penulis terhapus).
- findById(id): Promise<Comment | null> (belum soft-deleted)
- softDelete(id, actorId): Promise<boolean>  (pola WordRepository)
- listAdmin({ status, limit, cursor }): Promise<CursorPage<Comment>>
  semua status (belum terhapus), LEFT JOIN users.
- review(id, decision, reviewerId): Promise<boolean>
  SATU UPDATE ... WHERE id = :id AND status = 'pending_review' AND
  deleted_at IS NULL SET status, reviewed_by, reviewed_at RETURNING id
  → false = tidak ada / sudah direview (race dua moderator → 409).

ENDPOINT:

1. GET /api/v1/words/:wordId/comments (list komentar yang tampil)

Middleware: rateLimit 100/menit per IP (tier baca publik, Section 15).
Dipasang di route sendiri - MOUNT DI /api/v1/words SEBELUM
createPublicWordRoutes (pola media routes: urutan mount penting supaya
tidak tertelan routes.use('*') public word).
Query: limit (default 20, max 50), cursor (optional).
Response 200:
{
  "success": true,
  "data": [
    {
      "id": "01JCMT...",
      "word_id": "01JDWORD...",
      "user_id": "01JUSER...",
      "username": "budi",        // null kalau penulis terhapus
      "body": "Kata ini juga sering saya dengar di Sambas",
      "created_at": "2026-09-18T10:00:00Z",
      "upvotes": 3,              // vote count per komentar
      "downvotes": 0             // (VoteRepository.countMany)
    }
  ],
  "meta": { "limit": 20, "next_cursor": null, "has_more": false }
}

2. POST /api/v1/words/:wordId/comments (tulis komentar)

Middleware: authenticate (semua role) + rateLimit 30/menit per user_id
(tier tulis login, Section 15).
Body:
{
  "body": "Kata ini juga sering saya dengar di Sambas"
  // WAJIB, trim, 1..1000 karakter - kosong/panjang → 400
}
Use case: CreateCommentUseCase - urutan WAJIB:
a. wordRepo.findById(wordId) → null → 404 WORD_NOT_FOUND (kata
   tidak ada / soft-deleted; TIDAK memfilter status - konsisten
   dengan vote 08).
b. commentRepo.create({ wordId, userId, body, status:
   'pending_review' }).
c. AUDIT (Section 21): action 'create', entity_type 'comment',
   new_data { word_id, body }, request_id dari context.
Response 201:
{
  "success": true,
  "data": {
    "id": "01JCMT...", "word_id": "...", "user_id": "...",
    "username": "budi", "body": "...",
    "status": "pending_review",
    "created_at": "2026-09-18T10:00:00Z"
  }
}
Status pending_review = belum tampil publik (client menampilkan
pesan "menunggu moderasi").

3. DELETE /api/v1/comments/:id (hapus komentar sendiri / oleh admin)

Middleware: authenticate (semua role) + rateLimit 30/menit per user_id.
Use case: DeleteCommentUseCase:
a. commentRepo.findById(id) → null → 404 COMMENT_NOT_FOUND.
b. Otorisasi: actor adalah PENULIS ATAU role admin/root/reviewer -
   selain itu → 403 FORBIDDEN (penulis lain tidak bisa menghapus
   komentar orang).
c. commentRepo.softDelete(id, actorId) → false → 404 (race).
d. AUDIT: action 'delete', entity_type 'comment', old_data
   { word_id, body, status }, request_id.
Response 200: { "success": true, "data": null }

4. GET /api/v1/admin/comments?status=pending_review&limit=&cursor=

(antrean moderasi - pola antrean contribution 03)
Middleware: authenticate + authorizeRole('admin', 'root', 'reviewer')
+ rateLimit 500/menit per IP (tier admin, Section 15).
Query: status optional (default 'pending_review'; nilai valid:
pending_review | published | rejected), limit (default 20, max 50),
cursor.
Response 200: data = komentar + username + status + reviewed_by +
reviewed_at, meta cursor standar.

5. POST /api/v1/admin/comments/:id/approve

(tampilkan komentar - satu-satunya jalur ke published)
Middleware: sama dengan #4.
Use case: ReviewCommentUseCase(decision 'approve'):
a. commentRepo.findById → null → 404 COMMENT_NOT_FOUND.
b. commentRepo.review(id, 'approve', reviewerId) → false →
   409 COMMENT_ALREADY_REVIEWED (sudah punya keputusan / terhapus;
   race dua moderator klik bersamaan → satu 200, satu 409).
c. AUDIT: action 'approve', entity_type 'comment', old_data
   { status: 'pending_review' }, new_data { status: 'published' }.
Response 200: { "success": true, "data": { "id": "...",
  "status": "published", "reviewed_by": "...", "reviewed_at": "..." } }

6. POST /api/v1/admin/comments/:id/reject

Middleware: sama dengan #4.
Use case: ReviewCommentUseCase(decision 'reject') - langkah IDENTIK
dengan #5 (hanya decision & status akhir 'rejected'). TIDAK ada
kolom alasan (lihat KEPUTUSAN SEMANTIK).
Response 200: bentuk sama dengan #5, status "rejected".

KEPUTUSAN SEMANTIK (disengaja):
- TERIKAT LEMMA, bukan per makna/contoh: kolom word_id FK langsung -
  bentuk paling sederhana yang match keputusan produk; kalau kelak
  mau komentar per makna, itu fitur baru (target polymorphic ala
  votes), JANGAN digenapi sekarang.
- PRE-MODERATION penuh: tidak ada jalur komentar langsung tampil.
  Kosakata status IKUT konten Section 22 (pending_review/published/
  rejected), BUKAN kosakata workflow contributions (pending/approved)
  - komentar adalah konten, bukan record perubahan entitas.
- ANTREAN SENDIRI, tidak menyusup ke contributions: tabel
  contributions tetap murni entitas kamus (word/pronunciation/
  word_image/example); komentar punya lifecycle & pemilik antrean
  berbeda. Dua antrean di UI admin, dua endpoint list terpisah.
- REJECT TANPA alasan: keputusan moderasi komentar tidak butuh
  justifikasi tertulis untuk penulis (beda dari kontribusi kata yang
  reject WAJIB comment - kontributor menginvestasikan konten, komenter
  berkomentar ringan). Kelak mau notifikasi + alasan → fitur terpisah.
- PENULIS bisa hapus SENDIRI sebelum/sesudah moderasi: soft-delete
  menyembunyikan dari semua list; admin tetap bisa lihat jejaknya di
  audit trail. Admin/root/reviewer bisa hapus komentar SIAPA PUN.
- TIDAK ADA edit komentar: salah tulis → hapus + tulis ulang. Sederhana,
  dan riwayat moderasi tetap bersih.
- TIDAK ADA threading/balasan: flat, terbaru dulu. Kalau produk minta
  balasan → fitur parent_id terpisah.
- username DITAMPILKAN (bukan anonim): komentar login-only, identitas
  terang; penulis terhapus → username null, client menampilkan
  "pengguna terhapus".
- word pending_review BOLEH dikomentari (konsisten vote 08: cek ada +
  belum terhapus saja) - komentar tidak tampil di mana pun selama
  word-nya belum published.
- VOTE ke komentar: hanya menambah 'comment' pada enum target vote +
  case cek eksistensi (findById belum soft-deleted). TIDAK ada
  endpoint vote baru - toggle/counts/my (08) langsung jalan.
- COMMENT count TIDAK ditambahkan ke response word detail - client
  cukup fetch list komentar (badge jumlah = nice-to-have UI, kalau
  produk minta → endpoint counts komentar terpisah).

CARA DEFINISI ENDPOINT (WAJIB - Section 9): createRoute() +
app.openapi() di comment.routes.ts / admin-comment.routes.ts, tags
['Comments'] / ['Comments', 'Admin'], schema Zod request DAN response
(sukses + error). Factory ({ controller, authenticate }) - pola
word.routes.ts. Enum status response 3 nilai (pending_review |
published | rejected) supaya OpenAPI spec tidak basi. Mount di app.ts:
- /api/v1/words (route komentar per kata) SEBELUM createPublicWordRoutes
- /api/v1/comments (DELETE)
- /api/v1/admin/comments (moderasi)
Dokumentasi otomatis di /docs.

KEAMANAN & CATATAN:
- user_id & reviewer SELALU dari token, tidak pernah dari body.
- Rate limit terpasang sejak endpoint dibuat (Section 15).
- Log event bisnis ('comment created', 'comment approved', dst.)
  level info + request_id (Section 14).
- Body komentar DISIMPAN apa adanya (plain text, di-trim) - client
  bertanggung jawab escape saat render (API tidak mengirim HTML).
- Deliverable termasuk Bruno: http/comment/list-comments.bru,
  create-comment.bru, delete-comment.bru, admin-list-comments.bru,
  approve-comment.bru, reject-comment.bru (Section 20) dengan blok
  tests (201 → list kosong → approve → list tampil; 403, 404, 409).

TESTING (Section 10):
- Unit per use case (mock repository):
  * create: word hilang → 404; status awal 'pending_review'; audit
    'create' tercatat (new_data.body)
  * delete: penulis sendiri OK; user lain → 403; admin OK;
    id tidak ada → 404
  * review: decision approve/reject menulis reviewed_by/at; review
    false → 409 COMMENT_ALREADY_REVIEWED; audit old_data.status +
    new_data.status
- Integration comment.repository.impl.test.ts: listByWord hanya
  published & belum terhapus + urut terbaru; listAdmin filter status;
  review WHERE status pending (review ganda → false).
- E2E comment.e2e.test.ts:
  * happy path: login → POST → 201 pending_review → GET list KOSONG
    → login admin → approve → GET list TAMPIL (username + counts)
  * reject → tidak tampil di list publik
  * 401 tanpa token, 403 contributor akses antrean admin,
    404 word tidak ada / komentar tidak ada, 409 approve dua kali,
    penulis lain delete → 403
  * vote komentar: toggle-vote target_type 'comment' → counts di
    list komentar bertambah
```

---

## Catatan Implementasi

- Modul comment bergantung ke DUA interface modul lain via constructor:
  `WordRepository` (cek kata ada) dan `VoteRepository` (counts per
  komentar) - preseden: auditRepo di-inject lintas modul dari app.ts.
- ReviewCommentUseCase SATU use case dua decision (param 'approve' |
  'reject') - langkah identik, duplikasi use case hanya menambah file.
- Otorisasi delete (penulis ATAU verifikator) ada di USE CASE (butuh
  data komentar), bukan middleware - berbeda dari endpoint admin murni.
- listByWord + counts digabung di use case: page → kumpulkan id →
  voteRepo.countMany → tempel ke response. Dua query, bukan N+1.
- UI admin (antrean komentar + tombol approve/reject) & area komentar
  mobile menyusul di docs/admin + docs/mobile - di luar cakupan
  dokumen ini.

## Referensi Terkait

- `08-api-upvote-downvote.md` - modul vote yang digenapi (+target
  'comment') dan sumber counts per komentar
- `03-api-kontribusi-verifikasi.md` - pola antrean review admin &
  race double-review → 409
- `07-api-delete-kata.md` - pola soft-delete repository
- `api-base-stack.md` - Section 22 (approval gate - komentar ikut
  model ini), Section 15, 21
- `02-api-audit-logs.md` - kontrak audit (action create/delete/
  approve/reject, entity_type 'comment')
- `docs/dbdiagram.dbml` - skema database (tabel comments BARU)
- `ERROR_CODES.md` - katalog error code (+COMMENT_NOT_FOUND 404,
  +COMMENT_ALREADY_REVIEWED 409)
- repo `http/` - http/comment/*.bru
