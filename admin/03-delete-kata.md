# Admin UI - Hapus Kata (Soft-delete)

Tombol "Hapus" di daftar kata - menghapus (soft-delete) satu entri
kosakata. Mengonsumsi API yang dispesifikasikan di
`docs/api/07-api-delete-kata.md`, dengan konvensi backend di
`docs/api/api-base-stack.md` dan konvensi frontend di
`docs/admin/admin-base-stack.md` (clean architecture 3 lapis, cursor
list, antd v6). Koleksi uji fungsional endpoint-nya ada di repo `http/`
(`http/word/delete-word.bru`).

Status fondasi saat dokumen ini dibuat (JANGAN diduplikasi):

- Backend soft-delete **SUDAH LENGKAP**: `DELETE /api/v1/admin/words/:id`
  + use case `SoftDeleteWordUseCase` + audit `delete` (old_data) + tests
  unit/e2e - lihat `docs/api/07-api-delete-kata.md`.
- Frontend admin: kata dihapus tidak bisa diedit lagi (404 dari backend),
  tapi tombol "Hapus" di `words-page.tsx` **BELUM ADA** - fitur ini
  menambahkannya di kolom aksi daftar kata, sejajar tombol "Ubah".

---

## Prompt

```text
Buatkan tombol "Hapus" untuk fitur soft-delete kata di daftar kata admin
(halaman /words). Ini aksi destruktif satu klik: HANYA verifikator
(admin/editor/root/reviewer) yang boleh - contributor disembunyikan
(dan backend tetap 403 kalau dipaksa).

KONTEKS:
Soft-delete = kata hilang dari kamus publik DAN daftar admin, tapi
baris data dipertahankan backend untuk audit/recovery. Tidak ada
jaringan jalan "pulihkan" di UI iterasi ini - tombol harus
mengomunikasikan itu.

STRUKTUR (pola 3 lapis admin-base-stack Section 5):

1. infrastructure/word-api.ts
   TAMBAH deleteWordRequest(id): Promise<void> → client.delete
   `/admin/words/:id` (envelope { success, data: null } diabaikan).
   Rate limit 30/menit per user di backend - tidak ada penanganan khusus.

2. application/use-delete-word.ts  (BARU)
   useMutation + useQueryClient:
   onSuccess → removeQueries(['words','detail',id]) + invalidateQueries
   (['words']) - POLA PERSIS use-update-word.ts (hapus cache detail agar
   halaman edit tidak menyajikan kata yang sudah dihapus; list/search
   di-refresh).

3. presentation/words-page.tsx
   - Kolom aksi, SEJALAR tombol "Ubah" (hidden contributor):
     tombol "Hapus" type="link" danger, dibungkus Popconfirm antd:
       title = `Hapus kata "<lemma>"?`
       description = "Kata akan hilang dari kamus publik & daftar admin.
       Soft-delete: data tetap disimpan untuk audit/recovery."
       okText = "Hapus", okButtonProps danger, cancelText = "Batal"
   - onConfirm → mutateAsync(id):
       onSuccess → message.success(`Kata "<lemma>" dihapus`)
       onError → message.error(err.message)  (404 untuk kata yang sudah
         dihapus orang lain - pesan backend "Kata dengan id tersebut
         tidak ditemukan" sudah cukup jelas)
   - loading per-baris: state deletingId (ulid row yang sedang diproses)
     → tombol yang bersangkutan loading + tombol Hapus lain disabled
     (cegah delete ganda bersamaan).

TIDAK ADA perubahan router/rute. TIDAK ADA unit test baru (tidak ada
logika murni yang bisa di-test di luar hook - konsisten dengan
use-update-word yang juga tanpa test).

Command verifikasi: pnpm typecheck && pnpm lint && pnpm test && pnpm build
```

---

## Catatan Implementasi

- `useDeleteWord` TIDAK ditaruh di `useUpdateWord` yang sama - satu use
  case satu file (pola `use-create-word.ts` / `use-update-word.ts`).
- Per-baris loading memakai `deletingId` state, bukan `mutation.isPending`
  global, supaya tombol Hapus di baris lain tidak ikut loading.
- Popconfirm adalah konfirmasi, bukan window.confirm - konsisten dengan
  Popconfirm di create-word-page/edit-word-page.
- Halaman edit (/words/:id/edit) TIDAK diberi tombol Hapus: kata yang
  sedang diedit mungkin tidak tampil di daftar (deep link), dan pola
  manajemen = hapus dari daftar. Kalau kelak mau ditambahkan, cukup
  pindah tombol + Popconfirm yang sama.

## Referensi Terkait

- `docs/api/07-api-delete-kata.md` - kontrak DELETE /api/v1/admin/words/:id
- `docs/admin/02-edit-kata.md` - halaman edit; tombol "Ubah" yang duduk
  bersebelahan dengan "Hapus" di kolom aksi
- `docs/admin/admin-base-stack.md` - stack, clean architecture 3 lapis,
  kolom aksi fixed right (Section 13), testing (Section 15)
- Repo `http/` - `http/word/delete-word.bru`
- `src/features/words/` - words-page.tsx (kolom aksi), use-update-word.ts
  (pola mutation + removeQueries), word-api.ts (request layer)