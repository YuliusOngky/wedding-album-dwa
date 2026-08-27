# Cursor Build Prompt — Digital Wedding Album (DWA)

> **Cara pakai:** buka folder repo di Cursor, lalu paste isi file ini ke Composer/Agent (mode Agent, model paling kuat).
> Kerjakan **milestone demi milestone**. Jangan lompat. Setiap milestone selesai → jalankan acceptance test → commit → lanjut.

---

## 1. Konteks

Saya punya **prototipe UI statis** untuk produk "Digital Wedding Album" — album pernikahan digital yang bisa dibuat pengantin, lalu tamu undangan ikut menyumbang foto lewat QR code.

Prototipe ini **hanya mockup click-through**: tidak ada backend, tidak ada upload beneran, tidak ada database. Semua foto adalah CSS gradient, semua tombol hanya mengganti layar.

**Tugasmu: bangun aplikasi sungguhan dari prototipe ini, sampai bisa dipakai membuat album asli.**

Prototipe adalah **sumber kebenaran untuk desain** — warna, tipografi, spacing, alur layar, dan copywriting harus diikuti. Jangan mendesain ulang.

---

## 2. Aset yang sudah ada

Semua ada di root repo ini:

| File | Isi | Pakai sebagai |
|---|---|---|
| `dwa-prototype.html` | Style guide + 10 layar landing/onboarding | **Design token & komponen** |
| `dwa-r2-s1.html` | Creation Flow (6 layar): landing → setup → culture → upload → processing → ready | Alur onboarding pengantin |
| `dwa-r2-s2.html` | Desktop Album Editor: chapter list, live mobile preview, panel AI/Layout/Photos/Story | Layout editor |
| `dwa-r2-s3.html` | Mobile Album Viewer (9 halaman: cover → our story → journey → akad → tradisi → keluarga → resepsi → film → guest memories) | Tampilan album jadi |
| `dwa-r2-s4.html` | Guest Experience (5 layar): QR landing → upload → nama & relasi → uploading → thank you | Alur tamu |
| `index.html` | Landing index prototipe | — |

**Design token** (ambil dari `dwa-prototype.html`, blok `:root`):

```
--ink:#0E0E10  --espresso:#1C1915  --stone:#6B5E52  --muted:#9C8C80
--champagne:#E8DCD0  --ivory:#F7F2EA  --gold:#C9A96E  --gold-dk:#A08848
```

Font: **Cormorant Garamond** (display/serif) + **DM Sans** (body/UI). Keduanya dari Google Fonts.

Bahasa UI produk: **Bahasa Indonesia**. Konteks budaya: pernikahan adat Indonesia (Jawa, Sunda, Bali, Minang, Batak) — istilah seperti Akad Nikah, Siraman, Midodareni, Panggih, Resepsi dipakai apa adanya.

---

## 3. Keputusan yang sudah terkunci — jangan diubah tanpa tanya

| Aspek | Keputusan |
|---|---|
| Backend | **Node.js 20 + Express** (bukan Nest, bukan Fastify) |
| Database | **PostgreSQL 16**, akses via `pg` + query SQL eksplisit. Boleh pakai `node-pg-migrate` untuk migration. **Jangan pakai ORM berat** (Prisma/TypeORM/Sequelize). |
| Frontend | **React 18 + Vite**, JavaScript (bukan TypeScript), React Router, plain CSS + CSS variables (bukan Tailwind — token sudah ada di prototipe) |
| Auth | **Email + password**, hash `argon2`, sesi via **JWT di httpOnly cookie** (SameSite=Lax, 30 hari) |
| Storage | Filesystem lokal di volume Docker NAS (`/data/media`). Abstraksi lewat satu modul `storage.js` supaya nanti gampang pindah ke S3/R2. |
| AI | **AI dasar tanpa LLM** — lihat §7. Tidak ada panggilan ke OpenAI/Claude/Gemini di MVP ini. |
| Deploy | **Docker Compose di NAS** (Windows host, 192.168.1.20), di belakang nginx yang sudah ada |
| Testing | `node:test` + `supertest` untuk API. Vitest untuk util frontend. Tidak perlu E2E dulu. |

---

## 4. Arsitektur

```
wedding-album-dwa/
├─ prototypes/              # 5 file HTML lama dipindah ke sini (arsip, jangan diubah)
├─ server/
│  ├─ src/
│  │  ├─ index.js           # entry, express app
│  │  ├─ db.js              # pool pg + helper query
│  │  ├─ config.js          # baca env, validasi saat boot
│  │  ├─ storage.js         # put/get/delete file — satu-satunya yang sentuh disk
│  │  ├─ middleware/        # auth.js, error.js, ratelimit.js, validate.js
│  │  ├─ routes/            # auth, albums, chapters, pages, photos, guests, public
│  │  ├─ services/          # album.js, photo.js, qr.js
│  │  └─ ai/                # quality.js, dedup.js, faces.js, curate.js, layout.js
│  ├─ migrations/
│  └─ test/
├─ web/                     # React app (pengantin: onboarding + editor)
│  ├─ src/
│  │  ├─ styles/tokens.css  # SALIN PERSIS dari dwa-prototype.html
│  │  ├─ components/
│  │  ├─ pages/
│  │  └─ api/client.js
│  └─ vite.config.js
├─ guest/                   # React app terpisah, bundle kecil (tamu: QR upload)
├─ docker/
│  ├─ docker-compose.yml
│  ├─ Dockerfile.server
│  └─ nginx-wedding.conf
└─ docs/
```

**Kenapa `guest/` terpisah:** tamu membuka lewat HP di kondangan, sering sinyal jelek. Bundle harus < 150 KB gzip dan tidak ikut membawa kode editor.

---

## 5. Skema database

Buat sebagai migration bernomor. Semua id `uuid` (`gen_random_uuid()`, aktifkan `pgcrypto`). Semua tabel punya `created_at timestamptz default now()`.

```sql
-- pemilik album
users(id, email unique citext, password_hash, name, created_at)

-- album
albums(
  id, owner_id → users,
  slug unique,                    -- "yulius-ananda", dipakai di URL publik
  bride_name, groom_name,
  event_date date,
  venue,
  culture,                        -- 'jawa'|'sunda'|'bali'|'minang'|'batak'|'umum'
  style,                          -- 'editorial'|'cinematic'|'romantic'|'minimal'
  cover_photo_id → photos null,
  status,                         -- 'draft'|'published'
  guest_upload_enabled bool default true,
  guest_upload_closes_at timestamptz null,
  privacy,                        -- 'public'|'unlisted'|'password'
  view_password_hash null,
  published_at, created_at, updated_at
)

-- struktur album
chapters(id, album_id → albums, idx int, title, subtitle, kind, created_at)
  -- kind: 'cover'|'story'|'journey'|'ceremony'|'tradition'|'family'|'reception'|'film'|'guest'
  -- unique(album_id, idx)

pages(id, chapter_id → chapters, idx int, layout_key, caption text, created_at)
  -- layout_key contoh: 'full-bleed'|'duo'|'grid-2x2'|'hero-plus-2'|'quote'|'mosaic-3'
  -- unique(chapter_id, idx)

page_slots(id, page_id → pages, slot_idx int, photo_id → photos null, crop jsonb)
  -- unique(page_id, slot_idx)

-- media
photos(
  id, album_id → albums,
  source,                         -- 'owner'|'guest'
  guest_id → guests null,
  storage_key,                    -- path relatif di storage
  original_name, mime, bytes, width, height,
  phash char(16),                 -- perceptual hash untuk dedup
  sharpness real,                 -- variance of Laplacian
  brightness real,
  face_count int,
  quality_score real,             -- 0..100, hasil ai/quality.js
  status,                         -- 'processing'|'ready'|'rejected'
  taken_at timestamptz null,      -- dari EXIF
  created_at
)
  -- index (album_id, quality_score desc), index (album_id, phash)

-- tamu
guests(id, album_id → albums, name, relation, message text null, ip_hash, created_at)
  -- relation: 'keluarga_pengantin_pria'|'keluarga_pengantin_wanita'|'teman_kuliah'|
  --           'teman_kerja'|'tetangga'|'lainnya'

-- audit ringan
album_views(id, album_id, viewed_at, ip_hash)
```

Aturan integritas:

- Hapus album → cascade ke chapters, pages, page_slots, photos, guests.
- `photos.storage_key` tidak boleh dihapus dari DB tanpa menghapus filenya (buat satu fungsi `deletePhoto()` yang mengurus keduanya dalam transaksi + cleanup file setelah commit).
- Semua query yang menyentuh album **wajib** difilter `owner_id` dari sesi, kecuali route publik.

---

## 6. API

Prefix `/api`. Semua response JSON. Error format seragam:

```json
{ "error": { "code": "ALBUM_NOT_FOUND", "message": "Album tidak ditemukan." } }
```

### Auth
```
POST   /api/auth/register        { email, password, name }
POST   /api/auth/login           { email, password }
POST   /api/auth/logout
GET    /api/auth/me
```

### Album (butuh sesi)
```
GET    /api/albums
POST   /api/albums               { bride_name, groom_name, event_date, venue, culture, style }
GET    /api/albums/:id
PATCH  /api/albums/:id
DELETE /api/albums/:id
POST   /api/albums/:id/publish
POST   /api/albums/:id/unpublish
GET    /api/albums/:id/qr           → PNG QR code (juga SVG via ?format=svg)
```

### Upload & foto
```
POST   /api/albums/:id/photos/init   { files:[{name,size,mime}] } → { uploads:[{photo_id, upload_url}] }
PUT    /api/upload/:token            → binary body, simpan file
GET    /api/albums/:id/photos        ?source=&status=&sort=quality|date&limit=&cursor=
DELETE /api/photos/:id
GET    /api/media/:photoId/:variant  → 'thumb'|'preview'|'full'  (cache-control panjang, immutable)
```

Upload harus **resumable-friendly sederhana**: init → PUT per file → server memproses async (queue in-process cukup, jangan pasang Redis untuk MVP). Status foto `processing` → `ready`.

### Struktur album
```
GET    /api/albums/:id/chapters
POST   /api/albums/:id/chapters       { title, subtitle, kind, idx }
PATCH  /api/chapters/:id
DELETE /api/chapters/:id
POST   /api/albums/:id/chapters/reorder  { ids: [...] }

POST   /api/chapters/:id/pages        { layout_key, idx }
PATCH  /api/pages/:id                 { layout_key, caption }
DELETE /api/pages/:id
PUT    /api/pages/:id/slots           { slots: [{slot_idx, photo_id, crop}] }
```

### AI dasar
```
POST   /api/albums/:id/curate         → pilih foto terbaik, buat draft chapter+page. Body: { target_pages }
POST   /api/pages/:id/autolayout      → pilih layout_key yang cocok untuk jumlah & orientasi foto
GET    /api/albums/:id/duplicates     → grup foto mirip berdasarkan phash
```

### Publik (tanpa login)
```
GET    /api/public/:slug              → album published (struktur + URL media)
POST   /api/public/:slug/view         → catat view
GET    /api/public/:slug/guest        → cek apakah guest upload masih dibuka
POST   /api/public/:slug/guest        { name, relation, message } → { guest_id, upload_token }
POST   /api/public/:slug/guest/:guestId/photos   → upload foto tamu
```

**Rate limit wajib** pada semua route `/api/public/*`: 30 req/menit per IP, dan maksimal 20 foto per guest per album.

---

## 7. Modul AI dasar (tanpa LLM)

Semua di `server/src/ai/`. Pakai **`sharp`** untuk image processing. Semua jalan di worker thread supaya tidak memblokir event loop.

### `quality.js` — skor kualitas foto
Untuk tiap foto hitung, lalu simpan ke kolomnya:

1. **Sharpness** — variance of Laplacian. Resize ke lebar 512, grayscale, konvolusi kernel Laplacian `[0,1,0, 1,-4,1, 0,1,0]`, hitung variance. Rendah = blur.
2. **Brightness** — rata-rata luminance 0..255. Buang yang < 25 (terlalu gelap) atau > 235 (over-exposed).
3. **Face count** — pakai `@vladmandic/face-api` dengan model TinyFaceDetector (model di-bundle, jangan download saat runtime).

`quality_score` = kombinasi ternormalisasi:
```
score = 45 * norm(sharpness) + 20 * brightnessOk + 25 * min(face_count,3)/3 + 10 * resolutionOk
```
Kalibrasi bobotnya, jangan hardcode buta — tulis test dengan beberapa foto sampel (sertakan foto blur & gelap buatan di `server/test/fixtures/`).

### `dedup.js` — deteksi foto kembar
Perceptual hash **dHash 64-bit**: resize 9×8 grayscale, bandingkan pixel bertetangga horizontal → 64 bit → simpan hex 16 char.
Dua foto dianggap duplikat kalau **Hamming distance ≤ 8**. Kelompokkan, ambil yang `quality_score` tertinggi sebagai wakil.

### `curate.js` — pilih foto untuk album
Input: semua foto `ready`, target jumlah halaman.
Aturan:
- Buang duplikat (sisakan wakil terbaik), buang `quality_score < 35`.
- Kelompokkan per **waktu** (`taken_at`, gap > 20 menit = momen baru) supaya album mengikuti kronologi acara.
- Jaga keragaman: maksimal 3 foto per momen, prioritaskan yang `face_count` 2 (kemungkinan foto berdua pengantin) untuk chapter cover/story.
- Petakan momen → chapter sesuai `albums.culture`. Untuk `jawa`: Siraman → Midodareni → Akad → Panggih → Resepsi.

### `layout.js` — pilih layout halaman
Input: daftar foto untuk satu halaman.
Aturan deterministik berdasarkan jumlah + orientasi (portrait/landscape/square):
```
1 landscape           → 'full-bleed'
1 portrait            → 'hero-plus-caption'
2 apapun              → 'duo'
3 (1 landscape + 2)   → 'hero-plus-2'
4 seragam             → 'grid-2x2'
5-6                   → 'mosaic-3'
```
Tulis sebagai tabel keputusan, bukan if-else bercabang panjang.

> **Batas jujur:** ini heuristik, bukan machine learning. Jangan menamai fungsi/UI seolah ada model AI yang dilatih. Di UI, label "AI" boleh dipertahankan sesuai prototipe, tapi di kode dan komentar sebut apa adanya.

---

## 8. Guest flow

1. Pengantin publish album → sistem generate QR ke `https://optimadigitalselaras.com/wedding_album/a/{slug}`.
2. QR dicetak (kartu meja / standing banner). Endpoint `/api/albums/:id/qr` kembalikan PNG 1024px dan SVG untuk cetak.
3. Tamu scan → app `guest/` terbuka → layar sesuai `dwa-r2-s4.html`:
   - S1 QR Landing → S2 Upload → S3 Nama & Relasi → S4 Uploading → S5 Thank You
4. **Tanpa login.** Identitas tamu = baris di `guests` + token di `sessionStorage`.
5. Foto tamu masuk `photos` dengan `source='guest'`, langsung diproses `quality.js`, tampil di chapter "Guest Memories".
6. Pengantin bisa **moderasi**: sembunyikan/hapus foto tamu dari editor.

Batasan yang wajib diterapkan: maks 20 foto/tamu, maks 15 MB/foto, hanya `image/jpeg|png|heic|webp`, tolak selain itu dengan pesan Indonesia yang jelas.

---

## 9. Editor & Viewer

**Editor** (`web/`, desktop) — ikuti `dwa-r2-s2.html` persis:
- Kiri: daftar chapter, drag untuk reorder, indikator progres per chapter.
- Tengah: **live mobile preview** — render viewer sungguhan di dalam frame HP, bukan gambar statis.
- Kanan: tab AI Tools / Layouts / Photos / Story.
- Autosave dengan debounce 800 ms, indikator "All changes saved" seperti di prototipe.
- Undo/redo minimal 20 langkah (simpan patch di memori, tidak perlu persist).

**Viewer** (`guest/` atau route publik di `web/`) — ikuti `dwa-r2-s3.html`:
- Swipe/keyboard antar halaman, preload halaman berikutnya.
- Gambar `srcset` 3 ukuran, lazy load, `content-visibility:auto`.
- Harus enak dibuka di HP kelas menengah dengan 4G lemot. **Target: LCP < 2.5 s di Slow 4G.**

---

## 10. Deployment

NAS di `192.168.1.20` (host Windows, Docker Desktop). Sudah ada container `optima-web` (nginx) dengan mount:

```
C:/Users/NAS GIOS/websites/optimadigitalselaras  →  /usr/share/nginx/html
```

Situs statis prototipe saat ini dilayani di `optimadigitalselaras.com/wedding_album/`.

Buat `docker/docker-compose.yml` dengan tiga service:

```yaml
services:
  dwa-db:      # postgres:16-alpine, volume ./data/pg
  dwa-server:  # build Dockerfile.server, port internal 3000, volume ./data/media
  # nginx yang sudah ada dipakai sebagai reverse proxy
```

Tambahkan blok proxy ke nginx yang sudah berjalan (`docker/nginx-wedding.conf`, untuk di-include):

```nginx
location /wedding_album/api/ { proxy_pass http://dwa-server:3000/api/; ... }
location /wedding_album/a/   { try_files $uri /wedding_album/guest/index.html; }
location /wedding_album/app/ { try_files $uri /wedding_album/app/index.html; }
```

Build frontend dengan `base: '/wedding_album/app/'` dan `'/wedding_album/guest/'`.

**Env** (`.env.example` wajib dibuat, `.env` masuk `.gitignore`):
```
DATABASE_URL=
JWT_SECRET=
MEDIA_ROOT=/data/media
PUBLIC_BASE_URL=https://optimadigitalselaras.com/wedding_album
MAX_UPLOAD_MB=15
GUEST_MAX_PHOTOS=20
```

Sediakan `npm run deploy` yang: build → upload ke NAS via SFTP → `docker compose up -d --build`. Kredensial dibaca dari env/`.env`, **jangan pernah di-hardcode atau di-commit**.

Backup: satu script `scripts/backup.sh` yang `pg_dump` + tar `/data/media` ke folder bertanggal.

---

## 11. Milestone

Kerjakan berurutan. **Jangan mulai milestone berikutnya sebelum acceptance criteria terpenuhi.** Commit di akhir tiap milestone dengan pesan `feat(Mx): ...`.

### M0 — Fondasi
Pindahkan 5 HTML prototipe ke `prototypes/`. Scaffold `server/`, `web/`, `guest/`. Docker compose jalan lokal. Migration pertama (users, albums). Health check `GET /api/health`.
**Selesai kalau:** `docker compose up` → `/api/health` balas `{ok:true}`, dan `npm test` hijau.

### M1 — Auth + album kosong
Register, login, logout, sesi cookie. CRUD album. Halaman React: login, daftar album, form buat album (ikuti layar Album Setup di `dwa-r2-s1.html`).
**Selesai kalau:** bisa daftar → login → buat album → refresh browser masih login → logout.

### M2 — Upload & pipeline foto
Init upload, PUT file, simpan, generate 3 varian (thumb 400 / preview 1200 / full 2400) via sharp, baca EXIF, hitung sharpness/brightness/phash/face_count, status `processing`→`ready`.
**Selesai kalau:** upload 50 foto sekaligus tidak memblokir server, semua jadi `ready`, `GET /api/albums/:id/photos` mengembalikan skor, dan file varian ada di disk.

### M3 — Struktur album + editor
Chapter & page CRUD, reorder, slot assignment. Editor React sesuai `dwa-r2-s2.html`, autosave, live preview.
**Selesai kalau:** bisa menyusun album 8 chapter / 12 halaman sepenuhnya lewat UI, refresh → struktur tetap.

### M4 — Viewer
Route publik `/a/:slug` sesuai `dwa-r2-s3.html`. Publish/unpublish. Privacy unlisted & password.
**Selesai kalau:** album published bisa dibuka di HP tanpa login, Lighthouse mobile performance ≥ 85.

### M5 — Guest flow + QR
App `guest/` 5 layar sesuai `dwa-r2-s4.html`. QR generator. Rate limit. Moderasi foto tamu di editor.
**Selesai kalau:** scan QR dari HP lain → upload 3 foto → muncul di chapter Guest Memories → pengantin bisa menyembunyikannya.

### M6 — AI dasar
`quality.js`, `dedup.js`, `curate.js`, `layout.js` + endpoint `/curate`, `/autolayout`, `/duplicates`. Tombolnya sudah ada di panel kanan editor.
**Selesai kalau:** upload 200 foto mentah → tekan "Regenerate Layout" → keluar draft album 8 chapter yang urut kronologis, tanpa foto blur dan tanpa duplikat.

### M7 — Deploy & hardening
Deploy ke NAS, nginx config, backup script, `.env.example`, README cara setup dari nol.
**Selesai kalau:** `optimadigitalselaras.com/wedding_album/app/` bisa dipakai membuat album asli dari awal sampai publish, dari komputer lain.

---

## 12. Aturan kerja

**Lakukan:**
- Baca file prototipe yang relevan **sebelum** membuat komponen — ambil warna, ukuran, teks, dan struktur dari sana.
- Tulis test untuk setiap fungsi di `ai/` dan setiap route yang mengubah data.
- Validasi semua input dengan `zod` di middleware, jangan di dalam handler.
- Query SQL parameterized. Tidak ada string concatenation ke SQL, sekalipun untuk nilai internal.
- Pesan error yang dilihat user dalam Bahasa Indonesia; log internal boleh Inggris.
- Commit kecil dan sering, satu perubahan logis per commit.
- Kalau ada keputusan desain yang ambigu, tulis di `docs/decisions.md` lalu lanjut — jangan berhenti menunggu.

**Jangan:**
- Jangan menambah dependency di luar yang disebut tanpa alasan yang ditulis di `docs/decisions.md`.
- Jangan mengubah desain visual prototipe "supaya lebih bagus".
- Jangan pakai `localStorage` untuk data yang harus persist di server.
- Jangan menaruh secret di kode atau di commit.
- Jangan menandai milestone selesai kalau test merah atau fitur cuma separuh.
- Jangan bikin foto placeholder berupa gradient di produk jadi — itu hanya untuk prototipe.

---

## 13. Definition of done

Aplikasi dianggap jadi kalau seseorang bisa, tanpa bantuan:

1. Daftar akun, buat album, isi data pengantin & adat.
2. Upload 200+ foto dari HP atau laptop.
3. Menekan satu tombol dan mendapat draft album yang masuk akal.
4. Merapikan chapter, mengganti foto, menulis narasi.
5. Publish, mencetak QR, dan tamu bisa menyumbang foto tanpa install apa pun.
6. Membuka album di HP dan enak dilihat.
7. Semua ini bertahan setelah server restart.

---

**Mulai dari M0. Laporkan setiap milestone selesai dengan ringkasan singkat apa yang dibangun dan hasil acceptance test-nya.**
