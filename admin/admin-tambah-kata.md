
Buatkan UI halaman admin "Tambah Kata Baru" untuk aplikasi Kamus Digital
Sambas-Indonesia. Ini adalah form CRUD untuk content management, dipakai
admin/kontributor menambahkan satu entri kosakata lengkap dalam satu
submit.

KONTEKS:
Aplikasi kamus mirip SpanishDictionary.com, tapi untuk bahasa Sambas
(bahasa daerah Kalimantan Barat) ke Indonesia. Skema database sudah
mendukung: kata dasar, dialek, kelas kata (hierarkis), makna/definisi,
terjemahan makna, contoh kalimat, kategori/glosarium, sinonim,
pengucapan, dan sumber data.

STRUKTUR FORM (bagi jadi beberapa section/step):

1. DATA KATA DASAR

   - Kata Sambas / Lemma (text input, wajib)
   - Bahasa (dropdown, default: "Sambas", disabled kalau cuma 1 bahasa)
   - Dialek (dropdown/searchable, opsional — misal "Umum", "Sambas Kota",
     "Sambas Pesisir")
   - Catatan tambahan (textarea, opsional)
2. MAKNA / ARTI (bisa lebih dari satu, tombol "+ Tambah Makna")
   Untuk setiap makna:

   - Kelas kata (dropdown searchable: Nomina, Verba, Adjektiva, dst —
     tampilkan hierarki kalau ada, misal "Verba > Verba Transitif")
   - Definisi konseptual (textarea, wajib)
   - Terjemahan ke Indonesia (text input, wajib) — ini yang tampil
     sebagai arti utama
   - Tipe terjemahan (dropdown: Langsung/Deskriptif/Idiomatik, default
     "Langsung")
   - Urutan tampil (angka, auto-increment tapi bisa diubah manual)
3. CONTOH KALIMAT (nested di dalam tiap makna, tombol "+ Tambah Contoh")

   - Kalimat dalam bahasa Sambas (textarea, wajib)
   - Terjemahan kalimat ke Indonesia (textarea, wajib)
   - Sumber contoh (dropdown: Penutur Asli/Buku/Korpus/Wawancara/Lainnya)
4. KATEGORI / GLOSARIUM (multi-select searchable, dengan opsi
   "+ buat kategori baru" langsung dari dropdown)

   - Contoh tag yang sudah ada: Kekerabatan, Alam, Makanan, dst
5. RELASI KATA (opsional, collapsible section)

   - Sinonim (searchable multi-select ke kata lain yang sudah ada)
   - Antonim (sama)
6. PENGUCAPAN (opsional, collapsible section)

   - Notasi IPA (text input)
   - Audio upload (placeholder untuk fitur nanti, tampilkan sebagai
     "Coming Soon" atau upload file biasa)

AKSI FORM:

- Tombol "Simpan sebagai Draft" (status belum diverifikasi)
- Tombol "Simpan & Publikasikan" (langsung tayang, untuk role admin)
- Validasi: field wajib ditandai, tampilkan error inline, jangan biarkan
  submit kalau makna/lemma kosong

PERILAKU UX:

- Bagian "Makna" dan "Contoh Kalimat" bisa ditambah/dihapus dinamis
  (accordion/card list dengan tombol tambah & hapus)
- Autosave draft setiap beberapa detik (opsional, sebutkan sebagai nice-
  to-have)
- Setelah submit sukses, tampilkan toast konfirmasi + redirect ke daftar
  kata atau halaman detail kata yang baru dibuat

GAYA VISUAL:

- Bersih, mirip dashboard admin modern (referensi: Notion database view
  atau form builder seperti Airtable)
- Gunakan card/section terpisah per bagian data biar tidak terasa
  seperti satu form panjang yang membingungkan
- Responsive, prioritaskan desktop tapi tetap bisa dipakai di tablet
