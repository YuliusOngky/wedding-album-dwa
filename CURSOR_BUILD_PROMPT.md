# Cursor Build Prompt — DWA Album Factory

> **Cara pakai:** buka folder repo ini di Cursor → Composer mode Agent → paste seluruh isi file ini.
> Kerjakan **milestone berurutan M0 → M7**. Jangan lompat. Tiap milestone: bangun → jalankan acceptance test → commit → lapor → lanjut.

---

## 1. Apa yang dibangun

**Bukan** SaaS tempat pengantin mendaftar sendiri. Ini **pabrik album** — alat produksi internal yang dijalankan oleh satu operator (pemilik studio).

Alur kerjanya:

```
1. Klien datang menyerahkan foto & video
2. Operator buat project baru, isi data klien
3. Operator masukkan semua foto + 1 wedding film
4. Sistem menyeleksi & menyusun otomatis → tampil di preview
5. Operator merapikan di preview: buang foto jelek, tukar, geser, ganti layout
6. Export → album statis siap deploy
7. Deploy ke mana, ditentukan per klien
```

Ditambah satu komponen kecil terpisah: **Guest Intake** — halaman QR tempat tamu undangan menyumbang foto saat hari-H. Hasilnya ditarik masuk ke studio.

---

## 2. Tiga komponen, beda tempat jalan

| Komponen | Jalan di mana | Sifat | Ukuran kerja |
|---|---|---|---|
| **Studio** | `localhost` di komputer operator | Berat: ingest, AI, preview, editor, export | ~75% dari pekerjaan |
| **Guest Intake** | Server online (NAS), aktif ~1 minggu seputar hari-H | Sangat tipis: QR landing + terima upload. Tidak ada editor. | ~15% |
| **Album terbit** | Statis, deploy per klien | Tidak ada server. HTML + gambar. | ~10% |

**Prinsip yang mengikat semuanya:** yang online seminimal mungkin. Semua pekerjaan berat terjadi di localhost, di mana tidak ada biaya bandwidth, tidak ada batas waktu request, dan tidak ada yang bisa jebol.

---

## 3. Aset yang sudah ada

Lima prototipe HTML di root repo. **Ini sumber kebenaran desain — ikuti, jangan didesain ulang.**

| File | Isi | Dipakai untuk |
|---|---|---|
| `dwa-prototype.html` | Style guide, token, komponen | Ambil `:root` → jadi `tokens.css` |
| `dwa-r2-s1.html` | Creation Flow 6 layar | Alur buat project baru di Studio |
| `dwa-r2-s2.html` | Desktop Album Editor | **Layar utama Studio** — chapter list, live preview, panel kanan |
| `dwa-r2-s3.html` | Mobile Album Viewer 9 halaman | Template album terbit + isi live preview |
| `dwa-r2-s4.html` | Guest Experience 5 layar | Guest Intake |

Token warna (`dwa-prototype.html`, blok `:root`):
```
--ink:#0E0E10  --espresso:#1C1915  --stone:#6B5E52  --muted:#9C8C80
--champagne:#E8DCD0  --ivory:#F7F2EA  --gold:#C9A96E  --gold-dk:#A08848
```
Font: **Cormorant Garamond** (display) + **DM Sans** (UI/body), Google Fonts.
Bahasa UI: **Indonesia**. Konteks: pernikahan adat Indonesia — Akad Nikah, Siraman, Midodareni, Panggih, Resepsi dipakai apa adanya.

---

## 4. Keputusan terkunci

Keputusan ini **menggantikan** versi spec sebelumnya. Jangan diubah tanpa menuliskan alasannya di `docs/decisions.md`.

| Aspek | Keputusan | Kenapa |
|---|---|---|
| Studio runtime | **Node.js 20 + Express**, buka di browser `http://localhost:4000` | Bukan Electron. Lebih sederhana, tidak perlu bundling desktop. |
| Database Studio | **SQLite** via `better-sqlite3` | Aplikasi satu operator di satu mesin. PostgreSQL + Docker jadi beban tanpa manfaat. Satu file `.db`, gampang di-backup. |
| Frontend | **React 18 + Vite**, JavaScript (bukan TS), plain CSS + CSS variables | Token sudah ada, Tailwind cuma menambah lapisan. |
| Auth Studio | **Tidak ada.** Bind ke `127.0.0.1` saja. | Localhost, satu operator. Login = keamanan teater. |
| Image | `sharp` | — |
| Video | `ffmpeg` via `ffmpeg-static` + spawn langsung | Jangan `fluent-ffmpeg`, argumen eksplisit lebih mudah di-debug. |
| Guest Intake | Node + Express + SQLite, container Docker kecil di NAS | Harus online, tapi permukaannya sengaja dibuat sempit. |
| Album terbit | **Statis murni** — HTML + CSS + JS + gambar. Tanpa server, tanpa build step saat runtime. | Deploy ke mana saja, hidup selamanya, tidak bisa mati karena NAS mati. |
| Testing | `node:test` + `supertest` (server), Vitest (util frontend) | — |

### Aturan portabilitas export — wajib

Album hasil export **harus jalan di path apa pun tanpa dikonfigurasi ulang**, karena tujuan deploy ditentukan belakangan per klien:

- Semua path di HTML/CSS/JS **relatif** (`./media/...`), tidak pernah diawali `/`
- Tidak ada base URL yang di-hardcode
- Tidak ada request ke domain luar kecuali Google Fonts — dan sediakan fallback font lokal
- Buka `index.html` langsung lewat `file://` harus tetap menampilkan album (boleh tanpa fitur share)

Ini berarti bundle yang sama bisa ditaruh di `optimadigitalselaras.com/album/nama/`, di subdomain, di hosting klien, atau di flashdisk — tanpa build ulang.

---

## 5. Struktur folder

```
wedding-album-dwa/
├─ prototypes/               # 5 HTML lama dipindah ke sini. Arsip. Jangan diubah.
├─ studio/
│  ├─ server/
│  │  ├─ src/
│  │  │  ├─ index.js         # express, bind 127.0.0.1:4000
│  │  │  ├─ db.js            # better-sqlite3 + migration runner
│  │  │  ├─ paths.js         # semua path project ada di sini, jangan tebar string path
│  │  │  ├─ routes/          # projects, media, chapters, pages, curate, export, intake
│  │  │  ├─ pipeline/        # ingest.js, image.js, video.js, worker.js
│  │  │  ├─ ai/              # quality.js, dedup.js, faces.js, curate.js, layout.js
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

**Aturan `originals/`:** file asli klien hanya ditulis sekali saat ingest, sesudah itu read-only. Semua operasi bekerja di `derived/`. Kalau operator salah langkah, foto asli tidak pernah hilang.

---

## 6. Skema database (SQLite)

Satu file `.db` per project di `data/projects/<slug>/project.db`, plus satu `data/studio.db` berisi daftar project.

`studio.db`:
```sql
projects(id, slug UNIQUE, client_name, bride_name, groom_name,
         event_date, venue, culture, style, status,
         created_at, updated_at, last_opened_at)
-- culture: 'jawa'|'sunda'|'bali'|'minang'|'batak'|'umum'
-- style:   'editorial'|'cinematic'|'romantic'|'minimal'
-- status:  'draft'|'curating'|'ready'|'exported'
```

`project.db`:
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

guests(id, name, relation, message, created_at);
-- relation: 'keluarga_pria'|'keluarga_wanita'|'teman_kuliah'|'teman_kerja'|'tetangga'|'lainnya'

settings(key PRIMARY KEY, value);
-- wedding_film_media_id, cover_media_id, intake_token, intake_url, dll
```

Aturan:
- Operator **tidak pernah menghapus permanen** dari preview — set `hidden=1`. Ada tombol terpisah "Hapus permanen" dengan konfirmasi, dan itu pun tidak menyentuh `originals/`.
- Semua akses `media_id` di `slots` harus `ON DELETE SET NULL`, jangan cascade — menghapus foto tidak boleh merusak struktur halaman.

---

## 7. Pipeline ingest

Ini bagian paling menentukan apakah alat ini enak dipakai. Operator akan menyeret 300–800 file sekaligus.

### Foto
1. Salin ke `originals/` dengan nama asli dipertahankan (tambah suffix kalau bentrok)
2. Baca EXIF: `taken_at`, orientasi, dimensi
3. Auto-rotate sesuai EXIF, lalu buat varian ke `derived/`:
   - `thumb` 400px lebar, WebP q75
   - `preview` 1200px, WebP q82
   - `full` 2400px, WebP q86 + JPEG fallback q88
4. Hitung `sharpness`, `brightness`, `face_count`, `phash`, `quality_score` (§8)
5. `status` → `ready`

### Video (satu wedding film per album)
1. Salin ke `originals/`
2. Probe dengan `ffprobe`: durasi, resolusi, codec
3. Transcode ke `derived/`:
   - `film-1080.mp4` — H.264 High, CRF 21, AAC 128k, `-movflags +faststart`
   - `film-720.mp4` — H.264 Main, CRF 23, untuk koneksi lemot
   - `poster.jpg` — frame di detik ke-3 (atau 10% durasi kalau video pendek)
4. Simpan `wedding_film_media_id` di `settings`

### Aturan eksekusi
- Jalankan di **worker thread pool**, jumlah worker = `cpus().length - 1`, minimal 1
- Progress real-time ke UI via **SSE** (`GET /api/ingest/stream`), bukan polling
- **Bisa dilanjutkan**: kalau proses mati di tengah, jalankan lagi → lewati yang sudah `ready`, ulangi yang `ingesting`/`failed`
- File yang gagal → `status='failed'` + `error_message`, **jangan hentikan batch**. Tampilkan daftar gagal di akhir.
- Format diterima: `jpg jpeg png heic heif webp tif tiff` (foto), `mp4 mov mkv avi` (video). HEIC dari iPhone **wajib** jalan — pakai `sharp` dengan libheif, kalau environment tidak mendukung, fallback ke ffmpeg.

---

## 8. Modul AI (heuristik lokal, tanpa LLM)

Semua di `studio/server/src/ai/`. Karena jalan di localhost, tidak ada batas waktu request dan tidak ada biaya API — boleh lebih teliti daripada kalau di server.

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
Fotografer pernikahan menembak burst — ini akan memangkas 30–50% file. Wajib benar.

### `curate.js`
Input: semua media `ready` & tidak `hidden`. Output: draft chapter + page + slot.
- Buang duplikat (sisakan wakil), buang `quality_score < 35`
- Kelompokkan per **momen** pakai `taken_at`, jeda > 20 menit = momen baru → album jadi kronologis
- Maksimal 3 foto per momen, jaga keragaman
- Foto dengan `face_count == 2` diprioritaskan untuk cover & chapter story
- Petakan momen → chapter sesuai `culture`. Jawa: Siraman → Midodareni → Akad → Panggih → Resepsi
- Kalau `taken_at` kosong di sebagian besar file, fallback ke urutan nama file

**Foto tamu (`source='guest'`) memakai aturan berbeda — jangan disatukan:**

| | Foto klien (fotografer) | Foto tamu |
|---|---|---|
| Ambang kualitas | `quality_score < 35` dibuang | `< 20` saja yang dibuang |
| Dedup | Hamming ≤ 8, agresif | Hanya **dalam satu tamu**. Antar tamu jangan di-dedup. |
| Kuota | Dikurasi ketat | Setiap tamu **dijamin minimal 1 foto tampil** |
| Masuk chapter | Sesuai momen | Selalu ke chapter `kind='guest'` |

Alasannya bukan teknis. Foto tamu diambil pakai HP di ruangan remang — kalau dipakai ambang yang sama dengan foto fotografer, hampir semua terbuang. Dan dua tamu yang memotret momen sama dari sudut berbeda itu **bukan duplikat**, itu dua orang yang sama-sama hadir. Tujuan chapter ini keterwakilan, bukan kurasi.

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

> **Sebut apa adanya di kode.** Ini heuristik, bukan model terlatih. Label "AI" di UI boleh dipertahankan sesuai prototipe, tapi nama fungsi, komentar, dan `docs/` harus jujur.

### Titik ekstensi untuk model vision/LLM nanti

Model vision **tidak dibangun sekarang**, tapi tempatnya disiapkan sekarang — karena menyisipkannya belakangan tanpa persiapan berarti membongkar `curate.js`.

Cukup tiga hal, jangan lebih:

**1. Satu interface scorer.** Semua penilai foto memenuhi kontrak yang sama:

```js
// ai/scorers/index.js
// score(media, filePath) → { scores: {...}, tags: [...] }
//   scores: angka 0..1 per dimensi, bebas kuncinya
//   tags:   label bebas, contoh ['pelaminan','grup','outdoor']
export const heuristicScorer = { name: 'heuristic', version: 1, score }
```
Sekarang cuma ada `heuristicScorer`. Nanti tinggal tambah `visionScorer` dan gabungkan hasilnya — tanpa menyentuh `curate.js`.

**2. Dua kolom disiapkan sejak migration pertama**, biarkan NULL:
```sql
ALTER TABLE media ADD COLUMN tags_json TEXT;      -- hasil tagging, NULL sekarang
ALTER TABLE media ADD COLUMN embedding BLOB;      -- vektor, NULL sekarang
ALTER TABLE media ADD COLUMN scorer_version TEXT; -- penanda siapa yang menilai
```
Menambah kolom ke SQLite yang sudah berisi ratusan project itu merepotkan. Membiarkannya NULL sekarang tidak berbiaya apa pun.

**3. `curate.js` membaca `scores` dan `tags`, bukan `sharpness` langsung.** Kalau `curate.js` menyebut `media.sharpness` di mana-mana, dia terikat ke heuristik selamanya.

**Berhenti di situ.** Jangan bikin plugin system, registry, atau abstraksi berlapis untuk sesuatu yang belum ada. Satu interface, tiga kolom, satu aturan baca — itu cukup untuk menyisipkan model nanti, dan tidak cukup besar untuk jadi beban sekarang.

---

## 9. Preview & editor — jantung aplikasi

Ikuti `dwa-r2-s2.html`. Ini layar yang paling lama dipakai operator, jadi paling menentukan.

- **Kiri**: daftar chapter, drag-reorder, indikator progres
- **Tengah**: live mobile preview — render **template album terbit yang sesungguhnya** di dalam frame HP, bukan tiruan. Satu template, dipakai preview dan export. Kalau berbeda, operator akan tertipu.
- **Kanan**: tab AI Tools / Layouts / Photos / Story

Yang wajib ada di preview (poin 5 konsep):
- Klik foto di preview → langsung terpilih di panel kanan
- **Sembunyikan** foto (satu klik, bisa di-undo) dan **ganti** foto dari grid kandidat
- Geser posisi foto antar slot dengan drag
- Ganti layout halaman, tambah/hapus halaman
- **Undo/redo minimal 30 langkah** — operator bekerja cepat dan akan salah klik. Simpan patch di memori.
- Autosave debounce 800 ms + indikator "Semua perubahan tersimpan" seperti prototipe

Grid kandidat harus bisa disortir: skor kualitas, waktu, jumlah wajah, sumber (klien/tamu), dan filter "hanya yang belum dipakai".

---

## 10. Export

`POST /api/export` → menghasilkan `data/projects/<slug>/export/`:

```
export/
├─ index.html          # self-contained, path relatif semua
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
- Video pakai `preload="none"` dengan poster — jangan pernah autoload
- **Target: LCP < 2,5 detik di Slow 4G**, dan total bundle album 12 halaman < 25 MB
- Sediakan `npm run export -- --slug=<slug> --zip` untuk menghasilkan `.zip` sekalian

---

## 11. Guest Intake

Service terpisah di `intake/`, container Docker di NAS. **Tipis sengaja** — tidak ada editor, tidak ada preview, tidak menyimpan apa pun selain foto tamu dan identitasnya.

Layar mengikuti `dwa-r2-s4.html`: QR Landing → Upload → Nama & Relasi → Uploading → Terima Kasih.

- URL per album: `https://<host>/i/<slug>`, dibuka dari QR
- **Tanpa login.** Identitas tamu = baris `guests` + token di `sessionStorage`
- Batas: maks **20 foto per tamu**, **15 MB per file**, hanya `image/jpeg|png|heic|webp`
- Rate limit **30 request/menit per IP**
- Jendela waktu: `opens_at` / `closes_at` per album. Di luar jendela → halaman sopan "Pengumpulan foto sudah ditutup"
- Tolakan harus berpesan Indonesia yang jelas, bukan kode error

### Sinkronisasi ke Studio
```
GET  /api/intake/pull        # Studio menarik foto tamu baru dari intake
POST /api/intake/close       # Studio menutup jendela dari jarak jauh
```
Studio menyimpan `intake_url` + `intake_token` di `settings`. Tarikan bersifat **incremental** (berdasarkan `created_at` terakhir) dan **idempoten** — menarik dua kali tidak menggandakan foto. Setelah masuk, foto tamu diproses pipeline yang sama dengan `source='guest'`.

Setelah album diexport dan diserahkan, container intake untuk album itu **boleh dimatikan**. Tidak ada yang hilang — semua sudah ada di Studio.

### Chapter "Guest Memories"

Chapter `kind='guest'` adalah tujuan akhir foto tamu, dan tampilannya **beda dari chapter lain**. Ikuti halaman 9 `dwa-r2-s3.html`: mosaik 3 kolom + kartu pesan.

Layout khusus, hanya untuk chapter ini:
```
'mosaic-masonry'   → mosaik rapat, tinggi bervariasi, 15-40 foto per halaman
'message-cards'    → kutipan pesan tamu + nama & relasi
'mosaic-mixed'     → mosaik diselingi 2-3 kartu pesan
```

Ketentuan:
- Setiap foto membawa atribusi: **nama tamu + relasi** (contoh: "Budi Santoso · Keluarga Ananda"), tampil saat di-tap/hover
- `guests.message` yang tidak kosong menjadi kartu pesan. Potong di 180 karakter dengan elipsis.
- Urutkan **per tamu**, bukan per waktu — supaya kiriman satu orang berkelompok
- Chapter ini boleh **beberapa halaman** kalau tamu banyak. Pecah otomatis tiap ~30 foto.
- Kalau tidak ada satu pun foto tamu, chapter ini **tidak dibuat** — jangan tampilkan halaman kosong

Operator tetap bisa menyembunyikan foto tamu satu per satu dari preview (moderasi). Foto yang disembunyikan tidak ikut export.

---

## 12. API Studio

Prefix `/api`. Tanpa auth (localhost). Error seragam:
```json
{ "error": { "code": "PROJECT_NOT_FOUND", "message": "Project tidak ditemukan." } }
```

```
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
POST   /api/:slug/intake/pull
```

---

## 13. Milestone

Commit di akhir tiap milestone: `feat(Mx): ...`. **Jangan tandai selesai kalau test merah atau fitur separuh.**

### M0 — Fondasi
Pindahkan 5 prototipe ke `prototypes/`. Scaffold `studio/server` + `studio/web`. SQLite + migration runner. `GET /api/health`. `npm start` → buka `localhost:4000`.
**Selesai kalau:** `npm start` jalan dari clone bersih tanpa setup manual, `/api/health` balas `{ok:true}`, `npm test` hijau.

### M1 — Project & klien
CRUD project. Layar buat project baru mengikuti Album Setup di `dwa-r2-s1.html` (nama klien, pengantin, tanggal, venue, adat, style). Daftar project di layar awal.
**Selesai kalau:** buat 3 project, tutup app, buka lagi, ketiganya masih ada dengan folder `data/projects/<slug>/` lengkap.

### M2 — Ingest foto & video
Pipeline §7 lengkap: salin, EXIF, varian, transcode film, worker pool, SSE progress, resumable, laporan gagal.
**Selesai kalau:** drop **500 foto + 1 video 3 menit** → semua `ready` tanpa memblokir UI, progress jalan real-time, matikan proses di tengah lalu jalankan lagi → lanjut bukan mengulang, dan `originals/` byte-identical dengan file sumber.

### M3 — AI kurasi
`quality.js`, `dedup.js`, `curate.js`, `layout.js` + endpoint. Test dengan fixture.
**Selesai kalau:** 500 foto mentah → `POST /curate` → keluar draft 8 chapter / 12 halaman yang urut kronologis, tanpa foto blur, tanpa duplikat dalam satu halaman. Waktu proses < 90 detik.

### M4 — Preview & editor
Layar utama sesuai `dwa-r2-s2.html`. Live preview memakai template export yang sama. Sembunyikan/ganti/geser foto, ganti layout, undo-redo 30 langkah, autosave.
**Selesai kalau:** operator bisa menyusun album 12 halaman sepenuhnya lewat UI tanpa menyentuh database, undo mengembalikan 30 langkah dengan benar, dan refresh browser tidak kehilangan apa pun.

### M5 — Export statis
Builder §10. Portabilitas §4 dipenuhi.
**Selesai kalau:** hasil export dibuka **tiga cara** — `file://` langsung, dari `http://localhost:8080/`, dan dari `http://localhost:8080/sub/folder/dalam/` — ketiganya tampil identik tanpa mengubah satu baris pun. Lighthouse mobile performance ≥ 90.

### M6 — Guest Intake
Service `intake/` + Docker. 5 layar sesuai `dwa-r2-s4.html`. QR generator. Rate limit. Jendela waktu. Pull incremental ke Studio.
Ditambah chapter Guest Memories: layout mosaik + kartu pesan, atribusi nama & relasi, pecah otomatis tiap ~30 foto, aturan kurasi khusus foto tamu (§8).
**Selesai kalau:** tiga HP berbeda scan QR → masing-masing upload 3 foto + tulis pesan → `POST /intake/pull` di Studio → 9 foto masuk sebagai `source='guest'`, chapter Guest Memories terbentuk sendiri dengan **ketiga tamu terwakili**, pesan mereka tampil sebagai kartu, dan pull kedua kalinya tidak menggandakan apa pun.

### M7 — Serah terima
Script deploy album ke target (NAS via SFTP, atau folder mana pun). `npm run backup` (tar project + db). README lengkap dari nol. `docs/decisions.md`.
**Selesai kalau:** dari komputer bersih — clone repo, `npm install`, `npm start`, buat project, ingest foto, curate, edit, export, deploy — seluruhnya jalan mengikuti README tanpa perlu bertanya.

---

## 14. Aturan kerja

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
- Jangan menambah dependency di luar yang disebut tanpa alasan tertulis.
- Jangan mengubah desain visual prototipe "supaya lebih bagus".
- Jangan pakai gradient CSS sebagai pengganti foto di produk jadi — itu hanya untuk prototipe.
- Jangan menaruh path absolut atau base URL di hasil export.
- Jangan menaruh secret di kode atau di commit. `.env` masuk `.gitignore`, sediakan `.env.example`.
- Jangan menghapus file media milik operator tanpa konfirmasi eksplisit.

---

## 15. Selesai kalau

Operator bisa, sendirian, tanpa membuka terminal selain `npm start`:

1. Membuat project baru untuk klien A
2. Menyeret 500 foto + 1 wedding film, menunggu, dan semuanya terproses
3. Menekan satu tombol dan mendapat draft album yang masuk akal secara kronologis
4. Merapikan di preview — membuang yang jelek, menukar, menggeser, mengganti layout
5. Mencetak QR agar tamu menyumbang foto, lalu menariknya masuk
6. Export dan mendapat satu folder yang bisa ditaruh di hosting mana pun
7. Mengulang semuanya untuk klien B tanpa yang lama terganggu

---

**Mulai dari M0.** Setiap milestone selesai, laporkan singkat: apa yang dibangun, hasil acceptance test, dan apa yang perlu diputuskan sebelum lanjut.
