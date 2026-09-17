# Base Stack - Kamus Digital Sambas-Indonesia (Backend)

Dokumen ini jadi acuan tetap untuk semua prompt/fitur backend selanjutnya.
Setiap prompt fitur baru (auth, admin content, search, dll) akan mengikuti
struktur dan konvensi di sini, jadi tidak perlu dijelaskan ulang tiap kali.

---

## 1. Tech Stack

| Layer                    | Pilihan                                        | Alasan                                                                                      |
| ------------------------ | ---------------------------------------------- | ------------------------------------------------------------------------------------------- |
| Runtime                  | Node.js (v20+ LTS) **dan Cloudflare Workers** (dual entry - lihat Section 8) | Kompatibilitas library terluas + opsi edge; satu composition root untuk dua runtime |
| Framework                | Hono                                           | Ringan, cepat, tidak memaksa struktur (cocok clean architecture)                            |
| Bahasa                   | TypeScript                                     | Type safety, wajib untuk clean architecture yang solid                                      |
| ORM/Query Builder        | Drizzle ORM + drizzle-kit                      | Type-safe, ringan, migrasi eksplisit, cocok dgn Hono                                        |
| DB Driver                | `pg` via `drizzle-orm/node-postgres`       | Koneksi TCP standar PostgreSQL -**generik, no vendor lock-in** (lihat Section 9)     |
| Database                 | PostgreSQL (hosting: Neon)                     | Sesuai skema DBML yang sudah dirancang; provider bisa diganti kapan saja                    |
| Validasi & API Docs      | Zod +`@hono/zod-openapi`                     | Validasi schema-first, sekaligus generate OpenAPI spec otomatis dari schema yang sama       |
| API Reference UI         | Scalar (`@scalar/hono-api-reference`)        | Render dokumentasi API interaktif dari OpenAPI spec, auto-update tiap endpoint baru         |
| Auth                     | JWT (RS256) via`jose`                        | Access + refresh token, lihat`00-api-auth.md`                                             |
| ID Generation            | `ulid`                                       | ULID - time-sortable, URL-safe, 26 char, no DB sequence needed (lihat Section 19)          |
| Package manager          | pnpm                                           | Instalasi cepat, hemat disk, cocok untuk struktur modular                                   |
| Testing                  | Vitest (unit + integration + E2E)              | Cepat, native ESM/TS support - lihat Section 10 untuk strategi lengkap                     |
| HTTP client (fungsional) | Bruno - koleksi di repo`http/`              | Plain file`.bru` tersimpan di git, bisa dijalankan CLI (CI) & GUI - lihat Section 20     |
| Audit trail              | tabel`audit_logs` + modul `modules/audit/` | Setiap mutasi data tercatat siapa-kapan-apa, bisa ditelusuri - lihat Section 21            |
| Image hosting            | ImageKit via`ImageStoragePort`               | Wrapper provider-agnostic - ganti provider tinggal ganti impl (pola port, lihat Section 8) |
| Lint/Format              | ESLint + Prettier                              | Konsistensi kode antar kontributor                                                          |

---

## 2. Prinsip Clean Architecture yang Dipakai

Empat lapisan, dengan **arah dependency selalu ke dalam** (outer layer boleh
tahu inner layer, tidak sebaliknya):

```
Presentation → Application → Domain
Infrastructure → Application & Domain (implementasi interface)
```

- **Domain**: Inti bisnis. Tidak tahu apa-apa soal database, HTTP, atau
  framework. Murni TypeScript.
- **Application**: Use case / business logic. Bergantung pada Domain,
  dan pada *interface* (abstraksi) dari Infrastructure - bukan
  implementasinya.
- **Infrastructure**: Implementasi konkret - Drizzle, JWT, email service,
  dll. Mengimplementasikan interface yang didefinisikan di Application
  (ports) maupun Domain (repository contract).
- **Presentation**: Hono routes, controllers, middleware, request/response
  mapping. Titik masuk HTTP.

Composition root (`main.ts`) yang merakit semua dependency (manual DI,
tanpa framework DI tambahan - cukup untuk skala proyek ini).

---

## 3. Struktur Folder - Feature-Based (Modular) + Clean Architecture

Pendekatan: **bungkus per modul/fitur**, dan di dalam tiap modul tetap
menerapkan clean architecture (domain → application → infrastructure →
presentation). Kadang disebut *screaming architecture* - struktur folder
langsung "berteriak" fitur apa saja yang ada, bukan pola arsitektur
abstrak semata.

Alasan pilih ini dibanding layer-based murni: begitu jumlah fitur
bertambah (word, auth, category, contribution, translation, dst),
layer-based akan membuat file dari fitur berbeda tercampur dalam satu
folder flat (`domain/entities/word.entity.ts`,
`domain/entities/user.entity.ts`, `domain/entities/category.entity.ts`,
dst) - sulit dilihat cakupan satu fitur, dan rawan konflik saat
dikerjakan banyak kontributor. Feature-based menjaga arah dependency
clean architecture tetap sama, hanya di-scope per modul.

```
src/
├── modules/
│   ├── word/
│   │   ├── domain/
│   │   │   ├── entities/
│   │   │   │   ├── word.entity.ts
│   │   │   │   └── meaning.entity.ts
│   │   │   └── repositories/
│   │   │       └── word.repository.ts        # interface (contract)
│   │   ├── application/
│   │   │   ├── use-cases/
│   │   │   │   ├── create-word.use-case.ts
│   │   │   │   ├── get-word-by-id.use-case.ts
│   │   │   │   └── search-words.use-case.ts
│   │   │   └── dto/
│   │   │       └── create-word.dto.ts
│   │   ├── infrastructure/
│   │   │   └── word.repository.impl.ts        # implementasi Drizzle
│   │   └── presentation/
│   │       └── v1/                             # API versioning (Section 11)
│   │           ├── word.routes.ts
│   │           ├── word.controller.ts
│   │           └── validators/
│   │               └── create-word.validator.ts   # Zod schema
│   │
│   ├── auth/
│   │   ├── domain/
│   │   │   ├── entities/
│   │   │   │   └── user.entity.ts
│   │   │   ├── value-objects/
│   │   │   │   ├── email.vo.ts
│   │   │   │   └── password.vo.ts
│   │   │   └── repositories/
│   │   │       └── user.repository.ts          # interface
│   │   ├── application/
│   │   │   ├── use-cases/
│   │   │   │   ├── register-user.use-case.ts
│   │   │   │   ├── login-user.use-case.ts
│   │   │   │   └── refresh-token.use-case.ts
│   │   │   ├── dto/
│   │   │   │   └── login.dto.ts
│   │   │   └── ports/                          # interface service eksternal
│   │   │       ├── token-service.port.ts
│   │   │       └── password-hasher.port.ts
│   │   ├── infrastructure/
│   │   │   ├── user.repository.impl.ts
│   │   │   ├── jwt-token.service.ts             # implements token-service.port
│   │   │   └── argon2-password.service.ts       # implements password-hasher.port
│   │   └── presentation/
│   │       └── v1/
│   │           ├── auth.routes.ts
│   │           ├── auth.controller.ts
│   │           └── validators/
│   │               └── login.validator.ts
│   │
│   ├── category/
│   │   └── ... (struktur sama: domain/application/infrastructure/presentation)
│   │
│   └── contribution/
│       └── ... (struktur sama, dipakai nanti untuk workflow review)
│
├── shared/                            # HANYA yang benar-benar lintas modul
│   ├── database/
│   │   └── drizzle/
│   │       ├── schema/                # semua tabel terpusat (kebutuhan Drizzle)
│   │       │   ├── languages.schema.ts
│   │       │   ├── words.schema.ts
│   │       │   ├── meanings.schema.ts
│   │       │   ├── users.schema.ts
│   │       │   └── ...
│   │       ├── migrations/            # auto-generated drizzle-kit
│   │       └── client.ts              # koneksi DB
│   ├── middlewares/
│   │   ├── authenticate.middleware.ts
│   │   ├── authorize-role.middleware.ts
│   │   └── error-handler.middleware.ts
│   ├── errors/
│   │   └── app-error.ts                   # lihat Section 13
│   ├── utils/
│   ├── constants/
│   └── config/
│       └── env.ts                     # load & validasi environment variables
│
└── main.ts                            # composition root - rakit semua modul
```

**Catatan penting:** schema Drizzle tetap diletakkan terpusat di
`shared/database/drizzle/schema/` (karena Drizzle butuh semua tabel terdaftar
untuk relasi & migrasi), meskipun repository *implementasinya* tetap
tinggal di masing-masing modul (`modules/word/infrastructure/`,
`modules/auth/infrastructure/`, dst).

---

## 4. Contoh Alur Request (biar kebayang aliran datanya)

`POST /api/admin/words` →

1. **Presentation**: `modules/word/presentation/word.routes.ts` terima
   request → validasi body pakai Zod
   (`modules/word/presentation/create-word.validator.ts`) → panggil
   `modules/word/presentation/word.controller.ts`
2. **Controller**: mapping request ke DTO → panggil
   `CreateWordUseCase.execute(dto)` dari
   `modules/word/application/use-cases/create-word.use-case.ts`
3. **Application**: use case jalankan business logic (cek duplikasi, dll)
   → panggil `WordRepository.save()` - interface dari
   `modules/word/domain/repositories/word.repository.ts`
4. **Infrastructure**: `modules/word/infrastructure/word.repository.impl.ts`
   (implementasi konkret) eksekusi query Drizzle ke PostgreSQL
5. Response balik naik ke Controller → dikembalikan sebagai JSON

Domain & Application di dalam modul manapun **tidak pernah** import apa
pun dari Drizzle atau Hono secara langsung - semua lewat interface. Ini
yang membuat business logic bisa di-test tanpa perlu database beneran,
dan gampang ganti ORM/framework di masa depan kalau perlu.

Antar modul (misal `word` butuh cek `user` yang login) berkomunikasi
lewat interface yang diekspor modul lain, bukan saling import langsung
ke internal (domain/infrastructure) modul lain - jaga batas modul tetap
jelas.

---

## 5. Konvensi Penamaan File

| Tipe                    | Konvensi                                              | Contoh                               |
| ----------------------- | ----------------------------------------------------- | ------------------------------------ |
| Entity                  | `*.entity.ts`                                       | `word.entity.ts`                   |
| Use case                | `*.use-case.ts`                                     | `create-word.use-case.ts`          |
| Repository interface    | `*.repository.ts`                                   | `word.repository.ts`               |
| Repository implementasi | `*.repository.impl.ts`                              | `word.repository.impl.ts`          |
| DTO                     | `*.dto.ts`                                          | `create-word.dto.ts`               |
| Validator (Zod)         | `*.validator.ts`                                    | `create-word.validator.ts`         |
| Route                   | `*.routes.ts`                                       | `word.routes.ts`                   |
| Controller              | `*.controller.ts`                                   | `word.controller.ts`               |
| Port/interface service  | `*.port.ts`                                         | `token-service.port.ts`            |
| Folder modul            | `modules/<nama-fitur>/` (singular, kebab/lowercase) | `modules/word/`, `modules/auth/` |

---

## 6. Yang Perlu Disiapkan Sebelum Coding Dimulai

- [ ] Inisialisasi project (`pnpm init`, install Hono, Drizzle, Zod, dll)
- [ ] Setup `drizzle.config.ts` + koneksi PostgreSQL
- [ ] Generate schema Drizzle terpusat (`shared/database/drizzle/schema/`)
  dari file `.dbml` yang sudah dibuat
- [ ] Setup environment variables (`.env`) - DB connection string, JWT
  private/public key
- [ ] Setup `error-handler.middleware.ts` global untuk Hono di `shared/middlewares/`
- [ ] Buat folder `modules/` kosong dengan modul pertama: `auth/` dan `word/`
- [ ] Setup `main.ts` sebagai composition root yang me-register semua
  route dari tiap modul ke instance Hono

---

## 7. Strategi Migration Database

Migration tool yang dipakai: **Drizzle Kit** - satu ekosistem dengan
Drizzle ORM, bukan tool terpisah.

**Prinsip:** skema database didefinisikan sebagai kode (file
`*.schema.ts` di `shared/database/drizzle/schema/`), bukan diubah manual
lewat GUI/psql langsung ke database. Setiap perubahan skema **wajib**
lewat migration file yang tercatat, di-review, dan di-commit ke git -
sama seperti perubahan kode aplikasi biasa.

### Alur kerja setiap ada perubahan skema

```
1. Ubah file schema Drizzle
   (shared/database/drizzle/schema/words.schema.ts, dst)
        ↓
2. Jalankan: pnpm drizzle-kit generate
   → Drizzle Kit bandingkan schema baru vs migration history
   → Generate file SQL baru di shared/database/drizzle/migrations/
        ↓
3. REVIEW file SQL yang di-generate (WAJIB, jangan asal jalankan)
   → Pastikan tidak ada perubahan tak diinginkan
   → Perhatikan khusus statement ALTER/DROP yang bisa hilangkan data
        ↓
4. Jalankan: pnpm drizzle-kit migrate
   → Eksekusi migrasi ke database target (local/staging/production)
        ↓
5. Commit KEDUANYA ke git dalam satu PR:
   - File schema yang diubah
   - File migration SQL yang baru di-generate
```

### Aturan Wajib

- **Semua tabel WAJIB implementasi soft delete** (`deleted_at` timestamp +
  `deleted_by` varchar FK users) - TIDAK ADA hard delete di level aplikasi.
  Pengecualian HANYA untuk:
  - `audit_logs` - jejak audit bersifat immutable (menghapusnya mengalahkan tujuannya)
  - `refresh_tokens` / `password_reset_tokens` - ephemeral, lifecycle via
    `is_revoked`/`is_used`/`expires_at` + cleanup berkala
  - `word_categories` (junction) - hanya `deleted_at` tanpa `deleted_by`
    (link bukan entri; visibility mengikuti parent word yang di-soft-delete)

  Query baca WAJIB filter `WHERE deleted_at IS NULL` untuk mengecualikan
  baris yang sudah di-soft-delete.

- **TIDAK BOLEH mengubah nama ATAU isi migration yang sudah pernah
  dijalankan** di environment manapun (termasuk local dev milik
  kontributor lain) - KECUALI mendapat persetujuan eksplisit seluruh tim.
  Kalau ada kesalahan, buat migration baru untuk memperbaikinya - jangan
  ubah yang lama. Ini menjaga riwayat migrasi tetap konsisten di semua
  environment.
- **Nama migration WAJIB deskriptif sesuai isinya** - gunakan flag
  `--name` saat generate. DILARANG menyimpan migration dengan nama
  random yang dihasilkan drizzle-kit tanpa `--name`.
  Format: `<nomor>_<kebab-case-deskriptif>.sql`
  Contoh benar: `0007_add_soft_delete.sql`, `0008_create_audit_logs.sql`
  Contoh salah: `0007_mature_captain_universe.sql` (random, tak bermakna)
- **Migration dijalankan otomatis saat deploy** (bagian dari CI/CD atau
  startup script), bukan manual oleh developer di server production.
- **Kontributor baru** cukup jalankan `pnpm drizzle-kit migrate` setelah
  clone project - database lokal mereka otomatis sinkron dengan skema
  terbaru tanpa setup manual.
- Untuk perubahan yang berisiko hilangkan data (drop kolom, ubah tipe
  data), buat migration bertahap: tambah kolom baru → migrasi data →
  baru hapus kolom lama, dilakukan di PR/rilis terpisah.

### Command Referensi

| Command                       | Fungsi                                                                     |
| ----------------------------- | -------------------------------------------------------------------------- |
| `pnpm drizzle-kit generate --name=<deskriptif>` | Generate file migrasi SQL - **WAJIB pakai `--name`** (tanpa itu hasilnya nama random tak deskriptif) |
| `pnpm drizzle-kit migrate`  | Jalankan migrasi yang belum diterapkan ke database                         |
| `pnpm drizzle-kit studio`   | Buka GUI ringan untuk lihat isi database (dev only)                        |
| `pnpm drizzle-kit drop`     |

Contoh penamaan yang benar:

    pnpm drizzle-kit generate --name=add-soft-delete
    pnpm drizzle-kit generate --name=create-audit-logs

Hasil: 0007_add_soft_delete.sql - terbaca jelas di diff/PR apa isinya. Hapus migration file terakhir yang belum dijalankan (kalau salah generate) |

---

## 8. Portability - Tidak Vendor Lock-in

Prinsip: **provider database, hosting, dan infrastruktur pendukung
lainnya harus bisa diganti tanpa mengubah business logic**, hanya
konfigurasi/koneksi di layer paling luar (`infrastructure/`).

### Database Driver: Generic, Bukan Driver Eksklusif Provider

Drizzle punya beberapa pilihan driver PostgreSQL. Yang dipakai proyek
ini:

```
drizzle-orm/node-postgres  (pakai library "pg")
```

**Bukan** `drizzle-orm/neon-http` atau `neon-serverless` - driver
tersebut memang dioptimalkan untuk Neon (HTTP-based, cocok
edge/serverless function), tapi mengunci kode ke API khusus Neon.

Karena base stack kita jalan di **Node.js runtime biasa** (bukan
Cloudflare Workers/Vercel Edge), driver generik `pg` sudah paling tepat
- berjalan sama persis di Neon, Supabase, AWS RDS, self-hosted
PostgreSQL, atau provider mana pun yang bicara protokol PostgreSQL
standar.

```ts
// shared/database/drizzle/client.ts
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
});

export const db = drizzle(pool);
```

**Satu-satunya file yang perlu diubah kalau pindah provider:** file di
atas + environment variable `DATABASE_URL`. Tidak ada file lain
(`*.repository.impl.ts`, use case, entity) yang menyentuh driver
koneksi secara langsung - semua akses lewat instance `db` yang
di-export dari sini. Primary key juga DB-agnostic: semua tabel pakai
ULID - lihat Section 19. Contoh konkret pembagiannya (Docker di lokal,
Neon di staging/production) ada di Section 17.

### Dua Runtime: Node + Cloudflare Workers (Dual Entry)

Backend bisa dijalankan di DUA runtime dengan satu composition root
(`app.ts`) yang sama:

| Runtime | Entry | Koneksi DB | Email | Catatan |
| --- | --- | --- | --- | --- |
| Node (default dev/test) | `main.ts` (`@hono/node-server`) | `pg` TCP langsung (`DATABASE_URL`) | SMTP (nodemailer) atau Resend | `pnpm dev` |
| Cloudflare Workers | `worker.ts` (`export default { fetch }`) | **driver Neon serverless (WebSocket)** via secret `DATABASE_URL` - pool PER-REQUEST (AsyncLocalStorage, lihat `client.ts`) | Resend (HTTP) | `pnpm dev:worker` / `pnpm deploy` |

Keputusan penting (hasil diagnosis staging 2026-09-17):

- **Workers ≠ driver `pg`/node:net** - terbukti flaky ±25% (socket bisu
  "Query read timeout", konsisten dengan/tanpa Hyperdrive). Jalur yang
  stabil: `@neondatabase/serverless` (WebSocket, transaksi didukung).
- **WebSocket = I/O milik request pembuatnya** - Workers melarang
  objek I/O dipakai lintas request ("Cannot perform I/O on behalf of a
  different request"). Karena itu `client.ts` di Workers adalah FACADE
  per-request: middleware `requestDb` membuat pool per request dan
  menutupnya via `waitUntil`; seluruh modul lain tetap `import { db }`
  tanpa perubahan. Type disatukan lewat satu cast (query builder dan
  transaction kedua driver identik di runtime).
- **Konsekuensi disadari**: jalur Workers terikat adapter Neon (§8 direvisi
  dari posisi awal yang menolak driver Neon). Mitigasi: DB tetap Postgres
  standar (dump/restore ke mana pun) + entry Node tetap memakai `pg`
  generik - keluar dari Neon = keluar dari Workers, bukan rewrite.
- **Hyperdrive tidak dipakai jalur Workers** saat ini (driver WS tidak
  bicara protokol wire Postgres; dan kombinasi Hyperdrive+pg kena flaky
  di atas). `neon-http` tetap ditolak: transaksi tidak didukung. D1
  tetap ditolak: ganti dialek total (`ilike`, `selectDistinctOn`,
  error code PG, riwayat migration).
- Impl provider runtime-agnostic: password hashing **PBKDF2 via Web
  Crypto** (100.000 iterasi - plafon Workers; hash-wasm/argon2 TIDAK
  bisa: Workers melarang kompilasi WASM dinamis, bahkan saat
  module-init), signing ImageKit pakai Web Crypto, logger = JSON via
  `console` (Workers Logs).
- `worker.ts` memakai **lazy import**: isi `process.env` dari bindings
  dulu, baru `import('./app')` - karena `env.ts`/`client.ts` membaca
  env saat module load. Migration TETAP dari CI Node (`drizzle-kit
  migrate` dengan direct URL).
- Rate limiter in-memory di Workers bersifat **per-isolate** - jalan
  untuk awal, upgrade ke Durable Objects/Redis kalau disalahgunakan.

### Hal yang Perlu Dihindari agar Tetap Portable

- ❌ Jangan pakai fitur eksklusif provider di dalam application logic
  (misal Neon database branching untuk alur bisnis, bukan cuma
  CI/preview environment)
- ❌ Jangan hardcode connection string atau nama provider di kode manapun
  selain `client.ts` dan `shared/config/env.ts`
- ❌ Jangan pakai tipe data/ekstensi PostgreSQL yang tidak didukung
  provider lain tanpa mendokumentasikannya
- ✅ Environment variable (`DATABASE_URL`) adalah satu-satunya sumber
  informasi soal provider mana yang sedang dipakai
- ✅ File migration SQL yang di-generate Drizzle Kit adalah SQL standar
  PostgreSQL - portable ke provider mana pun tanpa modifikasi
- ✅ Provider eksternal (SMTP, ImageKit, dst) HANYA boleh dipanggil lewat
  port di `application/ports/` + implementasi di `infrastructure/`
  (pola `MailerPort`, `ImageStoragePort`) - kode hanya bergantung pada
  interface, ganti provider = ganti satu file impl

---

## 9. Dokumentasi API Otomatis (OpenAPI + Scalar)

Prinsip: **dokumentasi API tidak ditulis manual terpisah** - di-generate
otomatis dari schema Zod yang sudah dipakai untuk validasi request.
Tidak ada dua sumber kebenaran (validasi vs dokumentasi) yang bisa
saling tidak sinkron.

### Library yang Dipakai

| Library                        | Fungsi                                                                                                                     |
| ------------------------------ | -------------------------------------------------------------------------------------------------------------------------- |
| `@hono/zod-openapi`          | Pengganti Hono biasa untuk route - schema Zod yang sama dipakai untuk validasi request/response DAN generate OpenAPI spec |
| `@scalar/hono-api-reference` | Middleware Hono yang render UI dokumentasi interaktif dari OpenAPI spec (menggantikan Swagger UI)                          |

### Perubahan pada Struktur Validator

Sebelumnya validator ditulis terpisah dengan `@hono/zod-validator`.
Dengan `@hono/zod-openapi`, definisi route + schema + docs jadi satu
kesatuan, tetap diletakkan di `presentation/` tiap modul:

```ts
// modules/auth/presentation/v1/auth.routes.ts
import { createRoute, OpenAPIHono } from '@hono/zod-openapi';
import { loginSchema, loginResponseSchema } from './validators/login.validator';

const app = new OpenAPIHono();

const loginRoute = createRoute({
  method: 'post',
  path: '/login',
  tags: ['Auth'],
  summary: 'Login user dan dapatkan access token',
  request: {
    body: {
      content: { 'application/json': { schema: loginSchema } },
    },
  },
  responses: {
    200: {
      description: 'Login berhasil',
      content: { 'application/json': { schema: loginResponseSchema } },
    },
    401: { description: 'Email atau password salah' },
  },
});

app.openapi(loginRoute, async (c) => {
  const body = c.req.valid('json');   // sudah tervalidasi Zod otomatis
  // panggil use case, dst
});

export default app;
```

Setiap kali sebuah endpoint baru dibuat dengan pola `createRoute` +
`app.openapi(...)`, dokumentasinya **otomatis muncul** di halaman
Scalar tanpa langkah tambahan apa pun - cukup isi `summary`, `tags`,
dan schema request/response yang memang sudah wajib ditulis untuk
validasi.

### Setup di Composition Root

```ts
// main.ts
import { OpenAPIHono } from '@hono/zod-openapi';
import { apiReference } from '@scalar/hono-api-reference';

const app = new OpenAPIHono();

// register semua route modul
app.route('/api/auth', authRoutes);
app.route('/api/words', wordRoutes);

// generate OpenAPI spec (JSON) di /openapi.json
app.doc('/openapi.json', {
  openapi: '3.0.0',
  info: {
    title: 'Kamus Digital Sambas-Indonesia API',
    version: '1.0.0',
  },
});

// halaman dokumentasi interaktif Scalar
app.get(
  '/docs',
  apiReference({
    spec: { url: '/openapi.json' },
  }),
);
```

Dengan ini, `GET /docs` selalu menampilkan dokumentasi API yang
mencerminkan kode saat itu juga - setiap prompt fitur baru yang
dijalankan (mengikuti pola `createRoute` di atas) otomatis menambah
entri baru di dokumentasi tanpa perlu ditulis manual.

### Aturan untuk Setiap Prompt Fitur Baru

Mulai sekarang, setiap prompt endpoint baru (yang sudah ada maupun akan
datang) **wajib** mengikuti pola `@hono/zod-openapi` di atas:

- Ganti `Hono()` biasa jadi `OpenAPIHono()` di tiap `*.routes.ts`
- Definisikan tiap endpoint dengan `createRoute()`, isi `summary`,
  `tags`, dan schema request/response
- Gunakan `app.openapi(route, handler)`, bukan `app.post(path, handler)`
  biasa

---

## 10. Strategi Testing

Cakupan: **unit test + integration test + E2E test**, ketiganya
dijalankan dengan **Vitest**. File test **terpisah di folder
`__tests__/` per modul** - tidak co-located dengan source, dan tidak
digabung jadi satu folder `tests/` di root.

### Struktur Folder per Modul

```
modules/auth/
├── domain/
├── application/
├── infrastructure/
├── presentation/
└── __tests__/
    ├── unit/
    │   ├── register-user.use-case.test.ts
    │   ├── login-user.use-case.test.ts
    │   └── password.vo.test.ts
    ├── integration/
    │   ├── user.repository.impl.test.ts
    │   └── refresh-token.repository.impl.test.ts
    └── e2e/
        └── auth.e2e.test.ts
```

Pola yang sama berlaku untuk `modules/word/__tests__/`, dan modul
lainnya.

### 1. Unit Test - Domain & Application Layer

**Target:** use case, entity, value object. **Tidak menyentuh database
atau HTTP sama sekali.**

Karena `application/` hanya bergantung pada *interface*
(`*.repository.ts`, `*.port.ts`), dependency di-mock manual dengan
`vi.fn()` - tidak perlu database beneran, tidak perlu jalankan server.

```ts
// modules/auth/__tests__/unit/login-user.use-case.test.ts
import { describe, it, expect, vi } from 'vitest';
import { LoginUserUseCase } from '../../application/use-cases/login-user.use-case';

describe('LoginUserUseCase', () => {
  it('menolak login jika password salah', async () => {
    const mockUserRepo = {
      findByEmail: vi.fn().mockResolvedValue({ id: '01HXYZABCDEF1234567890', password_hash: 'hash' }),
    };
    const mockHasher = {
      compare: vi.fn().mockResolvedValue(false),
    };
    const useCase = new LoginUserUseCase(mockUserRepo, mockHasher, /* ... */);

    await expect(
      useCase.execute({ email: 'a@a.com', password: 'salah' }),
    ).rejects.toThrow('INVALID_CREDENTIALS');
  });
});
```

Ini lapisan test **paling banyak jumlahnya** dan **paling cepat
dijalankan** - jadi prioritas utama coverage.

### 2. Integration Test - Infrastructure Layer

**Target:** implementasi repository (`*.repository.impl.ts`) yang
benar-benar bicara ke PostgreSQL. Butuh **database test terpisah**
(bukan database dev/production).

Setup:

- Docker Compose menyediakan PostgreSQL khusus test (port berbeda dari
  dev)
- Migration dijalankan (`drizzle-kit migrate`) ke database test sebelum
  test suite berjalan
- Tiap test dibungkus transaction yang di-rollback setelah selesai
  (atau truncate tabel di `afterEach`) - supaya test tidak saling
  mempengaruhi data

```ts
// modules/auth/__tests__/integration/user.repository.impl.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { UserRepositoryImpl } from '../../infrastructure/user.repository.impl';
import { testDb } from '@/shared/database/drizzle/test-client';

describe('UserRepositoryImpl', () => {
  const repo = new UserRepositoryImpl(testDb);

  beforeEach(async () => {
    await testDb.delete(usersTable);
  });

  it('menyimpan dan mengambil user by email', async () => {
    await repo.save({ username: 'budi', email: 'budi@test.com', /* ... */ });
    const found = await repo.findByEmail('budi@test.com');
    expect(found?.username).toBe('budi');
  });
});
```

### 3. E2E Test - Full HTTP Flow

**Target:** endpoint lengkap dari HTTP request masuk sampai response
keluar, lewat instance Hono yang sesungguhnya (pakai `hono/testing`,
bukan server yang benar-benar listen di port).

```ts
// modules/auth/__tests__/e2e/auth.e2e.test.ts
import { describe, it, expect } from 'vitest';
import { testClient } from 'hono/testing';
import app from '@/main';

describe('POST /api/auth/login (E2E)', () => {
  it('mengembalikan access_token untuk kredensial valid', async () => {
    const res = await testClient(app).api.auth.login.$post({
      json: { email: 'seeded@test.com', password: 'Password123' },
    });
    expect(res.status).toBe(200);
    const body = await res.json();
    expect(body.data.access_token).toBeDefined();
  });

  it('menolak kredensial salah dengan pesan generik', async () => {
    const res = await testClient(app).api.auth.login.$post({
      json: { email: 'seeded@test.com', password: 'salah' },
    });
    expect(res.status).toBe(401);
  });
});
```

E2E test butuh database test yang sudah di-**seed** data awal (user
contoh, kata contoh) sebelum suite berjalan - biasanya lewat script
`seed-test-db.ts` yang dijalankan di `globalSetup` Vitest.

### Command & Konfigurasi

```jsonc
// package.json (scripts)
{
  "test": "vitest run",
  "test:unit": "vitest run --dir '**/__tests__/unit'",
  "test:integration": "vitest run --dir '**/__tests__/integration'",
  "test:e2e": "vitest run --dir '**/__tests__/e2e'",
  "test:watch": "vitest"
}
```

Integration & E2E test butuh `DATABASE_URL` yang mengarah ke database
test (bukan dev/production) - diset lewat `.env.test`, dimuat khusus
saat `NODE_ENV=test`.

### Aturan Wajib

- **Setiap use case baru wajib punya unit test** sebelum PR di-merge -
  ini yang paling murah dan cepat, tidak ada alasan untuk skip
- **Setiap repository implementasi baru wajib punya integration test**
  minimal untuk operasi create & read
- **Setiap endpoint baru wajib punya minimal 1 E2E test** untuk
  happy path + 1 untuk skenario gagal (validasi/auth)
- Integration & E2E test **tidak boleh** jalan ke database
  dev/production - selalu ke database test terisolasi
- CI (GitHub Actions) menjalankan `test:unit` di setiap push/PR, dan
  `test:integration` + `test:e2e` sebelum job migration/deploy
  (lihat Section 7 & diskusi CI/CD sebelumnya)

---

## 11. API Versioning

Prinsip: **breaking change tidak boleh langsung mengubah endpoint yang
sudah dipakai client lain** (aplikasi admin, aplikasi mobile nanti, atau
pihak ketiga yang mengonsumsi API kamus/translator ini). Versi lama
tetap jalan sampai resmi di-deprecate, versi baru dijalankan
berdampingan.

### Strategi: URI Versioning

Dipilih **URI path versioning** (`/api/v1/...`) - bukan header-based
(`Accept-Version: v1`) atau query param (`?version=1`). Alasan: paling
eksplisit, gampang di-debug (kelihatan langsung di URL/log), gampang
di-routing terpisah di Hono, dan paling umum dipahami konsumen API.

```
/api/v1/auth/login
/api/v1/auth/register
/api/v1/words
/api/v1/words/:id

/api/v2/words        ← breaking change, v1 tetap jalan berdampingan
```

### Dampak ke Struktur Folder

Versioning **tidak** menduplikasi seluruh modul. Domain & application
layer (business logic) tetap satu - yang berubah karena breaking change
biasanya di **presentation layer** (bentuk request/response) atau
kadang di use case tertentu saja.

```
modules/word/
├── domain/                      # tetap satu, tidak per-versi
├── application/
│   └── use-cases/
│       ├── create-word.use-case.ts        # dipakai v1 & v2
│       └── search-words-v2.use-case.ts    # hanya ada jika logic v2 beda total
├── infrastructure/               # tetap satu
└── presentation/
    ├── v1/
    │   ├── word.routes.ts
    │   ├── word.controller.ts
    │   └── validators/
    │       └── create-word.validator.ts
    └── v2/
        ├── word.routes.ts        # bentuk request/response baru
        ├── word.controller.ts    # tetap panggil use case yang sama,
        │                         # cuma mapping DTO yang beda
        └── validators/
            └── create-word.validator.ts
```

**Aturan:** kalau breaking change **hanya** soal bentuk request/response
(rename field, ubah struktur JSON), cukup buat controller & validator
baru di `presentation/v2/` yang tetap memanggil use case v1 yang sama.
Kalau breaking change menyangkut **business logic**, baru buat use case
baru - tapi ini jarang terjadi dan sebaiknya dihindari (business logic
harusnya versi-agnostic).

### Registrasi di Composition Root

```ts
// main.ts
import { OpenAPIHono } from '@hono/zod-openapi';
import authRoutesV1 from '@/modules/auth/presentation/v1/auth.routes';
import wordRoutesV1 from '@/modules/word/presentation/v1/word.routes';
import wordRoutesV2 from '@/modules/word/presentation/v2/word.routes';

const app = new OpenAPIHono();

app.route('/api/v1/auth', authRoutesV1);
app.route('/api/v1/words', wordRoutesV1);
app.route('/api/v2/words', wordRoutesV2);   // v1/words tetap aktif

app.doc('/openapi.json', {
  openapi: '3.0.0',
  info: { title: 'Kamus Digital Sambas-Indonesia API', version: '2.0.0' },
});

app.get('/docs', apiReference({ spec: { url: '/openapi.json' } }));
```

> **Catatan:** semua contoh path di section-section sebelumnya
> (`/api/auth/login`, `/api/admin/words`, dst) mengasumsikan prefix
> `/api/v1/` - anggap `/api/v1/auth/login`, `/api/v1/admin/words`, dan
> seterusnya.

### Apa yang Termasuk Breaking Change (wajib naik versi)

- Menghapus atau me-rename field di response
- Mengubah tipe data field (misal `id: number` → `id: string`)
- Menambah field **wajib** baru di request body
- Mengubah struktur nested object
- Mengubah kode status HTTP untuk skenario yang sama

### Apa yang TIDAK Perlu Naik Versi (non-breaking, aman ditambah ke versi berjalan)

- Menambah field **opsional** baru di response
- Menambah endpoint baru
- Menambah field **opsional** baru di request body
- Memperbaiki bug tanpa mengubah kontrak API

### Kebijakan Deprecation

1. Saat `v2` rilis, `v1` **tidak langsung dimatikan** - beri masa
   transisi (misal 3-6 bulan, sesuaikan jumlah konsumen API)
2. Endpoint versi lama diberi response header penanda:
   `Deprecation: true` dan `Sunset: <tanggal-mati>` (standar HTTP,
   `RFC 8594`)
3. Dokumentasikan migrasi v1 → v2 di changelog terpisah
   (`CHANGELOG-API.md`) - apa yang berubah, contoh before/after
4. Setelah masa transisi berakhir, endpoint v1 dihapus di rilis
   berikutnya (bukan tiba-tiba, harus ada pengumuman)

### Dampak ke Testing (Section 10)

Setiap versi punya E2E test terpisah:

```
modules/word/__tests__/
├── unit/                    # tidak perlu duplikasi, business logic sama
├── integration/             # tidak perlu duplikasi
└── e2e/
    ├── v1/
    │   └── words.e2e.test.ts
    └── v2/
        └── words.e2e.test.ts
```

Ini memastikan v1 yang masih dipakai konsumen lama **tidak diam-diam
rusak** ketika ada perubahan untuk keperluan v2.

---

## 12. Environment & Secrets Management

Prinsip: **satu sumber kebenaran untuk konfigurasi**, tervalidasi saat
aplikasi start (bukan gagal diam-diam di tengah request), dan **tidak
pernah** ada secret ter-commit ke git.

### Validasi Environment via Zod

```ts
// shared/config/env.ts
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'staging', 'production']),
  PORT: z.coerce.number().default(3000),

  DATABASE_URL: z.string().url(),

  JWT_PRIVATE_KEY: z.string().min(1),
  JWT_PUBLIC_KEY: z.string().min(1),
  JWT_ACCESS_TOKEN_TTL: z.coerce.number().default(900),        // 15 menit
  JWT_REFRESH_TOKEN_TTL: z.coerce.number().default(2592000),   // 30 hari

  SMTP_HOST: z.string().optional(),
  SMTP_PORT: z.coerce.number().optional(),
  SMTP_USER: z.string().optional(),
  SMTP_PASSWORD: z.string().optional(),

  CORS_ALLOWED_ORIGINS: z.string(),   // comma-separated, di-parse jadi array
});

export const env = envSchema.parse(process.env);
// Aplikasi CRASH saat start kalau ada env wajib yang hilang/salah format
// - lebih baik gagal cepat di awal daripada error tak jelas di production
```

Semua kode lain **wajib** import `env` dari file ini - dilarang akses
`process.env` langsung di file manapun selain `env.ts`.

### File `.env.example`

Wajib ada di root repo (di-commit ke git), berisi daftar semua variable
**tanpa nilai asli**, sebagai dokumentasi + template:

```
NODE_ENV=development
PORT=3000

DATABASE_URL=postgresql://user:password@localhost:5432/db_sambasku

JWT_PRIVATE_KEY=
JWT_PUBLIC_KEY=
JWT_ACCESS_TOKEN_TTL=900
JWT_REFRESH_TOKEN_TTL=2592000

SMTP_HOST=
SMTP_PORT=
SMTP_USER=
SMTP_PASSWORD=

CORS_ALLOWED_ORIGINS=http://localhost:5173
```

### Strategi per Environment

| Environment         | Sumber env var                                                                   | Catatan                                                                                              |
| ------------------- | -------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| Local dev           | `.env` (git-ignored)                                                           | Copy dari`.env.example`, isi manual                                                                |
| Test                | `.env.test` (git-ignored)                                                      | `DATABASE_URL` mengarah ke DB test terpisah (Section 10)                                           |
| CI (GitHub Actions) | GitHub Secrets                                                                   | Diset lewat Settings → Secrets, di-inject sebagai env di workflow                                   |
| Staging/Production (Node)  | GitHub Environments + secrets provider hosting (Railway/Render/Fly.io vars, dst) | Terpisah per environment, akses dibatasi + butuh approval untuk deploy production (lihat Section 16) |
| Production (Workers) | `wrangler.toml [vars]` (non-secret) + `wrangler secret put` (secret) + binding Hyperdrive | Setup lengkap di Section 17 - "Deploy ke Cloudflare Workers"; DB lewat `HYPERDRIVE.connectionString` |

### Aturan Wajib

- `.env` dan `.env.test` **wajib** masuk `.gitignore` - hanya
  `.env.example` yang di-commit
- **Rotasi JWT key**: jika `JWT_PRIVATE_KEY` pernah bocor/ter-commit
  tidak sengaja, generate pasangan key baru - ini akan invalidate semua
  access token yang beredar (aman by design, karena access token umur
  pendek/15 menit)
- Untuk tim yang berkembang, pertimbangkan secrets manager terpisah
  (Infisical, Doppler, atau 1Password Secrets) alih-alih env var mentah
  di platform hosting - belum wajib di skala proyek saat ini, tapi
  dicatat sebagai upgrade path

---

## 13. Global Error Handling & Response Envelope

Prinsip: **satu bentuk response** untuk semua endpoint, sukses maupun
gagal, di seluruh modul - supaya konsumen API (frontend admin, aplikasi
mobile nanti) hanya perlu menangani satu pola parsing.

### Format Envelope Standar

**Sukses (single resource):**

```json
{
  "success": true,
  "data": { "id": "01HXYZABCDEF1234567890", "lemma": "makatn" }
}
```

**Sukses (list, dengan pagination cursor - lihat subsection "Pagination" di bawah):**

```json
{
  "success": true,
  "data": [ { "id": "01HXYZABCDEF1234567890", "lemma": "makatn" } ],
  "meta": {
    "limit": 20,
    "next_cursor": "01HXYZABCDEF1234567891",
    "has_more": true
  }
}
```

`next_cursor: null` + `has_more: false` = halaman terakhir.

**Gagal:**

```json
{
  "success": false,
  "error_code": "WORD_NOT_FOUND",
  "message": "Kata dengan id tersebut tidak ditemukan",
  "details": null
}
```

`details` diisi array kalau error validasi (banyak field sekaligus):

```json
{
  "success": false,
  "error_code": "VALIDATION_ERROR",
  "message": "Data yang dikirim tidak valid",
  "details": [
    { "field": "lemma", "message": "Kata tidak boleh kosong" },
    { "field": "meanings", "message": "Minimal harus ada 1 makna" }
  ]
}
```

### Pagination - Cursor-Based (WAJIB untuk semua endpoint list)

Semua endpoint yang mengembalikan list memakai **cursor-based
pagination**, bukan offset - `LIMIT/OFFSET` dan `COUNT(*)` dilarang:

- `OFFSET n` memindai dan membuang n baris - makin dalam halaman,
  makin lambat (O(n) per halaman)
- `COUNT(*)` untuk `total_items`/`total_pages` makin mahal seiring
  tabel membesar - dan angkanya langsung basi begitu ada insert lain

**Cursor = ULID `id` item terakhir halaman sebelumnya.** Ini nyaris
gratis karena ULID lexicographically sortable by waktu (Section 19) -
`ORDER BY id` = urut waktu pembuatan, dan `id` selalu ter-index (PK).

Kontrak per endpoint list:

| Aspek                           | Bentuk                                                                                                       |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| Query                           | `?limit=20&cursor=<ULID opsional>` (halaman pertama tanpa cursor)                                          |
| Urutan                          | `ORDER BY id DESC` (terbaru dulu) kecuali disebut lain                                                     |
| Halaman berikutnya              | `WHERE id < cursor ORDER BY id DESC LIMIT limit + 1`                                                       |
| `has_more`                    | hasil fetch`limit + 1` → lebih dari `limit` berarti `true`, item ke-n+1 dibuang                       |
| `meta`                        | `{ "limit": 20, "next_cursor": "<ULID>\|null", "has_more": bool }` - TANPA `total_items`/`total_pages` |
| Sort kolom lain (mis.`lemma`) | hanya boleh kalau cursor tetap unik & ter-index - paling aman tetap`id`                                   |

Konsumen (web/admin) memakai pola *infinite scroll* / tombol
"Muat lagi": render `data`, kalau `has_more` tampilkan tombol yang
mengirim `cursor = next_cursor`.

### Hierarki Error di Domain/Application Layer

```ts
// shared/errors/app-error.ts
export abstract class AppError extends Error {
  abstract readonly statusCode: number;
  abstract readonly errorCode: string;
}

export class ValidationError extends AppError {
  statusCode = 400;
  errorCode = 'VALIDATION_ERROR';
  constructor(public details: { field: string; message: string }[]) {
    super('Data yang dikirim tidak valid');
  }
}

export class NotFoundError extends AppError {
  statusCode = 404;
  constructor(errorCode: string, message: string) {
    super(message);
    this.errorCode = errorCode;
  }
  errorCode: string;
}

export class UnauthorizedError extends AppError {
  statusCode = 401;
  errorCode: string;
  // errorCode bisa dioverride untuk kode spesifik: INVALID_CREDENTIALS, TOKEN_EXPIRED
  constructor(errorCode = 'UNAUTHORIZED', message = 'Tidak terautentikasi') {
    super(message);
    this.errorCode = errorCode;
  }
}

export class ForbiddenError extends AppError {
  statusCode = 403;
  errorCode = 'FORBIDDEN';
}

export class ConflictError extends AppError {
  statusCode = 409;
  errorCode: string;
  // errorCode bisa dioverride untuk kode spesifik: EMAIL_ALREADY_EXISTS, dst
  constructor(errorCode = 'CONFLICT', message = 'Konflik data') {
    super(message);
    this.errorCode = errorCode;
  }
}
```

Use case di `application/` melempar error class ini langsung (misal
`throw new NotFoundError('WORD_NOT_FOUND', 'Kata tidak ditemukan')`) -
**tidak** tahu soal Hono atau HTTP status code, cuma tahu jenis
errornya.

### Middleware Global (Satu Tempat, Dipakai Semua Modul)

```ts
// shared/middlewares/error-handler.middleware.ts
import type { ErrorHandler } from 'hono';
import { AppError } from '@/shared/errors/app-error';
import { logger } from '@/shared/logging/logger';   // lihat Section 14

export const errorHandler: ErrorHandler = (err, c) => {
  if (err instanceof AppError) {
    return c.json(
      {
        success: false,
        error_code: err.errorCode,
        message: err.message,
        details: 'details' in err ? err.details : null,
      },
      err.statusCode as any,
    );
  }

  // Error tak terduga - jangan bocorkan detail internal ke client
  logger.error({ err, request_id: c.get('requestId') }, 'Unhandled error');
  return c.json(
    { success: false, error_code: 'INTERNAL_ERROR', message: 'Terjadi kesalahan pada server' },
    500,
  );
};
```

Didaftarkan sekali di `main.ts` via `app.onError(errorHandler)` -
**tidak perlu try-catch manual di tiap controller.**

### Route Tidak Ditemukan (404)

Hono punya **dua jalur error yang berbeda** - keduanya wajib di-handle:

| Jalur            | Pemicu                               | Handler                    |
| ---------------- | ------------------------------------ | -------------------------- |
| `app.onError`  | Error dilempar dari handler/use case | `errorHandler` (di atas) |
| `app.notFound` | Request tidak cocok route manapun    | wajib didaftarkan sendiri  |

Tanpa `app.notFound`, Hono membalas plain text `404 Not Found` -
**melanggar prinsip envelope satu bentuk response**. Daftarkan sekali
di composition root:

```ts
// main.ts / app.ts
app.notFound((c) =>
  c.json(
    {
      success: false,
      error_code: 'NOT_FOUND',
      message: 'Route tidak ditemukan',
      details: null,
    },
    404,
  ),
);
```

Root (`GET /`) juga jangan dibiarkan 404 - cukup satu endpoint info
kecil yang menunjuk ke dokumentasi (meta route, tidak perlu ikut
OpenAPI spec):

```ts
app.get('/', (c) =>
  c.json({
    success: true,
    data: { name: 'Kamus Digital Sambas-Indonesia API', version: '1.0.0', docs: '/docs' },
  }),
);
```

### Katalog Error Code

Wajib dijaga sebagai dokumen hidup terpisah (`ERROR_CODES.md`), diupdate
tiap kali ada `errorCode` baru ditambahkan - supaya konsumen API punya
referensi lengkap tanpa harus baca kode:

| error_code                   | HTTP Status | Contoh Kapan Muncul                                                 |
| ---------------------------- | ----------- | ------------------------------------------------------------------- |
| `VALIDATION_ERROR`         | 400         | Body request tidak lolos Zod schema                                 |
| `INVALID_CREDENTIALS`      | 401         | Login gagal                                                         |
| `UNAUTHORIZED`             | 401         | Token tidak ada/invalid, atau refresh token tidak valid             |
| `TOKEN_EXPIRED`            | 401         | Access token kadaluarsa                                             |
| `RESET_TOKEN_INVALID`      | 401         | Token reset password tidak valid, kadaluarsa, atau sudah dipakai    |
| `FORBIDDEN`                | 403         | Role tidak diizinkan akses endpoint                                 |
| `NOT_FOUND`                | 404         | Route/endpoint tidak ditemukan (via`app.notFound`)                |
| `WORD_NOT_FOUND`           | 404         | Kata tidak ditemukan by id                                          |
| `MEANING_NOT_FOUND`        | 404         | Makna tidak ditemukan by id (kontribusi contoh kalimat)             |
| `CONTRIBUTION_NOT_FOUND`   | 404         | Kontribusi tidak ditemukan by id (antrean review)                   |
| `SEARCH_MISS_NOT_FOUND`    | 404         | Pencarian kosong tidak ditemukan by id (dismiss panel admin)        |
| `EMAIL_ALREADY_EXISTS`     | 409         | Registrasi dengan email yang sudah dipakai                          |
| `CONTRIBUTION_ALREADY_REVIEWED` | 409    | Kontribusi sudah punya keputusan (approve/reject/correct)           |
| `USERNAME_ALREADY_EXISTS`  | 409         | Registrasi dengan username yang sudah dipakai                       |
| `RATE_LIMITED`             | 429         | Terlalu banyak percobaan (lihat tabel limit Section 15)             |
| `INTERNAL_ERROR`           | 500         | Error tak terduga (bug, koneksi DB putus, dst)                      |
| `IMAGE_UPLOAD_UNAVAILABLE` | 503         | Provider penyimpanan gambar belum dikonfigurasi (env`IMAGEKIT_*`) |

### Aturan Wajib

- Semua endpoint baru **wajib** ikut format envelope ini - tidak ada
  pengecualian "response custom" per modul
- Error code baru **wajib** ditambahkan ke `ERROR_CODES.md` di PR yang
  sama
- Pesan `message` untuk end-user (Bahasa Indonesia, ramah), detail
  teknis (stack trace, dst) hanya masuk log - tidak pernah ke response

---

## 14. Logging & Observability

Prinsip: **structured logging** (JSON, bukan `console.log` string bebas)
supaya bisa di-query/filter nanti, dan **setiap request bisa dilacak**
lewat satu request ID yang konsisten dari masuk sampai keluar -
termasuk terhubung ke tabel `audit_logs.request_id` yang sudah ada di
skema database.

### Library: Pino

```ts
// shared/logging/logger.ts
import pino from 'pino';
import { env } from '@/shared/config/env';

export const logger = pino({
  level: env.NODE_ENV === 'production' ? 'info' : 'debug',
  transport:
    env.NODE_ENV !== 'production'
      ? { target: 'pino-pretty' }   // human-readable saat development
      : undefined,                  // JSON murni saat production
  redact: ['password', 'password_hash', 'token', 'access_token', 'refresh_token'],
});
```

`redact` mencegah data sensitif ikut ter-log meski developer lupa -
lapisan pengaman tambahan di atas aturan "jangan pernah log password"
yang sudah disebut di modul auth.

### Request ID Middleware

```ts
// shared/middlewares/request-id.middleware.ts
import { createMiddleware } from 'hono/factory';
import { randomUUID } from 'crypto';

export const requestIdMiddleware = createMiddleware(async (c, next) => {
  const requestId = c.req.header('X-Request-Id') ?? randomUUID();
  c.set('requestId', requestId);
  c.header('X-Request-Id', requestId);
  await next();
});
```

`requestId` ini yang dipakai sebagai `contributions.request_id` /
`audit_logs.request_id` saat use case melakukan perubahan data -
menyambungkan log aplikasi dengan jejak audit di database.

### Apa yang Di-log

| Level     | Kapan dipakai                                                                                               |
| --------- | ----------------------------------------------------------------------------------------------------------- |
| `debug` | Detail teknis untuk development (query yang dijalankan, dst) - mati di production                          |
| `info`  | Request masuk/keluar (method, path, status, durasi), event bisnis penting (user register, kata dipublikasi) |
| `warn`  | Kondisi tak normal tapi tidak fatal (rate limit tercapai, percobaan login gagal)                            |
| `error` | Exception tak tertangani, kegagalan koneksi eksternal (DB, SMTP)                                            |

### Yang TIDAK BOLEH Di-log

- Password (plain maupun hash)
- Access token / refresh token utuh
- Isi email lengkap pengguna di log level rendah (cukup user_id)
- Data pribadi lain yang tidak perlu untuk debugging

### Observability Lanjutan (Upgrade Path, Belum Wajib Sekarang)

- **Error tracking**: Sentry - cukup tambah SDK, otomatis capture
  exception dari `error-handler.middleware.ts`
- **Metrics/tracing**: OpenTelemetry - baru relevan kalau traffic sudah
  signifikan atau butuh debug performa lintas service
- Dicatat di sini sebagai pengingat, **tidak perlu diimplementasi di
  fase awal** - cukup structured logging dulu

---

## 15. Rate Limiting Global

Prinsip: **semua endpoint publik/menulis data dilindungi**, bukan cuma
login. Tujuannya cegah spam submission (misal bot submit ribuan kata
palsu) dan brute force di endpoint sensitif.

### Strategi Bertingkat

| Kategori endpoint           | Contoh                                                             | Limit                                   |
| --------------------------- | ------------------------------------------------------------------ | --------------------------------------- |
| Publik, baca saja           | `GET /api/v1/words`, `GET /api/v1/words/:id`                   | 100 request/menit per IP                |
| Publik, tulis (belum login) | `POST /api/v1/auth/register`, `POST /api/v1/contributions/words` (submit kata anonim) | 5 request/jam per IP                    |
| Auth sensitif               | `POST /api/v1/auth/login`, `POST /api/v1/auth/forgot-password` | 5 percobaan/15 menit per email+IP       |
| Sudah login, tulis data     | `POST /api/v1/words` (submit kata), `POST /api/v1/words/:id/pronunciations` / `/:id/images`, `POST /api/v1/meanings/:id/examples` (kontribusi media) | 30 request/menit per user_id            |
| Admin                       | Endpoint role admin/editor (termasuk antrean review `GET/POST /api/v1/admin/contributions/...`) | Longgar (500/menit) atau tidak dibatasi |

### Implementasi

Untuk single instance/awal: in-memory store cukup. Begitu ada rencana
scale ke multi-instance, **wajib pindah ke Redis-backed** supaya limit
konsisten lintas instance.

```ts
// shared/middlewares/rate-limit.middleware.ts
import { createMiddleware } from 'hono/factory';
import { RateLimiterMemory } from 'rate-limiter-flexible';
// ganti RateLimiterMemory -> RateLimiterRedis begitu multi-instance

export function rateLimit(opts: { points: number; duration: number; keyFn?: (c: any) => string }) {
  const limiter = new RateLimiterMemory({ points: opts.points, duration: opts.duration });

  return createMiddleware(async (c, next) => {
    const key = opts.keyFn ? opts.keyFn(c) : (c.req.header('x-forwarded-for') ?? 'unknown');
    try {
      await limiter.consume(key);
      await next();
    } catch {
      c.header('Retry-After', String(opts.duration));
      return c.json(
        { success: false, error_code: 'RATE_LIMITED', message: 'Terlalu banyak percobaan, coba lagi nanti' },
        429,
      );
    }
  });
}
```

Dipasang per route, bukan global blanket, supaya limit-nya sesuai
kategori di atas:

```ts
// modules/auth/presentation/v1/auth.routes.ts
authRoutes.use(
  '/login',
  rateLimit({
    points: 5,
    duration: 900,
    // ponytail: key IP saja - gabung dengan email (parse body) kalau perlu ketat
    keyFn: (c) => `login:${c.req.header('x-forwarded-for')}`,
  }),
);
authRoutes.openapi(loginRoute, loginHandler);   // Section 9: bukan app.post
```

### Aturan Wajib

- Endpoint baru yang menerima **write** dari publik/user biasa **wajib**
  dipasangi rate limit sejak dibuat, bukan ditambah belakangan setelah
  ada insiden
- Response 429 **wajib** sertakan header `Retry-After`
- Di belakang reverse proxy/load balancer, PASTIKAN `x-forwarded-for`
  diteruskan (proxy_set_header / set_real_ip_from di nginx, dst) -
  tanpa itu semua klien berbagi satu bucket rate limit (`unknown`)
  dan kolektif saling mengunci
- Limit untuk role admin/editor boleh lebih longgar, tapi tidak boleh
  benar-benar tanpa batas (jaga-jaga akun admin diretas)

---

## 16. CI/CD Pipeline Lengkap

Menyatukan yang sudah dibahas terpisah (migration di Section 7, testing
di Section 10) jadi satu alur utuh, dari commit sampai deploy.

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
  push:
    branches: [main, staging]

jobs:
  lint-and-typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm typecheck   # tsc --noEmit

  unit-test:
    needs: lint-and-typecheck
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile
      - run: pnpm test:unit

  integration-and-e2e-test:
    needs: unit-test
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: db_sambasku_test
        ports: ['5432:5432']
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile
      - run: pnpm drizzle-kit migrate
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/db_sambasku_test
      - run: pnpm test:integration && pnpm test:e2e
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/db_sambasku_test
```

```yaml
# .github/workflows/deploy-staging.yml
name: Deploy Staging

on:
  push:
    branches: [staging]

jobs:
  migrate:
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile
      - run: pnpm drizzle-kit migrate
        env:
          # Neon: direct (non-pooler) URL - lihat Section 17
          DATABASE_URL: ${{ secrets.STAGING_DATABASE_URL }}

  deploy:
    needs: migrate
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - run: echo "deploy ke staging di sini"
```

```yaml
# .github/workflows/deploy-production.yml
name: Deploy Production

on:
  push:
    tags: ['v*']   # deploy production hanya lewat git tag rilis

jobs:
  backup:
    runs-on: ubuntu-latest
    environment: production   # butuh manual approval (GitHub Environment protection rule)
    steps:
      - name: Backup database
        run: pg_dump ${{ secrets.PRODUCTION_DATABASE_URL }} > backup-$(date +%Y%m%d-%H%M%S).sql
      - uses: actions/upload-artifact@v4
        with: { name: db-backup, path: backup-*.sql }

  migrate:
    needs: backup
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - run: pnpm drizzle-kit migrate
        env:
          # Neon: direct (non-pooler) URL - lihat Section 17
          DATABASE_URL: ${{ secrets.PRODUCTION_DATABASE_URL }}

  deploy:
    needs: migrate
    runs-on: ubuntu-latest
    environment: production
    steps:
      - run: echo "deploy ke production di sini"
```

### Ringkasan Alur

```
PR dibuka          → lint + typecheck + unit test (cepat, feedback instan)
PR mau di-merge    → + integration test + e2e test (pakai Postgres service container)
Merge ke staging   → auto migrate + auto deploy staging (tanpa approval)
Tag rilis (v*)     → backup DB → manual approval → migrate → deploy production
```

### Aturan Wajib

- **Production hanya di-deploy lewat git tag**, bukan otomatis dari
  `main` - supaya rilis production selalu proses sadar/sengaja
- **`environment: production`** di GitHub wajib diset dengan *required
  reviewers* - tidak ada migrate/deploy production tanpa approval manusia
- Job `migrate` selalu `needs` sebelum job `deploy` di semua workflow -
  aplikasi tidak pernah start dengan skema database yang belum sinkron
- Backup **wajib** jalan sebelum migrate di production (tidak perlu di
  staging)

---

## 17. Strategi Multi-Environment

Prinsip: **minimal 3 environment terpisah total** - infrastruktur,
database, dan kredensial masing-masing tidak saling menyentuh. Tidak
ada environment yang berbagi database dengan environment lain, bahkan
untuk "sekadar coba cepat".

### Tier Environment

| Environment                              | Tujuan                                                                                           | Siapa yang akses               | Database                                                                             |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------ | ------------------------------ | ------------------------------------------------------------------------------------ |
| **Local (Development)**            | Coding sehari-hari di komputer masing-masing developer                                           | Developer individu             | PostgreSQL lokal (Docker Compose) atau branch Neon pribadi                           |
| **Development (opsional, shared)** | Integrasi awal antar developer sebelum masuk staging - bisa dilewati kalau tim masih kecil/solo | Semua developer, internal only | DB`dev`, boleh sering direset                                                      |
| **Staging**                        | Simulasi production - tempat QA, demo ke stakeholder, test sebelum rilis                        | Tim internal + tester          | DB`staging`, data mirip production (anonim/sample), **bukan** data asli user |
| **Production**                     | Yang dipakai pengguna sungguhan                                                                  | Publik                         | DB`production`, data asli - paling dilindungi                                     |

> Untuk skala tim saat ini (solo/kecil), **Development tier boleh
> dilewati** - cukup Local → Staging → Production. Tambahkan tier
> Development kalau nanti tim berkembang dan butuh tempat integrasi
> sebelum staging.

### Pemetaan Branch Git → Environment

```
feature/xxx  ──PR──▶  develop (opsional)  ──PR──▶  staging  ──tag rilis──▶  main (production)
     │                      │                         │                         │
  local dev            auto-deploy               auto-deploy              manual approval
  (tidak ada                ke dev                 ke staging              + backup dulu
   auto-deploy)          (jika dipakai)                                    → deploy production
```

- `main` = mencerminkan kondisi production saat ini
- `staging` = kandidat rilis berikutnya, sudah lolos semua test CI
- `develop` = opsional, integrasi harian (skip kalau tim kecil)
- `feature/*` = branch kerja per fitur, hanya dites di local + CI (belum
  ter-deploy ke environment manapun)

### Perbedaan Konfigurasi per Environment

Ini poin yang paling sering terlewat - bukan cuma `DATABASE_URL` yang
beda, tapi juga:

| Config                          | Local                                         | Staging                              | Production                                                 |
| ------------------------------- | --------------------------------------------- | ------------------------------------ | ---------------------------------------------------------- |
| `NODE_ENV`                    | `development`                               | `staging`                          | `production`                                             |
| Log level (Section 14)          | `debug`                                     | `info`                             | `info`                                                   |
| Format log                      | Human-readable (`pino-pretty`)              | JSON                                 | JSON                                                       |
| Rate limit (Section 15)         | Longgar/dimatikan                             | Sama seperti production              | Ketat sesuai tabel                                         |
| `CORS_ALLOWED_ORIGINS`        | `http://localhost:5173`                     | `https://staging.kamus-sambas.app` | `https://kamus-sambas.app`                               |
| Email (SMTP)                    | Mailtrap/Ethereal (sandbox, tidak kirim asli) | Mailtrap/sandbox provider            | SMTP asli (mengirim ke user sungguhan)                     |
| JWT key pair                    | Key dummy per developer                       | Key khusus staging                   | Key khusus production (paling dilindungi)                  |
| Error detail di response        | Boleh lebih verbose untuk debug               | Sama seperti production              | Pesan generik saja, detail cuma di log                     |
| Swagger/Scalar docs (`/docs`) | Aktif                                         | Aktif                                | Pertimbangkan lindungi dengan auth atau nonaktifkan publik |

### Database: Docker di Lokal, Neon di Staging/Production

Bisa, dan memang dirancang begitu (Section 8): **satu driver `pg`, satu
`DATABASE_URL`** - antar environment hanya nilainya yang berubah, kode
dan migration SQL-nya identik.

| Environment          | Database                                            | Sumber connection string     |
| -------------------- | --------------------------------------------------- | ---------------------------- |
| Local dev            | Docker Compose service`postgres` (port 5432)      | `.env`                     |
| Local test           | Docker Compose service`postgres-test` (port 5433) | `.env.test`                |
| CI (test job)        | Service container`postgres:16` di workflow        | env di workflow (Section 16) |
| Staging / Production | **Neon** (TCP standar via `pg`)             | GitHub secrets → env deploy |

Aturan Neon (wajib dipatuhi semua prompt/deploy):

- **App runtime** pakai connection string **pooled** - host-nya
  mengandung `-pooler` (PgBouncer Neon yang mengelola koneksi)
- **Migration** (`pnpm drizzle-kit migrate` di job CI/CD) pakai
  connection string **direct** (non-pooler) - DDL lewat transaction
  pooling bisa gagal. Jadi per environment Neon ada DUA secret:
  direct URL untuk job `migrate`, pooled URL untuk runtime app
- Selalu sertakan `?sslmode=require` di kedua URL
- **Tidak perlu** ganti driver ke `drizzle-orm/neon-http` - itu hanya
  untuk edge/serverless; runtime Node.js biasa cukup `pg` via TCP
  (Section 8), jadi kode tetap portable kalau suatu saat pindah dari
  Neon ke RDS/self-hosted
- Kalau ada instalasi PostgreSQL lain yang sudah memakai port 5432 di
  mesin lokal (mis. Homebrew), matikan dulu sebelum `docker compose up`
  - atau ubah mapping port di `docker-compose.yml` + `DATABASE_URL`

### Deploy ke Cloudflare Workers (Production)

Lanjutan tabel di atas - production bisa memilih Workers sebagai runtime
(lihat "Dua Runtime" di Section 8). Urutan setup PERTAMA kali:

```text
1. npx wrangler login
2. Secrets (jangan pernah di wrangler.toml):
     npx wrangler secret put DATABASE_URL       # Neon POOLED URL ?sslmode=require
                                                # (driver Neon serverless WS - lihat Section 8)
     npx wrangler secret put JWT_PRIVATE_KEY    # PEM (key pair khusus environment)
     npx wrangler secret put JWT_PUBLIC_KEY
     npx wrangler secret put RESEND_API_KEY     # email HTTP (opsional)
     npx wrangler secret put IMAGEKIT_PRIVATE_KEY # opsional
3. Migration tetap dari CI Node (GitHub Actions):
     pnpm drizzle-kit migrate  # Neon DIRECT URL (Section 16) - bukan via Workers
4. pnpm deploy   # = wrangler deploy (setelah CI hijau + backup DB, Section 16)
```

Aturan wajib tambahan (setara aturan Neon di atas):

- `[vars]` di `wrangler.toml` HANYA untuk config non-secret; semua
  kredensial lewat `wrangler secret put` - konsisten Section 12
  (`.env`/secrets tidak pernah masuk git)
- Simpan `wrangler.toml` + `worker.ts` di repo
- Log production = Workers Logs (logger JSON via `console` otomatis
  terekap; `wrangler tail --format json` untuk streaming + diagnosis)
- Smoke test pasca-deploy: `cd http && bruno run --env <env> auth/
  language/ word/ contribution/ search-miss/ category/ audit/`
  (environment Bruno lokal, tidak di-commit - Section 20)

### Kaitan dengan Section Sebelumnya

- **Section 12 (Env & Secrets)**: `envSchema` sudah mendukung
  `NODE_ENV: z.enum(['development', 'test', 'staging', 'production'])`
  - jadi validasi env sudah siap multi-environment sejak awal
- **Section 16 (CI/CD)**: workflow `deploy-staging.yml` dan
  `deploy-production.yml` sudah dipisah total, masing-masing pakai
  `secrets.STAGING_DATABASE_URL` vs `secrets.PRODUCTION_DATABASE_URL` -
  tinggal tambah `deploy-dev.yml` dengan pola sama kalau tier Development
  dipakai
- **Section 7 (Migration)**: migration dijalankan terpisah per
  environment (`migrate` job masing-masing workflow) - skema staging dan
  production bisa saja beda sementara di masa transisi rilis, tapi harus
  konvergen

### Aturan Wajib

- **Tidak ada environment yang memakai `DATABASE_URL` environment lain**
  - bahkan untuk debug cepat. Kalau perlu data mirip production di
  staging, lakukan **anonymized data seeding**, bukan copy langsung dari
  production
- **JWT key pair wajib berbeda** per environment - token yang di-generate
  di staging tidak boleh valid di production
- **Kredensial pihak ketiga** (SMTP, dsb) staging **wajib** pakai mode
  sandbox/test provider - jangan sampai testing di staging mengirim email
  asli ke user
- Setiap environment baru yang ditambahkan (misal tier Development)
  **wajib** didaftarkan di tabel Section 12 (env source) dan tabel
  konfigurasi di atas - jangan biarkan environment baru berjalan dengan
  konfigurasi "asal jalan"

---

## 18. Referensi Prompt Terkait

- `docs/admin/admin-tambah-kata.md` - UI admin (frontend)
- `01-api-tambah-kata.md` - prompt modul word (sudah diselaraskan)
- `00-api-auth.md` - prompt modul auth (sudah diimplementasikan)
- `02-api-audit-logs.md` - prompt modul audit (auditor: admin & root)
- `03-api-kontribusi-verifikasi.md` - prompt modul contribution (antrean
  review approve/reject/correct + kontribusi media: gambar, pronounce,
  contoh kalimat)
- `04-api-sinonim-inline.md` - sinonim BARU secara inline saat create word
  di POST /admin/words (inherit makna induk by default + override per makna)
- `docs/dbdiagram.dbml` - skema database lengkap
- `ERROR_CODES.md` - katalog error code (buat terpisah, lihat Section 13)
- repo `http/` - koleksi Bruno untuk uji fungsional semua endpoint (Section 20)

---

## 19. Strategi ID - ULID

Prinsip: **semua primary key dan foreign key memakai ULID** (Universally
Unique Lexicographically Sortable Identifier), bukan auto-increment
integer. ULID di-generate di aplikasi (bukan di database), sehingga
tidak bergantung pada sequence database.

### Mengapa ULID, Bukan Auto-Increment

| Aspek        | Auto-Increment (`serial`/`bigint`)                     | ULID (`varchar(26)`)                                                   |
| ------------ | ---------------------------------------------------------- | ------------------------------------------------------------------------ |
| Distribusi   | Bergantung pada satu DB sequence - sulit shard/partisi    | Generate di mana saja, tidak butuh sequence                              |
| Sortability  | Urut tapi tidak terkait waktu                              | Lexicographically sortable berdasarkan waktu (karena timestamp embedded) |
| Keamanan     | ID bisa di-detect (1, 2, 3...) - rawan enumeration        | 26 char random - tidak bisa di-guess                                    |
| Portabilitas | Banyak DB punya sequence, tapi mekanisme beda-beda         | Tipe data`varchar` - universal di semua DB                            |
| Multi-client | Butuh locking/transaction untuk generate ID sebelum insert | Bisa di-generate sebelum insert, tanpa conflict                          |

### Library: `ulid`

```bash
pnpm add ulid
```

```ts
// shared/utils/ulid.ts
import { ulid } from 'ulid';

export function generateId(): string {
  return ulid();
}
```

ULID format: `01HXYZABCDEF1234567890` - 26 karakter, kombinasi
48-bit timestamp (milisecond) + 80-bit entropy. Lexicographically
sortable: ID yang di-generate lebih awal selalu lexicographically
lebih kecil dari yang di-generate belakangan.

### Dampak ke Skema Database

Semua tabel memakai `varchar(26)` sebagai primary key, bukan
`serial`/`bigint`:

```sql
-- Sebelum (auto-increment)
CREATE TABLE words (
  id SERIAL PRIMARY KEY,
  language_id INT NOT NULL REFERENCES languages(id),
  ...
);

-- Sesudah (ULID)
CREATE TABLE words (
  id VARCHAR(26) PRIMARY KEY,
  language_id VARCHAR(26) NOT NULL REFERENCES languages(id),
  ...
);
```

### Dampak ke Drizzle Schema

```ts
// shared/database/drizzle/schema/words.schema.ts
import { varchar, text, timestamp } from 'drizzle-orm/pg-core';
import { generateId } from '@/shared/utils/ulid';

export const words = pgTable('words', {
  id: varchar('id', { length: 26 }).primaryKey().$defaultFn(() => generateId()),
  languageId: varchar('language_id', { length: 26 }).notNull().references(() => languages.id),
  lemma: varchar('lemma', { length: 255 }).notNull(),
  notes: text('notes'),
  createdBy: varchar('created_by', { length: 26 }).references(() => users.id),
  updatedBy: varchar('updated_by', { length: 26 }).references(() => users.id),
  createdAt: timestamp('created_at').notNull().defaultNow(),
  updatedAt: timestamp('updated_at'),
  deletedAt: timestamp('deleted_at'),
  deletedBy: varchar('deleted_by', { length: 26 }).references(() => users.id),
});
```

`$defaultFn(() => generateId())` membuat Drizzle otomatis generate
ULID saat insert tanpa perlu diisi manual - use case/repository tidak
perlu memikirkan ID generation.

### Dampak ke DBML (dbdiagram.dbml)

Semua definisi di `docs/dbdiagram.dbml` diubah:

```dbml
// Sebelum
Table words {
  id int [pk, increment]
  language_id int [not null, ref: > languages.id]
  ...
}

// Sesudah
Table words {
  id varchar(26) [pk]
  language_id varchar(26) [not null, ref: > languages.id]
  ...
}
```

Aturan konversi DBML:

- `id int [pk, increment]` → `id varchar(26) [pk]`
- `id bigint [pk, increment]` → `id varchar(26) [pk]`
- Semua `int [ref: > ...]` → `varchar(26) [ref: > ...]`
- Hapus `increment` dari PK definition
- `word_categories` (junction table, composite PK) tetap pakai
  composite PK tanpa `increment`:
  ```dbml
  Table word_categories {
    word_id varchar(26) [not null, ref: > words.id]
    category_id varchar(26) [not null, ref: > categories.id]
    Indexes {
      (word_id, category_id) [pk]
    }
  }
  ```

### Dampak ke Response API

ID yang dikembalikan di response selalu dalam format string:

```json
{
  "success": true,
  "data": {
    "id": "01HXYZABCDEF1234567890",
    "lemma": "makatn",
    "language_id": "01HXYZABCDEF1234567891"
  }
}
```

**Tidak ada** numerik ID di response - konsisten dengan tipe
`varchar(26)` di database.

### Dampak ke Validation (Zod)

```ts
// Validator: ID harus string 26 karakter
const idSchema = z.string().length(26);

// Endpoint params
const paramsSchema = z.object({
  id: idSchema, // /api/v1/words/:id
  wordId: idSchema, // nested: /api/v1/words/:wordId/meanings
  meaningId: idSchema, // nested deeper
});
```

### Aturan Wajib

- **Semua tabel baru wajib pakai `varchar(26)` ULID sebagai PK** -
  tidak ada pengecualian untuk tabel baru
- **Migrasi dari auto-increment ke ULID** untuk tabel yang sudah ada
  dilakukan bertahap: tambah kolom `ulid` → migrate data → swap PK →
  drop kolom lama (dalam PR terpisah)
- **Jangan pernah expose integer ID** di response API - selalu pakai
  ULID string
- **`$defaultFn(() => generateId())`** di Drizzle schema adalah satu-
  satunya tempat ID di-generate - repository/use case tidak perlu
  generate ID manual
- **Tidak ada sequence database** yang dipakai untuk ID - jika ada
  tabel yang masih pakai `serial`/`bigint`, itu adalah teknikal debt
  yang harus segera di-migrate

---

## 20. Koleksi HTTP Request (Bruno)

Prinsip: **setiap endpoint harus bisa diuji fungsionalnya dari koleksi
HTTP yang tersimpan di git** - bukan cuma dari kode. Koleksi Bruno hidup
di repo `http/` (submodule `sambasku-http`), berdampingan dengan
`api/`, dan **wajib di-sync setiap kali endpoint berubah**.

### Mengapa Bruno, bukan Postman/Insomnia

- Koleksi berupa **plain file `.bru` per request** - diff-friendly,
  reviewable di PR, tanpa export/import JSON raksasa
- Bisa dijalankan **CLI** (`bruno run`) untuk CI maupun **GUI** desktop
  untuk eksplorasi manual
- Tidak butuh akun/cloud - semua di git

### Struktur Koleksi

```text
http/
├── bruno.json              # manifest collection
├── README.md               # cara pakai + aturan sync
├── environments/
│   └── local.bru           # baseUrl + kredensial DEV (non-secret saja)
└── auth/                   # satu folder per modul, file per endpoint
    ├── register.bru
    ├── login.bru
    ├── refresh.bru
    ├── logout.bru
    ├── logout-all-devices.bru
    ├── forgot-password.bru
    └── reset-password.bru
```

Modul baru (word, category, dst) membuat folder sendiri:
`http/word/create-word.bru`, `http/language/list-languages.bru`, dst.

### Konvensi File `.bru`

- `meta.seq` = urutan menjalankan request dalam satu folder (alur logis:
  register → login → token-dependent)
- Request berantai memakai collection variable: `Login` menyimpan
  `access_token` (via `vars:post_response`) dan `refresh_token`
  (via `script:post_response` membaca `Set-Cookie`) - request berikutnya
  tinggal `{{access_token}}` / header `Cookie: refresh_token=…`
- **Setiap request wajib punya blok `tests`** minimal: status code +
  bentuk envelope (`res.body.success`, field kunci di `data`)
- Path + body HARUS sama persis dengan definisi `createRoute()` di kode -
  inilah "kontrak" fungsionalnya

### Aturan Sinkronisasi (WAJIB, satu PR yang sama dengan perubahan API)

| Perubahan di`api/`                       | Yang harus dilakukan di`http/`              |
| ------------------------------------------ | --------------------------------------------- |
| Endpoint baru                              | File`.bru` baru di folder modul + `tests` |
| Field request/response berubah             | Update`body:json` + assertion `tests`     |
| Endpoint dihapus / path berubah            | Hapus / rename file`.bru`                   |
| Error code baru yang memengaruhi assertion | Update`tests`                               |

Environment selain `local` TIDAK di-commit - staging/production
dikelola lokal lewat fitur environment Bruno (kredensial sungguhan
tidak pernah masuk git, konsisten Section 12).

### Menjalankan

```bash
# GUI: buka folder http/ sebagai collection (File → Open Collection)

# CLI (mis. smoke test setelah deploy):
cd http && bruno run --env local auth/
```

---

## 21. Audit Trail - Setiap Mutasi Data Tercatat

Prinsip: **setiap operasi yang mengubah data domain (create / update /
delete) WAJIB menghasilkan satu baris `audit_logs`** - siapa melakukan,
kapan, apa yang berubah, dari request mana. Auditor internal membacanya
lewat modul audit yang hanya bisa diakses role **admin** dan **root**.

### Tabel `audit_logs`

| Kolom           | Isi                                                                                           |
| --------------- | --------------------------------------------------------------------------------------------- |
| `id`          | ULID (Section 19)                                                                             |
| `user_id`     | ULID pelaku (FK users, nullable untuk aksi sistem)                                            |
| `action`      | `'create'` \| `'update'` \| `'delete'` \| `'password_change'` \| `'publish'` \| `'verify'`/`'unverify'` \| `'approve'` \| `'reject'` \| `'correct'` \| dst |
| `entity_type` | `'user'` \| `'word'` \| `'category'` \| dst                                             |
| `entity_id`   | ULID entitas yang diubah                                                                      |
| `old_data`    | snapshot sebelum perubahan (JSON,`null` untuk create)                                       |
| `new_data`    | snapshot sesudah perubahan (JSON)                                                             |
| `request_id`  | dari`requestIdMiddleware` (Section 14) - menyambung log aplikasi ↔ audit DB               |
| `created_at`  | timestamp                                                                                     |

### Konvensi Penulisan (WAJIB)

- **Yang menulis: use case**, lewat `AuditLogRepository` - interface
  diekspor `modules/audit/domain/repositories/audit-log.repository.ts`
  dan di-inject ke use case modul lain (pola komunikasi antar modul
  Section 4: lewat interface yang di-export, bukan import internal).
- **`request_id` wajib diisi** - controller mengambilnya dari context
  (`c.get('requestId')`) dan meneruskannya ke use case.
- **`old_data`/`new_data` TIDAK BOLEH berisi**: password (plain/hash),
  token, atau kredensial apa pun - cukup field aman (mis. reset password
  → `new_data: { changed: true }`).
- **`record()` bersifat best-effort**: implementasi tidak boleh
  melempar error ke caller - kegagalan insert hanya di-log `error`
  (request user tidak ikut gagal). *Upgrade path*: kalau compliance
  mewajibkan atomic, pindahkan insert ke transaksi yang sama dengan
  mutasi utamanya.
- `created_by`/`updated_by` di tiap tabel (lihat dbdiagram) tetap
  dipertahankan - itu jejak cepat per baris; `audit_logs` adalah jejak
  lengkap peristiwa (termasuk `old_data`) + penyambung ke log aplikasi
  via `request_id`.

### Sisi Pembaca (Auditor)

Endpoint `GET /api/v1/admin/audit-logs` (modul `audit`) - hanya role
`admin` dan `root` (`authenticate` + `authorizeRole('admin','root')`),
lengkap dengan filter + pagination. Detail kontraknya ada di
`02-api-audit-logs.md`.

---

## 22. Model Publikasi & Verifikasi Konten (Approval Gate)

Prinsip: **setiap kontribusi WAJIB melewati verifikasi admin sebelum
tayang**. Publikasi adalah gerbang (`status`), kepercayaan adalah jejak
(`is_verified` / `is_corrected`) - verifikator bisa menyetujui, menolak
dengan alasan, atau **mengoreksi langsung** isi kontribusi saat review.

> Model ini **menggantikan** keputusan awal "publish by default" (kontribusi
> langsung tayang). Pembalikan ini sadar dan disengaja: UI admin sejak awal
> memang mengharapkan antrean review, dan kualitas isi kamus diutamakan
> daripada kecepatan tayang. Berlaku untuk SEMUA konten yang bisa
> dikontribusikan user - kata, gambar, pronounce, contoh kalimat (sample),
> dan jenis konten lain yang menyusul.

### Kontrak

| Kolom (tabel konten: `words`, `pronunciations`, `word_images`, `examples`) | Makna |
| --- | --- |
| `status` | `'draft' \| 'pending_review' \| 'published' \| 'rejected'` - draft = masih dikerjakan penulis; **pending_review = menunggu keputusan verifikator, TIDAK tayang**; published = tayang publik; rejected = ditolak (terminal - kirim ulang sebagai kontribusi baru). `pending_review`/`rejected` hanya di-set sistem; request user tetap `draft` \| `published` |
| `is_verified` | `false` = belum diverifikasi; `true` = sudah diverifikasi tim verifikator |
| `verified_by` / `verified_at` | siapa & kapan verifikasi dilakukan (hanya di `words`; identitas reviewer konten anak ada di `contribution_reviews`) |
| `is_corrected` | `true` = isi konten pernah **dikoreksi verifikator** saat review; snapshot sebelum koreksi tersimpan di audit `old_data` (action `correct`) |

### Role Matrix

| Role | Submit (kata & media) | Review antrean (approve/reject/correct) | verify/unverify pasca-publikasi |
| --- | --- | --- | --- |
| root / admin / reviewer | langsung `published` + `is_verified: true` (self-verified) | ✅ | ✅ |
| editor | langsung `published` + `is_verified: true` (self-verified) | ❌ | ❌ |
| contributor | **`pending_review` → masuk antrean** (tidak tayang) | ❌ | ❌ |

Hanya role `contributor` yang masuk antrean - reviewer sudah berwenang
menyetujui kontribusi siapa pun, mengantrekan karyanya sendiri tidak
menambah integritas.

### Alur Wajib

- Contributor submit non-draft → entity `pending_review` + baris
  `contributions` dengan `status: 'pending'` → muncul di antrean review.
  Endpoint publik **tidak menampilkannya** (list/search kata filter
  `status = 'published'`; anak kata - pronounce/gambar/contoh - juga
  hanya tampil bila `status = 'published'`)
- Submit oleh admin/editor/root/reviewer → langsung `published` +
  `is_verified: true` (mereka bagian dari tim verifikator - self-verified)
- `draft` tetap draft untuk semua role (tidak tayang, tidak masuk antrean)
- Keputusan verifikator (role **admin, root, reviewer**) lewat antrean
  `GET /api/v1/admin/contributions`:
  - **approve** → entity `published` + `is_verified: true`
  - **reject** → entity `rejected` - `comment` (alasan) WAJIB
  - **correct** → verifikator mengirim isi yang sudah dikoreksi →
    entity diperbarui, `is_corrected: true`, lalu published + verified
  - Setiap keputusan menulis baris `contribution_reviews`
    (`reviewer_id`, `status approved/rejected/corrected`, `comment`) dan
    memindahkan `contributions.status`
- `POST /api/v1/admin/words/:id/verify` dan `/unverify` **tetap
  tersedia** untuk memberi/mencabut kepercayaan PASCA-publikasi
  (kontrak di `01-api-tambah-kata.md`)
- Kontribusi konten anak pada kata yang sudah tayang (gambar, pronounce,
  contoh kalimat) punya endpoint sendiri dan mengikuti alur yang sama -
  kontrak lengkap di `03-api-kontribusi-verifikasi.md`
- Audit trail (Section 21) mencatat aksi `approve`/`reject`/`correct`
  (correct membawa `old_data` snapshot pra-koreksi) dan `is_verified` /
  `is_corrected` di `new_data`

*Rationale: dua dimensi tetap terpisah - `status` menjawab "boleh tayang?",
`is_verified` menjawab "dipercaya?", `is_corrected` menjawab "pernah diubah
verifikator?". Gerbang publikasi di tangan verifikator; penolakan selalu
bersalah; koreksi tidak menghapus jejak.*


---

## 23. Auth Eksternal (Google OAuth) - Kontrak Desain (menyusul diimplementasi)

Prinsip: client app (web admin / mobile / web user) yang menjalankan alur
"Sign in with Google" via SDK Google; backend hanya MENERIMA Google ID
token, MEMVERIFIKASI, lalu menerbitkan JWT milik kita sendiri (RS256,
pola token Section 00-api-auth.md). Backend TIDAK menjadi target redirect
OAuth - pola ini paling cocok untuk multi-client dan tetap sederhana.

### Alur

```text
1. Client   : Sign in with Google (SDK/browser) -> ID token (JWT RS256 Google)
2. Client   : POST /api/v1/auth/google  { "id_token": "..." }
3. Backend  : verifikasi signature via JWKS Google (jose, Web Crypto),
              cek iss, aud == GOOGLE_CLIENT_ID, exp, email_verified
4. Backend  : cocokkan/link/daftar user (aturan di bawah)
5. Backend  : terbitkan access + refresh token - response SAMA PERSIS
              dengan login biasa (client tidak perlu tahu bedanya)
```

### Perubahan skema database (migration 0008 kelak)

Tabel BARU `auth_identities` (satunya tempat identitas eksternal):

| Kolom | Isi |
| --- | --- |
| `id` | ULID (Section 19) |
| `user_id` | FK users |
| `provider` | `'google'` (terbuka: `'github'`, `'apple'`, dst) |
| `provider_user_id` | Google `sub` - identitas STABIL (email Google bisa berubah) |
| `email_at_provider` | snapshot email saat link (nullable) |
| soft delete standar | Section 7 |

UNIQUE `(provider, provider_user_id)` + index `user_id`.

`users.password_hash` menjadi **NULLABLE** - user OAuth-only tidak punya
password; login password hanya berlaku untuk akun yang lahir dari
register. Login wajib menolak `password_hash IS NULL`
(INVALID_CREDENTIALS), dan compare() mengembalikan false untuk hash kosong.

### Aturan link akun (urutan pemeriksaan)

1. `auth_identities (google, sub)` ada -> login user terkait. Selesai.
2. Tidak ada, tapi `users.email` cocok (email Google sudah `email_verified`)
   -> LINK: buat baris auth_identities menempel ke akun lama (kepemilikan
   email terverifikasi = bukti cukup).
3. Tidak ada sama sekali -> buat user baru: username unik (turunan
   email/nama + sufiks angka bila bentrok), role `'contributor'`
   (default, sama seperti register), `password_hash` NULL.

Role admin/editor TIDAK PERNAH diturunkan dari provider - tetap penugasan
manual (DB/panel admin). Google hanya membuktikan identitas, bukan wewenang.

### Endpoint & konfigurasi

- `POST /api/v1/auth/google` - publik; rate limit 5/15 menit per IP
  (setara login, Section 15)
- Body: `{ "id_token": string }`; response = envelope login standar
- Error code BARU saat implementasi: `INVALID_GOOGLE_TOKEN` (401) -
  daftarkan di ERROR_CODES.md di PR yang sama
- Env BARU: `GOOGLE_CLIENT_ID` (wajib saat fitur aktif; secret di
  Workers, var di dev)
- Port: `GoogleTokenVerifierPort` di `auth/application/ports/` - impl
  memakai `jose` `createRemoteJWKSet` + `jwtVerify` (Web Crypto: jalan
  IDENTIK di Node dan Workers; JWKS di-fetch dan di-cache oleh jose)

### Catatan keamanan (wajib)

- SELALU cek `aud` (client ID kita) dan `iss` - ID token tanpa keduanya
  bukan bukti apa pun (token milik app lain bisa "dipakai ulang")
- Identitas stabil = `sub`, bukan email
- `password_hash NULL` tidak boleh punya jalur login password
- Refresh token rotasi + httpOnly cookie: reuse mekanisme login biasa

### Urutan implementasi (satu PR per langkah bila besar)

1. Migration 0008: tabel `auth_identities` + `password_hash` nullable
   (+ update dbdiagram.dbml dan Section 7 flow)
2. Port + impl verifier + env GOOGLE_CLIENT_ID
3. `LoginWithGoogleUseCase` + controller + route + validator (Zod)
4. Test: unit (mock port: link ketiga cabang aturan), e2e (happy path
   + token invalid 401)
5. Bruno `auth/login-google.bru` + sample `docs/json/auth/` - satu PR
   (Section 20: tiga sumber sinkron)
