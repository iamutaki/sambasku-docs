# Sample Response JSON - Kamus Digital Sambas-Indonesia API

Penyimpanan **kanonik** contoh response untuk semua endpoint dan kasus yang
didukung. Setiap file adalah JSON valid dan siap dipakai: mock frontend,
fixture test, contract testing, atau referensi manual.

## Konvensi penamaan

```text
<endpoint>.<status>[.<varian>].json

create-word.201.json                    → sukses standar (verifikator: published + is_verified true)
create-word.201.pending-review.json     → sukses, kontributor (pending_review - antrean review)
create-word.400.validation.json         → gagal, varian penyebab
shared/error-envelope.401.json          → envelope error generik (dipakai semua endpoint)
```

Folder per modul: `auth/ language/ word/ contribution/ search-miss/
category/ audit/ image/ vote/ comment/` - `shared/` berisi envelope error
yang identik di semua endpoint (401/403/429).

## Pemetaan endpoint → sample

| Endpoint | Sample |
| --- | --- |
| `GET /` | `auth/root.200.json` |
| `GET /api/v1/ping` | `misc/ping.200.json` |
| route tak dikenal | `auth/not-found.404.json` |
| `POST /api/v1/auth/register` | `auth/register.201.json`, `.400`, `.409` |
| `POST /api/v1/auth/login` | `auth/login.200.web.json`, `.200.mobile`, `.401` |
| `POST /api/v1/auth/refresh` | `auth/refresh.200.web.json`, `.200.mobile`, `.401` |
| `POST /api/v1/auth/logout` & `/logout-all-devices` | `auth/logout.200.json`, `auth/logout-all-devices.200.json` |
| `POST /api/v1/auth/forgot-password` | `auth/forgot-password.200.json` (response selalu sama - anti-enumeration) |
| `POST /api/v1/auth/reset-password` | `auth/reset-password.200.json`, `.401` |
| `GET /api/v1/languages` | `language/list-languages.200.json` |
| `GET /api/v1/dialects?language_id=` | `language/list-dialects.200.json`, `.400` |
| `POST /api/v1/admin/words` | `word/create-word.201.json`, `.201.pending-review`, `.201.warnings`, `.201.peribahasa`, `.400.validation`, `.400.referensi`, `.400.has-component-silang` |
| `GET /api/v1/words/:id` | `word/get-word-detail.200.json`, `.200.peribahasa`, `.200.component`, `.404` |
| `GET /api/v1/words/search` | `word/search-words.200.json`, `word/search-reverse.200.json` |
| `GET /api/v1/word-classes` | `word/word-classes.200.json` |
| `POST /api/v1/words/:wordId/pronunciations` | `word/add-pronunciation.201.json`, `.201.pending-review` |
| `POST /api/v1/words/:wordId/images` | `word/add-word-image.201.json`, `.201.pending-review` |
| `POST /api/v1/meanings/:meaningId/examples` | `word/add-example.201.json`, `.201.pending-review`, `.404` |
| `POST /api/v1/admin/words/:id/verify` | `word/verify-word.200.json`, `.404` |
| `POST /api/v1/admin/words/:id/unverify` | `word/unverify-word.200.json` |
| `GET /api/v1/admin/contributions` | `contribution/list-contributions.200.json` |
| `GET /api/v1/admin/contributions/:id` | `contribution/get-contribution-detail.200.json`, `.404` |
| `POST /api/v1/admin/contributions/:id/approve` | `contribution/approve-contribution.200.json`, `.409` |
| `POST /api/v1/admin/contributions/:id/reject` | `contribution/reject-contribution.200.json`, `.400` |
| `POST /api/v1/admin/contributions/:id/correct` | `contribution/correct-contribution.200.json` (publish) , `.200.publish-false` (koreksi saja) |
| `POST /api/v1/contributions/words` | `contribution/submit-word-anon.201.json` |
| `GET /api/v1/search-misses` | `search-miss/list-search-misses.200.json` |
| `GET /api/v1/admin/search-misses` | `search-miss/admin-list-search-misses.200.json` |
| `POST /api/v1/admin/search-misses/:id/dismiss` | `search-miss/dismiss-search-miss.200.json`, `.404` |
| `GET /api/v1/categories` | `category/list-categories.200.json` |
| `GET /api/v1/admin/audit-logs` | `audit/list-audit-logs.200.json`, `.200.password-change` |
| `GET /api/v1/admin/images/upload-token` | `image/upload-token.200.json`, `.503` |
| `POST /api/v1/votes` | `vote/toggle-vote.200.json`, `.200.toggle-off`, `.404` |
| `GET /api/v1/votes/counts?targets=` | `vote/vote-counts.200.json` |
| `GET /api/v1/votes/my?targets=` | `vote/my-votes.200.json` |
| `GET /api/v1/words/:wordId/comments` | `comment/list-comments.200.json` |
| `POST /api/v1/words/:wordId/comments` | `comment/create-comment.201.json`, `.400.validation` |
| `DELETE /api/v1/comments/:id` | `comment/delete-comment.200.json` |
| `GET /api/v1/admin/comments?status=` | `comment/admin-list-comments.200.json` |
| `POST /api/v1/admin/comments/:id/approve` | `comment/approve-comment.200.json`, `.409` |
| `POST /api/v1/admin/comments/:id/reject` | `comment/reject-comment.200.json` |

## Aturan sinkronisasi (WAJIB)

Endpoint berubah (field baru, bentuk response beda, kasus baru) → file
sample di sini diupdate **dalam PR yang sama**, bersama koleksi Bruno
(`http/`, blok `docs {}` pada request terkait). Tiga sumber contoh -
kode (OpenAPI schema), Bruno docs, dan folder ini - jangan saling
 tertinggal.

Semua sample mengikuti envelope standar `api-base-stack.md` Section 13.
