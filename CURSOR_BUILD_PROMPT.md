# DWA Album Factory — Build Spec

> **Cara pakai:** buka folder repo ini di Cursor → Composer mode Agent → paste seluruh isi file ini.

---

## 0. Sebelum menulis kode

Lakukan tiga hal ini dulu, berurutan:

1. **Baca kelima file prototipe HTML** di root repo. Itu sumber kebenaran desain, bukan referensi longgar.
2. **Buat `README.md` dan `docs/decisions.md`.** README berisi cara menjalankan dari nol; decisions berisi satu entri pertama: "Spec awal diterima, lihat CURSOR_BUILD_PROMPT.md".
3. **Laporkan rencana M0** dalam 5 baris, lalu langsung kerjakan. Jangan menunggu persetujuan.

Setelah itu kerjakan **M0 → M7 berurutan**. Jangan lompat. Tiap milestone: bangun → jalankan acceptance test → commit → laporkan singkat → lanjut.

Format laporan tiap milestone:
```
Mx selesai
- Dibangun: ...
- Acceptance test: [lulus/gagal] + hasilnya
- Keputusan yang saya ambil sendiri: ...
- Perlu diputuskan sebelum lanjut: ... (atau: tidak ada)
```

---

## 1. Apa yang dibangun

**Bukan** SaaS tempat pengantin mendaftar sendiri. Ini **pabrik album** — alat produksi internal yang dijalankan satu operator (pemilik studio). Klien tidak punya akun dan tidak pernah melihat editor; klien hanya menerima album jadi.

Alur kerja operator:

```
1. Klien menyerahkan foto & video
2. Operator buat project baru, isi data klien
3. Operator masukkan semua foto + 1 wedding film
4. Sistem menyeleksi & menyusun otomatis → tampil di preview
5. Operator merapikan di preview: buang yang jelek, tukar, geser, ganti layout
6. Export → album statis siap deploy
7. Tujuan deploy ditentukan per klien
```

Ditambah satu komponen kecil terpisah: **Guest Intake** — halaman QR tempat tamu undangan menyumbang foto saat hari-H, lalu ditarik masuk ke Studio.

### Tiga komponen, beda tempat jalan

| Komponen | Jalan di mana | Sifat | Porsi kerja |
|---|---|---|---|
| **Studio** | `localhost` di komputer operator | Berat: ingest, kurasi, preview, editor, export | ~75% |
| **Guest Intake** | Server online (NAS), aktif ~1 minggu seputar hari-H | Sangat tipis: QR landing + terima upload. Tidak ada editor. | ~15% |
| **Album terbit** | Statis, deploy per klien | Tanpa server. HTML + gambar. | ~10% |

**Prinsip yang mengikat semuanya:** yang online seminimal mungkin. Pekerjaan berat terjadi di localhost — tidak ada biaya bandwidth, tidak ada batas waktu request, tidak ada yang bisa jebol.

---

## 2. Aset yang sudah ada

Lima prototipe HTML di root repo. **Ikuti, jangan didesain ulang.**

| File | Isi | Dipakai untuk |
|---|---|---|
| `dwa-prototype.html` | Style guide, token, komponen | Ambil `:root` → jadi `tokens.css` |
| `dwa-r2-s1.html` | Creation Flow, 6 layar | Alur buat project baru di Studio |
| `dwa-r2-s2.html` | Desktop Album Editor | **Layar utama Studio** — chapter list, live preview, panel kanan |
| `dwa-r2-s3.html` | Mobile Album Viewer, 9 halaman | Template album terbit + isi live preview |
| `dwa-r2-s4.html` | Guest Experience, 5 layar | Guest Intake |

Token warna (dari blok `:root` di `dwa-prototype.html`):
```
--ink:#0E0E10  --espresso:#1C1915  --stone:#6B5E52  --muted:#9C8C80
--champagne:#E8DCD0  --ivory:#F7F2EA  --gold:#C9A96E  --gold-dk:#A08848
```

Font: **Cormorant Garamond** (display) + **DM Sans** (UI/body), Google Fonts.
Bahasa UI: **Indonesia**. Konteks: pernikahan adat Indonesia — Akad Nikah, Siraman, Midodareni, Panggih, Resepsi dipakai apa adanya.

---

## 3. Keputusan terkunci

Jangan diubah tanpa menuliskan alasannya di `docs/decisions.md`.

| Aspek | Keputusan | Kenapa |
|---|---|---|
| Studio runtime | **Node.js 20 + Express**, dibuka di browser `http://localhost:4000` | Bukan Electron. Tidak perlu bundling desktop. |
| Database Studio | **SQLite** via `better-sqlite3` | Satu operator, satu mesin. Postgres + Docker jadi beban tanpa manfaat. Satu file, gampang di-backup. |
| Frontend | **React 18 + Vite**, JavaScript (bukan TypeScript), plain CSS + CSS variables | Token sudah ada; Tailwind cuma menambah lapisan. |
| Auth Studio | **Tidak ada.** Bind ke `127.0.0.1`. | Localhost, satu operator. Login = keamanan teater. |
| Image | `sharp` | — |
| Video | `ffmpeg` via `ffmpeg-static`, spawn langsung | Jangan `fluent-ffmpeg` — argumen eksplisit lebih mudah di-debug. |
| Guest Intake | Node + Express + SQLite, container Docker kecil di NAS | Harus online, tapi permukaannya sengaja sempit. |
| Album terbit | **Statis murni** — HTML + CSS + JS + gambar. Tanpa server. | Deploy ke mana saja, hidup selamanya, tidak ikut mati kalau NAS mati. |
| Testing | `node:test` + `supertest` (server), Vitest (util frontend) | — |

### Aturan portabilitas export — wajib

Album hasil export **harus jalan di path apa pun tanpa dikonfigurasi ulang**, karena tujuan deploy ditentukan belakangan per klien:

- Semua path di HTML/CSS/JS **relatif** (`./media/...`), tidak pernah diawali `/`
- Tidak ada base URL yang di-hardcode
- Tidak ada request ke domain luar kecuali Google Fonts — sediakan fallback font lokal
- Membuka `index.html` lewat `file://` harus tetap menampilkan album (boleh tanpa fitur share)

Bundle yang sama harus bisa ditaruh di `domain.com/album/nama/`, di subdomain, di hosting klien, atau di flashdisk — tanpa build ulang.

---

## 4. Struktur folder

```
wedding-album-dwa/
├─ prototypes/               # 5 HTML dipindah ke sini di M0. Arsip. Jangan diubah.
├─ studio/
│  ├─ server/
│  │  ├─ src/
│  │  │  ├─ index.js         # express, bind 127.0.0.1:4000
│  │  │  ├─ db.js            # better-sqlite3 + migration runner
│  │  │  ├─ paths.js         # SEMUA path project ada di sini
│  │  │  ├─ routes/          # projects, media, chapters, pages, curate, export, intake
│  │  │  ├─ pipeline/        # ingest.js, image.js, video.js, worker.js
│  │  │  ├─ ai/              # quality.js, dedup.js, faces.js, curate.js, layout.js, scorers/
│  │  │  └─ export/          # build.js, template/  ← template album terbit
│  │  └─ test/
│  └─ web/                   # React: Studio UI
├─ intake/                   # Node + React, service tamu (Docker)
├─ docs/
└─ data/                     # .gitignore — project operator ada di sini
   └─ projects/<slug>/
      ├─ project.db
      ├─ originals/          # file asli, TIDAK PERNAH diubah
      ├─ derived/            # thumb/preview/full/video hasil olahan
      └─ export/             # bundle siap deploy
```

**Aturan `originals/`:** file asli klien ditulis sekali saat ingest, sesudah itu read-only. Semua operasi bekerja di `derived/`. Kalau operator salah langkah, foto asli tidak pernah hilang — dan foto pernikahan tidak bisa difoto ulang.

### npm scripts yang harus ada

```
npm start                          # jalankan Studio, buka localhost:4000
npm test                           # semua test
npm run export -- --slug=x [--zip] # export album
npm run deploy -- --slug=x --target=<sftp|dir>
npm run backup                     # tar semua project + db, bernama tanggal
npm run intake:build               # build image Docker guest intake
```

---

## 5. Skema database (SQLite)

Satu `data/studio.db` berisi daftar project, plus satu `project.db` per project.

**`studio.db`**
```sql
projects(id, slug UNIQUE, client_name, bride_name, groom_name,
         event_date, venue, culture, style, status,
         created_at, updated_at, last_opened_at)
-- culture: 'jawa'|'sunda'|'bali'|'minang'|'batak'|'umum'
-- style:   'editorial'|'cinematic'|'romantic'|'minimal'
-- status:  'draft'|'curating'|'ready'|'exported'
```

**`project.db`**
```sql
media(
  id, kind,                    -- 'photo'|'video'
  source,                      -- 'client'|'guest'
  guest_id,                    -- NULL kalau dari klien
  original_path,               -- relatif terhadap folder project
  original_name, mime, bytes,
  width, height, duration_ms,  -- duration hanya untuk video
  taken_at,                    -- dari EXIF, NULL kalau tidak ada
  phash,                       -- dHash 64-bit hex, foto saja
  sharpness, brightness, face_count,
  quality_score,               -- 0..100
  status,                      -- 'ingesting'|'ready'|'failed'
  hidden INTEGER DEFAULT 0,    -- disembunyikan operator, bukan dihapus
  error_message,
  tags_json,                   -- NULL sekarang, lihat §7
  embedding,                   -- NULL sekarang, lihat §7
  scorer_version,              -- NULL sekarang, lihat §7
  created_at
);
CREATE INDEX ON media(quality_score DESC);
CREATE INDEX ON media(phash);
CREATE INDEX ON media(taken_at);

chapters(id, idx, title, subtitle, kind, created_at);
-- kind: 'cover'|'story'|'journey'|'ceremony'|'tradition'|'family'|'reception'|'film'|'guest'
-- UNIQUE(idx)

pages(id, chapter_id, idx, layout_key, caption, created_at);
-- UNIQUE(chapter_id, idx)

slots(id, page_id, slot_idx, media_id, crop_json);
-- UNIQUE(page_id, slot_idx)
-- media_id: ON DELETE SET NULL

guests(id, name, relation, message, created_at);
-- relation: 'keluarga_pria'|'keluarga_wanita'|'teman_kuliah'|'teman_kerja'|'tetangga'|'lainnya'

settings(key PRIMARY KEY, value);
-- wedding_film_media_id, cover_media_id, intake_token, intake_url, dll

decisions(
  id, at,
  action,            -- 'hide'|'unhide'|'swap'|'pick'|'reorder'|'delete'|'set_cover'|'undo'
  media_id,          -- yang dipilih / dikenai aksi
  rejected_json,     -- ARRAY media_id yang tersedia tapi TIDAK dipilih
  context_json,      -- {chapter_kind, layout_key, slot_idx, page_idx, source}
  snapshot_json      -- skor & tag SEMUA kandidat saat itu, chosen + rejected
);
CREATE INDEX ON decisions(at);
```

Aturan:
- Operator **tidak pernah menghapus permanen** dari preview — set `hidden=1`. Ada tombol terpisah "Hapus permanen" dengan konfirmasi, dan itu pun tidak menyentuh `originals/`.
- `slots.media_id` **`ON DELETE SET NULL`**, jangan cascade — menghapus foto tidak boleh merusak struktur halaman.

---

## 6. Pipeline ingest

Bagian yang paling menentukan alat ini enak dipakai atau tidak. Operator akan menyeret 300–800 file sekaligus.

### Foto
1. Salin ke `originals/`, nama asli dipertahankan (tambah suffix kalau bentrok)
2. Baca EXIF: `taken_at`, orientasi, dimensi
3. Auto-rotate sesuai EXIF, lalu buat varian ke `derived/`:
   - `thumb` 400px lebar, WebP q75
   - `preview` 1200px, WebP q82
   - `full` 2400px, WebP q86 + JPEG fallback q88
4. Hitung `sharpness`, `brightness`, `face_count`, `phash`, `quality_score` (§7)
5. `status` → `ready`

### Video (satu wedding film per album)
1. Salin ke `originals/`
2. Probe `ffprobe`: durasi, resolusi, codec
3. Transcode ke `derived/`:
   - `film-1080.mp4` — H.264 High, CRF 21, AAC 128k, `-movflags +faststart`
   - `film-720.mp4` — H.264 Main, CRF 23, untuk koneksi lemot
   - `poster.jpg` — frame detik ke-3 (atau 10% durasi kalau video pendek)
4. Simpan `wedding_film_media_id` di `settings`

### Aturan eksekusi
- **Worker thread pool**, jumlah worker = `cpus().length - 1`, minimal 1
- Progress real-time via **SSE** (`GET /api/:slug/ingest/stream`), bukan polling
- **Bisa dilanjutkan**: kalau proses mati di tengah, jalankan lagi → lewati yang sudah `ready`, ulangi yang `ingesting`/`failed`
- File gagal → `status='failed'` + `error_message`, **jangan hentikan batch**. Tampilkan daftar gagal di akhir.
- Format diterima: `jpg jpeg png heic heif webp tif tiff` (foto), `mp4 mov mkv avi` (video)
- **HEIC dari iPhone wajib jalan** — `sharp` dengan libheif; kalau environment tidak mendukung, fallback ke ffmpeg

---

## 7. Modul kurasi

Semua di `studio/server/src/ai/`. Ini **heuristik, bukan model terlatih**. Label "AI" di UI boleh dipertahankan sesuai prototipe, tapi nama fungsi, komentar, dan `docs/` harus jujur.

Karena jalan di localhost: tidak ada batas waktu request dan tidak ada biaya API, jadi boleh lebih teliti daripada kalau di server.

### `quality.js`
- **Sharpness** — variance of Laplacian: resize lebar 512 → grayscale → konvolusi `[0,1,0, 1,-4,1, 0,1,0]` → variance. Rendah = blur.
- **Brightness** — rata-rata luminance 0..255. Tandai buruk kalau < 25 atau > 235.
- **Face count** — `@vladmandic/face-api`, model TinyFaceDetector, **model di-bundle di repo**, tidak diunduh saat runtime.

```
quality_score = 45*norm(sharpness) + 20*brightnessOk + 25*min(faces,3)/3 + 10*resolutionOk
```
Kalibrasi bobotnya pakai fixture nyata di `studio/server/test/fixtures/` (sertakan foto blur & gelap buatan). Jangan hardcode tanpa uji.

### `dedup.js`
dHash 64-bit: resize 9×8 grayscale → bandingkan pixel bertetangga horizontal → 64 bit → hex 16 char.
Duplikat kalau **Hamming distance ≤ 8**. Kelompokkan, wakil = `quality_score` tertinggi.
Fotografer pernikahan menembak burst — ini memangkas 30–50% file. Wajib benar.

### `curate.js`
Input: semua media `ready` & tidak `hidden`. Output: draft chapter + page + slot.

- Buang duplikat (sisakan wakil), buang `quality_score < 35`
- Kelompokkan per **momen** pakai `taken_at`; jeda > 20 menit = momen baru → album jadi kronologis
- Maksimal 3 foto per momen, jaga keragaman
- Foto dengan `face_count == 2` diprioritaskan untuk cover & chapter story
- Petakan momen → chapter sesuai `culture`. Jawa: Siraman → Midodareni → Akad → Panggih → Resepsi
- Kalau `taken_at` kosong di sebagian besar file, fallback ke urutan nama file

**Foto tamu (`source='guest'`) memakai aturan berbeda — jangan disatukan:**

| | Foto klien (fotografer) | Foto tamu |
|---|---|---|
| Ambang kualitas | buang `< 35` | buang `< 20` saja |
| Dedup | Hamming ≤ 8, agresif | Hanya **dalam satu tamu**. Antar tamu jangan di-dedup. |
| Kuota | Dikurasi ketat | Setiap tamu **dijamin minimal 1 foto tampil** |
| Masuk chapter | Sesuai momen | Selalu ke chapter `kind='guest'` |

Alasannya bukan teknis. Foto tamu diambil pakai HP di ruangan remang — kalau dipakai ambang yang sama dengan foto fotografer, hampir semua terbuang, dan tamu yang sudah repot scan QR tidak menemukan fotonya. Dua tamu yang memotret momen sama dari sudut berbeda **bukan duplikat** — itu dua orang yang sama-sama hadir. Tujuan chapter ini keterwakilan, bukan kurasi.

### `layout.js`
Tabel keputusan berdasarkan jumlah + orientasi, **bukan** if-else bercabang:
```
1 landscape          → 'full-bleed'
1 portrait           → 'hero-plus-caption'
2 apa pun            → 'duo'
3 (1 landscape + 2)  → 'hero-plus-2'
4 seragam            → 'grid-2x2'
5-6                  → 'mosaic-3'
```

### Titik ekstensi untuk model vision nanti

Model vision **tidak dibangun sekarang**, tapi tempatnya disiapkan sekarang — menyisipkannya belakangan tanpa persiapan berarti membongkar `curate.js`.

**a. Satu interface scorer.** Semua penilai foto memenuhi kontrak yang sama:
```js
// ai/scorers/index.js
// score(media, filePath) → { scores: {...}, tags: [...] }
//   scores: angka 0..1 per dimensi, kuncinya bebas
//   tags:   label bebas, contoh ['pelaminan','grup','outdoor']
export const heuristicScorer = { name: 'heuristic', version: 1, score }
```
Sekarang cuma ada `heuristicScorer`. Nanti tinggal tambah `visionScorer` dan gabungkan hasilnya — tanpa menyentuh `curate.js`.

**b. Tiga kolom disiapkan sejak migration pertama, biarkan NULL** — `tags_json`, `embedding`, `scorer_version` (sudah ada di §5). Menambah kolom ke SQLite yang sudah berisi puluhan project itu merepotkan; membiarkannya NULL sekarang gratis.

**c. `curate.js` membaca `scores` dan `tags`, bukan `sharpness` langsung.** Kalau `curate.js` menyebut `media.sharpness` di mana-mana, dia terikat ke heuristik selamanya.

**d. Rekam keputusan operator sejak M4** — lihat §8.

**Berhenti di situ.** Jangan bikin plugin system, registry, atau abstraksi berlapis untuk sesuatu yang belum ada.

---

## 8. Preview & editor — jantung aplikasi

Ikuti `dwa-r2-s2.html`. Ini layar yang paling lama dipakai operator, jadi paling menentukan.

- **Kiri**: daftar chapter, drag-reorder, indikator progres
- **Tengah**: live mobile preview — render **template album terbit yang sesungguhnya** di dalam frame HP, bukan tiruan. Satu template, dipakai preview **dan** export. Kalau ditulis dua kali, cepat atau lambat preview akan berbohong dan baru ketahuan setelah album dikirim ke klien.
- **Kanan**: tab AI Tools / Layouts / Photos / Story

Yang wajib bisa dilakukan di preview:
- Klik foto di preview → langsung terpilih di panel kanan
- **Sembunyikan** foto (satu klik, bisa di-undo) dan **ganti** foto dari grid kandidat
- Geser posisi foto antar slot dengan drag
- Ganti layout halaman, tambah/hapus halaman
- **Undo/redo minimal 30 langkah** — operator bekerja cepat dan akan salah klik. Simpan patch di memori.
- Autosave debounce 800 ms + indikator "Semua perubahan tersimpan" seperti prototipe

Grid kandidat harus bisa disortir: skor kualitas, waktu, jumlah wajah, sumber (klien/tamu), plus filter "hanya yang belum dipakai".

### Pencatatan keputusan operator

Setiap kali operator membuang atau menukar foto, dia menyatakan selera yang tidak diketahui sistem. Catat ke tabel `decisions`. **Tidak dipakai sekarang** — ini bahan untuk model vision nanti.

**Yang menentukan berguna atau tidaknya data ini: catat yang ditolak, bukan cuma yang dipilih.**

"Foto A disembunyikan" hampir tidak bernilai. "Untuk slot ini, dari kandidat A, B, C — operator memilih B" adalah **pasangan preferensi**, dan itu yang bisa dilatih. Karena itu `rejected_json` wajib diisi setiap kali ada pilihan nyata:

| Aksi | `media_id` | `rejected_json` |
|---|---|---|
| Ganti foto di slot | pengganti | foto lama + kandidat yang terlihat di grid saat itu |
| Sembunyikan foto | yang disembunyikan | kosong — ini sinyal negatif tunggal |
| Pilih cover | yang dipilih | semua kandidat cover yang ditawarkan |
| Terima hasil curate apa adanya | tiap foto terpakai | yang dibuang curate di momen yang sama |

`snapshot_json` menyimpan skor & tag **semua** kandidat saat keputusan diambil. Kalau setahun lagi rumus skornya berubah, data lama tetap terbaca sendiri tanpa perlu menghitung ulang dari foto asli.

Ketentuan:
- **Tanpa UI.** Operator tidak boleh merasa sedang diawasi atau diminta melabeli apa pun. Kalau terasa seperti kerja tambahan, dia akan mulai malas dan datanya jadi bias.
- Tulis asinkron, jangan pernah memblokir interaksi editor
- Ikut `npm run backup`, **tidak** ikut export ke klien
- Undo dicatat sebagai `action` tersendiri — membatalkan berarti keputusan pertama salah, dan itu informasi
- Tidak ada data pribadi tamu di sini selain `media_id`

Perkiraan ukuran: ~500 keputusan per album, < 1 MB.

---

## 9. Export

`POST /api/:slug/export` → menghasilkan `data/projects/<slug>/export/`:

```
export/
├─ index.html          # self-contained, semua path relatif
├─ assets/
│  ├─ album.css
│  ├─ album.js         # navigasi halaman, lazy load
│  └─ fonts/           # fallback lokal kalau Google Fonts diblokir
├─ media/
│  ├─ p001-thumb.webp  p001-preview.webp  p001-full.webp  p001-full.jpg
│  └─ film-1080.mp4    film-720.mp4       poster.jpg
├─ album.json          # data album, supaya bisa di-rebuild tanpa Studio
└─ MANIFEST.txt        # jumlah file, total ukuran, tanggal export, versi builder
```

Ketentuan:
- Hanya media yang **dipakai** di halaman ikut diexport. Jangan bawa 800 foto mentah.
- `srcset` 3 ukuran + `loading="lazy"` + `content-visibility:auto`
- Video `preload="none"` dengan poster — jangan pernah autoload
- **Target: LCP < 2,5 detik di Slow 4G**; total bundle album 12 halaman < 25 MB
- Foto yang `hidden` tidak pernah ikut export

---

## 10. Guest Intake

Service terpisah di `intake/`, container Docker di NAS. **Tipis sengaja** — tidak ada editor, tidak ada preview, tidak menyimpan apa pun selain foto tamu dan identitasnya.

Layar mengikuti `dwa-r2-s4.html`: QR Landing → Upload → Nama & Relasi → Uploading → Terima Kasih.

- URL per album: `https://<host>/i/<slug>`, dibuka dari QR
- **Tanpa login.** Identitas tamu = baris `guests` + token di `sessionStorage`
- Batas: maks **20 foto per tamu**, **15 MB per file**, hanya `image/jpeg|png|heic|webp`
- Rate limit **30 request/menit per IP**
- Jendela waktu `opens_at` / `closes_at` per album. Di luar jendela → halaman sopan "Pengumpulan foto sudah ditutup"
- Penolakan harus berpesan Indonesia yang jelas, bukan kode error

Tamu adalah **corong satu arah**: foto masuk, tidak ada yang keluar. Tamu tidak bisa melihat album, tidak bisa menghapus, tidak bisa melihat unggahan orang lain. Foto mereka tidak langsung tampil — masuk ke Studio sebagai kandidat, operator yang memutuskan.

### Sinkronisasi ke Studio
```
POST /api/:slug/intake/pull     # Studio menarik foto tamu baru
POST /api/:slug/intake/close    # Studio menutup jendela dari jarak jauh
```
Studio menyimpan `intake_url` + `intake_token` di `settings`. Tarikan bersifat **incremental** (berdasarkan `created_at` terakhir) dan **idempoten** — menarik dua kali tidak menggandakan foto. Setelah masuk, foto tamu lewat pipeline yang sama dengan `source='guest'`.

Setelah album diexport dan diserahkan, container intake untuk album itu **boleh dimatikan**. Tidak ada yang hilang — semua sudah ada di Studio.

### Chapter "Guest Memories"

Tujuan akhir foto tamu, dan tampilannya **beda dari chapter lain**. Ikuti halaman 9 `dwa-r2-s3.html`: mosaik 3 kolom + kartu pesan.

Layout khusus, hanya untuk chapter ini:
```
'mosaic-masonry'   → mosaik rapat, tinggi bervariasi, 15-40 foto per halaman
'message-cards'    → kutipan pesan tamu + nama & relasi
'mosaic-mixed'     → mosaik diselingi 2-3 kartu pesan
```

Ketentuan:
- Tiap foto membawa atribusi **nama tamu + relasi** ("Budi Santoso · Keluarga Ananda"), tampil saat di-tap/hover
- `guests.message` yang tidak kosong jadi kartu pesan. Potong di 180 karakter dengan elipsis.
- Urutkan **per tamu**, bukan per waktu — supaya kiriman satu orang berkelompok
- Boleh **beberapa halaman** kalau tamu banyak. Pecah otomatis tiap ~30 foto.
- Kalau tidak ada satu pun foto tamu, chapter ini **tidak dibuat** — jangan tampilkan halaman kosong

Operator tetap bisa menyembunyikan foto tamu satu per satu dari preview (moderasi).

---

## 11. API Studio

Prefix `/api`. Tanpa auth (localhost). Error seragam:
```json
{ "error": { "code": "PROJECT_NOT_FOUND", "message": "Project tidak ditemukan." } }
```

```
GET    /api/health
GET    /api/projects                    POST   /api/projects
GET    /api/projects/:slug              PATCH  /api/projects/:slug
DELETE /api/projects/:slug              # arsip, bukan hapus file

POST   /api/:slug/ingest                # { paths:[...] } folder lokal / drag-drop
GET    /api/:slug/ingest/stream         # SSE progress
GET    /api/:slug/media                 ?kind=&source=&hidden=&sort=&unused=
PATCH  /api/:slug/media/:id             # { hidden, crop }
DELETE /api/:slug/media/:id             # hapus permanen, konfirmasi wajib
GET    /api/:slug/duplicates

POST   /api/:slug/curate                # { target_pages }
POST   /api/:slug/pages/:id/autolayout

GET    /api/:slug/chapters              POST   /api/:slug/chapters
PATCH  /api/:slug/chapters/:id          DELETE /api/:slug/chapters/:id
POST   /api/:slug/chapters/reorder
POST   /api/:slug/chapters/:id/pages    PATCH  /api/:slug/pages/:id
DELETE /api/:slug/pages/:id             PUT    /api/:slug/pages/:id/slots

POST   /api/:slug/export                GET    /api/:slug/export/status
GET    /api/:slug/intake/qr             # PNG 1024px + ?format=svg untuk cetak
POST   /api/:slug/intake/pull           POST   /api/:slug/intake/close
```

---

## 12. Milestone

Commit di akhir tiap milestone: `feat(Mx): ...`. **Jangan tandai selesai kalau test merah atau fitur separuh.**

### M0 — Fondasi
Pindahkan 5 prototipe ke `prototypes/`. Scaffold `studio/server` + `studio/web`. SQLite + migration runner. `GET /api/health`. `README.md` + `docs/decisions.md`.
**Selesai kalau:** dari clone bersih, `npm install && npm start` jalan tanpa setup manual, `/api/health` balas `{ok:true}`, `npm test` hijau.

### M1 — Project & klien
CRUD project. Layar buat project baru mengikuti Album Setup di `dwa-r2-s1.html` (nama klien, pengantin, tanggal, venue, adat, style). Daftar project di layar awal.
**Selesai kalau:** buat 3 project, tutup app, buka lagi — ketiganya masih ada dengan folder `data/projects/<slug>/` lengkap.

### M2 — Ingest foto & video
Pipeline §6 lengkap: salin, EXIF, varian, transcode film, worker pool, SSE progress, resumable, laporan gagal.
**Selesai kalau:** drop **500 foto + 1 video 3 menit** → semua `ready` tanpa memblokir UI, progress jalan real-time, matikan proses di tengah lalu jalankan lagi → melanjutkan bukan mengulang, dan `originals/` byte-identical dengan file sumber.

### M3 — Kurasi
`quality.js`, `dedup.js`, `curate.js`, `layout.js`, `scorers/` + endpoint. Test dengan fixture.
**Selesai kalau:** 500 foto mentah → `POST /curate` → keluar draft 8 chapter / 12 halaman yang urut kronologis, tanpa foto blur, tanpa duplikat dalam satu halaman. Waktu proses < 90 detik.

### M4 — Preview & editor
Layar utama sesuai `dwa-r2-s2.html`. Live preview memakai template export yang sama. Sembunyikan/ganti/geser foto, ganti layout, undo-redo 30 langkah, autosave. Pencatatan `decisions` di latar belakang, tanpa UI.
**Selesai kalau:** operator bisa menyusun album 12 halaman sepenuhnya lewat UI tanpa menyentuh database, undo mengembalikan 30 langkah dengan benar, refresh browser tidak kehilangan apa pun — **dan** setelah sesi itu tabel `decisions` berisi baris `swap` yang `rejected_json`-nya **tidak kosong** serta `snapshot_json` memuat skor kandidat terpilih maupun yang ditolak.

### M5 — Export statis
Builder §9. Portabilitas §3 dipenuhi.
**Selesai kalau:** hasil export dibuka **tiga cara** — `file://` langsung, `http://localhost:8080/`, dan `http://localhost:8080/sub/folder/dalam/` — ketiganya tampil identik tanpa mengubah satu baris pun. Lighthouse mobile performance ≥ 90.

### M6 — Guest Intake
Service `intake/` + Docker. 5 layar sesuai `dwa-r2-s4.html`. QR generator. Rate limit. Jendela waktu. Pull incremental. Chapter Guest Memories dengan aturan kurasi khusus (§7).
**Selesai kalau:** tiga HP berbeda scan QR → masing-masing upload 3 foto + tulis pesan → `POST /intake/pull` → 9 foto masuk sebagai `source='guest'`, chapter Guest Memories terbentuk sendiri dengan **ketiga tamu terwakili**, pesan tampil sebagai kartu, dan pull kedua tidak menggandakan apa pun.

### M7 — Serah terima
`npm run deploy` (SFTP atau folder). `npm run backup` (tar project + db, bernama tanggal). README lengkap dari nol. `docs/decisions.md` terisi.
**Selesai kalau:** dari komputer bersih — clone, install, start, buat project, ingest, curate, edit, export, deploy — seluruhnya jalan hanya dengan mengikuti README, tanpa perlu bertanya.

---

## 13. Aturan kerja

**Lakukan:**
- Baca file prototipe yang relevan **sebelum** membuat komponen. Ambil warna, ukuran, teks, struktur dari sana.
- Satu template album, dipakai preview **dan** export. Jangan pernah menulis dua.
- Test untuk setiap fungsi di `ai/` dan `pipeline/`, dan setiap route yang mengubah data.
- Validasi input dengan `zod` di middleware, bukan di dalam handler.
- Query SQLite parameterized. Tidak ada string concatenation ke SQL.
- Pesan ke operator dalam Bahasa Indonesia; log internal boleh Inggris.
- Semua path lewat `paths.js`. Jangan tebar string path di mana-mana.
- Commit kecil, satu perubahan logis per commit.
- Keputusan ambigu → tulis di `docs/decisions.md`, lalu lanjut. Jangan berhenti menunggu jawaban.

**Jangan:**
- Jangan menyentuh isi `originals/` setelah ingest. Sekali tulis, sesudah itu read-only.
- Jangan menambah dependency di luar yang disebut tanpa alasan tertulis di `docs/decisions.md`.
- Jangan mengubah desain visual prototipe "supaya lebih bagus".
- Jangan pakai gradient CSS sebagai pengganti foto di produk jadi — itu hanya untuk prototipe.
- Jangan menaruh path absolut atau base URL di hasil export.
- Jangan menaruh secret di kode atau di commit. `.env` masuk `.gitignore`, sediakan `.env.example`.
- Jangan menghapus file media milik operator tanpa konfirmasi eksplisit.
- Jangan menamai fungsi atau komentar seolah ada model AI terlatih. Sebut heuristik apa adanya.

---

## 14. Selesai kalau

Operator bisa, sendirian, tanpa membuka terminal selain `npm start`:

1. Membuat project baru untuk klien A
2. Menyeret 500 foto + 1 wedding film, menunggu, dan semuanya terproses
3. Menekan satu tombol dan mendapat draft album yang masuk akal secara kronologis
4. Merapikan di preview — membuang yang jelek, menukar, menggeser, mengganti layout
5. Mencetak QR agar tamu menyumbang foto, lalu menariknya masuk
6. Export dan mendapat satu folder yang bisa ditaruh di hosting mana pun
7. Mengulang semuanya untuk klien B tanpa yang lama terganggu

---

**Mulai dari §0, lalu M0.**
