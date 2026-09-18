# API Word - Soft-delete Kata (Admin)

Mengikuti `api-base-stack.md`: Section 3 (struktur folder), 9
(`@hono/zod-openapi` + Scalar), 10 (testing), 11 (versioning `/api/v1/`),
13 (envelope & error), 15 (rate limiting), 19 (ULID), 21 (audit trail),
22 (approval gate). KEPUTUSAN SEMANTIK mengikuti `05-api-edit-kata.md` -
dokumen ini hanya mendefinisikan endpoint DELETE-nya.

Status fondasi saat dokumen ini dibuat (JANGAN diduplikasi):
- `WordRepositoryImpl.softDelete(id, actorId)` SUDAH ADA di impl - set
  `deleted_at` + `deleted_by`, filter `isNull(deleted_at)`, return apakah
  ada baris yang di-update. SEMUA query `findDetailById`/`findById`/
  `search`/`findMissingReferences` sudah memfilter `isNull(deletedAt)`
  → setelah soft-delete, kata otomatis hilang dari publik & admin.
- Yang BELUM: `softDelete` di interface repository, use case, endpoint
  DELETE, controller, tests, Bruno, UI admin (tombol "Hapus" di daftar
  kata - `docs/admin/03-delete-kata.md`).

---

## Prompt

```text
Buatkan endpoint soft-delete kata untuk modul word, mengikuti pola
VerifyWordUseCase (aksi + audit sederhana) dan edit kata (05 doc:
role verifier team, approval gate Section 22, audit Section 21).

LOKASI MODUL: modules/word/ (sudah ada - LENGKAPI, jangan buat modul baru)

STRUKTUR FILE YANG PERLU DIBUAT/DIUBAH:

modules/word/
├── application/
│   └── use-cases/
│       └── soft-delete-word.use-case.ts   # BARU
├── domain/repositories/word.repository.ts # UBAH: +softDelete di interface
├── presentation/v1/
│   ├── word.routes.ts                     # UBAH: +route DELETE di grup /:id
│   ├── word.controller.ts                 # UBAH: +deleteWord
│   └── validators/                        # TIDAK ADA validator baru (tanpa body)
└── __tests__/
    ├── unit/soft-delete-word.use-case.test.ts   # BARU
    └── e2e/v1/delete-word.e2e.test.ts           # BARU

PERUBAHAN REPOSITORY (SATU-SATUNYA - interface)
- Tambahkan softDelete(id: string, actorId: string): Promise<boolean>
  pada interface WordRepository. Implementasi Drizzle SUDAH ADA
  (word.repository.impl.ts) - tidak ada perubahan impl.

TIDAK ADA perubahan skema database / migration / tabel baru (kolom
deleted_at & deleted_by sudah ada). TIDAK ADA error code baru
(WORD_NOT_FOUND, UNAUTHORIZED, FORBIDDEN sudah ada di ERROR_CODES.md).

ENDPOINT:

DELETE /api/v1/admin/words/:id
  (soft-delete kata - hilang dari publik & admin, baris dipertahankan)

Middleware: authenticate + authorizeRole('admin', 'editor', 'root',
'reviewer') + rate limit 30/menit per user_id - DAFTARKAN route di grup
/:id yang SUDAH memakai middleware persis ini (group yang sama dengan
GET prefill & PUT edit), jangan buat grup baru.
Body: TIDAK ADA (tanpa body, tanpa validator baru).
Params: id ULID 26 char (Section 19) - z.object({ id: z.string().length(26) }).

Use case: SoftDeleteWordUseCase - urutan WAJIB:

a. MUAT SNAPSHOT: wordRepo.findById(cmd.wordId)
   → null (tidak ada / SUDAH soft-deleted) → 404 WORD_NOT_FOUND.
   Snapshot dipakai utk audit old_data. SEMUA status boleh dihapus
   (draft/pending_review/published/rejected) - findById tidak memfilter
   status, hanya deleted_at.
b. SOFT-DELETE: wordRepo.softDelete(cmd.wordId, actorId)
   → false (kalah race: dihapus antara langkah a & b) → 404 WORD_NOT_FOUND.
c. AUDIT (Section 21): action 'delete', entity_type 'word',
   entity_id, old_data { lemma, word_type, language_id, status,
   is_verified } dari snapshot (a), request_id dari context.
   new_data TIDAK ADA (baris tetap utuh; tidak relevan).

Response sukses (200) - envelope standar Section 13, bentuk sama dengan
verify/unverify: { "success": true, "data": null }.

Response gagal: envelope standar (401/403/404/429/500) - pola persis
update; controller TIDAK menulis handler 500 sendiri.

KEPUTUSAN SEMANTIK (disengaja):
- SOFT-DELETE, bukan hard delete: baris + children tetap utuh untuk
  audit/recovery; tanpa migration, banyak query yang sudah memfilter
  deleted_at jadi otomatis konsisten. Kalau kelak mau "restore",
  tinggal set deleted_at = null (fitur terpisah - JANGAN masuk prompt
  ini).
- IDEMPOTEN di sisi user tapi 404 di sisi server: delete kata yang
  sudah dihapus → 404 WORD_NOT_FOUND (bukan 200 ganda). Konsisten
  dengan race soft-delete di update/verify, dan memberi sinyal jelas.
- Role SAMA dengan edit kata (admin/editor/root/reviewer) - bagian tim
  verifikator. Contributor 403: menghapus entri bukan jalur kontribusi.
  (Kalau kelak mau mempersempit ke admin/root saja, itu keputusan
  produk terpisah - cukup ubah authorizeRole.)
- children (meanings/pronunciations/images/examples/relations/variants)
  TIDAK di-touch: soft-delete hanya memflip kata. Kata lain yang
  "menunjuk" kata terhapus (lewat related_words) akan mendapat
  VALIDATION_ERROR saat diedit (findMissingReferences menolak id yang
  sudah soft-deleted) - self-healing, pengguna tinggal menghapus relasi.
- Kata yang dihapus TIDAK bisa diedit/di-verify lagi: findById &
  findDetailById memfilter deleted_at → 404.

CARA DEFINISI ENDPOINT (WAJIB - Section 9): createRoute() + app.openapi()
di word.routes.ts (DAFTARKAN dalam grup /:id yang sudah memakai
middleware yang sama, sejajar adminWordDetailRoute & updateWordRoute),
params Zod langsung di route (tanpa validator file baru), tags
['Words', 'Admin'], summary jelas (soft-delete + konsekuensi). Response
200 memakai okNullResponseSchema (sudah dipakai verify/unverify).
Dokumentasi otomatis di /docs.

KEAMANAN & CATATAN:
- deleted_by SELALU dari token (c.get('user')), tidak pernah dari body.
- Rate limit terpasang sejak endpoint dibuat (Section 15) - grup /:id
  sudah membawa rateLimit 30/menit per user_id.
- Log event bisnis 'word soft-deleted' level info + request_id (Section 14).
- Controller memakai withActor (pola addPronunciation/addWordImage) →
  401 otomatis kalau token tidak ada.
- Deliverable termasuk Bruno: http/word/delete-word.bru (Section 20)
  dengan blok tests (status 200 + data null, delete kedua → 404).

TESTING (Section 10):
- Unit soft-delete-word.use-case.test.ts (mock repository):
  * softDelete(id, actorId) dipanggil + audit action 'delete' memuat
    old_data { lemma, word_type, language_id, status, is_verified }
  * findById null → 404 WORD_NOT_FOUND, softDelete TIDAK dipanggil,
    audit TIDAK direcord
  * softDelete false (kalah race) → 404 WORD_NOT_FOUND, audit TIDAK
    direcord
- E2E delete-word.e2e.test.ts:
  * happy path: login admin → create → DELETE → 200 { success, data:
    null } → GET publik 404 & GET admin detail 404 → audit 'delete'
    tercatat (old_data.lemma)
  * idempotent: DELETE kedua → 404
  * 404 id tidak dikenal (WORD_NOT_FOUND), 403 contributor, 401 tanpa
    token
  * DRAFT juga bisa dihapus (semua status boleh)
```

---

## Catatan Implementasi

- Use case TIDAK boleh import Drizzle - soft-delete tetap satu UPDATE
  singkat di impl (`softDelete`), atomicity & filter `isNull(deletedAt)`
  hidup di repository.
- Interface `softDelete` diposisikan tepat di bawah `setVerified` -
  pasangan aksi status yang sama-sama "flip kolom + false → 404".
- Route DELETE DI DALAM grup `/:id` (bukan grup `/`): otomatis mewarisi
  `authorizeRole('admin','editor','root','reviewer')` + rate limit yang
  sama dengan GET prefill & PUT edit - tidak ada middleware duplikat.
- Response `data: null` dengan `okNullResponseSchema` - konsisten dengan
  verify/unverify; `promise<Promise<void>>` di sisi kontrak.
- `softDelete` tidak akan balik ke interface yang rusak: impl sudah
  mengembalikan boolean; kosong → false; bukan ekspor.
- Frontend admin mengonsumsi endpoint ini lewat tombol "Hapus"
  (Popconfirm) di daftar kata - lihat `docs/admin/03-delete-kata.md`.

## Referensi Terkait

- `05-api-edit-kata.md` - keputusan role verifier team & pola
  use case/audit yang digenapi dokumen ini
- `01-api-tambah-kata.md` - kontrak induk bentuk kata/status
- `api-base-stack.md` - stack, envelope, audit, approval gate, rate limit
- `03-api-kontribusi-verifikasi.md` - antrean review (jalur perubahan
  contributor atas entri existing - DELETE TIDAK buka jalur ini)
- `02-api-audit-logs.md` - kontrak bukti audit (action 'delete')
- `docs/dbdiagram.dbml` - skema database (kolom deleted_at/deleted_by,
  TIDAK berubah)
- `ERROR_CODES.md` - katalog error code (tidak ada kode baru)
- repo `http/` - `http/word/delete-word.bru`
- `docs/admin/03-delete-kata.md` - UI admin tombol "Hapus"