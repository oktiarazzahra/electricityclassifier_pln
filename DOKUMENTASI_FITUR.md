# 📊 DOKUMENTASI FITUR DETEKSI PENCURIAN LISTRIK

## Pendahuluan

Sistem ini menggunakan **27 fitur** untuk mendeteksi pola pencurian listrik. Fitur-fitur ini dibagi menjadi 2 kategori utama:
1. **13 Fitur Statistik Dasar** - Menganalisis pola pemakaian secara keseluruhan
2. **14 Fitur Temporal** - Mendeteksi PERUBAHAN pola dari waktu ke waktu

---

## ⚠️ ATURAN ELIGIBILITY: MINIMAL 12 BULAN DATA AKTIF

### Mengapa Diperlukan Minimal 12 Bulan?

Model deteksi pencurian listrik **HANYA boleh memprediksi** pelanggan yang memiliki **minimal 12 bulan data konsumsi AKTIF berturut-turut** karena:

1. **Pola pencurian membutuhkan observasi jangka panjang** untuk dibedakan dari pola normal
2. Fitur seperti `year_change_pct` membutuhkan perbandingan tahun pertama vs tahun terakhir
3. **Risiko tinggi false positive/negative** jika data kurang dari 12 bulan

### Proses Filtering Eligibility:

```
LANGKAH 1: Buang pelanggan yang sudah berhenti
           (kolom bulan terakhir = N/A)
           → Status: INACTIVE_STOPPED
           
LANGKAH 2: Trim ke periode aktif
           (dari bulan pertama bukan N/A sampai bulan terakhir bukan N/A)
           
LANGKAH 3: Hitung jumlah bulan aktif
           active_months = end_month - start_month + 1
           
LANGKAH 4: Filter berdasarkan eligibility
           - active_months >= 12 → ELIGIBLE (bisa diprediksi)
           - active_months < 12  → INSUFFICIENT_HISTORY (tidak diprediksi)
```

### Status Eligibility:

| Status | Deskripsi | Aksi |
|--------|-----------|------|
| ✅ ELIGIBLE | ≥12 bulan data aktif, masih berlangganan | **DIPREDIKSI** oleh model |
| ⚠️ INSUFFICIENT_HISTORY | <12 bulan data aktif (pelanggan baru) | **TIDAK DIPREDIKSI** - tunggu sampai cukup data |
| ❌ INACTIVE_STOPPED | Bulan terakhir = N/A (sudah berhenti) | **TIDAK DIPREDIKSI** - tidak relevan |

### Statistik dari Data:

| Kategori | Jumlah | Persentase |
|----------|--------|------------|
| ELIGIBLE | 132,738 | 86.8% |
| INSUFFICIENT_HISTORY | 5,916 | 3.9% |
| INACTIVE_STOPPED | 14,320 | 9.4% |
| **Total** | **152,974** | **100%** |

---

### ⚠️ PENTING: Mengapa Perlu Fitur Temporal?

Data yang tersedia adalah history pemakaian dari **2021 hingga 2026** (59 bulan). Pencuri tidak mencuri sejak awal - mereka mungkin:
- Normal di 2021-2023
- Mulai mencuri di 2024

Jika hanya melihat rata-rata keseluruhan, pencuri ini bisa terlihat "normal". Oleh karena itu, fitur temporal membandingkan **periode awal vs periode akhir** untuk mendeteksi perubahan perilaku.

---

## 📋 KATEGORI 1: FITUR STATISTIK DASAR (13 Fitur)

### 1. `usage_mean` - Rata-rata Pemakaian

**Deskripsi:** Rata-rata pemakaian listrik (kWh) selama periode aktif pelanggan.

**Rumus:**
```
usage_mean = Σ(pemakaian bulan i) / jumlah bulan aktif
```

**Contoh:**
| Bulan | Jan | Feb | Mar | Apr | Mei | Jun | Mean |
|-------|-----|-----|-----|-----|-----|-----|------|
| Pelanggan Normal | 200 | 180 | 220 | 190 | 210 | 200 | **200** |
| Pelanggan Pencuri | 50 | 40 | 30 | 60 | 45 | 55 | **46.7** |

**Insight:** Pencuri cenderung memiliki rata-rata pemakaian lebih rendah karena sebagian listrik tidak tercatat di meteran.

---

### 2. `usage_std` - Standar Deviasi Pemakaian

**Deskripsi:** Ukuran variasi/fluktuasi pemakaian bulanan.

**Rumus:**
```
usage_std = √[Σ(xi - mean)² / n]
```

**Contoh:**
| Pelanggan | Pola Pemakaian | Std Dev |
|-----------|----------------|---------|
| Stabil | 200, 200, 200, 200, 200 | **0** |
| Normal | 180, 220, 190, 210, 200 | **15.8** |
| Tidak Stabil | 300, 50, 280, 20, 350 | **147.2** |

**Insight:** Pencuri yang kadang mencuri, kadang tidak, akan memiliki std dev tinggi.

---

### 3. `usage_min` - Pemakaian Minimum

**Deskripsi:** Nilai pemakaian terendah dalam periode aktif.

**Contoh:**
- Pelanggan Normal: Minimum = 150 kWh (bulan hemat)
- Pelanggan Pencuri: Minimum = 0 kWh (bulan meteran dirusak)

**Insight:** Pencuri sering memiliki bulan dengan pemakaian sangat rendah atau nol.

---

### 4. `usage_max` - Pemakaian Maksimum

**Deskripsi:** Nilai pemakaian tertinggi dalam periode aktif.

**Insight:** Digunakan bersama usage_min untuk menghitung range pemakaian.

---

### 5. `usage_range` - Rentang Pemakaian

**Deskripsi:** Selisih antara pemakaian maksimum dan minimum.

**Rumus:**
```
usage_range = usage_max - usage_min
```

**Contoh:**
| Pelanggan | Min | Max | Range |
|-----------|-----|-----|-------|
| Normal | 150 | 250 | **100** |
| Pencuri | 0 | 300 | **300** |

**Insight:** Range besar menunjukkan ketidakstabilan pola - bisa jadi indikasi pencurian.

---

### 6. `coefficient_variation` - Koefisien Variasi

**Deskripsi:** Rasio standar deviasi terhadap rata-rata. Mengukur seberapa "stabil" pola pemakaian relatif terhadap rata-ratanya.

**Rumus:**
```
coefficient_variation = usage_std / usage_mean
```

**Contoh:**
| Pelanggan | Mean | Std | CV |
|-----------|------|-----|-----|
| Stabil | 200 | 20 | **0.10** |
| Tidak Stabil | 100 | 80 | **0.80** |

**Insight:** CV tinggi menunjukkan pola tidak konsisten - mencurigakan.

---

### 7. `zero_usage_count` ⭐ SANGAT PENTING

**Deskripsi:** Jumlah bulan dengan pemakaian = 0 kWh dalam periode aktif.

**Contoh:**
```
Pelanggan A: [200, 180, 0, 220, 0, 0, 190, 200]
zero_usage_count = 3

Pelanggan B: [200, 180, 220, 190, 210, 200, 195, 205]
zero_usage_count = 0
```

**Insight:** 
- Rumah yang masih berpenghuni TIDAK MUNGKIN 0 kWh (minimal kulkas, lampu)
- Banyak zero usage = INDIKASI KUAT meteran dirusak

**Statistik dari data:**
- Known Fraud rata-rata: 3.5 bulan zero usage
- Pelanggan Normal rata-rata: 0.2 bulan zero usage

---

### 8. `max_drop` - Penurunan Maksimum Antar Bulan

**Deskripsi:** Penurunan terbesar antara 2 bulan berturut-turut.

**Rumus:**
```
max_drop = max(pemakaian[i] - pemakaian[i+1]) untuk semua drop > 0
```

**Contoh:**
```
Bulan:    Jan   Feb   Mar   Apr   Mei   Jun
kWh:      200   180   190   50    60    45
Drop:           20   -10   140   -10    15

max_drop = 140 (dari Mar ke Apr)
```

**Insight:** Penurunan drastis tiba-tiba bisa menandakan awal pencurian.

---

### 9. `trend_slope` - Kemiringan Tren

**Deskripsi:** Slope dari regresi linear pemakaian terhadap waktu.

**Rumus:**
```
trend_slope = koefisien dari y = ax + b (dimana a = slope)
```

**Contoh:**
| Tren | Pola | Slope |
|------|------|-------|
| Naik | 100 → 150 → 200 → 250 | **+50** |
| Stabil | 200 → 200 → 200 → 200 | **0** |
| Turun | 300 → 200 → 100 → 50 | **-83** |

**Insight:** Trend menurun bisa mengindikasikan pencurian yang semakin berani.

---

### 10. `recent_disconnect` - Disconnect 3 Bulan Terakhir

**Deskripsi:** Jumlah bulan dengan pemakaian < 10 kWh dalam 3 bulan terakhir.

**Contoh:**
```
3 bulan terakhir: [5, 200, 3]
recent_disconnect = 2 (bulan dengan < 10 kWh)
```

**Insight:** Disconnect di bulan terakhir sangat mencurigakan - bisa jadi sedang dicabut meternya.

---

### 11. `inactive_months` - Bulan Tidak Aktif

**Deskripsi:** Jumlah bulan dengan pemakaian < 5 kWh.

**Insight:** Mirip dengan zero_usage_count tapi dengan threshold lebih tinggi.

---

### 12. `ue_encoded` - Kode Unit Pelanggan

**Deskripsi:** Encoding numerik dari Unit Entitas (UE) pelanggan.

**Insight:** Beberapa wilayah/unit memiliki tingkat fraud yang lebih tinggi.

---

### 13. `ue_fraud_risk` - Risk Score per Unit

**Deskripsi:** Persentase historis fraud di unit tersebut.

**Rumus:**
```
ue_fraud_risk = jumlah fraud di UE / total pelanggan di UE
```

**Contoh:**
| UE | Total Pelanggan | Fraud | Risk Score |
|----|-----------------|-------|------------|
| AAE= | 1000 | 5 | **0.005** |
| AAM= | 500 | 10 | **0.020** |

**Insight:** UE dengan fraud rate tinggi perlu perhatian lebih.

---

## 📋 KATEGORI 2: FITUR TEMPORAL (14 Fitur)

### STRATEGI UTAMA: Bandingkan PERIODE AWAL vs PERIODE AKHIR

Fitur temporal membandingkan pola di **12 bulan pertama** dengan **12 bulan terakhir** untuk mendeteksi perubahan perilaku.

```
Timeline: 2021 ─────────────────────────────────────────── 2026
          |← 12 bulan pertama →|    |← 12 bulan terakhir →|
          [  NORMAL ???  ]          [  MENCURI ???  ]
```

---

### 14. `first_year_mean` - Rata-rata Tahun Pertama

**Deskripsi:** Rata-rata pemakaian 12 bulan pertama periode aktif.

**Contoh:**
```
Bulan 1-12: [200, 220, 180, 210, 190, 200, 230, 175, 195, 205, 215, 185]
first_year_mean = 200.4 kWh
```

**Insight:** Ini adalah "baseline" pemakaian normal pelanggan.

---

### 15. `last_year_mean` - Rata-rata Tahun Terakhir

**Deskripsi:** Rata-rata pemakaian 12 bulan terakhir periode aktif.

**Contoh:**
```
Bulan terakhir 1-12: [100, 80, 50, 90, 0, 60, 70, 0, 40, 55, 45, 50]
last_year_mean = 53.3 kWh
```

**Insight:** Jika jauh lebih rendah dari first_year_mean → mencurigakan!

---

### 16. `year_change_abs` - Perubahan Absolut

**Deskripsi:** Selisih antara rata-rata tahun terakhir dan tahun pertama.

**Rumus:**
```
year_change_abs = last_year_mean - first_year_mean
```

**Contoh:**
```
first_year_mean = 200.4 kWh
last_year_mean = 53.3 kWh
year_change_abs = 53.3 - 200.4 = -147.1 kWh
```

**Insight:** Nilai negatif besar = penurunan usage yang signifikan.

---

### 17. `year_change_pct` ⭐ SANGAT PENTING

**Deskripsi:** Perubahan persentase antara tahun pertama dan terakhir.

**Rumus:**
```
year_change_pct = ((last_year_mean - first_year_mean) / first_year_mean) × 100
```

**Contoh:**
```
first_year_mean = 200 kWh
last_year_mean = 50 kWh
year_change_pct = ((50 - 200) / 200) × 100 = -75%
```

| Pelanggan | First Year | Last Year | Change % | Interpretasi |
|-----------|------------|-----------|----------|--------------|
| Normal | 200 | 210 | **+5%** | Sedikit naik, wajar |
| Normal | 200 | 180 | **-10%** | Sedikit turun, wajar |
| **PENCURI** | 200 | 50 | **-75%** | Turun drastis, CURIGA! |

**Insight:** 
- Perubahan ±20% masih wajar (musiman, gaya hidup)
- Penurunan > 50% sangat mencurigakan
- Ini fitur PALING PENTING karena langsung menunjukkan transisi ke perilaku fraud

---

### 18. `first_year_zero` - Zero Usage Tahun Pertama

**Deskripsi:** Jumlah bulan dengan 0 kWh di 12 bulan pertama.

**Contoh:**
```
12 bulan pertama: [200, 180, 190, 200, 210, 195, 205, 185, 200, 190, 195, 200]
first_year_zero = 0
```

---

### 19. `last_year_zero` - Zero Usage Tahun Terakhir

**Deskripsi:** Jumlah bulan dengan 0 kWh di 12 bulan terakhir.

**Contoh:**
```
12 bulan terakhir: [100, 0, 50, 0, 0, 60, 0, 30, 0, 40, 0, 20]
last_year_zero = 6
```

---

### 20. `zero_increase` ⭐ PENTING

**Deskripsi:** Peningkatan jumlah zero usage dari tahun pertama ke tahun terakhir.

**Rumus:**
```
zero_increase = last_year_zero - first_year_zero
```

**Contoh:**
| Pelanggan | First Year Zero | Last Year Zero | Increase |
|-----------|-----------------|----------------|----------|
| Normal | 0 | 0 | **0** |
| Normal | 1 | 1 | **0** |
| **PENCURI** | 0 | 6 | **+6** |

**Insight:** Peningkatan zero usage menunjukkan meteran mulai dirusak.

---

### 21. `stability_first` - Stabilitas Tahun Pertama

**Deskripsi:** Standar deviasi pemakaian di 12 bulan pertama.

---

### 22. `stability_last` - Stabilitas Tahun Terakhir

**Deskripsi:** Standar deviasi pemakaian di 12 bulan terakhir.

---

### 23. `stability_change` - Perubahan Stabilitas

**Deskripsi:** Perubahan stabilitas antara periode pertama dan terakhir.

**Rumus:**
```
stability_change = stability_last - stability_first
```

**Contoh:**
| Pelanggan | Stability First | Stability Last | Change |
|-----------|-----------------|----------------|--------|
| Normal | 20 | 22 | **+2** |
| **PENCURI** | 15 | 80 | **+65** |

**Insight:** Pencuri yang tidak konsisten mencurinya akan punya stability_change tinggi.

---

### 24. `max_window_drop` ⭐ PENTING

**Deskripsi:** Penurunan terbesar antara dua periode 6-bulanan berturut-turut.

**Cara Hitung:**
1. Bagi data menjadi window 6 bulan
2. Hitung rata-rata tiap window
3. Cari penurunan terbesar antar window

**Contoh:**
```
Data: [200, 210, 190, 205, 195, 200,    180, 190, 170, 185, 175, 180,    50, 40, 30, 60, 45, 55]
       └────── Window 1 ──────┘         └────── Window 2 ──────┘         └──── Window 3 ────┘
       Mean = 200                       Mean = 180                        Mean = 46.7

Drop 1→2 = 200 - 180 = 20
Drop 2→3 = 180 - 46.7 = 133.3

max_window_drop = 133.3
```

**Insight:** Ini mendeteksi KAPAN pencurian dimulai (titik perubahan perilaku).

---

### 25. `volatility_change` - Perubahan Volatilitas

**Deskripsi:** Perbedaan variabilitas antar periode 6-bulanan antara paruh pertama dan paruh kedua.

**Insight:** Pencuri yang perilakunya berubah-ubah akan menunjukkan peningkatan volatilitas.

---

### 26. `trend_break_score` - Skor Patahan Tren

**Deskripsi:** Perbedaan slope tren antara paruh pertama dan paruh kedua periode aktif.

**Cara Hitung:**
1. Hitung trend (slope) paruh pertama
2. Hitung trend (slope) paruh kedua
3. Selisihkan

**Contoh:**
```
Paruh Pertama: Trend sedikit naik (slope = +1)
Paruh Kedua: Trend turun tajam (slope = -5)

trend_break_score = 1 - (-5) = 6
```

**Insight:** Skor tinggi menunjukkan ada "patahan" tren - kemungkinan titik awal pencurian.

---

### 27. `recent_anomaly_score` - Skor Anomali Terbaru

**Deskripsi:** Z-score dari rata-rata 6 bulan terakhir dibandingkan keseluruhan periode.

**Rumus:**
```
recent_anomaly_score = (mean_6_bulan_terakhir - mean_keseluruhan) / std_keseluruhan
```

**Contoh:**
```
Mean keseluruhan = 180 kWh
Std keseluruhan = 50 kWh
Mean 6 bulan terakhir = 55 kWh

recent_anomaly_score = (55 - 180) / 50 = -2.5
```

**Interpretasi:**
| Score | Interpretasi |
|-------|--------------|
| -1 s/d +1 | Normal (dalam 1 std dev) |
| -2 s/d -1 | Sedikit di bawah normal |
| < -2 | **SANGAT DI BAWAH NORMAL** |

**Insight:** Score negatif besar = 6 bulan terakhir sangat menyimpang dari pola normal pelanggan.

---

## 🎯 BAGAIMANA MODEL MENGGUNAKAN FITUR-FITUR INI?

### 1. TIDAK MEMBANDINGKAN ANTAR PELANGGAN

Model TIDAK bilang: "Rumah A mencuri karena pakai lebih sedikit dari Rumah B"

### 2. MEMBANDINGKAN PELANGGAN DENGAN DIRINYA SENDIRI

Model bilang: "Pelanggan A mencurigakan karena polanya BERUBAH DRASTIS dari kebiasaan A sendiri"

### Contoh Deteksi:

**Pelanggan Normal:**
```
2021-2022: [200, 210, 190, 205, 195, 200, ...] → Mean = 200 kWh
2025-2026: [195, 205, 180, 210, 200, 190, ...] → Mean = 197 kWh
year_change_pct = -1.5% → WAJAR
zero_increase = 0 → BAGUS
Probabilitas Fraud = 5%
```

**Pelanggan Pencuri:**
```
2021-2022: [300, 320, 280, 310, 290, 305, ...] → Mean = 300 kWh
2025-2026: [50, 0, 40, 0, 30, 0, 20, 0, ...] → Mean = 23 kWh
year_change_pct = -92% → SANGAT MENCURIGAKAN!
zero_increase = +5 → ADA BANYAK ZERO USAGE BARU
max_window_drop = 250 → PENURUNAN DRASTIS
Probabilitas Fraud = 95%
```

---

## 📈 TOP 10 FITUR PALING BERPENGARUH

Berdasarkan Feature Importance dari model:

| Rank | Fitur | Kategori | Importance |
|------|-------|----------|------------|
| 1 | `ue_fraud_risk` | Statistik | ⭐⭐⭐⭐⭐ |
| 2 | `ue_encoded` | Statistik | ⭐⭐⭐⭐ |
| 3 | `usage_std` | Statistik | ⭐⭐⭐ |
| 4 | `year_change_pct` | Temporal | ⭐⭐⭐ |
| 5 | `usage_min` | Statistik | ⭐⭐⭐ |
| 6 | `recent_anomaly_score` | Temporal | ⭐⭐⭐ |
| 7 | `coefficient_variation` | Statistik | ⭐⭐ |
| 8 | `stability_first` | Temporal | ⭐⭐ |
| 9 | `stability_last` | Temporal | ⭐⭐ |
| 10 | `max_drop` | Statistik | ⭐⭐ |

---

## 🔴 INDIKATOR PENCURIAN KUAT

### Level CRITICAL (Probabilitas Fraud ≥ 70%):
- Zero usage saat baseline > 50 kWh
- year_change_pct < -70%
- zero_increase ≥ 4

### Level HIGH (Probabilitas Fraud 50-70%):
- Drop ≥ 70% dari baseline
- max_window_drop > 100 kWh
- recent_anomaly_score < -2

### Level MEDIUM (Probabilitas Fraud 30-50%):
- Drop 50-70% dari baseline
- Pemakaian < 20% dari baseline
- zero_increase ≥ 2

---

## 📝 KESIMPULAN

1. **27 fitur** bekerja bersama untuk mendeteksi pola pencurian
2. **Fitur temporal** sangat penting untuk mendeteksi PERUBAHAN perilaku
3. Model melihat **SELURUH history** (2021-2026), bukan hanya 12 bulan terakhir
4. Setiap pelanggan dibandingkan dengan **dirinya sendiri**, bukan pelanggan lain
5. **Kombinasi fitur** lebih akurat daripada satu fitur tunggal

---

*Dokumentasi ini dibuat untuk membantu memahami bagaimana model ML mendeteksi potensi pencurian listrik.*
