# QUORUM — Live Paper Trading + Backtest (serverless, Rp0)

Bot paper-trading **BTCUSDT 5m** berbasis **uji Monte Carlo**: setiap entry
harus lolos 10.000 path simulasi dari **dua generator independen** (Moving
Block Bootstrap + FHS-GARCH) yang menghargai posisi ber-TP/SL sebagai **opsi
barrier ganda** — bukan prediksi, tapi pricing. Dua generator harus sepakat
(**quorum**); kalau ribut, yang pesimis menang. Verdict dievaluasi RulesJudge
profil `conservative` (margin 12pp di atas breakeven, EV pesimis minimal
+0,50%, divergensi generator maks 8pp). Uang virtual $10.000. **Tidak ada —
dan tidak akan ada — kode order live di repo ini.**

## Arsitektur (semua server gratisan)

```
Pemicu (3 lapis, sejak 2026-09-27):
  1. cron-job.org tiap 15 menit → POST /repos/rfypych/quorum/dispatches {"event_type":"tick"}
     (dispatch API SELALU menciptakan run — cron internal GitHub terukur me-drop ~90% tick saat sibuk)
  2. schedule GitHub "4,19,34,49 * * * *" (cadangan otomatis)
  3. tombol Run workflow di tab Actions (manual)
   │
   ▼
GitHub Actions (repo publik = menit gratis tak terbatas)
   │  python live_catchup.py
   │    1. tarik semua candle TERTUTUP sejak run terakhir (catch-up, REST publik)
   │    2. replay tiap candle: cek TP/SL posisi (tie=SL) → kalau flat → UJI
   │    3. verdict ENTER → beli uang virtual, tulis ledger
   │    4. commit data/ kembali ke repo ini (record = bagian dari git history)
   ▼
GitHub Pages (domain gratis: https://rfypych.github.io/quorum/)
   └─ index.html + web/ membaca data/*.jsonl + reports/backtest-*.json
      → candlestick chart harga (telemetry data/klines.jsonl), ekuitas, P(TP)
        per trial, tabel trade, perbandingan profil backtest, heartbeat. HP-friendly.
```

Setup/pemulihan pemicu (termasuk langkah cron-job.org): lihat `setup/SETUP.md`.

**Kenapa pemicu dari luar?** Cron internal GitHub me-drop hampir semua tick
saat jam sibuk (terukur 2 hari: hanya ~7 dari 96 tick/hari nyala), jadi
dashboard tampak mati berjam-jam padahal data utuh. Dispatch API dari
cron-job.org selalu menciptakan run. Delay antrean beberapa menit tetap
mungkin — dan tetap aman, karena (lihat bawah) hasil paper deterministik
terhadap data, bukan jam.

**Kenapa catch-up, bukan loop 24/7?** Scheduler mana pun bisa telat. Karena
fill ditetapkan di **harga close candle sinyal** dan exit di level barrier,
hasil paper **deterministik terhadap data** — jitter pemicu hanya menggeser
*kapan* angka tercatat, bukan *berapa* angkanya. Desain ini menukar presisi
jam dengan ketahanan macet, dan tidak kehilangan apa pun.

## File penting

| File | Peran |
|---|---|
| `replay.py` | Satu logika replay untuk backtest & live (konsistensi penuh) |
| `backtest.py` | Walk-forward historis (`--months 6`), laporan per-profil di `reports/` |
| `live_catchup.py` | Loop paper otomatis untuk pemicu cron / dijalankan manual |
| `klines_log.py` | Telemetry OHLCV untuk chart dashboard — BUKAN buku keputusan (aman OOS) |
| `backfill_klines.py` | Isi historis chart harga sekali (`--days 5`) |
| `sidang/` | Engine MC, generator, judge, ledger, feed multi-host Binance — **BEKU sejak OOS mulai** (nama paket lawas dipertahankan: ganti nama = sentuh engine = jam OOS reset) |
| `data/` | Record live: `state.json`, `ledger.jsonl`, `trials.jsonl`, `runs.jsonl`, `klines.jsonl` — **di-commit** (ini bukunya) |
| `web/`, `index.html` | Dashboard Pages (chart candlestick + kalibrasi profil), nol dependensi eksternal, bahasa visual BoardUI |
| `setup/` | Kit aktifasi otomatisasi: workflow template + panduan pemicu eksternal |

## Menjalankan manual

```bash
pip install -r requirements.txt
python backtest.py --months 6 --profile conservative   # replay historis
python live_catchup.py                                 # satu kali catch-up
python run_live.py --symbol BTCUSDT --interval 5m      # mode WS di laptop (opsional)
python tests/test_plumbing.py                          # tes sanitasi
```

Trigger backtest di cloud: tab **Actions → quorum-backtest → Run workflow**
(input `months`), hasilnya di-commit ke `reports/`.

## Kejujuran (baca dulu sebelum memantau)

- **Paper-only.** Semua angka = uang kertas. Bukan saran keuangan.
- **Fee dihitung jujur** 0,25% roundtrip + tie intrabar = SL + timeout di
  harga close. Tidak ada fill ajaib.
- **Backtest bukan bukti edge**: parameter tidak di-fit ke historis, tapi
  struktur dipilih dengan pengetahuan pasar baru-baru ini (weak-form
  evidence). Bukti sejati = **live paper OOS mulai hari bot dinyalakan**.
- **Gate uang nyata**: umur 18 (KYC venue legal) DAN track record OOS
  3–6 bulan (Sharpe harian > 1,5; PF > 1,3; tanpa bulan negatif parah).
  Sampai keduanya hijau, duit nyata = Rp0.
- Bot yang diam (semua STAND_DOWN) = bot yang sehat. Market choppy +
  fee = tidak ada kasus; menolak trade adalah fitur termahal sistem ini.

## Ganti nama (2026-09-27)

Repo ini sebelumnya `sidang-live` — di-rename jadi `quorum` atas keputusan
PM ("terlalu ke indo-indoan dan dilebih-lebihkan"). Link lama otomatis
di-redirect GitHub. Yang diganti: nama repo, workflow, dashboard, docs.
Yang TIDAK disentuh: engine (`sidang/`, `replay.py`, `backtest.py`,
`live_catchup.py`) dan seluruh `data/` — jam OOS tetap jalan dari
2026-09-23T20:00Z tanpa reset.

## Lisensi & etika

Kode milik pemilik repo. Data dari endpoint publik Binance (market data
saja, tanpa API key, tanpa KYC). Jangan pakai repo ini untuk mengklaim
profit live — semua yang tercatat di sini adalah simulasi.
