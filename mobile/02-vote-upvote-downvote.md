# Mobile - Upvote/Downvote (Vote Polymorphic)

Mengikuti `mobile-base-stack.md`: Section 2 (3 lapis per fitur), 5 (pola
fitur lengkap), 6 (networking retrofit + envelope), 10 (testing), 11
(mapping error_code). Kontrak API: `08-api-upvote-downvote.md` - toggle +
batch counts + vote milik user; target polymorphic
`word|meaning|example|pronunciation|word_image|comment` (key lookup
`"type:id"`).

Status fondasi saat dokumen ini dibuat (JANGAN diduplikasi):
- SUDAH ADA (API): `POST /api/v1/votes` (toggle 1/-1/batal), `GET
  /api/v1/votes/counts` (batch counts publik), `GET /api/v1/votes/my`
  (vote user login) - 1 user 1 vote (unique), toggle + ganti arah,
  count on-read, rate limit 60/menit per user.
- SUDAH ADA (mobile, fitur lain): auth status provider (isAuth), pola
  fitur 3+1 lapis, retrofit + `ApiResponse`, riverpod codegen, mapping
  `AppError` → failure dengan `errorCode`.
- Yang belum dulu: fitur vote itu sendiri (modul `features/vote`), DTO,
  wiring ke detail kata & komentar.

Selesai diimplementasikan (verifikasi terakhir): `flutter analyze` bersih,
`flutter test` hijau (termasuk `test/features/vote/`), codegen retrofit
+ riverpod tanpa konflik.

---

## Prompt

```text
Buatkan fitur vote (upvote/downvote) di mobile mengikuti pola fitur 3
lapis + presentation (Section 2 & 5 mobile-base-stack).

LOKASI: lib/features/vote/ (fitur baru)

STRUKTUR:
├── domain/
│   ├── entities/vote_target.dart    # VoteTarget(type,id) + key "type:id"
│   │                                #   + isComment
│   ├── entities/vote_view.dart      # VoteView{target,upvotes,downvotes,
│   │                                #   myVote(1|-1|null),score,hasVoted,
│   │                                #   copyWith} + VoteCounts
│   ├── failures/vote_failure.dart   # VoteFailure{message,errorCode};
│   │                                #   isNotFound, isRateLimited
│   ├── repositories/vote_repository.dart  # interface: toggle/counts/my
│   ├── providers/vote_domain_providers.dart # root: 3 use case provider
│   └── usecases/  toggle_vote / get_vote_counts / get_my_votes
├── data/
│   ├── models/  vote_toggle_request_dto / vote_toggle_response_dto /
│   │            vote_count_dto / my_vote_dto   (freezed + json + retrofit)
│   ├── datasources/vote_remote_datasource.dart  # retrofit:
│   │            POST /api/v1/votes, GET /counts, GET /my
│   ├── repositories/vote_repository_impl.dart
│   └── providers/vote_data_providers.dart
└── presentation/
    ├── providers/vote_providers.dart  # VoteController (family by VoteTarget)
    └── widgets/vote_buttons.dart      # pasangan tombol up/down + count

PERILAKU:
1. VoteController.build(target): load counts (publik) → 0/0 kalau target
   kosong; kalau isAuth, load my_votes juga → VoteView. (Anonim cuma
   lihat jumlah - tanpa myVote.)
2. toggle(value): panggil POST /votes; SUKSES → state = AsyncData(view
   hasil server) (count+myVote final, siap dipakai UI); gagal → kembalikan
   VoteFailure? supaya caller bisa tampilkan toast - state lama TIDAK
   berubah jadi error (komentar/vote tetap utuh).
3. VoteButtons: state-icon murni (up/down aktif dari myVote, count dari
   upvotes/downvotes); menahan tap saat busy/onVote berjalan (anti
   double-tap); labeled 'Upvote'/'Downvote'.
4. Guard login: caller (word detail / komentar) memeriksa isAuth sebelum
   onVote; anonim → prompt login (tidak call toggle).

WIRING:
- Kata: lib/features/dictionary/presentation/pages/word_detail_page.dart
  - _WordVoteBar → VoteTarget('word', wordId) + VoteController +
    VoteButtons (auth: toggle; anonim: prompt login + lihat counts).
- Komentar: lib/features/comment/presentation/widgets/
  word_comments_section.dart - tiap baris komentar VoteButtons dengan
  VoteTarget('comment', commentId) + guard login (08-api §komentar).

TESTING: test/features/vote/vote_dto_test.dart (deserialisasi DTO),
vote_use_cases_test.dart (toggle output, mapping counts/my, failure).
Fixture: test/fixtures/json/vote/*.

Command verifikasi: flutter analyze && flutter test
```

---

## Catatan Implementasi

- Key lookup = `VoteTarget.key` ("type:id") - satu-satunya bentuk konsisten
  di counts & my-votes; `VoteController` family memakai `VoteTarget`
  sebagai parameter (codegen: provider otomatis `(VoteTarget target)`).
- `VoteView.copyWith` memakai sentinel (`_unset`) supaya `myVote = null`
  (batal vote) bisa ditetapkan eksplisit tanpa dianggap "tidak diubah".
- Fetch my_votes digate `authStatusProvider.isAuth` - anonim tidak pernah
  memanggil endpooint /votes/my (backend mengharuskan login; UI memakai
  counts publik saja).
- Toggle memakai RESPONSE server sebagai source of truth (bukan optimistik):
  count + myVote final dari backend. Menghindari drift dua arah dan tetap
  benar saat race/toggle-off.
- Error toggle ditangani caller via nilai return `VoteFailure?` (bukan
  throw) supaya UI bisa bedakan: toast vs state error permanen.
- Target komentar memakai tipe `'comment'` (ditambahkan ke enum endpoint
  setelah proposal awal 08) - cermin `WordComment.voteTarget`.

## Referensi Terkait

- `docs/api/08-api-upvote-downvote.md` - kontrak vote (toggle/counts/my,
  keputusan semantic, rate limit 60/menit).
- `docs/api/09-api-comment.md` - vote pada komentar (public list membawa
  upvotes/downvotes).
- `docs/admin/07-vote-detail-kata.md` - sisi admin (read-only counts pada
  detail kata, endpoint counts yang sama).
- `mobile-base-stack.md` - Section 2, 5 (pola fitur), 6 (networking), 10
  (testing), 11 (mapping error_code → UI).
- `mobile/test/fixtures/json/vote/` - vote-counts/my-votes/toggle fixture
  (dipakai juga oleh tes DTO).

## Catatan Kesenjangan

- Fitur komentar mobile (tulis/daftar/moderation-prompt di
  `lib/features/comment/`) SUDAAH diimplementasikan tetapi BELUM memiliki
  dokumen tersendiri di `docs/mobile/` (dokumen ini hanya vote; rencana:
  `docs/mobile/03-comment.md`).