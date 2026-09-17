# API Word - Sinonim Baru Inline (Inline Synonym Creation)

Mengikuti `api-base-stack.md`: Clean Architecture feature-based
(`modules/word/`) - Section 3 (struktur folder), 9 (`@hono/zod-openapi`),
10 (testing), 11 (versioning `/api/v1/`), 13 (envelope & error), 15 (rate
limiting), 19 (ULID), 21 (audit trail), 22 (approval gate).

Dokumen ini **memperluas** kontrak `01-api-tambah-kata.md`. Fitur baru:
saat membuat kata baru, `related_words` kini menerima sinonim yang **juga
baru (belum ada di database)** dan dibuat sekaligus dalam satu request -
bukan hanya menautkan ke kata yang sudah ada.

Kata sinonim baru ini secara **default mewarisi definisi/makna induknya**
(inherit), lalu bisa di-ubah definisi/maknanya **satu per satu** bila
diperlukan (override). Ini menutup celah UX: "saya buat kata X, tapi
sinonimnya belum ada sama sekali".

---

## Prompt

```text
PERLUAS modul "word" (fitur Tambah Kata Baru) pada POST /api/v1/admin/words
agar related_words mendukung DUA bentuk entri (union), mengikuti struktur
clean architecture feature-based dan kontrak yang sudah ada di
01-api-tambah-kata.md.

STACK (tidak berubah): Hono + OpenAPIHono (Section 9), Drizzle ORM +
PostgreSQL, Zod, Drizzle Kit (Section 7), ULID (Section 19).

LOKASI MODUL: modules/word/ (perluas file yang sudah ada)

------------------------------------------------------------------------
1. KONTRAK related_words — DUA BENTUK
------------------------------------------------------------------------

Array related_words sekarang menerima dua bentuk per item. DALAM SATU
REQUEST BOLEH CAMPUR keduanya (mis. 2 link ke kata lama + 1 sinonim baru).
Masing-masing item WAJIB tepat SATU dari dua bentuk (salah satu, bukan
dua-duanya / bukan kosong):

  Form A — LINK ke entri yang SUDAH ada (tidak berubah dari 01):
  {
    "relation_type": "synonym",
    "word_id": "01HXYZAB..."            // ULID kata yang sudah ada
  }

  Form B — BUAT entri baru secara INLINE (BARU):
  {
    "relation_type": "synonym",
    "word": {
      "lemma": "ngamakn",
      "word_type": "word",                       // default 'word'
      "notes": "Varian lisan, lafal daerah",

      "inherit_meanings": true,                   // DEFAULT true — ikut definisi induk
      "meaning_overrides": [                      // opsional, hanya saat inherit=true
        {
          "meaning_index": 0,                     // indeks 0-based makna induk
          "definition": "Mengunyah makanan",
          "word_class_id": "01HXYZAC...",
          "translations": [
            { "language_id": "01HXYZAD...",
              "translation_text": "kunyah",
              "translation_type": "direct" }
          ],
          "examples": [
            { "source_language_id": "01HXYZAE...",
              "source_sentence": "Ikam ngamakn nasi.",
              "target_language_id": "01HXYZAF...",
              "target_sentence": "Dia mengunyah nasi.",
              "source_type": "native_speaker" }
          ]
        }
      ],
      "meanings": [ ... ],                        // wajib DAN HANYA saat inherit_meanings=false

      "pronunciation": { "notation": "ipa", "value": "/ŋamakn/" },
      "variants": [ ... ],
      "images": [ ... ],
      "category_ids": [ ... ],
      "status": "draft"                           // default: ikut status induk (Section 22)
    }
  }

CONTOH BODY LENGKAP (induk + 1 sinonim inline):

{
  "language_id": "01HXYZAB...",
  "lemma": "makatn",
  "meanings": [
    {
      "word_class_id": "01HXYZAC...",
      "definition": "Aktivitas memasukkan makanan ke mulut",
      "order_index": 1,
      "translations": [
        { "language_id": "01HXYZAD...",
          "translation_text": "makan",
          "translation_type": "direct" }
      ],
      "examples": [
        { "source_language_id": "01HXYZAE...",
          "source_sentence": "Kami udah makatn tadi.",
          "target_language_id": "01HXYZAF...",
          "target_sentence": "Kami sudah makan tadi.",
          "source_type": "native_speaker" }
      ]
    }
  ],
  "word_type": "word",
  "category_ids": ["01HXYZAG..."],
  "related_words": [
    { "relation_type": "synonym",
      "word": {
        "lemma": "ngamakn",
        "inherit_meanings": true,                 // default — ikut makna "makatn"
        "meaning_overrides": [
          { "meaning_index": 0,
            "definition": "Mengunyah makanan" }   // hanya makna ke-0 yang beda
        ],
        "status": "draft"
      } }
  ],
  "status": "published"
}

------------------------------------------------------------------------
2. SEMANTIK INHERIT MAKNA (BY DEFAULT IKUT INDUK)
------------------------------------------------------------------------

inherit_meanings: true (DEFAULT):
- Server MENYALIN lengkap graph makna induk yang dikirim DI DTO request
  (bukan query ulang ke DB): untuk tiap kali meanings induk dibuat baris
  meanings baru milik sinonim dengan isi & order_index identik, berikut
  meaning_translations dan examples-nya.
- Hasil salinan disimpan asli (materialized), bukan link/pointer runtime —
  supaya query detail, search, dan alur review (Section 22) tetap
  sederhana dan independen antar entri.
- Tiap baris meanings hasil salinan mencatat sumbernya di kolom baru
  `meanings.inherited_from_meaning_id` (provenance: "ini ikut def. induk
  mana") — untuk audit & fitur "reset ke induk"/re-sync di masa depan.
- meaning_overrides[] bisa memodifikasi makna hasil salinan SATU PER SATU
  (lihat aturan).
- Menu "kelola makna" di UI memperlihatkan makna mana yang masih ikut
  induk (inherited_from_meaning_id terisi) dan mana yang sudah di-override
  (null) — override berarti makna itu SELESAI "mengikuti" induknya.

inherit_meanings: false:
- Sinonim TIDAK mewarisi apa-apa; field "meanings" WAJIB diisi penuh
  dengan bentuk yang sama seperti meanings induk (semua aturan validasi
  meanings di 01 berlaku: minimal 1 makna, dst).

inherit_meanings: true + "meanings" diisi → VALIDATION_ERROR (berbenturan).
inherit_meanings: false + "meaning_overrides" diisi → VALIDATION_ERROR
(hanya masuk akal saat inherit).

------------------------------------------------------------------------
3. ATURAN & VALIDASI (tambahan di create-word.validator.ts)
------------------------------------------------------------------------

1. Pemilahan bentuk: item ber-"word_id" = Form A; item ber-"word" = Form B;
   punya keduanya / tak punya sama sekali → VALIDATION_ERROR.
2. Form B hanya sah untuk relation_type yang menghubungkan dua kata
   (semua nilai "synonym" | "antonym" | "has_component" | "derived_from"
   — sama dengan Form A).
3. inherit_meanings: true → "meanings" dilarang; result SINONIM dijamin
   punya makna (karena induk WAJIB minimal 1 makna) — guard defensif tetap
   ada.
4. meaning_index WAJIB valid (0 <= index < jumlah meanings induk);
   index out of range → VALIDATION_ERROR field
   "related_words.N.word.meaning_overrides.M.meaning_index". Override
   boleh menyebut ulang index sama → yang menang = yang terakhir (last
   one wins), atau ditolak sebagai duplikat — PILIH: ditolak
   (deterministik, hindari user salah faham).
5. Lemma Form B WAJIB ada (aturan lemma yang sama seperti 01) dan:
   - lemma inline == lemma induk (case-insensitive) → VALIDATION_ERROR
     (kata tidak bisa jadi sinonim dirinya sendiri).
   - lemma inline duplikat antar Form B dalam satu request →
     VALIDATION_ERROR.
   - lemma inline duplikat dengan kata LAIN yang sudah ada di language itu
     → warning (bukan error), perilaku sama seperti duplikat lemma induk
     (01 bagian c).
6. Guard jumlah: maksimal 5 Form B per request → VALIDATION_ERROR.
7. Referensi eksternal Form B (word_class_id dari override/meanings,
   language_id dari translations/examples, category_ids, variant
   dialect_id, pronunciation) divalidasi eksistensinya bersama referensi
   induk — SATU panggilan findMissingReferences dengan kumpulan gabungan
   semua id, lalu error dipetakan ke field path yang benar
   (related_words.N.word....). Form A tetap divalidasi sebagai kata
   existing (harus ada, belum soft-deleted).
8. Model publikasi (Section 22) diterapkan PER ENTITAS (induk & tiap
   kata inline) dengan aturan role yang sama — lihat bagian 7.
   status Form B default = status yang dikirim di body induk; boleh
   di-override per Form B.

------------------------------------------------------------------------
4. PERUBAHAN DATABASE (migration 0010)
------------------------------------------------------------------------

SATU perubahan: tambah kolom provenance pada tabel meanings.

  ALTER TABLE meanings
    ADD COLUMN inherited_from_meaning_id varchar(26) REFERENCES meanings(id);
  CREATE INDEX meanings_inherited_from_idx ON meanings(inherited_from_meaning_id);

- Self-referencing FK (makna → makna), NULLABLE (makna "biasa"/sudah
  di-override), tidak ada DROP/hard delete (Section 7), tidak memengaruhi
  baris lama.
- Sync ke docs/dbdiagram.dbml (kolom + note "04: sinonim inline").
- Jalankan: pnpm drizzle-kit generate --name=add-inherited-from-meaning
  → REVIEW SQL → migrate → commit schema + migration dalam satu PR
  (Section 7).

------------------------------------------------------------------------
5. PERUBAHAN DTO / USE CASE / REPOSITORY
------------------------------------------------------------------------

DTO (create-word.dto.ts):
- CreateWordRelatedDto → union:
  { wordId: string; relationType: RelationType }            // Form A
  | { relationType: RelationType; word: InlineWordDto; }    // Form B
- InlineWordDto = subset CreateWordDto + inherit_meanings:
  { lemma, notes?, wordType?, categoryIds?,
    inheritMeanings?: boolean,                 // default true
    meaningOverrides?: MeaningOverrideDto[],   // hanya saat inherit=true
    meanings?: CreateWordMeaningDto[],         // hanya saat inherit=false
    variants?, pronunciation?, images?,
    status?: 'draft' | 'published' }
- MeaningOverrideDto = { meaningIndex: number; definition?;
  wordClassId?; translations?; examples? } (translate + replace; field
  yang tidak disebut TETAP memakai hasil salinan).

USE CASE (create-word.use-case.ts):
1. Pembagian bentuk; untuk tiap Form B resolusi makna:
   - inherit=true  → copy meanings induk (dari DTO), lalu terapkan
     meaning_overrides (update field yang disebut; translations/examples
     yang dikirim = replace total hasil salinan).
   - inherit=false → pakai meanings yang dikirim apa adanya.
2. Validasi gabungan referensi (di atas), duplikat lemma induk + tiap
   lemma inline, warning per kata.
3. resolvePublication dijalankan per entitas → sinhala status/isVerified
   Masing-masing (induk + tiap kata inline) dipasang ke
   WordToSave/WordsResolved.
4. Panggil repository untuk SIMPAN KLUSTER dalam satu transaksi atomik.

REPOSITORY (word.repository.impl.ts):
- Perluas kontrak saveWithRelations → saveWithRelationsAndRelated(
    word, actorId, related: Array<{ relationType, inlineWord: WordToSave,
    inheritedFrom: Record<meaningIndex, parentMeaningId> }>, actorId )
  atau method baru bernama jelas (mis. saveWithInlineRelations) — SEMUA
  insert berada dalam SATU db.transaction() yang sama (Section 4:
  atomicity di kontrak repository, use case tidak tahu transaction).
  Rollback jika salah satu insert gagal → TIDAK ada baris tersisa.
- Urutan insert dalam transaksi:
  1) kata INDUK (words) + makna2 induk (meanings + translations +
     examples + word_categories + variants + pronunciations + images)
     — DIAMBIL id tiap makna induk untuk dipakai provenance.
  2) untuk TIAP Form B: insert kata inline (words) + makna hasil
     resolusi (meanings, tiap baris diisi inherited_from_meaning_id =
     id makna induk terkait; lalu translations/examples), + kategori +
     variants + pronunciation + images.
  3) lexical_relations: source = kata induk, target = tiap kata inline
     (relation_type sesuai request; invers tetap TIDAK disimpan).
  4) contributions: SATU baris per entitas yang dibuat (induk + tiap
     kata inline, entity_type 'word', action 'create', status sesuai
     publication masing-masing).

AUDIT (Section 21): SATU entri audit per entitas yang dibuat
action 'create', entity_type 'word'. Untuk kata inline sertakan
new_data { lemma, word_type, status, is_verified, meanings_count,
inherited_meanings_count, overridden_meanings_count }. request_id tetap
dari context.

------------------------------------------------------------------------
6. PERUBAHAN RESPONSE
------------------------------------------------------------------------

createWordResponseSchema diperluas dengan field opsional
"inline_created_words" — memuat HASIL tiap kata inline yang dibuat (bukan
dipisah dari data induk), urut sesuai request:

{
  "success": true,
  "data": {
    "word_id": "01HXYZAA...",
    "lemma": "makatn",
    "word_type": "word",
    "status": "published",
    "is_verified": true,
    "created_at": "2026-09-18T10:00:00Z",
    "warnings": [],                        // duplikat lemma induk
    "inline_created_words": [
      {
        "word_id": "01HXYZBA...",
        "lemma": "ngamakn",
        "relation_type": "synonym",
        "word_type": "word",
        "status": "draft",
        "is_verified": false,
        "meanings_count": 1,
        "inherited_meanings_count": 1,     // berapa makna yang masih ikut induk
        "overridden_meanings_count": 1,    // berapa makna yang di-override
        "warnings": []                     // duplikat lemma kata ini (jika ada)
      }
    ]
  }
}

GET /api/v1/words/:id — response meanings[] diperluas dengan field
opsional "inherited_from_meaning_id": string | null — membedakan makna
yang masih "mengikuti" induk dengan yang sudah mandiri (untuk UI).

------------------------------------------------------------------------
7. MODEL PUBLIKASI & VERIFIKASI (Section 22 — per entitas)
------------------------------------------------------------------------

- TIAP kata (induk & tiap kata inline) diproses sendiri oleh
  resolvePublication dgn aturan role TANPA perubahan:
  admin/editor/root/reviewer → published + is_verified true;
  contributor + published → pending_review (bukan tayang).
- Contributor yang submit "published" → induk DAN semua kata inline masuk
  antrean review (baris contributions 'pending') — masing-masing entitas.
- Verifikator menyetujui/menolak SATU-PERSATU lewat antrean (modul
  contribution, 03-api-kontribusi-verifikasi.md) — menyetujui induk TIDAK
  otomatis menyetujui sinonim inline.
- Relasi synonym baru TAMPAK di detail kedua arah hanya saat KEDUA entitas
  published (query relasi sudah memfilter status — lihat findDetailById).
  Sinonim masih pending_review → belum muncul di related_words induk.
- Soft-delete & verify/unverify berlaku per entitas biasa (tidak ada
  aturan cascade baru).

------------------------------------------------------------------------
8. TESTING (Section 10 — wajib)
------------------------------------------------------------------------

UNIT (create-word.use-case + validator):
- Form B inherit=true → outcomes: makna sinonim = salinan makna induk
  (termasuk translations/examples), baris provenance mengarah benar;
  override definisi/translasi satu makna hanya mengubah makna itu.
- inherit=false + meanings penuh → valid; + meaning_overrides → error;
  tanpa meanings → error.
- Pemilahan bentuk: word_id+word dua-duanya → error; kosong → error.
- lemma inline == lemma induk → error; duplikat antar Form B → error;
  duplikat dengan kata existing → warning.
- indeks override out-of-range / duplikat → error; guard maksimal 5.
- Role matrix: contributor → induk + semua inline pending_review.

INTEGRATION (word.repository.impl):
- Satu transaksi: induk + N kata inline + relations + N+1 contributions
  semua tersimpan; relasi source=induk target=inline.
- ROLLBACK: insert inline ke-2 gagal (mis. FK word_class_id palsu di
  override) → TIDAK ada baris tersisa (induk pun).
- findDetailById: makna inline memuat inherited_from_meaning_id yang
  benar; kata inline pending_review belum muncul sebagai related_words
  induk yang published.

E2E: POST admin/words dengan campuran Form A + Form B → response memuat
inline_created_words (jumlah, status, count); 201; gagal validasi → 400
dengan field path related_words.N.word.*.

------------------------------------------------------------------------
9. DELIVERABLES (satu PR yang sama dengan kode — Section 20)
------------------------------------------------------------------------

- http/word/create-word-with-inline-synonym.bru (+ tests: status 201,
  inline_created_words length, status per role, warnings).
- http/word/create-word-link-existing.bru (regresi Form A).
- docs/json/word/create-word.201.inline-synonym.json (contoh respons).
- Update docs/admin/admin-tambah-kata.md (UI: UI daftar sinonim bisa
  memilih kata lama/baru — kontrak 04).
- Migration 0010 + dbdiagram.dbml + apibase-stack Section 18 (tambah
  referensi 04 ini).
- ERROR_CODES.md TIDAK berubah (tidak ada error code baru; semua pakai
  VALIDATION_ERROR).
```

---

## Catatan Implementasi

- **Materialized, bukan live link.** Makna sinonim disalin penuh saat
  request; kolom `inherited_from_meaning_id` hanya provenance + dasar
  fitur "reset ke definisi induk" di masa depan — evaluasi read, search,
  dan review tidak perlu JOIN runtime. Kompromi sadar: setelah pembuatan,
  mengubah makna induk TIDAK otomatis mengubah makna sinonim kecuali
  lewat override/update ulang (jalan: endpoint update kata yang sudah
  direncanakan di 01, atau fitur re-sync menyusul).
- Update kata inline (mengubah maknanya "1 per 1" belakangan) dilakukan
  lewat endpoint update/soft-delete yang sama dengan kata biasa (sudah
  ada `updateWithRelations` / planned prompt di 01) — sesudah inline
  dibuat, kata itu 100% entri mandiri dan tidak butuh mekanisme khusus.
- Sinonim inline tidak perlu endpoint terpisah (tidak ada
  POST /synonyms) — cukup perluasan related_words di create word.

## Referensi Terkait

- `01-api-tambah-kata.md` - kontrak dasar create word yang diperluas di sini
- `api-base-stack.md` - Section 22 (approval gate), 21 (audit), 18 (daftar
  referensi prompt)
- `03-api-kontribusi-verifikasi.md` - antrean review (kata inline masuk
  sebagai entitas mandiri)
- `docs/admin/admin-tambah-kata.md` - UI form yang mengonsumsi fitur ini
- `docs/dbdiagram.dbml` - skema (kolom baru inherited_from_meaning_id)