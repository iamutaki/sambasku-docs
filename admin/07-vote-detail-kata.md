# Admin UI - Ringkasan Vote pada Detail Kata (read-only)

Mengikuti `admin-base-stack.md`: Section 2-3 (clean architecture feature-based),
6 (router/guard), 7-8 (session + axios), 10 (React Query), 13 (error handling).
Kontrak API yang dikonsumsi: `08-api-upvote-downvote.md` - `GET
/api/v1/votes/counts` (batch counts publik) untuk target `word:<id>`, SELURUHNYA
read-only. TIDAK ada tombol vote di admin.

Status fondasi saat dokumen ini dibuat (JANGAN diduplikasi):
- SUDAH ADA (API): modul vote polymorphic (word|meaning|example|
  pronunciation|word_image|comment), `GET /api/v1/votes/counts?targets=...`
  (target tanpa vote = 0/0, dedupe + maks 50 target), count SELALU on-read
  (tanpa counter denormalized), authenticate semua role.
- SUDAH ADA (admin): halaman detail kata `/words/:id` dengan section 1-7,
  pola feature-based + React Query + envelope + `ApiError`, role gate
  verifikator di page.
- SUDAH ADA (mobile): UI vote interaktif (tombol up/down/toggle) di detail
  kata & baris komentar - konsumen AUTHENTICATED.
- Yang BELUM: admin sama sekali tidak menampilkan vote (tidak ada UI vote di
  mana pun di `admin/`).

## Keputusan Produk (admin)

- Admin = pengawas engagement, BUKAN voter. Hanya menampilkan COUNTS
  (up/down/skor) - tanpa tombol vote. Voting interaktif adalah domain
  aplikasi mobile (user biasa).
- Hanya tingkat kata yang ditampilkan (`word:<id>`, cermin mobile di detail
  kata). Vote makna/contoh/pelafalan/gambar TIDAK ditampilkan (scope sengaja
  dikunci - lihat opsi yang ditolak di bawah).
- Data diambil langsung dari `GET /api/v1/votes/counts` (endpoint publik,
  semua role auth) - TIDAK ada perubahan backend: `GET /admin/words/:id`
  tidak ikut membawa votes.
- Role gate: hanya verifikator (reviewer/admin/root) yang memicu request
  counts (mengikuti gate halaman detail yang sama).

## Prompt

```text
Tampilkan ringkasan vote kata (read-only) di halaman Detail Kata admin
(/words/:id), mengikuti pola feature-based.

KONTEKS:
- Backend: GET /api/v1/votes/counts?targets=word:<id> → data[] item
  { target_type, target_id, upvotes, downvotes } (target tanpa vote = 0/0;
  query "type:id" dipisah koma, dedupe, maks 50).
- Admin TIDAK memilih - tampilkan saja counts (up↑, down↓, skor) sebagai
  bagian "Informasi dasar" (section 1b) halaman detail.
- Halaman TIDAK memanggil client langsung - lewat infrastructure request fn
  (aturan base-stack) + React Query dengan cache stabil (counts jarang
  berubah dalam sesi moderasi).
- Error jaringan/target tidak ada → tampilkan teks abu "Vote tidak tersedia"
  (jangan merusak seluruh halaman detail).

STRUKTUR:
1. hooks query useVoteCounts(['word:<id>']): queryKey ['votes','counts',
   targets], queryFn getVoteCountsRequest(buildTargetsQuery(targets)),
   enabled = ada role verifikator && targets.length > 0, staleTime 60s.
2. Widget WordVoteCount: loading → Skeleton.Node; error/empty → "Vote tidak
   tersedia"; sukses → Tag hijau (LikeOutlined + upvotes), Tag merah
   (DislikeOutlined + downvotes), "Skor {up - down}".
3. Pasang section "1b. Vote" di WordDetailContent (detail.id = target_id).
TIDAK ADA perubahan API di task ini (endpoint counts sudah ada).

FILE:
- src/features/votes/domain/vote.ts                    # BARU: VoteCountsItem, MAX_VOTE_TARGETS, buildTargetsQuery
- src/features/votes/infrastructure/vote-api.ts        # BARU: getVoteCountsRequest (via client, envelope)
- src/features/votes/application/use-vote-counts.ts    # BARU: hook + toVoteCountMap + isVoteCountsReader
- src/features/votes/presentation/word-vote-count.tsx  # BARU: widget read-only
- src/features/words/presentation/word-detail-page.tsx # UBAH: section "1b. Vote"
- src/features/votes/__tests__/unit/vote-counts.test.ts # BARU: buildTargetsQuery (join/dedupe/cap 50) + toVoteCountMap

Command verifikasi: pnpm typecheck && pnpm lint && pnpm test && pnpm build
```

---

## Catatan Implementasi

- Target query dibangun di helper murni `buildTargetsQuery` (dedupe + cap
  50, selaras `targetsQuerySchema`) supaya gampang diuji tanpa network.
- `toVoteCountMap` memetakan item `{target_type,target_id,...}` → key
  "type:id"; widget fallback aman (`?? { upvotes: 0, downvotes: 0 }`) walau
  backend sudah menjamin 0/0 untuk target tanpa vote.
- Cache `staleTime: 60_000` karena counts adalah data engagement yang
  berubah pelan - cukup segar saat admin membuka detail, tanpa membanjiri
  endpoint di navigasi cepat antar kata.
- Ikon memakai `LikeOutlined`/`DislikeOutlined` (ThumbsUp/ThumbsDown tidak
  diekspor @ant-design/icons versi repositori ini).
- Halaman detail tetap berfungsi penuh saat request counts gagal/terblokir
  (widget menampilkan fallback abu-abu sendiri, tidak melempar error).

## Opsi yang Ditolak (beralasan)

- Tombol vote interaktif di admin: admin voter bukan keputusan produk -
  tetap di mobile.
- Kolom up/down di tabel moderasi komentar: ruang terbatas + out of scope
  keputusan "counts only". Bisa ditambah via batch `comment:<id>` tanpa
  perubahan backend bila nanti dibutuhkan.
- Vote makna/contoh/pelafalan/gambar di detail: belum dibutuhkan oleh alur
  moderasi; endpoint counts sudah mendukung poly-target kapan pun.

## Referensi Terkait

- `08-api-upvote-downvote.md` - target polymorphic, toggle/counts/my,
  keputusan semantic (1 user 1 vote, count on-read).
- `docs/json/vote/vote-counts.200.json` - bentuk respon counts.
- `http/vote/get-vote-counts.bru` - uji manual batch counts.
- `admin-base-stack.md` - Section 2-3, 8, 10, 13.
- `04-detail-kata.md` - halaman detail kata (tempat section 1b disisipkan).
- `06-comment-moderasi.md` - fitur moderasi komentar (pola hooks feature
  yang sama, termasuk role gate).