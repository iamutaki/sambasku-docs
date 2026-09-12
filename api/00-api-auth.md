# API Auth — Register & Login

Mengikuti `api-base-stack.md`: Clean Architecture feature-based (`modules/auth/`)
— Section 3 (struktur folder), 9 (`@hono/zod-openapi` + Scalar), 10 (testing),
11 (versioning `/api/v1/`), 13 (envelope & error), 15 (rate limiting).

---

## Prompt

```text
Buatkan modul autentikasi (Register & Login) untuk backend Kamus Digital 
Sambas-Indonesia, mengikuti struktur clean architecture feature-based 
yang sudah ditetapkan di api-base-stack.md.

STACK:
- Hono (Node.js runtime) — route pakai OpenAPIHono dari @hono/zod-openapi (Section 9)
- Drizzle ORM + PostgreSQL
- Zod untuk validasi request/response + generate OpenAPI spec (Section 9)
- JWT (algoritma RS256) via library `jose` — access token + refresh token
- argon2 untuk hashing password
- Migration: Drizzle Kit (drizzle-kit generate / migrate)

LOKASI MODUL: modules/auth/

STRUKTUR FILE YANG PERLU DIBUAT:

modules/auth/
├── domain/
│   ├── entities/
│   │   └── user.entity.ts
│   ├── value-objects/
│   │   ├── email.vo.ts              # validasi format email
│   │   └── password.vo.ts           # validasi kekuatan password
│   └── repositories/
│       └── user.repository.ts       # interface (contract)
├── application/
│   ├── use-cases/
│   │   ├── register-user.use-case.ts
│   │   ├── login-user.use-case.ts
│   │   ├── refresh-token.use-case.ts
│   │   ├── logout-user.use-case.ts
│   │   ├── logout-all-devices.use-case.ts
│   │   ├── forgot-password.use-case.ts
│   │   └── reset-password.use-case.ts
│   ├── dto/
│   │   ├── register.dto.ts
│   │   ├── login.dto.ts
│   │   └── reset-password.dto.ts
│   └── ports/
│       ├── token-service.port.ts       # interface generate/verify JWT
│       ├── password-hasher.port.ts     # interface hash/compare password
│       └── mailer.port.ts              # interface kirim email reset password
├── infrastructure/
│   ├── user.repository.impl.ts         # implementasi Drizzle
│   ├── refresh-token.repository.impl.ts
│   ├── password-reset-token.repository.impl.ts
│   ├── jwt-token.service.ts            # implements token-service.port
│   ├── argon2-password.service.ts      # implements password-hasher.port
│   └── smtp-mailer.service.ts          # implements mailer.port
├── presentation/
│   └── v1/                             # Section 11: API versioning
│       ├── auth.routes.ts              # OpenAPIHono + createRoute (Section 9)
│       ├── auth.controller.ts
│       └── validators/
│           ├── register.validator.ts   # Zod schema request + response
│           ├── login.validator.ts
│           └── reset-password.validator.ts
└── __tests__/                          # Section 10
    ├── unit/
    │   ├── register-user.use-case.test.ts
    │   ├── login-user.use-case.test.ts
    │   └── password.vo.test.ts
    ├── integration/
    │   ├── user.repository.impl.test.ts
    │   ├── refresh-token.repository.impl.test.ts
    │   └── password-reset-token.repository.impl.test.ts
    └── e2e/
        └── auth.e2e.test.ts

TABEL DATABASE TERKAIT (didefinisikan di 
shared/database/drizzle/schema/, dipakai lintas modul via repository):

- users (sudah ada di skema utama)
- refresh_tokens (baru, khusus modul auth)
- password_reset_tokens (baru, khusus modul auth)

Schema Drizzle untuk refresh_tokens & password_reset_tokens:

refresh_tokens {
  id, user_id (FK users), token_hash, device_info, ip_address,
  is_revoked (default false), expires_at, created_at
}

password_reset_tokens {
  id, user_id (FK users), token_hash, is_used (default false),
  expires_at, created_at
}

Setelah schema ditambahkan, WAJIB jalankan:
  pnpm drizzle-kit generate   → review SQL yang dihasilkan
  pnpm drizzle-kit migrate    → apply ke database
(lihat Section 7 api-base-stack.md untuk detail alur migration)

CARA DEFINISI ENDPOINT (WAJIB — Section 9 api-base-stack.md):
- Route file pakai OpenAPIHono, tiap endpoint didefinisikan dengan
  createRoute() + app.openapi() — BUKAN app.post() biasa
- Tiap endpoint isi: tags: ['Auth'], summary, schema Zod untuk request
  body DAN response (sukses + error) di validators/
- Schema response HARUS bentuk envelope standar (Section 13)
- Dokumentasi otomatis muncul di GET /docs (Scalar) tanpa langkah tambahan

ENDPOINT (semua di bawah prefix /api/v1 — Section 11):

1. POST /api/v1/auth/register
   Body: { username, email, password, confirm_password }
   
   Use case: RegisterUserUseCase
   - Validasi via register.validator.ts (Zod): email format valid, 
     password minimal 8 karakter kombinasi huruf+angka, 
     confirm_password harus sama
   - Cek username & email unik lewat UserRepository
   - Hash password via PasswordHasherPort (implementasi argon2id)
   - Role default: 'contributor'
   - Simpan user via UserRepository
   - Audit trail (Section 21): catat action 'create', entity_type 'user',
     new_data { username, email, role } — TANPA password/hash,
     request_id dari context
   - JANGAN pernah kembalikan password_hash di response
   
   Response sukses (201) — envelope standar Section 13, tanpa field custom
   (user_id berupa ULID string — lihat Section 8 api-base-stack.md):
   { "success": true,
     "data": { "user_id": "01ARZ3NDEKTSV4RRFFQ69G5FAV",
               "username": "...", "email": "..." } }

2. POST /api/v1/auth/login
   Body: { email, password, client_type? }
   client_type: 'web' (default) | 'mobile' — pilih kanal refresh token:

   Use case: LoginUserUseCase
   - Cari user by email via UserRepository
   - Bandingkan password via PasswordHasherPort.compare()
   - Pesan error generik "Email atau password salah" untuk SEMUA kasus
     gagal (email tidak ada / password salah / user soft-deleted /
     user dinonaktifkan is_active=false) — cegah user enumeration
   - Jika sukses:
     a. Generate access_token (TokenServicePort, TTL dari env 
        JWT_ACCESS_TOKEN_TTL — default 900 detik / 15 menit, 
        payload: { user_id, role })
     b. Generate refresh_token (random string), hash, simpan via 
        RefreshTokenRepository (TTL dari env JWT_REFRESH_TOKEN_TTL — 
        default 30 hari)
     c. WEB (default): refresh token dikirim sebagai httpOnly + secure +
        sameSite=strict cookie lewat Hono (setCookie dari hono/cookie)
     d. MOBILE (client_type='mobile'): refresh token dikirim di response
        body (data.refresh_token), TANPA Set-Cookie — client menyimpannya
        di secure storage perangkat (iOS Keychain / Android Keystore /
        expo-secure-store / flutter_secure_storage). Alasan: HttpOnly dan
        SameSite adalah mekanisme browser yang tidak berlaku di app
        native; threat model mobile adalah pencurian perangkat (at-rest
        encryption), bukan XSS
     e. access_token dikembalikan di response body (kedua kanal)
   - Rate limiting (Section 15): 5 percobaan/15 menit per email+IP 
     (middleware terpisah, lihat bagian Middleware)
   
   Response sukses (200):
   { "success": true,
     "data": { "access_token": "...", "expires_in": 900,
               "user": { "id": "01ARZ3NDEKTSV4RRFFQ69G5FAV",
                         "username": "...", "role": "..." } } }

3. POST /api/v1/auth/refresh
   - Sumber token (dua kanal, dipilih otomatis): body 
     { refresh_token: "..." } (mobile) ATAU cookie (web, default)
   - RefreshTokenUseCase: validasi hash cocok, belum expired, 
     is_revoked = false
   - Generate access_token baru
   - Rotate refresh_token: revoke yang lama, buat & simpan yang baru 
     (cegah replay attack)
   - Token rotasi dikembalikan via kanal yang sama: cookie (web) atau
     data.refresh_token di body (mobile)

4. POST /api/v1/auth/logout
   - Sumber token sama seperti refresh: body (mobile) atau cookie (web)
   - LogoutUserUseCase: revoke refresh_token yang sedang dipakai
   - Clear cookie (no-op untuk mobile)

5. POST /api/v1/auth/logout-all-devices
   - LogoutAllDevicesUseCase: revoke SEMUA refresh_token milik user 
     (butuh authenticate middleware, ambil user_id dari req)

6. POST /api/v1/auth/forgot-password
   Body: { email }
   - ForgotPasswordUseCase: generate token, hash, simpan ke 
     password_reset_tokens (expired 1 jam)
   - Kirim link reset via MailerPort (implementasi SMTP)
   - Response SAMA persis baik email terdaftar atau tidak (cegah 
     enumeration): "Jika email terdaftar, link reset telah dikirim"

7. POST /api/v1/auth/reset-password
   Body: { token, new_password }
   - ResetPasswordUseCase: validasi token belum expired & belum dipakai
   - Konsumsi token secara ATOMIK dulu (UPDATE ... WHERE is_used = false,
     cek affected row) — jaminan token hanya bisa dipakai SEKALI,
     termasuk terhadap request konkuren; yang kalah race ditolak
     RESET_TOKEN_INVALID
   - Update password_hash user (setelah token terbakar — kalau gagal,
     user minta link baru, failure mode aman)
   - Revoke SEMUA refresh token user — logout paksa semua perangkat
     (reset password biasanya berarti akun tercompromi; session lama
     milik pencuri tidak boleh selamat)
   - Audit trail (Section 21): action 'password_change', entity_type
     'user', new_data { changed: true } — hash password TIDAK PERNAH
     masuk audit

MIDDLEWARE (shared/middlewares/, dipakai lintas modul):

1. authenticate.middleware.ts
   - Ambil access_token dari header Authorization: Bearer <token>
   - Verifikasi via TokenServicePort
   - Set context: c.set('user', { user_id, role })
   - Token expired → response 401, error_code: "TOKEN_EXPIRED"

2. authorize-role.middleware.ts
   - Factory function: authorizeRole('admin', 'editor')
   - Cek c.get('user').role ada di daftar yang diizinkan
   - Dipakai proteksi endpoint modul lain, misal
     POST /api/v1/admin/words (modul word)

3. rate-limit.middleware.ts (shared/middlewares/, Section 15)
   - Pakai implementasi rateLimit() dari base-stack 
     (rate-limiter-flexible, in-memory dulu → Redis saat multi-instance)
   - Dipasang per route di auth.routes.ts sesuai tabel Section 15:
     * POST /register        → 5 request/jam per IP
     * POST /login           → 5 percobaan/15 menit per email+IP
     * POST /forgot-password → 5 percobaan/15 menit per email+IP
   - Response 429 wajib sertakan header Retry-After, 
     error_code: RATE_LIMITED

KEAMANAN WAJIB:
- Password policy: minimal 8 karakter, tolak password umum (pertimbangkan 
  library zxcvbn untuk cek kekuatan)
- Jangan pernah log password, bahkan di error log
- CORS whitelist origin eksplisit, jangan wildcard *
- Environment variable untuk JWT private/public key (.env, JANGAN 
  hardcode) — load & validasi via shared/config/env.ts (pakai Zod juga 
  untuk validasi env)
- Soft delete user (deleted_at) tidak boleh bisa login lagi — cek di 
  LoginUserUseCase

FORMAT RESPONSE — ENVELOPE STANDAR (Section 13 api-base-stack.md):
- Sukses: { "success": true, "data": { ... } }
- Gagal (dipakai error-handler.middleware.ts global via app.onError):
{ "success": false, "error_code": "INVALID_CREDENTIALS",
  "message": "Email atau password salah", "details": null }
- Use case melempar error class dari shared/errors/app-error.ts
  (UnauthorizedError, ConflictError, dst) — errorCode spesifik modul auth
  (INVALID_CREDENTIALS, TOKEN_EXPIRED, EMAIL_ALREADY_EXISTS,
  USERNAME_ALREADY_EXISTS, RESET_TOKEN_INVALID) wajib didaftarkan di
  ERROR_CODES.md di PR yang sama
```

---

## Catatan Implementasi

- Semua use case di `application/use-cases/` **tidak boleh** import 
  langsung dari Drizzle atau `jose`/`argon2` — selalu lewat interface 
  (`*.repository.ts`, `*.port.ts`). Implementasi konkret ada di 
  `infrastructure/`.
- `main.ts` (composition root) yang merakit:
  `new JwtTokenService()` → di-inject ke `new LoginUserUseCase(userRepo, tokenService, passwordHasher)`
  → di-inject ke `AuthController` → di-daftarkan ke `auth.routes.ts`.
  Route modul di-register dengan `app.route('/api/v1/auth', authRoutesV1)`
  (Section 11).
- Testing (Section 10): setiap use case wajib punya unit test, setiap
  repository implementasi wajib integration test (database test terpisah),
  setiap endpoint minimal 1 E2E test happy path + 1 skenario gagal
  (validasi/auth) — semua di `modules/auth/__tests__/`.
- Rujuk **Section 7 (Strategi Migration)** di `api-base-stack.md` setiap
  kali menambah/mengubah tabel `refresh_tokens` atau
  `password_reset_tokens`.

## Referensi Terkait

- `api-base-stack.md` — definisi stack & struktur folder lengkap
- `02-api-audit-logs.md` — sisi pembaca audit (auditor: admin & root);
  modul auth menulis audit untuk register & reset password
- repo `http/` — koleksi Bruno endpoint auth sudah tersedia
  (`http/auth/*.bru`, Section 20) — WAJIB di-update tiap kali endpoint
  auth berubah
- `kamus_sambas.dbml` — skema database utama (tabel `users`)
- `ERROR_CODES.md` — katalog error code (Section 13), wajib diupdate
  tiap ada errorCode baru
- Prompt selanjutnya: modul `word` untuk fitur admin tambah kata
