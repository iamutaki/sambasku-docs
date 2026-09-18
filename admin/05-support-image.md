# Admin UI - Gambar pada Tambah & Edit Kata (Direct Upload ImageKit)

Mengikuti `admin-base-stack.md`: Section 2-3 (clean architecture feature-based),
6 (router/guard), 7-8 (session + axios), 10 (React Query), 13 (error handling),
15 (responsive + action bar). Kontrak API yang dikonsumsi:
`01-api-tambah-kata.md` (images[] di POST /admin/words), `05-api-edit-kata.md`
(images[] di PUT full-replace), endpoint gambar modul image
(`GET /admin/images/upload-token`) + word media (`POST /words/:wordId/images`
- TIDAK dipakai fitur ini, satu POST create sudah membawa images[]).

Status fondasi saat dokumen ini dibuat (JANGAN diduplikasi):
- SUDAH ADA (API): images[] di create/update body (maks 10, maks 1 is_primary,
  superRefine 400), upload-token (signature HMAC-SHA1, TTL 30 menit, 503
  IMAGE_UPLOAD_UNAVAILABLE saat IMAGEKIT_* belum di-set), provider_file_id kini
  ikut di response detail (round-trip PUT).
- SUDAH ADA (admin): form create 6 section + halaman detail render images
  (thumbnail + tag Utama); form edit reuses blok create; infra axios `client`
  + envelope + ApiError.
- Yang BELUM: semua UI upload (tidak ada antd Upload/FormData di mana pun di
  admin), tipe images di form values, prefill images di edit, dan round-trip
  PUT (TANPA ini: sekali edit kata bergambar = semua gambar terhapus senyap
  karena PUT full-replace hard-delete word_images).

---

## Prompt

```text
Buatkan section upload gambar untuk form tambah & edit kata di admin,
mengikuti pola blok form yang ada (word-form-blocks.tsx).

KONTEKS:
- Backend TIDAK pernah melewati byte gambar. Alur per file:
  1. GET /admin/images/upload-token (lewat infra `client` + envelope)
     → { token, signature, expire, public_key, upload_endpoint }
  2. POST upload_endpoint (https://upload.imagekit.io/api/v1/files/upload)
     FormData: file, fileName, folder=/words, publicKey=public_key, token,
     expire, signature → ImageKit balas { url, fileId }
     (fetch POLOS - host berbeda, tanpa envelope, TANPA header Authorization)
  3. Hasil { url, provider_file_id: fileId } masuk array images form;
     submit create/update membawa images[] sebagaimana adanya.
- Upload TERJADI SAAT FILE DIPILIH (bukan saat submit). File yang ter-upload
  lalu form dibatalkan/gagal submit = orphan di CDN - DITERIMA (tidak ada
  delete API yang ter-wire; bersih manual dari dashboard ImageKit).
- 503 IMAGE_UPLOAD_UNAVAILABLE dari token endpoint = penyimpanan belum
  dikonfigurasi → section disabled + Alert penjelasan, form tetap bisa
  disubmit tanpa gambar.

STRUKTUR:
1. Section baru "7. Gambar (opsional)" - Collapse aktif default di create
   & edit (simetri dengan halaman detail "7. Gambar"):
   - antd Upload listType="picture-card", multiple, maks 10 file
     (upload-token rate limit 30/menit per IP - jangan bombardir)
   - beforeUpload: tolak non jpg/png/webp + >5MB (message.warning,
     return Upload.LIST_IGNORE)
   - customRequest → hook upload (token → ImageKit → {url, fileId})
   - tiap gambar selesai: Input alt_text (maks 500, opsional) + Switch
     "Utama" (is_primary) - EKSKLUSIF client-side: set satu true = clear
     lainnya (cermin superRefine API 'Hanya satu gambar yang boleh
     is_primary' → tidak pernah kena 400)
   - hapus dari daftar (sebelum submit) hanya menghapus dari form state;
     file CDN dibiarkan (orphan diterima)
2. Submit: buildCreateWordBody/buildUpdateBody memetakan images[] →
   { url, provider_file_id, alt_text, is_primary } (HANYA yang selesai
   upload; strip field client-only uid/status)
3. Edit page: wordDetailToFormValues memetakan detail.images (kini membawa
   provider_file_id) → nilai awal form - gambar existing TAMPIL sebagai
   item daftar (thumbnail + alt_text + Utama) dan ikut terkirim ulang di
   PUT (ini yang mencegah penghapusan senyap)
4. ROLE GATE: semua role yang bisa buka form (admin/editor/root/reviewer
   via edit; contributor via create) boleh upload - sama dengan API.
   Kontributor: images[] ikut gerbang kata (children pending_review
   bersama kata) - warning kontributor sudah ada di form, TIDAK perlu
   tambahan.
TIDAK ADA perubahan API di task ini (semua endpoint sudah ada).

FILE:
- src/features/words/domain/create-word.ts               # UBAH: +WordImageInput, images? di form values & request types
- src/features/words/domain/word-detail.ts               # UBAH: +provider_file_id di WordDetail.images
- src/features/words/infrastructure/image-api.ts         # BARU: getUploadToken() (via client) + uploadToProvider() (fetch polos)
- src/features/words/application/use-upload-word-image.ts # BARU: hook per-file (loading/error, 503 → flag off)
- src/features/words/presentation/word-images-field.tsx  # BARU: blok section 7 (Upload + alt_text + Utama + Alert 503)
- src/features/words/presentation/create-word-page.tsx   # UBAH: pasang section 7
- src/features/words/presentation/edit-word-page.tsx     # UBAH: pasang section 7
- src/features/words/application/create-word-utils.ts    # UBAH: body builder += images[]
- src/features/words/application/word-detail-mappers.ts  # UBAH: prefill images
- src/features/words/__tests__/create-word-utils.test.ts # UBAH: mapping images + hanya yang selesai upload

Command verifikasi: pnpm typecheck && pnpm lint && pnpm test && pnpm build
```

---

## Catatan Implementasi

- Upload component TIDAK memakai action/axios `client` - upload_endpoint
  milik ImageKit (CORS terbuka untuk upload bertanda tangan), response-nya
  bukan envelope sambasku.
- Token diambil SEKALI PER FILE saat upload dimulai (bukan di-cache) -
  sederhana, dan 30 menit TTL jauh di atas durasi hidup form; kalau kelak
  10-file burst jadi masalah (429 token), cache token sampai < expire.
- Eksklusivitas is_primary client-side WAJIB (API menolak >1 dengan 400
  superRefine); pada prefill edit, satu-satunya primary existing tetap
  terkirim apa adanya.
- fetch ImageKit gagal (network/non-200) → tandai item error di daftar +
  tombol retry; item error TIDAK ikut images[] saat submit.
- Halaman TIDAK memanggil client langsung - getUploadToken tetap lewat
  infrastructure request fn (aturan base-stack), hanya POST ke CDN yang
  fetch polos.

## Referensi Terkait

- `01-api-tambah-kata.md` - images[] di create + alur direct-upload (baris
  ~298-305)
- `05-api-edit-kata.md` - PUT full-replace (semantik yang memaksa round-trip
  images)
- `03-api-kontribusi-verifikasi.md` - status children per role
  (pending_review vs published)
- `docs/json/image/upload-token.200.json` + `word/add-word-image.201.json`
- `http/image/get-upload-token.bru` - uji token manual (503 saat belum
  dikonfigurasi)
- `admin-base-stack.md` - Section 2-3, 8, 13, 15
- `04-detail-kata.md` - section "7. Gambar" di halaman detail (tujuan render)
