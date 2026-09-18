# API Vote - Upvote/Downvote Elemen Kamus (Polymorphic)

Mengikuti `api-base-stack.md`: Section 3 (struktur folder), 7 (perubahan
skema), 9 (`@hono/zod-openapi` + Scalar), 10 (testing), 11 (versioning
`/api/v1/`), 13 (envelope & error), 15 (rate limiting), 19 (ULID). Pola
target polymorphic mengikuti tabel `contributions` (`entity_type` +
`entity_id` - preseden satu-satunya di codebase, dipertahankan).

Status fondasi saat dokumen ini dibuat (JANGAN diduplikasi):
- SUDAH ADA: semua tabel target (words, meanings, examples,
  pronunciations, word_images), authenticate + authorizeRole, envelope,
  rateLimit middleware, pola modul 4 lapis, audit trail.
- Yang BELUM: tabel `votes`, modul vote, endpoint, tests, Bruno,
  docs/json.

KEPUTUSAN PRODUK (2026-09-18):
- Vote HANYA user login (semua role termasuk contributor). 1 user = 1
  vote per target - unique constraint, bukan logika aplikasi.
- Perilaku TOGGLE + GANTI ARAH: vote searah kedua kali = batal; vote
  beda arah = replace. (Anonim hanya lihat jumlah.)
- Vote TIDAK di-audit (Section 21) - volume tinggi, bukan aksi admin;
  audit trail untuk aksi moderasi komentar ada di `09-api-comment.md`.

---

## Prompt

```text
Buatkan modul vote (upvote/downvote) untuk backend Kamus Digital
Sambas-Indonesia, mengikuti pola modul word (4 lapis: domain,
application, infrastructure, presentation - Section 3).

LOKASI MODUL: modules/vote/ (BARU - modul lengkap baru)

STRUKTUR FILE YANG PERLU DIBUAT/DIUBAH:

modules/vote/
├── domain/
│   └── repositories/vote.repository.ts        # BARU - interface
├── application/
│   └── use-cases/
│       ├── toggle-vote.use-case.ts            # BARU
│       ├── get-vote-counts.use-case.ts        # BARU
│       └── get-my-votes.use-case.ts           # BARU
├── infrastructure/
│   └── vote.repository.impl.ts                # BARU
└── presentation/v1/
    ├── vote.routes.ts                         # BARU
    ├── vote.controller.ts                     # BARU
    └── validators/vote.validator.ts           # BARU

shared/database/drizzle/schema/votes.schema.ts # BARU
shared/database/drizzle/schema/index.ts        # UBAH: +export votes
app.ts                                         # UBAH: wiring DI + mount

PERUBAHAN SKHEMA (WAJIB - ikuti alur Section 7):
- Tabel BARU `votes`:
    id varchar(26) PK (ULID, Section 19)
    user_id varchar(26) NOT NULL ref users.id
    entity_type varchar(50) NOT NULL  -- word | meaning | example |
                                       -- pronunciation | word_image
    entity_id varchar(26) NOT NULL    -- polymorphic, TANPA FK
                                       -- (preseden contributions)
    value smallint NOT NULL           -- 1 = upvote, -1 = downvote
    created_at timestamp NOT NULL default now
    updated_at timestamp              -- terisi saat ganti arah
- UNIQUE (user_id, entity_type, entity_id) - jaminan 1 user 1 vote,
  sekaligus penjaga race dua toggle bersamaan.
- INDEX (entity_type, entity_id) - agregasi count.
- TIDAK ADA kolom deleted_at/deleted_by: vote mati = baris di-DELETE
  hard (lihat KEPUTUSAN SEMANTIK). Tidak ada counter denormalized -
  jumlah SELALU dihitung on-read (count FILTER) supaya tidak bisa
  drift.
- Alur: pnpm drizzle-kit generate --name=votes → REVIEW SQL →
  pnpm drizzle-kit migrate. TANPA backfill (tabel baru).
- Update docs/dbdiagram.dbml di PR yang sama.

REPOSITORY - interface VoteRepository (domain):
- targetExists(entityType, entityId): Promise<boolean>
  Cek per tabel target by PK + isNull(deleted_at) (switch entity_type;
  tabel tidak ada → ValidationError). TIDAK memfilter status -
  cukup ada & belum dihapus.
- toggle(userId, entityType, entityId, value): Promise<
  { myVote: 1 | -1 | null; upvotes: number; downvotes: number }>
  SATU transaction: DELETE ... WHERE value = :value RETURNING id
  (kena → toggle OFF); kalau tidak kena → INSERT ... ON CONFLICT
  (user_id, entity_type, entity_id) DO UPDATE SET value, updated_at
  (baru / ganti arah). Lalu hitung counts di tx yang sama.
- countMany(targets): Promise<Record<string, { upvotes; downvotes }>>
  key "entity_type:entity_id". Query grouped
  WHERE (entity_type, entity_id) IN (...). Target tanpa vote ikut
  dikembalikan sebagai 0/0 (dilengkapi use case, bukan SQL).
- findUserVotes(userId, targets): Promise<Record<string, 1 | -1>>
  hanya target yang punya vote.

ENDPOINT:

1. POST /api/v1/votes (toggle vote - upvote/downvote/batal)

Middleware: authenticate (SEMUA role) + rateLimit 60/menit per user_id
(Section 15 - lebih longgar dari tier tulis 30/menit karena voting
adalah interaksi ringan, bukan penulisan konten).
Body:
{
  "target_type": "word",        // word | meaning | example |
                                // pronunciation | word_image
  "target_id": "01JDWORDMAKATN0000000000A",  // ULID 26 char
  "value": 1                    // 1 = upvote, -1 = downvote
}
Use case: ToggleVoteUseCase - urutan WAJIB:
a. targetRepo.targetExists → false → 404 VOTE_TARGET_NOT_FOUND
   (target tidak ada / sudah soft-deleted).
b. voteRepo.toggle(...) - satu transaction (lihat repository).
c. TIDAK ADA audit (KEPUTUSAN PRODUK di atas).
Response 200:
{
  "success": true,
  "data": {
    "target_type": "word",
    "target_id": "01JDWORDMAKATN0000000000A",
    "my_vote": 1,        // 1 | -1 | null (null = vote batal)
    "upvotes": 4,
    "downvotes": 1
  }
}

2. GET /api/v1/votes/counts?targets=word:01X,meaning:01Y,example:01Z

(batch jumlah vote - client sudah memegang semua id dari word detail,
TIDAK mengubah response endpoint existing)
Middleware: rateLimit 100/menit per IP (tier baca publik, Section 15).
Query: targets = daftar "type:id" dipisah koma, maks 50 pasang,
duplikat di-dedupe. Format salah → 400 VALIDATION_ERROR details
[{field: "targets"}].
Response 200:
{
  "success": true,
  "data": [
    { "target_type": "word", "target_id": "01X",
      "upvotes": 4, "downvotes": 1 },
    { "target_type": "meaning", "target_id": "01Y",
      "upvotes": 0, "downvotes": 0 }
  ]
}
Target tidak dikenal TIDAK divalidasi eksistensinya di endpoint baca
ini (no-enum check saja) - counts tidak dikenal = 0/0, murah & aman.

3. GET /api/v1/votes/my?targets=... (format sama dengan #2)

(vote milik user login - untuk merender state tombol UI)
Middleware: authenticate (semua role). Tanpa rate limit tambahan
(baca kecil bermakna user sendiri).
Response 200:
{
  "success": true,
  "data": [
    { "target_type": "word", "target_id": "01X", "value": 1 }
  ]
}
Hanya target yang dipilih user yang dikembalikan.

KEPUTUSAN SEMANTIK (disengaja):
- POLYMORPHIC entity_type + entity_id, bukan FK per tabel: satu tabel
  melayani 5+ jenis target; menambah jenis target baru (mis.
  komentar - lihat 09) = satu case di enum + switch cek eksistensi.
  Konsisten dengan contributions. KONSEKUENSI diterima: tidak ada FK
  database ke target (baris target yang soft-deleted dibiarkan -
  counts-nya berhenti dibaca karena client tidak lagi melihat id itu).
- TOGGLE di server, bukan state dari client: client kirim arah yang
  DIPILIH user; server yang memutuskan insert/update/delete. Double
  click = vote hidup-mati cepat; response selalu menyertakan state
  final (my_vote) sehingga UI bisa merekonsiliasi.
- EKSISTENSI target hanya dicek di TOGGLE (endpoint tulis): vote ke
  target yang sudah dihapus → 404 VOTE_TARGET_NOT_FOUND. Endpoint
  counts TIDAK mengecek (baca massal, biar murah).
- TIDAK memfilter status target: kata pending_review pun bisa
  di-vote by id - tidak berbahaya karena tidak pernah tampil publik.
- TANPA audit per vote (KEPUTUSAN PRODUK): volume tinggi (jutaan
  baris), bukan aksi admin, tidak ada nilai forensik. Kalau kelak
  butuh deteksi vote-brigading, analitik tabel votes sendiri cukup.
- VALUE disimpan smallint (+1/-1), bukan boolean: arah eksplisit,
  sum(value) siap dipakai kalau kelak mau skor/net-score.
- SORTING berdasarkan vote (mis. search) TIDAK masuk cakupan ini -
  tambah endpoint/param terpisah kalau produk minta.

CARA DEFINISI ENDPOINT (WAJIB - Section 9): createRoute() +
app.openapi() di vote.routes.ts, tags ['Votes'], schema Zod request
DAN response (sukses + error 400/401/403/404/429). Query targets
divalidasi dengan preprocess: string koma → array "type:id" (regex
^(word|meaning|example|pronunciation|word_image):[0-9A-HJKMNP-TV-Z]{26}$),
maks 50. Factory createVoteRoutes({ controller, authenticate }) -
pola word.routes.ts. Dokumentasi otomatis di /docs.

KEAMANAN & CATATAN:
- user_id SELALU dari token (c.get('user')), tidak pernah dari body.
- Mount /api/v1/votes di app.ts SETELAH route words (urutan tidak
  sensitif - tidak ada prefix bentrok).
- Log event bisnis 'vote toggled' level info + request_id (Section 14),
  TANPA audit trail.
- Deliverable termasuk Bruno: http/vote/toggle-vote.bru,
  get-vote-counts.bru, get-my-votes.bru (Section 20) dengan blok
  tests (toggle 200 → toggle ulang my_vote null; counts 200; 404
  target tidak dikenal; 401 tanpa token).

TESTING (Section 10):
- Unit toggle-vote.use-case.test.ts (mock repository):
  * toggle dipanggil dengan argumen benar; response diteruskan
  * targetExists false → 404 VOTE_TARGET_NOT_FOUND, toggle TIDAK
    dipanggil
- Unit get-vote-counts.use-case.test.ts: target tanpa vote
  dilengkapi 0/0; dedupe.
- Integration vote.repository.impl.test.ts (DB nyata):
  * toggle ON → myVote 1; toggle ulang searah → myVote null &
    baris hilang; toggle beda arah → baris ter-update (bukan baris
    baru); counts benar setelah tiap langkah
  * unique constraint: dua insert paralel user sama target sama
    → satu baris
- E2E vote.e2e.test.ts:
  * happy path: register → login → vote word → 200 my_vote 1 →
    vote ulang → my_vote null → vote -1 → my_vote -1
  * 404 VOTE_TARGET_NOT_FOUND (id target acak), 401 tanpa token,
    400 target_type tidak dikenal
  * counts publik tanpa token → 200; my tanpa token → 401
```

---

## Catatan Implementasi

- Use case TIDAK boleh import Drizzle - transaction toggle hidup di
  `VoteRepositoryImpl` (kontrak "DIJAMIN atomik" seperti WordRepository).
- `countMany` dan `findUserVotes` satu pola: parse targets → query
  `inArray` pasangan (tuple) → lengkapi nol di use case, BUKAN di SQL -
  SQL tetap sederhana.
- Controller tanpa withActor versi admin - cukup `c.get('user')`
  (semua role diizinkan; 401 dari authenticate).
- Query param `targets` panjang (50 x 33 char ≈ 1.7KB) - masih aman
  untuk query string; kalau kelak perlu lebih, naik ke POST batch
  (JANGAN sekarang).
- docs/json/vote/ + baris mapping di docs/json/README.md di PR sama.

## Referensi Terkait

- `api-base-stack.md` - stack, envelope, rate limit, ULID, testing
- `01-api-tambah-kata.md` - bentuk entitas word & children (target vote)
- `03-api-kontribusi-verifikasi.md` - preseden polymorphic
  entity_type/entity_id di tabel contributions
- `09-api-comment.md` - komentar (menambah 'comment' sebagai target
  vote baru di enum + cek eksistensi)
- `docs/dbdiagram.dbml` - skema database (tabel votes BARU)
- `ERROR_CODES.md` - katalog error code (+VOTE_TARGET_NOT_FOUND 404)
- repo `http/` - http/vote/*.bru
