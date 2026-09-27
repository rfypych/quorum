# Aktifasi otomatisasi (sekali, ±5 menit)

> **STATUS 2026-09-27: AKTIF + MIGRASI PEMICU.** Workflow live/backtest aktif.
> Sejak 2026-09-27 pemicu utama tick = **cron-job.org** (Jalur C di bawah) —
> cron internal GitHub terukur me-drop ~90% tick saat sibuk (data 2 hari:
> hanya ~7 dari 96 tick/hari nyala). Dokumen ini = prosedur pemulihan kalau
> workflow terhapus / repo di-clone ulang / akun cron-job.org hilang.

File `setup/live.yml` + `setup/backtest.yml` adalah workflow GitHub Actions.
Karena alasan keamanan, **file workflow hanya boleh dibuat lewat web GitHub
atau token dengan scope `workflow`** — bot/sandbox dengan token `repo` saja
akan ditolak.

## Pemicu tick (3 lapis)

| Lapis | Sumber | Peran |
|---|---|---|
| 1 | **cron-job.org** → `POST /repos/rfypych/quorum/dispatches` dengan body `{"event_type":"tick"}` | pemicu utama tiap 15 menit — dispatch API **selalu** menciptakan run (tidak kena drop antrean cron internal) |
| 2 | schedule GitHub `4,19,34,49 * * * *` | cadangan otomatis — kapan pun nyala, catch-up menutup lubang |
| 3 | tombol **Run workflow** di tab Actions | manual, kapan pun |

Run ganda di jendela sama aman: `concurrency` mengantri run berikutnya, dan
run kedua jadi heartbeat idle (0 candle baru) — tidak ada trial dobel.

## Jalur C — aktifkan cron-job.org (pemicu utama, sekali saja)

1. Daftar gratis di `https://cron-job.org` (tanpa kartu, tanpa KYC).
2. Buat **fine-grained PAT** di GitHub: Settings → Developer settings →
   Personal access tokens → **Fine-grained tokens** → Generate new token:
   - Repository access: **Only select repositories** → `rfypych/quorum`
   - Permissions: **Actions → Read and write** (cuma itu, jangan lebih)
   - Expiration: 90 hari (kalau kedaluwarsa, ulangi langkah ini)
3. Di cron-job.org → **Create cronjob**:
   - Title: `quorum tick`
   - URL: `https://api.github.com/repos/rfypych/quorum/dispatches`
   - Method: **POST**
   - Headers: `Authorization: Bearer <PAT-mu>` dan `Content-Type: application/json`
   - Body: `{"event_type":"tick"}`
   - Schedule: **every 15 minutes** (kalau tersedia opsi cron custom: `4,19,34,49 * * * *`)
4. Save. Verifikasi ≤15 menit: tab Actions repo dapat run baru dengan event
   `repository_dispatch`, pill dashboard hijau.

Kenapa PAT fine-grained khusus? Kalau sampai bocor di layanan pihak ketiga,
orang cuma bisa "menekan tombol run" di repo ini — tidak bisa baca kode,
tidak bisa push. Jangan pakai PAT classic scope-`repo` penuh untuk ini.

## Jalur A — lewat web GitHub (pemulihan workflow)

1. Buka `https://github.com/rfypych/quorum`
2. Klik **Add file → Create new file**
3. Ketik nama file: `.github/workflows/live.yml`
   (GitHub otomatis bikin folder `.github/workflows/` saat kamu mengetik `/`)
4. Buka file `setup/live.yml` di repo (raw), copy seluruh isinya, paste ke editor web
5. **Commit changes** (tombol hijau)
6. Ulangi langkah 2–5 untuk `.github/workflows/backtest.yml` dari `setup/backtest.yml`
7. Buka tab **Actions** → kalau ada tombol *"I understand my workflows, go
   ahead and enable them"* → klik sekali

## Jalur B — token scope `workflow` (pemulihan via agen)

1. GitHub → Settings → Developer settings → Personal access tokens (classic)
   → **Generate new token (classic)** → centang `repo` + `workflow`
2. Kirim token ke agen yang mengelola repo; dia push `.github/workflows/`
   dan trigger run pertama.

## Verifikasi harian

- Tab **Actions** di repo: `quorum-live` hijau tiap ±15 menit
  (kolom event = `repository_dispatch` / `schedule`).
- Dashboard (`https://rfypych.github.io/quorum/`): pill **BOT HIDUP**,
  tabel "Kesehatan Run" dapat baris baru tiap 15 menit.
- Kalau pill merah >75 menit: (1) cek cron-job.org dulu — job pause/failed?,
  (2) tekan **Run workflow** manual di tab Actions, (3) kalau tetap mati,
  ikuti Jalur A/B di atas.

## Notes

- Delay beberapa menit masih mungkin (antrean runner Actions) — aman: bot
  *catch-up*, tidak skip data; hasil paper deterministik terhadap data,
  bukan jam cron.
- Repo **harus publik** supaya menit Actions gratis tak terbatas. Repo privat
  = 2.000 menit/bulan (kurang untuk 96 run/hari).
- Menit per run ≈ 1–2 menit (install numpy/scipy ter-cache).
