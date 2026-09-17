# Base Stack Mobile - Kamus Digital Sambas-Indonesia (Flutter)

Dokumen ini jadi acuan tetap untuk semua prompt/fitur mobile selanjutnya,
mengikuti pola yang sudah terbukti di proyek `jnn_mobile` (repo pribadi),
disesuaikan dengan kontrak backend sambasku (`docs/api/api-base-stack.md`).
Setiap prompt fitur baru (auth, pencarian, kontribusi, dst) mengikuti
struktur dan konvensi di sini tanpa dijelaskan ulang.

---

## 1. Tech Stack

| Layer | Pilihan | Alasan |
| --- | --- | --- |
| Framework | Flutter (stable channel, Dart SDK ^3.x) | Satu kode, Android + iOS |
| Arsitektur | Clean Architecture, feature-first (3 lapis per fitur) | Sama filosofi dengan backend (base-stack Section 2); skala fitur tanpa campur aduk |
| State management | flutter_riverpod + riverpod_annotation (codegen) + flutter_hooks + hooks_riverpod | Provider terurut per lapisan; hooks memangkas boilerplate widget |
| Functional error | fpdart (`Either<Failure, T>`) | Repository/usecase mengembalikan hasil eksplisit, tanpa exception liar |
| Networking | dio + retrofit (codegen) | Datasource deklaratif via anotasi; interceptor untuk auth |
| Model / DTO | freezed + json_serializable | Immutable + fromJson/toJson tergenerate |
| Codegen | build_runner | Satu perintah untuk retrofit/freezed/riverpod/envied |
| Routing | go_router | Declarative, redirect auth, deep-link ready |
| UI kit | forui (+ forui_hooks) + gap | Konsisten dengan jnn_mobile; komponen siap pakai |
| Loading placeholder | skeletonizer | Skeleton saat fetch |
| Local storage | shared_preferences (via `AuthTokenStorage`) | Token sesi; sensitif cukup untuk access/refresh + flag isAuth |
| Env compile-time | envied (`.env` + flavor) | Secret tidak masuk repo |
| Flavor | flutter_flavorizr (staging / production) | Dua app berdampingan di satu HP |
| Ikon app | flutter_launcher_icons (per flavor) | Icon + nama dibedakan per flavor |
| Testing | flutter_test (+ widget test per halaman kunci) | Minimal: usecase unit + widget test form |
| Lint | flutter_lints + analysis_options.yaml | Konsistensi antar kontributor |

Firebase/FCM **tidak dipakai fase awal** (kamus belum butuh push) - dicatat
sebagai upgrade path, jangan dipasang spekulatif.

---

## 2. Prinsip Arsitektur (3 lapis per fitur)

Arah dependency selalu ke dalam, sama seperti backend:

```text
presentation (pages/notifier/state)
   → domain (entities/failures/repositories interface/usecases/providers)
       → data (datasources retrofit/models DTO/repositories impl/providers)

core/  = lintas fitur (network, router, constants, widgets umum)
shared/ = halaman/util lintas fitur (splash, dev tool)
```

- **domain**: murni Dart. Entity, `Failure` per fitur, interface repository,
  usecase (`call(params)`), plus provider tier domain.
- **data**: retrofit datasource (anotasi `@RestApi`), DTO freezed,
  repository impl yang memetakan `ApiResponse`/DioException ke `Either`,
  plus provider tier data.
- **presentation**: halaman (hook widget), state class `copyWith`,
  `Notifier` riverpod, plus provider tier presentation.

Komunikasi antar fitur lewat provider yang di-export, bukan import internal
ke file privat fitur lain (kejiran dengan aturan Section 4 backend).

---

## 3. Struktur Folder

```text
lib/
├── core/
│   ├── constants/env.dart            # envied: API host, ImageKit key
│   ├── models/api_response.dart      # envelope sambasku (Section 6)
│   ├── network/
│   │   ├── sambasku_api_client.dart  # Dio + interceptor (singleton)
│   │   ├── auth_token_storage.dart   # token + isAuth (singleton)
│   │   ├── imagekit_api_client.dart  # klien upload ImageKit
│   │   ├── interceptors/auth_interceptor.dart
│   │   └── network_providers.dart    # dioProvider dsb.
│   ├── router/
│   │   ├── app_router.dart           # agregasi + redirect auth
│   │   └── route_definer.dart        # path + name per route
│   ├── services/                     # device id, dsb (bila perlu)
│   └── widgets/                      # widget generik lintas fitur
│
├── features/
│   ├── auth/
│   │   ├── auth_router.dart
│   │   ├── domain/
│   │   │   ├── entities/auth_session.dart
│   │   │   ├── failures/auth_failure.dart
│   │   │   ├── repositories/auth_repository.dart     # abstract interface
│   │   │   ├── usecases/login_use_case.dart
│   │   │   └── providers/auth_domain_providers.dart
│   │   ├── data/
│   │   │   ├── datasources/auth_remote_datasource.dart   # retrofit + .g.dart
│   │   │   ├── models/                                   # DTO freezed + .g/.freezed
│   │   │   ├── repositories/auth_repository_impl.dart
│   │   │   └── providers/auth_data_providers.dart
│   │   └── presentation/
│   │       ├── pages/login_page.dart
│   │       ├── models/auth_login_state.dart
│   │       └── providers/auth_login_providers.dart       # Notifier
│   │
│   ├── dictionary/        # pencarian + detail kata + beranda miss
│   ├── contribution/      # submit kata (login & anonim), kontribusi media
│   ├── profile/           # profil + logout
│   └── ... (fitur baru menyusul, pola sama)
│
├── shared/
│   ├── splash/            # cek sesi → arahkan login/beranda
│   ├── pages/ widgets/ utils/
│   └── dev_tool/          # network monitor, storage inspector (debug)
│
├── flavors.dart           # enum Flavor + F (title/nama per flavor)
├── app.dart               # MaterialApp.router + ProviderScope
└── main.dart              # bootstrap: flavor, ApiClient, DeviceId
```

`main.dart` tetap ramping: init flavor, `SambaskuApiClient.instance`,
DeviceIdService, lalu `runApp(ProviderScope(child: App()))`.

---

## 4. Konvensi Penamaan File

| Tipe | Konvensi | Contoh |
| --- | --- | --- |
| Halaman | `*_page.dart` | `login_page.dart` |
| State halaman | `*_state.dart` (folder models presentation) | `auth_login_state.dart` |
| Provider tier | `<fitur>_data_providers.dart` / `_domain_providers.dart` / `_<halaman>_providers.dart` | `auth_data_providers.dart` |
| Entity | `*.dart` nomina domain | `auth_session.dart` |
| Failure | `<fitur>_failure.dart` | `dictionary_failure.dart` |
| Repository interface | `*_repository.dart` | `auth_repository.dart` |
| Repository impl | `*_repository_impl.dart` | `auth_repository_impl.dart` |
| Usecase | `*_use_case.dart` + `class XUseCase` | `login_use_case.dart` |
| Datasource | `*_remote_datasource.dart` (+ `.g.dart`) | `auth_remote_datasource.dart` |
| DTO | `*_dto.dart` (+ `.g.dart` + `.freezed.dart`) | `login_response_dto.dart` |
| Router per fitur | `<fitur>_router.dart` | `dictionary_router.dart` |
| File codegen | `*.g.dart`, `*.freezed.dart` - TIDAK diedit manual | |

Folder fitur: `features/<nama_fitur>/` (snake_case, singular: `auth`,
`dictionary`, `contribution`).

---

## 5. Pola Lengkap Satu Fitur (contoh: auth)

Alur dependency dan bentuk tiap file - semua fitur meniru pola ini.

**1. Entity + Failure (domain):**

```dart
class AuthSession {
  const AuthSession({required this.userId, required this.username, required this.role});
  final String userId; final String username; final String role;
}

class AuthFailure {
  const AuthFailure(this.message);      // + subkelas spesifik bila perlu
  final String message;
}
```

**2. Repository interface (domain) - semua method balikan `Either`:**

```dart
abstract interface class AuthRepository {
  Future<Either<AuthFailure, AuthSession>> login({
    required String email, required String password,
  });
}
```

**3. Usecase - satu aksi, `call(params)`, validasi/trim di sini:**

```dart
class LoginUseCase {
  const LoginUseCase(this._repository);
  final AuthRepository _repository;

  Future<Either<AuthFailure, AuthSession>> call(LoginParams params) =>
      _repository.login(email: params.email.trim(), password: params.password);
}
```

**4. Provider 3 tier (codegen `@riverpod`, `part '*.g.dart'`):**

```dart
// data: auth_data_providers.dart
@riverpod
AuthRemoteDatasource authRemoteDatasource(Ref ref) =>
    AuthRemoteDatasource(ref.watch(dioProvider));

@riverpod
AuthRepository authRepository(Ref ref) =>
    AuthRepositoryImpl(ref.watch(authRemoteDatasourceProvider),
                        ref.watch(authTokenStorageProvider));

// domain: auth_domain_providers.dart
@riverpod
LoginUseCase authLoginUseCase(Ref ref) =>
    LoginUseCase(ref.watch(authRepositoryProvider));
```

**5. State (presentation/models) - copyWith manual dengan `clearX`:**

```dart
class AuthLoginState {
  const AuthLoginState({this.isSubmitting = false, this.errorMessage, this.session});
  final bool isSubmitting; final String? errorMessage; final AuthSession? session;

  AuthLoginState copyWith({bool? isSubmitting, String? errorMessage,
      bool clearErrorMessage = false, AuthSession? session, bool clearSession = false}) =>
    AuthLoginState(
      isSubmitting: isSubmitting ?? this.isSubmitting,
      errorMessage: clearErrorMessage ? null : errorMessage ?? this.errorMessage,
      session: clearSession ? null : session ?? this.session,
    );
}
```

**6. Notifier (presentation/providers):**

```dart
@riverpod
class AuthLoginNotifier extends _$AuthLoginNotifier {
  @override
  AuthLoginState build() => const AuthLoginState();

  Future<void> submit({required String email, required String password}) async {
    state = state.copyWith(isSubmitting: true, clearErrorMessage: true, clearSession: true);
    final result = await ref.read(authLoginUseCaseProvider)(LoginParams(email: email, password: password));
    result.match(
      (failure) => state = state.copyWith(isSubmitting: false, errorMessage: failure.message, clearSession: true),
      (session) => state = state.copyWith(isSubmitting: false, session: session),
    );
  }
}
```

**7. Halaman - hook consumer widget, UI forui, skeletonizer saat load:**

```dart
class LoginPage extends HookConsumerWidget {
  const LoginPage({super.key});
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.watch(authLoginNotifierProvider);
    final email = useTextEditingController();
    // ... FButton (forui) disable saat state.isSubmitting, pesan error
  }
}
```

---

## 6. Networking - Kontrak sambasku

### Envelope standar (wajib dipahami semua fitur)

Backend memakai SATU bentuk response (`docs/api/api-base-stack.md` Section
13). Definisikan sekali di `core/models/api_response.dart`:

```dart
@freezed
class ApiResponse<T> with _$ApiResponse<T> {
  const factory ApiResponse.success({
    required bool success,
    required T? data,
    CursorMeta? meta,               // hanya endpoint list
  }) = ApiSuccess<T>;

  const factory ApiResponse.failure({
    required bool success,
    required String errorCode,      // "error_code" di JSON
    required String message,
    List<ApiErrorDetail>? details,
  }) = ApiFailure<T>;
}

@freezed
class CursorMeta with _$CursorMeta {
  const factory CursorMeta({
    required int limit,
    @JsonKey(name: 'next_cursor') String? nextCursor,
    @JsonKey(name: 'has_more') required bool hasMore,
  }) = _CursorMeta;
}
```

Aturan mapping di repository impl:
- `success == true` → `Either.right(entity)` (DTO → entity, BUKAN DTO bocor
  ke presentation)
- `success == false` → `Either.left(XFailure(message))`; `error_code` boleh
  disimpan untuk penanganan spesifik (tabel Section 11)
- `DioException` → map ke Failure generik (offline / timeout / 500)

### Datasource (retrofit)

```dart
@RestApi()
abstract interface class AuthRemoteDatasource {
  factory AuthRemoteDatasource(Dio dio, {String? baseUrl, ParseErrorLogger? errorLogger}) =
      _AuthRemoteDatasource;

  @POST('/api/v1/auth/login')
  Future<ApiResponse<LoginResponseDto>> login(@Body() LoginRequestDto body);

  @GET('/api/v1/words/search')
  Future<ApiResponse<List<WordSummaryDto>>> search(@Queries() Map<String, dynamic> query);
}
```

### Auth - client_type mobile (BEDA dari web)

Refresh token backend memakai **httpOnly cookie untuk web**; mobile TIDAK
bisa membaca cookie httpOnly, jadi backend menyediakan varian body
(lihat `http/auth/login-mobile.bru`):

- Login: `POST /api/v1/auth/login` + field `client_type: 'mobile'`
  → response berisi `refresh_token` di **body**
- Refresh: `POST /api/v1/auth/refresh` body `{ refresh_token, client_type: 'mobile' }`
  (rotasi: token lama mati, simpan yang baru)

### AuthInterceptor (pola jnn_mobile, disesuaikan)

- `onRequest`: sisipkan `Authorization: Bearer <accessToken>`
- `onError` 401: refresh SEKALI via `_refreshFuture` bersama (queue -
  beberapa request 401 bersamaan menunggu satu refresh), lalu retry;
  gagal refresh → clear token + `setIsAuth(false)` (redirect login oleh
  router)
- Timeouts: connect/receive/send 15 detik

### Pagination - cursor-based (WAJIB)

Semua list backend memakai `?limit=&cursor=` + `meta.next_cursor` +
`meta.has_more` - TIDAK ADA page number. Pola UI: infinite scroll /
tombol "Muat lagi" menyimpan `nextCursor` di state halaman.

---

## 7. Routing

- Tiap fitur punya `<fitur>_router.dart`: konstanta `RouteDefiner(path,
  name)` + `static final List<GoRoute> routes`
- `core/router/app_router.dart` mengagregasi semua, plus `redirect`:
  belum auth & di luar `/login` → login; sudah auth & di `/login` → splash
- Nama route: `'<Fitur>Router.<halaman>'` (mis. `DictionaryRouter.detail`)
- Detail kata butuh parameter id: path `/words/:id`, baca via
  `state.pathParameters['id']`

Route awal sambasku:

| Route | Halaman |
| --- | --- |
| `/login` | Login (email/password; Google menyusul §API 23) |
| `/register` | Register |
| `/` (beranda) | Pencarian + kartu "sedang dicari, belum ada artinya" (search-miss) |
| `/words/:id` | Detail kata (makna, contoh, pelafalan, gambar, relasi) |
| `/contribute` | Form usul kata baru (anonim atau login) |
| `/profile` | Profil, logout |

---

## 8. Flavor & Environment

- `flavors.dart`: `enum Flavor { staging, production }` + class `F`
  (`F.name`, `F.title`, `F.isStaging`)
- flutter_flavorizr membuat dua target: nama app
  "SambasKu (Staging)" / "SambasKu"
- `core/constants/env.dart` dengan envied membaca `.env` (git-ignored,
  ada `.env.example`):

```text
SAMBASKU_API_HOST_STAGING=https://sambasku-staging.iamutaki.com
SAMBASKU_API_HOST_PRODUCTION=      # diisi saat production rilis
IMAGEKIT_PUBLIC_KEY=public_xxx
IMAGEKIT_URL_ENDPOINT=https://ik.imagekit.io/apinull
```

Pemilihan host per flavor di `env.dart` (switch `F.appFlavor`) - TIDAK
ADA host hardcode di datasource.

---

## 9. Upload Gambar (pola direct upload)

Sama seperti web admin (base-stack Section 8 backend + bukti staging):

1. Login → `GET /api/v1/admin/images/upload-token`
   (Bearer) → `{ token, signature, expire, public_key, upload_endpoint }`
2. Upload file LANGSUNG ke `upload_endpoint` (multipart): `file`,
   `token`, `signature`, `expire`, `publicKey`, `fileName`, `folder=/words`
3. Respons → `fileId` + `url`
4. Kirim `url` + `provider_file_id` sebagai `images[]` saat submit kata

**Token upload SEKALI PAKAI** (terbukti di staging: request kedua dengan
token sama ditolak) - satu token per file; expire 30 menit.
`core/network/imagekit_api_client.dart` = Dio terpisah tanpa auth
interceptor backend (host ImageKit, bukan API host).

---

## 10. Strategi Testing

| Level | Target | Alat |
| --- | --- | --- |
| Unit | usecase (mock repository interface) + mapper DTO→entity | flutter_test |
| Widget | form kunci (login, contribute): state submitting/error/sukses | flutter_test + ProviderScope override |
| Integration (opsional) | repository impl vs API staging | dio asli + staging host |

Aturan: tiap usecase baru wajib unit test; tiap halaman form minimal 1
widget test (sukses + gagal). Jalankan `flutter test` di CI.

---

## 11. Mapping error_code → Perlakuan UI

| error_code (backend) | HTTP | Perlakuan mobile |
| --- | --- | --- |
| `VALIDATION_ERROR` | 400 | Tampilkan `details[].field` inline per field form |
| `INVALID_CREDENTIALS` / `UNAUTHORIZED` / `TOKEN_EXPIRED` | 401 | Pesan login gagal; interceptor handle refresh |
| `FORBIDDEN` | 403 | Sembunyikan aksi khusus role; pesan tanpa hak |
| `*_NOT_FOUND` | 404 | Empty state halaman |
| `EMAIL_ALREADY_EXISTS` dll | 409 | Inline di field terkait |
| `RATE_LIMITED` | 429 | Toast "coba lagi nanti" + hitung `Retry-After` |
| `IMAGE_UPLOAD_UNAVAILABLE` | 503 | Sembunyikan fitur upload gambar |
| `INTERNAL_ERROR` | 500 | Pesan generik + retry |

Katalog lengkap: `api/ERROR_CODES.md` (satu sumber kebenaran - jangan
menduplikasi daftar di mobile, cukup perlakuan umum per kelompok).

---

## 12. Urutan Fitur Mobile (roadmap)

1. **Bootstrap**: scaffold proyek + flavor + env + core (network, router,
   token storage) + splash + login/register (`00-api-auth.md`)
2. **Dictionary**: beranda + pencarian 2 arah (`search_in`) + infinite
   scroll cursor + detail kata
3. **Beranda miss**: kartu "sedang dicari" (`GET /search-misses`) + CTA
   kontribusi
4. **Kontribusi anonim**: form usul kata (`POST /contributions/words`)
   + status pending review di UI
5. **Kontribusi login**: media (pelafalan/gambar/contoh) pada kata
   existing + upload gambar ImageKit
6. **Profil**: info user, logout (revoke refresh)
7. Menyusul: Google sign-in (menunggu backend Section 23), notifikasi

---

## 13. Referensi Terkait

- `docs/api/api-base-stack.md` - kontrak backend (envelope Section 13,
  pagination, error, auth mobile varian)
- `docs/api/00-api-auth.md` - endpoint auth (termasuk client_type mobile)
- `docs/api/01-api-tambah-kata.md`, `03-api-kontribusi-verifikasi.md`
- `api/ERROR_CODES.md` - katalog error code
- `docs/json/` - sample response semua endpoint (mock untuk unit/widget
  test - JANGAN hardcode bentuk response di luar file ini)
- repo `http/` - koleksi Bruno (perilaku endpoint hidup, contoh chaining)
- repo referensi pola: `jnn_mobile` (iamutaki) - sumber konvensi asli
  dokumen ini
