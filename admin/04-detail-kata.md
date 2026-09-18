# Admin UI - Detail Kata

Halaman read-only `/words/:id` - ringkasan penuh satu entri kosakata
untuk SEMUA status (draft/pending_review/published/rejected). Mengonsumsi
endpoint admin yang SUDAH lengkap (`GET /api/v1/admin/words/:id`,
`docs/api/05-api-edit-kata.md`), dengan konvensi frontend di
`docs/admin/admin-base-stack.md`. TIDAK ada endpoint baru.

Status fondasi saat dokumen ini dibuat (JANGAN diduplikasi):

- Backend detail admin **SUDAH LENGKAP**: `GET /api/v1/admin/words/:id`
  mengembalikan detail semua status + children (prefill jujur) - lihat
  `docs/api/05-api-edit-kata.md`. Tombol "Detail" di `words-page.tsx`
  masih placeholder `message.info(...)`.
- Halaman edit `/words/:id/edit` SUDAH APA ADANYA - detail adalah
  "pintu baca" di depannya: cek status & isi entri dulu, baru pilih
  "Ubah Kata".

---

## Prompt

```text
Buatkan halaman admin "Detail Kata" (read-only) - /words/:id, dibuka dari
tombol "Detail" di daftar kata. Ringkasan penuh satu entri kosakata untuk
SEMUA status: lihat isi entri, cek status/verifikasi, lalu lompat ke
"Ubah Kata" untuk mengedit.

KONTEKS:
- Satu-satunya data: GET /api/v1/admin/words/:id (role verifikator:
  admin/editor/root/reviewer - contributor 403). Response = WordDetail
  (domain/word-detail.ts) yang SUDAH dipakai prefill edit - TIDAK ada
  request baru, TIDAK ada mapper baru.
- Tampilan read-only MENIRU contribution-entity-view.tsx (pola Section +
  Descriptions + Space yang sudah dipakai layar review) - konsisten, bukan
  bikin pola baru.

STRUKTUR:
1. useParams id → useWordDetail(id) (hook yang sama dengan edit -
   cache ['words','detail',id] terbagi).
2. TERJEMAHKAN id → nama: useLanguageOptions untuk bahasa + bahasa
   terjemahan, useDialectOptions(detail.language_id) untuk dialek anak
   (variants/pronunciations). Respons detail hanya membawa id.
3. Render:
   - PageHeader: lemma + StatusTag; subtitle = jenis entri · bahasa;
     extra = "Kembali ke Daftar" + "Ubah Kata" → navigate
     /words/$id/edit (passing id).
   - DESCRIPTIONS informasi dasar: bahasa, status (tag warna sama dengan
     words-page), tanda verifikasi/koreksi, dibuat, diperbarui.
   - Catatan Tambahan (jika ada)
   - MAKNA/ARTI per meaning: kelas kata (name + code), urutan, definisi,
     terjemahan (• teks · bahasa (jenis terjemahan)), contoh (sumber —
     terjemahan, (sumber contoh))
   - KATEGORI: tag per kategori; kosong → "Tidak ada"
   - RELASI KATA: lemma (jenis relasi) - label dari RELATION_TYPE_LABELS
   - MUNCUL DALAM (hanya kalau ada): sama dengan relasi, arah masuk
   - BENTUK TURUNAN: bentuk (jenis, afiks "nilai", dialek — catatan)
   - PENGUCAPAN: [nilai] /notasi/ (dialek)
   - GAMBAR: thumbnail (Image antd) + tag "Utama" + alt text
4. ROLE GATE sama persis edit kata:
   - tombol "Detail" di words-page DILIHATKAN hapus untuk contributor
     (sejajar Ubah/Hapus - endpoint 403)
   - navigasi manual contributor ke /words/:id → halaman 403 + tombol
     "Kembali ke Daftar" (jangan bakar request prefill)
5. STATE: isPending → Skeleton; isError/404 → Alert + "Kembali ke Daftar".

TIDAK ADA perubahan API. TIDAK ADA unit test (tampilan murni, tanpa
logika yang bisa diuji terpisah).

FILE:
- src/features/words/presentation/word-detail-page.tsx   # BARU
- src/features/words/presentation/words-page.tsx         # UBAH: tombol
                                                         #   Detail → navigate
- src/app/router.tsx                                     # UBAH: route
                                                         #   /words/$id
- src/shared/layouts/console-layout.tsx                 # UBAH: breadcrumb
                                                         #   "Detail Kata"

Command verifikasi: pnpm typecheck && pnpm lint && pnpm test && pnpm build
```

---

## Catatan Implementasi

- `/words/$id` dan `/words/new` TIDAK konflik: TanStack Router memberi
  prioritas segmen statis ('new') di atas segmen dinamis ($id).
- Bahasa lead detail dipakai untuk dialek: dialek terjemahan tidak
  tersedia di data referensi (dialek = per bahasa sumber) - best-effort,
  fallback ke id.
- `useWordDetail` staleTime 30s - membuka detail lalu edit dalam waktu
  dekat memakai cache yang sama (round-trip murah).
- Descriptions `column={{ xs: 1, md: 2 }}` - responsif (Section 17).
- Tanggal pakai dayjs format 'DD MMM YYYY HH:mm' (konsisten audit-logs).

## Referensi Terkait

- `docs/api/05-api-edit-kata.md` - kontrak GET /api/v1/admin/words/:id
- `docs/admin/02-edit-kata.md` - halaman edit; tombol "Ubah" yang dicapai
  dari ujung "Detail"
- `docs/admin/admin-base-stack.md` - stack, clean architecture, router,
  action bar responsif, testing
- `src/features/words/` - word-detail.ts (tipe konsumen), word-api.ts
  (request layer), words-page.tsx (kolom aksi), use-word-detail.ts (query)
- `src/features/contributions/presentation/contribution-entity-view.tsx`
  - pola tampilan read-only yang ditiru halaman ini