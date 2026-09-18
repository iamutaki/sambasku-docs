# Mobile - Device ID ULID (header X-Device-Id)

Mengikuti `mobile-base-stack.md`: dio + retrofit (Section 6 networking),
riverpod codegen, local storage via shared_preferences.

Konteks: backend membatasi submit anonim dengan rate limit **dual-bucket
per-device** (`docs/api/06-api-x-device-id.md`) karena 5/jam per-IP
mengunci manusia yang berbagi IP (CGNAT operator, kampus, warnet).
Mobile TIDAK memakai cookie - device diidentifikasi lewat header
`X-Device-Id` yang isinya ULID digenerate sekali per instalasi.

---

## Prompt

```text
Buatkan device ID (ULID) yang dikirim sebagai header X-Device-Id pada
semua request dio, untuk rate limiting anonim sisi API.

PACKAGE BARU (satu): ulid (pub.dev) - hanya untuk GENERATE id.
Penyimpanan TIDAK perlu package baru: shared_preferences, pola yang
sama dengan AuthTokenStorage (device id BUKAN rahasia - cuma kunci
rate limit & statistik; flutter_secure_storage = upgrade opsional
kalau nanti mau survive uninstall di iOS Keychain).

LOKASI: lib/core/ (bukan per-fitur - lintas fitur)

STRUKTUR FILE:
├── lib/core/device/
│   ├── device_id.dart            # baca/generate sekali + provider
│   └── device_id_interceptor.dart # dio interceptor: pasang header
└── lib/core/device_id_test.dart? # tidak - ikuti struktur test
    proyek (folder test/), lihat bagian TESTING

PERILAKU device_id.dart:
1. init(): baca key 'sambasku_device_id' dari shared_preferences
2. kalau ADA → pakai apa adanya. JANGAN pernah regenerate selama
   nilainya masih tersimpan (regenerate = bucket rate limit baru =
   melewati limit - justru menggagalkan tujuannya)
3. kalau KOSONG (instalasi pertama) → generate ulid() SATU kali,
   simpan, pakai
4. expose Riverpod provider (codegen @riverpod): String deviceIdProvider
   - siap SEBELUM dio pertama dipakai (await di bootstrap/main)

PERILAKU interceptor:
- extends InterceptorsWrapper (dio), onRequest:
  options.headers['X-Device-Id'] = ref.read(deviceIdProvider)
- dipasang di dioProvider (lib/core/network_providers.dart) - satu
  tempat, otomatis berlaku untuk SEMUA request (auth & anon); server
  mengabaikan header ini pada endpoint yang tidak memakainya
- tidak perlu retry/error handling khusus

ATURAN:
- JANGAN pakai device id sebagai identitas user / pengganti login -
  hanya kunci rate limit & statistik kasar (client self-asserted)
- uninstall app = id hilang (shared_preferences terhapus di kedua
  platform) → instalasi berikutnya dianggap device baru. TERMINAL,
  by design - friction yang cukup, bukan tembok
- jangan log isi id di level info ke service eksternal (cukup dev)

TESTING (folder test/ proyek):
- unit: pertama kali → tergenerate & tersimpan; panggil kedua →
  nilai SAMA (tidak regenerate)
- interceptor: request tiruan membawa header X-Device-Id nilainya id
```

---

## Catatan Implementasi

- Format ULID dipilih demi konsistensi konvensi proyek (semua id ULID,
  base-stack API Section 19) - secara fungsional, UUIDv4 sama baiknya;
  server memperlakukan header ini sebagai string opaque.
- Timestamp instalasi ter-encode di ULID (6 char pertama) - bocoran
  privasi sepele, dicatat sadar.
- Value properti `deviceIdProvider` harus deterministik setelah init
  (bukan stream) supaya interceptor bisa `ref.read` murah.

## Referensi Terkait

- `docs/api/06-api-x-device-id.md` - konsumen header ini (dual-bucket)
- `mobile-base-stack.md` - Section 6 (dio/interceptor), local storage
- `docs/api/api-base-stack.md` Section 15 - strategi rate limiting
