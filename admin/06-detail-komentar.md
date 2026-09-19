# Admin UI - Komentar pada Detail Kata (Moderasi Inline)

Mengikuti `admin-base-stack.md`: Section 2-3 (clean architecture), 10
(React Query), 13 (error handling). Kontrak API:
`09-api-comment.md` (antrean `GET /admin/comments` kini mendukung
`word_id` + status opsional).

Status fondasi saat dokumen ini dibuat (JANGAN diduplikasi):
- SUDAH ADA: halaman antrean moderasi `/comments` (tabs status, approve/
  reject inline, `features/comments/` lengkap), embed `WordVoteCount` di
  detail kata, `useReviewComment` dengan invalidasi prefix `['comments']`.
- Yang BELUM: section komentar di halaman detail kata (admin harus lompat
  ke antrean global untuk satu kata) + filter `word_id` di API antrean.

---

## Prompt

```text
Buatkan section "8. Komentar" untuk halaman detail kata admin, mengikuti
pola embed WordVoteCount (self-contained component + own hook).

KONTEKS:
- GET /admin/comments kini menerima word_id (ULID) dan status OPSIONAL
  (absen = semua status -embed detail; halaman antrean tetap kirim
  eksplisit via tab).
- Section menampilkan SEMUA status komentar kata itu (Tag warna dari
  COMMENT_STATUS_TAG_COLOR), terbaru dulu, cursor pagination.
- Komentar pending_review punya tombol Tolak/Setujui INLINE (pola
  halaman antrean): modal.confirm + useReviewComment; invalidasi prefix
  ['comments'] otomatis menyegarkan section ini DAN antrean global.
- 409 COMMENT_ALREADY_REVIEWED (race) → toast error + refetch.

FILE:
- src/features/comments/domain/comment.ts          # UBAH: ListCommentsParams +wordId
- src/features/comments/infrastructure/comment-api.ts # UBAH: kirim word_id
- src/features/comments/application/use-word-comments.ts # BARU: useCursorList key ['comments','word',wordId]
- src/features/comments/presentation/word-comments.tsx  # BARU: embed + moderasi inline
- src/features/words/presentation/word-detail-page.tsx  # UBAH: section 8 setelah Gambar

Command verifikasi: pnpm typecheck && pnpm lint && pnpm test && pnpm build
```

---

## Catatan Implementasi

- Key `['comments', 'word', id]` sengaja berbagi prefix `['comments']`
  dengan antrean - satu keputusan moderasi menyegarkan keduanya sekaligus.
- Busy-state per baris: `reviewMutation.variables?.id === cm.id` (pola
  halaman antrean) - tombol baris lain di-disable saat mutasi jalan.
- Body komentar pakai `Paragraph ellipsis` (3 baris + "selengkapnya") -
  komentar panjang tidak merusak layout halaman detail.

## Referensi Terkait

- `09-api-comment.md` - kontrak komentar + delta word_id/status-opsional
- `docs/admin/admin-base-stack.md` - Section 2-3, 10, 13
- `src/features/comments/presentation/comments-page.tsx` - halaman antrean
  (pola actions yang direuse)
- `src/features/votes/presentation/word-vote-count.tsx` - pola embed
