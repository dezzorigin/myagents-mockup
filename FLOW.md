# Flow Page myagents — Spesifikasi Text Flow

> Versi: draft v2. Sumber keputusan: diskusi desain (arsitektur repo, pricing, UI, kredensial, RenderCV, struktur multi-produk).
> Prinsip: www.myagents.id adalah rumah BRAND, /jobsearch adalah PRODUK pertama. Info gratis, aksi bayar. Tanpa login di depan.

---

## 0. Arah Web: Multi-Produk

www.myagents.id bukan landing satu fitur, tapi etalase brand yang jual produk AI (sekpri digital custom use case Hermes).

```
www.myagents.id                  <- Landing brand: siapa myagents, daftar produk
   |-- /jobsearch                <- PRODUK 1: AI job search (yang dibangun sekarang)
   |      |-- landing produk     <- halaman jualan jobsearch (mockup-landing.html)
   |      |-- /app               <- aplikasinya: chat, dashboard, kredensial
   |-- /oneclick (nanti)         <- PRODUK 2: custom use case Hermes tingkat enterprise
   |-- /[produk-berikutnya]      <- slot produk lain
```

Aturan arsitektur:
- Landing brand menaikkan kepercayaan: cerita, daftar produk, testimoni terkumpul dari semua produk.
- Tiap produk punya landing sendiri + aplikasi sendiri. Jangan campur.
- Nav global sederhana: Produk (dropdown), Cara kerja, Kontak WA. Tanpa daftar akun di landing brand.
- Satu sistem akun untuk semua produk (satu tabel users), kuota per produk terpisah di tabel langganan.
- Brand di www, produk di path. Kalau nanti produk besar boleh pindah subdomain (jobsearch.myagents.id), tapi jangan sekarang — satu deployment lebih murah.

## 1. Aktor

| Aktor | Keterangan |
|---|---|
| User | Pencari kerja (untuk jobsearch), target utama fresh graduate, akses dari HP |
| Agen | Persona chat myagents (sekpri digital), antarmuka utama user |
| Worker | Program di PC/homeserver, mengerjakan antrean malam hari (Playwright + RenderCV) |
| Admin | Kamu, verifikasi bayar via WhatsApp, flip status paket di dashboard internal |
| Portal | JobStreet, Glints, LinkedIn Indonesia (sumber lowongan + jalur apply) |

## 2. Peta Flow Utama (produk jobsearch)

```
[www.myagents.id]  (landing brand)
   |
   v  klik "Job Search" / CTA produk
[/jobsearch]  (landing produk: hero chat, bukti, harga, FAQ)
   |
   v
[Chat tanpa akun] --> agen jawab: jumlah cocok + skor + nama perusahaan (gratis, rate limit 20/hari)
   |
   |-- user tekan [Lamar] / [Auto-blast] / aksi apa pun
   v
[Sheet Bayar + Pendaftaran]
   |-- Daftar: nama + nomor WA + PIN 6 digit (tanpa email, satu akun untuk semua produk)
   |-- Pilih paket: Beta Rp10rb / Coba Rp25rb(20) / Bulanan Rp49rb(60)
   |
   v
[Chat WA admin] --> transfer --> bukti --> [Admin flip status di dashboard internal]
   |
   v
[Akun aktif berbayar]
   |
   v
[Branch A: belum punya CV]          [Branch B: sudah punya CV]
   |                                    |
   v                                    v
[CV Builder RenderCV]              [Unggah PDF]
   v                                   |
[YAML -> PDF ATS] --------------------+
   |
   v
[Konfirmasi Mode di Chat]
   |-- "Otomatis semua" --> antrean auto-blast
   |-- "Cek satu per satu" --> antrean 1-klik
   |
   v
[User pilih lowongan (checkbox, sesuai kuota)]
   |
   v
[Kredensial portal (opsional, utk jalur login)] -- terenkripsi + persetujuan risiko
   |
   v
[Antrean Supabase] --> worker on-demand (PC) mengerjakan 20.00-23.00
   |
   v
[Branch jalur apply per lowongan]
   |-- [A] Login-able: worker login pakai kredensial terenkripsi + profil browser
   |        terenkripsi di disk worker --> isi form --> submit
   |-- [B] Email HRD: kirim CV + pesan singkat dari mailer domain
   |-- [C] Form eksternal (Greenhouse/Lever/Google Form): isi + submit
   |-- [D] Portal terkunci / 2FA / login gagal: pindah ke antrean 1-klik
   |
   v
[Tracking Dashboard]
   status: antre -> dikerjakan -> terkirim / gagal(+alasan) / panggilan
   |
   v
[Notifikasi WA] ringkasan harian pagi
   |
   v
[Follow-up otomatis hari ke-7 untuk yang tanpa kabar]
   |
   v
[User lapor dapat panggilan] --> agen minta testimoni screenshot
   |
   v
[Testimoni di landing brand + landing produk] --> [Referral: bonus 5 lamaran] --> loop akuisisi
```

## 3. Detail Per Flow

### 3.1 Flow Landing Brand (www.myagents.id)
- Hero: satu kalimat apa itu myagents ("Sekpri digital: agen AI untuk pekerjaan repetitifmu") + CTA ke produk unggulan (jobsearch).
- Section produk: kartu jobsearch aktif + kartu "segera" (oneclick enterprise, custom use case Hermes).
- Section kepercayaan: counter progres live, testimoni lintas produk.
- Nav: Produk (dropdown), Cara kerja, Kontak WA. Tanpa daftar akun di sini.
- Bahasa Indonesia awam, tanpa emoji, tanpa klaim berlebihan.

### 3.2 Flow Chat Gratis (pra-daftar)
- Chat pill di landing produk /jobsearch. Tidak ada login.
- Agen jawab: "Ketemu N lowongan yang cocok dengan {intent}". Skor 1-5 tampil, nama perusahaan tampil penuh, tombol Lamar terkunci.
- Rate limit: 20 pencarian per hari per perangkat.
- Tujuan: dopamin < 60 detik.

### 3.3 Flow Pendaftaran + Bayar
- Trigger: tekan Lamar / Auto-blast / aksi keluar mana pun.
- Sheet: 3 field (nama, nomor WA, PIN 6 digit) + pilihan paket.
- Akun berlaku lintas produk (nanti login ke produk lain pakai akun sama), kuota per produk.
- Paket: Beta Rp10rb (10 lamaran, 10 orang pertama, wajib testimoni) / Coba Rp25rb (20 lamaran, 30 hari) / Bulanan Rp49rb (60 auto-blast/bln).
- Bayar: chat WA admin, transfer, kirim bukti. Admin flip status di dashboard internal (jam kerja).
- Anti-fraud: satu nomor WA = satu akun. Kuota tidak pindah akun. Sisa hangus saat kedaluwarsa.

### 3.4 Flow CV Builder (RenderCV)
- Trigger: user bilang belum punya CV, atau dari menu dashboard.
- Agen tanya: pendidikan, pengalaman (organisasi/magang), keahlian, kontak.
- Data -> struktur YAML RenderCV -> worker render PDF tema ATS-friendly.
- PDF disimpan di Supabase Storage sebagai CV user.
- CV tidak di-tailor per lowongan (keputusan MVP).

### 3.5 Flow Kredensial Portal
- Halaman terpisah di dashboard. Per portal: username, password, centang persetujuan risiko.
- Enkripsi at-rest (Supabase Vault/pgcrypto). Tidak pernah ditampilkan kembali. Bisa dihapus.
- Profil browser per user (cookies sesi portal) disimpan TERENKRIPSI di disk worker.
- 2FA/login mencurigakan: worker pause, agen minta kode via chat.
- Yang tidak mau menyerahkan password: tetap bisa 1-klik.

### 3.6 Flow Auto-blast (jalur worker)
- Klasifikasi: [A] login-able, [B] email HRD, [C] form eksternal, [D] terkunci.
- Antrean Supabase, worker on-demand 20.00-23.00.
- Throttle: maks 20 lamaran/hari/portal, jeda acak.
- Dedup + deteksi repost sebelum masuk antrean.
- Kuota dipotong hanya saat TERKIRIM. Gagal tidak memotong kuota.

### 3.7 Flow Tracking + Notifikasi
- Dashboard: tab status (Antre/Dikerjakan/Terkirim/Gagal/Panggilan), 4 kolom pipeline.
- Ringkasan harian ke WA (manual dulu, otomatis via Baileys saat infra siap).
- Follow-up otomatis hari ke-7.
- Tab Panggilan: tombol "Siapkan interview".

### 3.8 Flow Testimoni + Referral
- User lapor dapat panggilan -> screenshot -> tampil di landing brand DAN landing produk.
- Referral: kode per user, bonus 5 lamaran (kuota, bukan uang).

## 4. Halaman yang Perlu Dibangun (MVP)

| Halaman | Path | Isi | Prioritas |
|---|---|---|---|
| Landing brand | www.myagents.id | Etalase produk, kepercayaan, CTA ke jobsearch | P0 |
| Landing produk | /jobsearch | Hero chat, cara kerja, bukti, harga, FAQ | P0 |
| Chat + hasil | /jobsearch/app | Chat, kartu lowongan + skor, checkbox | P0 |
| Sheet bayar + daftar | modal | 3 field + paket + instruksi WA | P0 |
| Dashboard lamaran | /jobsearch/app/lamaran | Kuota, tab status, pipeline | P0 |
| Kredensial portal | /jobsearch/app/kredensial | Form per portal + persetujuan | P0 |
| CV builder | /jobsearch/app/cv | Wizard chat + preview PDF | P1 |
| Dashboard admin | /admin | Akun, flip status paket, verifikasi bayar | P0 |

## 5. Data Minimal (skema awal)

- users: id, nama, wa_number (unik), pin_hash, tier, created_at
- products: slug (jobsearch, oneclick, ...), nama, aktif
- subscriptions: user_id, product_slug, paket, quota_total, quota_used, expiry
- credentials: user_id, portal, username, password_encrypted, consent_at
- jobs: id, portal, title, company, location, salary_min, salary_max, apply_path (A/B/C/D), url, jd_text, posted_at, dedup_hash
- scores: user_id, job_id, score, reasons
- applications: user_id, job_id, status, path_used, sent_at, fail_reason, followup_at
- payments: user_id, product_slug, paket, amount, proof_url, verified_by_admin_at
- testimonials: user_id, product_slug, screenshot_url, published_at

## 6. Keputusan Terkunci

1. www.myagents.id = landing brand multi-produk; jobsearch = produk pertama di path /jobsearch.
2. Satu akun (nama + WA + PIN) untuk semua produk; kuota per produk terpisah.
3. Info gratis (skor + nama perusahaan), semua aksi keluar berbayar.
4. Tanpa login di depan. Satu nomor WA = satu akun.
5. CV apa adanya; builder RenderCV hanya untuk yang belum punya.
6. Auto-blast pakai kredensial user terenkripsi (model AIHawk, versi aman) + profil browser terenkripsi di disk worker.
7. Mode 1-klik tetap ada untuk yang tidak mau menyerahkan password.
8. Harga jobsearch: Beta Rp10rb(10, testimoni) / Coba Rp25rb(20) / Bulanan Rp49rb(60).
9. Worker on-demand di PC (20.00-23.00), migrasi ke homeserver/VPS nanti.
10. Bayar via WhatsApp manual, aktivasi oleh admin. Gateway otomatis nanti.
11. Notifikasi WA manual dulu, Baileys saat infra siap.
12. Produk berikutnya (oneclick enterprise, custom Hermes) hanya landing + waitlist; tidak dibangun sebelum jobsearch jalan.
