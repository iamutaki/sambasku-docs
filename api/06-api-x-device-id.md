# API - Rate Limit Anonim per-Device (header X-Device-Id)

Mengikuti `api-base-stack.md` Section 15 (rate limiting): pola middleware
`rateLimit({ points, duration, keyFn })` yang sudah ada - TIDAK ada
library, migration, atau error code baru.

MASALAH (hasil diskusi 2026-09-18): limit `POST /api/v1/contributions/words`
saat ini 5/jam **per-IP** - satu IP publik dipakai banyak manusia (CGNAT
operator seluler, kampus, warnet) sehingga pengguna sah saling mengunci.
Ganti ke per-device signature? Device id buatan client adalah
*self-asserted* - bot bisa generate id baru tanpa batas (friction, bukan
tembok). Solusi pragmatis: **dual-bucket** - per-device (ketat, fairness
untuk manusia di IP bersama) + per-IP (longgar, langit-langit bot).
Mobile mengirim `X-Device-Id` (ULID, lihat `docs/mobile/01-mobile-ulid-
device-id.md`) - TANPA cookie (mobile-first).

Pengingat skala: rate limit hanya membatasi BEBAN ANTREAN REVIEWER -
submit anonim tetap `pending_review` sampai disetujui (Section 22,
approval gate). Kualitas data publik tidak pernah bertaruh pada ini.

---

## Prompt

```text
Ubah rate limiting endpoint kontribusi anonim menjadi dual-bucket
per-device + per-IP.

ENDPOINT (scope): POST /api/v1/contributions/words
(modules/word/presentation/v1/anon-contribution.routes.ts - baris
rateLimit({ points: 5, duration: 3600 }) saat ini).
Endpoint anonim lain (register) menyusul kalau terbukti perlu - jangan
dicampur di PR ini.

KONTRAK HEADER:
- X-Device-Id: opsional, string OPAQUE
- TIDAK divalidasi format (ULID/UUID/apapun) - key bucket saja;
  memvalidasi bentuk justru memberi bot cetakan yang harus dipenuhi
- normalisasi minimal: trim; panjang di luar 8-64 karakter → header
  DIABAIKAN (jatuh ke bucket IP saja)
- tidak dipercaya sebagai identitas: bukan pengganti login, tidak
  dipakai untuk data domain (hanya key rate limit)

LIMIT BARU (dua middleware berantai, urutan: IP dulu, device kedua):
1. per-IP: 20 request/jam (key clientIpKey() yang sudah ada - dinaikkan
   dari 5; langit-langit bot yang mengacak device id dari satu IP)
2. per-device: 5 request/jam per nilai X-Device-Id (key
   `anon-dev:<id>`); header absen/abaikan → middleware ini loloskan
   (bucket IP tetap satu-satunya pengikat)
Kena salah satu → 429 RATE_LIMITED + header Retry-After (perilaku
middleware sekarang, tidak berubah).

IMPLEMENTASI:
- rate-limit.middleware.ts TIDAK diubah (sudah mendukung keyFn) - cukup
  dua pemanggilan rateLimit() berurutan di anon-contribution.routes.ts
- keyFn device membaca header + normalisasi; return string kosong bila
  absen → helper skip consume (guard kecil di middleware ATAU keyFn
  fallback unik-per-request untuk melewati; pilih yang paling kecil
  diff-nya, dokumentasikan di komentar)
- TANPA cookie, TANPA perubahan DB/schema, TANPA error code baru

KEAMANAN & CATATAN:
- X-Device-Id self-asserted - jangan pernah dijadikan dasar keputusan
  selain rate limit
- jangan log nilai id utuh di level info (privacy kecil tapi gratis)
- store in-memory = per-instance (limit konsisten per instance saja) -
  upgrade ke Redis saat multi-instance, sudah tercatat Section 15
- upgrade path (TERCATAT SAJA, jangan dibangun sekarang): attestation
  Play Integrity (Android) / App Attest (iOS) dengan nonce issue server
  → device id terverifikasi kriptografis; baru masuk akal kalau abuse
  anonim terukur

TESTING:
- unit (middleware/helper normalisasi): header 8-64 char → key valid;
  kosong/terlalu panjang → diabaikan
- e2e anon-contribution: 5 request dgn X-Device-Id sama → ke-6 429;
  ganti device id (IP sama) → lolos kembali (bukti bucket per-device);
  tanpa header → hanya kena langit-langit IP
- Bruno: update http/contribution/*.bru submit anonim - tambah header
  X-Device-Id + satu varian tanpa header; tests memastikan 429 membawa
  Retry-After (Section 20 - satu PR yang sama)
```

---

## Catatan Implementasi

- Angka 5/jam per-device = angka lama dipertahankan di bucket yang
  tepat (per manusia, bukan per IP); 20/jam per-IP = langit-langit
  anti-bot-rotate. Keduanya knob konfigurasi, bisa disetel belakangan.
- Dual-bucket ini pola yang sama dipakai endpoint sensitif Section 15
  (per email+IP) - hanya keyFn yang beda.

## Referensi Terkait

- `docs/mobile/01-mobile-ulid-device-id.md` - pengirim header (client)
- `api-base-stack.md` Section 15 - tabel limit & middleware rate limit
- `03-api-kontribusi-verifikasi.md` - antrean review tujuan submit anonim
  (approval gate = dinding sesungguhnya)
