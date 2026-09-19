# Admin UI - Moderasi Komentar (antrean approve/reject)

Mengikuti `admin-base-stack.md`: Section 2-3 (clean architecture feature-based),
6 (router/guard), 7-8 (session + axios), 10 (React Query), 13 (error handling),
15 (responsive + action bar). Kontrak API yang dikonsumsi: `09-api-comment.md`
Section 4-6 - `GET /api/v1/admin/comments` (antrean moderasi), `POST /api/v1/admin/comments/:id/approve|reject`, dan 409 `COMMENT_ALREADY_REVIEWED`
pada balapan dua moderator.

Status fondasi saat dokumen ini dibuat (JANGAN diduplikasi):

- SUDAH ADA (API): seluruh endpoint moderasi di backend (management route,
  role gate admin/root/reviewer, antrean per status + cursor pagination,
  approve/reject WHOLE-ROW, no reason column by design, soft-delete).
- SUDAH ADA (admin): pola feature-based (`features/contributions`), `DataTable`
  (TanStack → antd bridge), `PageHeader`, `useCursorList`, envelope + `ApiError`
  (409 punya `error_code` yang bisa dibaca), router manual + `MENU_ROUTES`.
- SUDAH ADA (mobile): fitur komentar (tulis/daftar/vote/hapus) selesai; semua
  komentar masuk `pending_review` dan hanya muncul setelah di-approve admin.
- Yang BELUM: seluruh UI moderasi di admin (tidak ada fitur comments sama
  sekali di `admin/`).

---

## Prompt

```text
Buatkan halaman "Moderasi Komentar" di admin, mengikuti pola fitur
contributions + komponen bersama (DataTable/PageHeader/useCursorList).

KONTEKS:
- Satu-satunya jalur komentar menjadi visible (published) adalah approve
  oleh admin/root/reviewer; antrean default = pending_review.
- Backend mensortir per status (pending_review|published|rejected) dengan
  cursor pagination - `GET /admin/comments?status=&limit=&cursor=`; item
  `{ id, word_id, user_id, username, body, status, reviewed_by,
    reviewed_at, created_at }` + meta `{ limit, next_cursor, has_more }`.
- Keputusan `POST /admin/comments/:id/approve|reject` TANPA body (no reason
  column by design) → 200 `{ id, status, reviewed_by, reviewed_at }`.
- Balapan dua moderator me-review komentar sama = 409 `COMMENT_ALREADY_REVIEWED`
  → tampilkan pesan error + REFETCH list (state backend sudah berubah).
- Halaman TIDAK memanggil client langsung - semuanya lewat infrastructure
  request fn (aturan base-stack); keputusan lewat mutation + invalidasi
  queryKey prefix ['comments'] (sama seperti review kontribusi).

STRUKTUR:
1. Halaman tunggal /comments - Tabs filter status (Menunggu/Diterbitkan/
   Ditolak, default Menunggu), list cursor-paginated via useCursorList,
   tombol "Muat lagi" saat has_more, Alert error + refetch.
   Kolom: komentar (ellipsis expandable), penulis (fallback 'Pengguna
   terhapus' saat username null), kata (link ke /words/$id), status (Tag),
   dikirim, direview, aksi (hanya baris pending_review).
2. Aksi approve/reject = modal konfirmasi (App.useApp().modal) + tombol
   CheckOutlined/CloseOutlined dengan loading per baris; onSuccess →
   message.success + invalidate ['comments']; onError (409) →
   message.error(normalizeError(err).message) + refetch.
3. ROLE GATE: list enabled hanya untuk admin/root/reviewer (Guard layout
   sudah melindungi route; ini mencegah request saat role tak berhak).
4. Menu sidebar 'Komentar' (CommentOutlined) + breadcrumb 'Komentar'.
TIDAK ADA perubahan API di task ini (semua endpoint sudah ada).

FILE:
- src/features/comments/domain/comment.ts                  # BARU: CommentStatus (pending_review|published|rejected), label, tag color, AdminCommentItem, ListCommentsParams, ReviewCommentResult
- src/features/comments/infrastructure/comment-api.ts      # BARU: listAdminCommentsRequest (+ normalize CursorPage), approveCommentRequest, rejectCommentRequest
- src/features/comments/application/use-comment-list.ts    # BARU: useCursorList queryKey ['comments', {status}], enabled role-gate
- src/features/comments/application/use-review-comment.ts  # BARU: mutation approve/reject + invalidate ['comments']
- src/features/comments/presentation/comments-page.tsx     # BARU: halaman moderasi (Tabs + DataTable + aksi)
- src/features/comments/__tests__/unit/comment-domain.test.ts # BARU: label & tag color lengkap per status
- src/app/router.tsx                                       # UBAH: route /comments (console-layout)
- src/shared/layouts/console-layout.tsx                    # UBAH: MENU_ROUTES '/comments' + breadcrumb 'Komentar'

Command verifikasi: pnpm typecheck && pnpm lint && pnpm test && pnpm build
```

---

## Catatan Implementasi

- Kosakata status komentar (pending_review|published|rejected) SENG AJA BUKAN
  workflow kontribusi (draft/pending/published/rejected) - ikut entity komentar
  backend, jangan dicampur.
- Aksi approve/reject hanya dirender untuk baris `pending_review`; baris
  published/rejected menampilkan '-' (tidak ada retract/undo di API).
- Per-baris loading memakai `reviewMutation.variables?.id` (mutation tunggal
  antar baris agar tak ada dua keputusan paralel pada baris sama).
- 409 race ditangani di onError: user melihat pesan backend (COMMENT_ALREADY_REVIEWED)
  dan list di-refetch supaya antrean akurat (item sudah pindah status).
- Kata "Buka kata" memakai kata { word_id } sebagai link menuju detail kata
  yang di-komentari (belum ada kata yang membawa lemma; cukup id).
- Belum ada halaman detail komentar tersendiri - keputusan inline di tabel
  sudah memenuhi spesifikasi (queue + action); tambah kolom/expanded row jika
  nanti butuh konteks lebih (mis. pratinjau kata induk).

## Referensi Terkait

- `09-api-comment.md` - Section 4-6: antrean moderasi, approve/reject,
  409 COMMENT_ALREADY_REVIEWED, DELETE komentar (verifikator).
- `docs/json/comment/admin-list-comments.200.json`,
  `approve-comment.200.json`, `reject-comment.200.json`,
  `approve-comment.409.json` - bentuk respon untuk tes/kontrak.
- `http/comment/admin-list-comments.bru` + `admin-review-comment.bru` - uji
  manual endpoint moderasi.
- `admin-base-stack.md` - Section 2-3, 8, 10, 13, 15.
- `04-detail-kata.md` - halaman /words/:id (tujuan link "Kata" dari antrean).
