# MTsN 1 Sijunjung Educational Platform

Aplikasi pembelajaran digital berbasis web untuk MTsN 1 Sijunjung yang menyatukan aktivitas pembelajaran, evaluasi, penugasan, penjadwalan ujian mingguan, dan game edukasi realtime dalam satu sistem yang koheren.

Dokumen acuan produk lengkap ada di **Product Requirements Document (PRD) — Final** dan **Project Development Roadmap — Version 1.0**. README ini hanya ringkasan operasional untuk menjalankan dan mengembangkan project.

> **Prinsip Utama:** Sederhana secara arsitektur, tetapi tidak pernah mengorbankan kebenaran (correctness), keamanan (security), integritas data (data integrity), dan keandalan (reliability).

---

## Ringkasan Produk

- **Model pengembangan:** Solo developer
- **Target deployment:** Single Node instance + PostgreSQL
- **Arsitektur:** Modular Monolith
- **Pengguna:** Admin, Guru, Siswa

Fitur utama: autentikasi & sesi, manajemen pengguna & data akademik, bank soal + import Excel, Quiz/Exam/Assignment, Attempt–Answer–Result, penjadwalan ujian mingguan (ExamSchedule), sistem violation & penguncian attempt, monitoring & audit log, serta game edukasi realtime (Survival dan Versus).

---

## Tech Stack

| Lapisan     | Teknologi                                                        |
| ----------- | ---------------------------------------------------------------- |
| Frontend    | Next.js (App Router), React, TypeScript, Tailwind CSS, shadcn/ui |
| Backend     | Next.js Route Handlers, TypeScript, Custom Node Server           |
| Database    | PostgreSQL (source of truth) + Prisma ORM                        |
| Realtime    | Socket.IO — khusus kebutuhan realtime (game edukasi)             |
| Autentikasi | Persistent database session (bukan JWT stateless)                |
| Validasi    | Server-side schema validation di seluruh endpoint                |
| Testing     | Unit Test, Integration Test, Critical E2E Test                   |
| Deployment  | Single Node instance + PostgreSQL                                |

**Sengaja tidak dipakai pada tahap ini** (bukan larangan permanen, hanya batas kompleksitas yang disengaja): Redis, microservices, message queue, horizontal scaling, object storage terpisah, generic permission/workflow engine, analytics engine terpisah.

---

## Arsitektur Singkat

Satu aplikasi Next.js yang menjalankan UI, API, autentikasi, dan Socket.IO dalam satu proses Node.js (`server.ts`), dengan PostgreSQL sebagai satu-satunya penyimpanan data permanen.

**Aturan arsitektur yang mengikat (WAJIB dipatuhi setiap kontribusi kode):**

1. **Database sebagai source of truth** — tidak ada data penting yang hanya hidup di client atau di memory proses server.
2. **Server authoritative** — skor, hasil, timer, HP, pemenang, status attempt/game, dan violation selalu ditentukan server, tidak pernah oleh client.
3. **Business logic di `modules/`** — satu file per domain, bukan di UI atau route handler. Logic gameplay realtime terpisah di `games/`.
4. **Validasi server-side wajib** di setiap endpoint, terlepas dari validasi di client.
5. **Rantai otorisasi tetap:** `Authentication → Role → Ownership → Business Rule`, ditulis eksplisit per endpoint (role di-hardcode di kode), **bukan** generic permission engine dinamis.
6. **Transaction wajib** pada operasi kritis: submit attempt, auto-submit, import Excel, create/update/publish/cancel ExamSchedule, grading Assignment, dan perubahan status penting lainnya.
7. **Submit attempt idempotent** — hanya satu transisi final `ACTIVE → SUBMITTED` yang boleh berhasil, walau ada double-click, retry, atau request bersamaan.
8. **Exam timeout berbasis timestamp server/database**, bukan timer client.
9. **Session tersimpan permanen di PostgreSQL** (hash token saja, bukan token mentah); cookie `HttpOnly` + `Secure` + `SameSite` yang sesuai pada production.

Detail lengkap ada di PRD bagian 6 (Arsitektur Sistem), 10 (Otorisasi & Keamanan), dan 21 (Aturan Pengembangan).

---

## Struktur Folder

```
mtsn-1-sijunjung-educational/
├── prisma/
│   ├── schema.prisma
│   ├── seed.ts
│   └── migrations/
├── app/
│   ├── (auth)/login/page.tsx
│   ├── admin/            # users/, academic/, activities/schedules/, results/, audit/
│   ├── guru/             # questions/, activities/, assignments/, results/, games/
│   ├── siswa/            # activities/, attempts/, assignments/, results/, games/
│   ├── games/            # survival/, versus/
│   └── api/              # auth/, users/, academic/, questions/, activities/,
│                         # exam-schedules/, attempts/, assignments/, results/, games/
├── components/
│   ├── ui/               # primitif shadcn (Button, Input, Form, Dialog, Table, Select, Alert, Badge)
│   ├── layout/           # shell/nav per role
│   └── shared/           # komponen lintas role
├── modules/              # satu file per domain (BUKAN folder per domain)
│   ├── auth.ts, users.ts, academic.ts, questions.ts
│   ├── activities.ts, exam-schedules.ts, attempts.ts
│   ├── assignments.ts, results.ts, games.ts, audit.ts
├── games/                # engine gameplay murni (state machine, bukan CRUD)
│   ├── survival.ts
│   └── versus.ts
├── lib/
│   ├── db.ts, auth.ts, validation.ts, rate-limit.ts
│   ├── utils.ts, api-error.ts, logger.ts
├── tests/
│   ├── unit/, integration/, e2e/
├── server.ts             # custom server: Next.js + Socket.IO dalam satu proses
├── middleware.ts         # auth guard per role
├── next.config.ts
├── package.json
├── .env.example
├── docker-compose.yml
└── README.md
```

Sebuah domain di `modules/` hanya dipecah menjadi sub-folder (mis. `modules/activities/{service,validation,types,index}.ts`) ketika file tersebut benar-benar sudah terlalu besar atau kompleks — bukan demi tampilan struktur.

---

## Menjalankan Project Secara Lokal

### Prasyarat

- Node.js 20+
- Docker (untuk PostgreSQL lokal) atau instance PostgreSQL yang sudah berjalan

### Langkah

```bash
# 1. Install dependencies
npm install

# 2. Siapkan environment variables
cp .env.example .env
# isi DATABASE_URL, session secret, dan variabel lain sesuai .env.example

# 3. Jalankan PostgreSQL lokal (via Docker Compose)
docker-compose up -d

# 4. Jalankan migration & seed
npx prisma migrate dev
npx prisma db seed

# 5. Jalankan development server (Next.js + Socket.IO dalam satu proses)
npm run dev
```

Buka [http://localhost:3000](http://localhost:3000).

### Script yang Tersedia

| Script                      | Fungsi                                                    |
| --------------------------- | --------------------------------------------------------- |
| `npm run dev`               | Menjalankan custom server (`server.ts`) untuk development |
| `npm run build`             | Production build                                          |
| `npm run start`             | Menjalankan custom server hasil build untuk production    |
| `npm run lint`              | Menjalankan ESLint                                        |
| `npm run format`            | Merapikan format kode dengan Prettier                     |
| `npm run format:check`      | Memeriksa format tanpa mengubah file                      |
| `npm run type-check`        | Memeriksa tipe TypeScript (`tsc --noEmit`) tanpa emit     |
| `npx prisma migrate dev`    | Membuat & menjalankan migration di development            |
| `npx prisma migrate deploy` | Menjalankan migration di production                       |

---

## Environment Variables

Lihat `.env.example` untuk daftar lengkap. **Jangan pernah commit file `.env`** — hanya `.env.example` yang masuk repository. Variabel minimal yang dibutuhkan:

- `DATABASE_URL` — connection string PostgreSQL
- Session secret / signing key untuk cookie sesi
- Konfigurasi Socket.IO (jika ada, mis. path/origin)

Production menggunakan mekanisme environment/secret management milik platform deployment, bukan file `.env`.

---

## Testing

Pengujian difokuskan pada risiko terhadap data, keamanan, Exam, Result, Game, dan pengalaman siswa — bukan mengejar coverage 100% tanpa alasan.

- **Unit test** (`tests/unit/`): perhitungan skor, validasi jawaban, aturan otorisasi, aturan Activity/Attempt, aturan timer, aturan violation, logic penilaian game.
- **Integration test** (`tests/integration/`): login/session, create/import Question, Activity/Activity Target, start/save/resume/submit Attempt, submission & grading Assignment, create/publish/cancel ExamSchedule, GameSession/GameResult.
- **Critical E2E test** (`tests/e2e/`): alur Quiz, Exam (termasuk resume & auto-submit), Assignment, Penjadwalan Ujian, dan Game.

---

## Aturan Kontribusi (Non-Negotiable)

- Tidak menambahkan fitur di luar cakupan MVP tanpa alasan yang jelas.
- Tidak membuat abstraksi (repository pattern, generic service factory, generic permission/workflow engine, dsb.) sebelum benar-benar dibutuhkan.
- Tidak meletakkan business logic penting di client/UI.
- Tidak mempercayai data penting (skor, role, ownership) yang dikirim dari client tanpa verifikasi ulang di server.
- Tidak melewati rantai otorisasi `Authentication → Role → Ownership → Business Rule`.
- Tidak menyimpan data penting hanya di memory proses.
- Tidak mengubah data kritis tanpa transaction.
- Tidak menambahkan Redis, microservices, atau message queue tanpa kebutuhan nyata.
- Setiap perubahan schema database wajib melalui Prisma migration.
- Setiap fitur kritis wajib memiliki automated test.
- Setiap perubahan arsitektur wajib memiliki alasan teknis atau bisnis yang jelas.

Definition of Done per fitur mengacu pada PRD bagian 17, dan checklist kesiapan implementasi ada pada PRD Lampiran A.

---

## Dokumen Acuan

- **Product Requirements Document (PRD) — Final**
- **Project Development Roadmap — Version 1.0** (Fase 0–10)
- **Prompt Library** — template prompt siap-pakai untuk tiap jenis pekerjaan pengembangan

Seluruh keputusan implementasi mengacu ke ketiga dokumen ini sebagai satu-satunya sumber kebenaran (source of truth) proyek.
