# PANDUAN LENGKAP DETEKSI PENCURIAN LISTRIK PLN
## Dokumentasi untuk Orang Awam

---

# APA INI?

Ini adalah sistem komputer yang bisa **mendeteksi pelanggan yang dicurigai mencuri listrik** berdasarkan pola pemakaian listrik bulanan mereka.

**Bayangkan begini:** 
Kalau seseorang mencuri listrik, pola pemakaiannya akan "aneh" dibanding orang normal. Misalnya:
- Tiba-tiba meteran menunjukkan 0 kWh padahal rumahnya masih menyala
- Pemakaian yang sangat tidak stabil (kadang tinggi, kadang rendah tidak wajar)
- Tren pemakaian yang terus menurun padahal tidak ada alasan

Sistem ini menganalisis 152,974 pelanggan dan menemukan 6,541 yang mencurigakan.

---

# TINGKAT KECURIGAAN (PRIORITY)

Setiap pelanggan yang dicurigai diberi **tingkat prioritas** berdasarkan seberapa besar kemungkinan mereka melakukan kecurangan:

## CRITICAL (Probabilitas >= 80%)
**Artinya:** SANGAT MENCURIGAKAN - Hampir pasti melakukan kecurangan

**Karakteristik pelanggan CRITICAL:**
- Pola pemakaian sangat tidak normal
- Banyak bulan dengan pemakaian 0 (meteran tidak bergerak)
- Penurunan pemakaian yang drastis secara tiba-tiba
- Trend pemakaian terus menurun tanpa alasan

**Contoh pola CRITICAL:**
```
Bulan:      Jan   Feb   Mar   Apr   Mei   Jun   Jul   Agu   Sep   Okt   Nov   Des
Pemakaian:  500   480   450   0     0     0     0     0     0     0     0     0
                              ↑
                         Tiba-tiba 0! Meteran diakali?
```

**Jumlah:** ~3,308 pelanggan (2.16% dari total)

---

## HIGH (Probabilitas 50% - 79.9%)
**Artinya:** CUKUP MENCURIGAKAN - Kemungkinan besar melakukan kecurangan

**Karakteristik pelanggan HIGH:**
- Pola pemakaian tidak wajar tapi tidak seekstrem CRITICAL
- Ada beberapa bulan dengan pemakaian 0
- Variasi pemakaian yang tidak masuk akal

**Contoh pola HIGH:**
```
Bulan:      Jan   Feb   Mar   Apr   Mei   Jun   Jul   Agu   Sep   Okt   Nov   Des
Pemakaian:  500   480   100   450   0     480   0     460   100   0     470   0
                        ↑                 ↑           ↑           ↑           ↑
                   Turun drastis    Kadang 0, kadang normal (tidak konsisten)
```

**Jumlah:** ~3,174 pelanggan (2.08% dari total)

---

## MEDIUM (Probabilitas 30% - 49.9%)
**Artinya:** AGAK MENCURIGAKAN - Perlu diperhatikan lebih lanjut

**Karakteristik pelanggan MEDIUM:**
- Ada anomali ringan dalam pola pemakaian
- Tidak seekstrem CRITICAL atau HIGH
- Mungkin ada penjelasan wajar, tapi tetap perlu dicek

**Contoh pola MEDIUM:**
```
Bulan:      Jan   Feb   Mar   Apr   Mei   Jun   Jul   Agu   Sep   Okt   Nov   Des
Pemakaian:  500   520   480   510   450   200   480   500   490   510   480   500
                                          ↑
                                   Turun signifikan tapi hanya 1 bulan
```

**Jumlah:** ~5,885 pelanggan (3.85% dari total)

---

## LOW (Probabilitas < 30%)
**Artinya:** KEMUNGKINAN NORMAL - Tidak ada indikasi kuat kecurangan

**Karakteristik pelanggan LOW:**
- Pola pemakaian stabil dan konsisten
- Variasi wajar (naik sedikit saat cuaca panas karena AC, dll)
- Tidak ada bulan dengan pemakaian 0 yang mencurigakan

**Contoh pola LOW (NORMAL):**
```
Bulan:      Jan   Feb   Mar   Apr   Mei   Jun   Jul   Agu   Sep   Okt   Nov   Des
Pemakaian:  500   520   480   510   550   600   580   560   520   500   480   490
                                    ↑               ↑
                              Naik saat musim panas (AC) - WAJAR
```

**Jumlah:** ~140,546 pelanggan (91.91% dari total)

---

# BAGAIMANA CARA KERJANYA?

## Langkah 1: Kumpulkan Data
Sistem mengambil data:
- **Data Temuan:** 61 kasus pencurian yang SUDAH TERBUKTI (tertangkap)
- **Data History:** 152,974 pelanggan dengan pemakaian listrik 59 bulan terakhir

## Langkah 2: Pelajari Pola Pencuri
Sistem mempelajari pola dari 61 pencuri yang sudah tertangkap:
- Bagaimana pola pemakaian mereka?
- Apa yang membuat mereka berbeda dari pelanggan normal?

## Langkah 3: Buat Model Kecerdasan Buatan
Sistem membuat "model AI" yang bisa mengenali pola pencurian:
- **Isolation Forest:** Mencari pelanggan yang "berbeda" dari mayoritas
- **Random Forest:** Mengklasifikasikan fraud/normal
- **XGBoost:** Model kedua untuk validasi

## Langkah 4: Prediksi Semua Pelanggan
Sistem memeriksa semua 152,974 pelanggan dan memberikan:
- **Probabilitas fraud (0-100%):** Seberapa besar kemungkinan mencuri
- **Priority (CRITICAL/HIGH/MEDIUM/LOW):** Tingkat urgensi untuk dicek

## Langkah 5: Hasilkan Laporan
Sistem menghasilkan file Excel berisi daftar pelanggan mencurigakan beserta:
- Pola pemakaian bulanan mereka
- Statistik pemakaian (rata-rata, variasi, dll)
- Probabilitas fraud

---

# APA SAJA YANG DIANALISIS? (13 FITUR)

Sistem menganalisis 13 aspek dari pola pemakaian:

| No | Fitur | Artinya | Contoh Mencurigakan |
|----|-------|---------|---------------------|
| 1 | **usage_mean** | Rata-rata pemakaian bulanan | Terlalu rendah dibanding rumah serupa |
| 2 | **usage_std** | Variasi pemakaian | Sangat tidak stabil (naik-turun drastis) |
| 3 | **usage_min** | Pemakaian terendah | = 0 (meteran tidak bergerak) |
| 4 | **usage_max** | Pemakaian tertinggi | Sangat tinggi lalu tiba-tiba turun |
| 5 | **usage_range** | Rentang (max - min) | Sangat besar (tidak konsisten) |
| 6 | **coefficient_variation** | Tingkat variabilitas | Sangat tinggi (tidak stabil) |
| 7 | **zero_usage_count** | Jumlah bulan = 0 | > 3 bulan = mencurigakan |
| 8 | **max_drop** | Penurunan terbesar | > 50% dari bulan sebelumnya |
| 9 | **trend_slope** | Tren pemakaian | Terus menurun drastis |
| 10 | **recent_disconnect** | 3 bulan terakhir ada 0? | Ya = mencurigakan |
| 11 | **inactive_months** | Bulan tidak aktif berturut-turut | > 2 bulan = sangat mencurigakan |
| 12 | **ue_encoded** | Kode Unit Elektrik | Beberapa unit lebih berisiko |
| 13 | **ue_fraud_risk** | Tingkat risiko per unit | Area dengan banyak kasus sebelumnya |

---

# CARA MEMBACA FILE EXCEL HASIL

Buka file Excel di folder `results/`:

## Kolom-kolom Penting:

### Identitas Pelanggan
- **IDE:** ID unik pelanggan (terenkripsi)
- **UE:** Unit Elektrik (wilayah)

### Hasil Prediksi
- **priority:** CRITICAL / HIGH / MEDIUM / LOW
- **ensemble_fraud_prob:** Probabilitas fraud (0.00 - 1.00)
  - 0.95 artinya 95% kemungkinan fraud
  - 0.30 artinya 30% kemungkinan fraud

### Statistik Pemakaian
- **usage_mean:** Rata-rata pemakaian (kWh/bulan)
- **usage_std:** Standar deviasi (makin tinggi = makin tidak stabil)
- **zero_usage_count:** Berapa bulan pemakaiannya 0
- **max_drop:** Penurunan pemakaian terbesar (kWh)
- **inactive_months:** Berapa bulan berturut-turut tidak aktif

### Pola Pemakaian Bulanan
- **kWh_Feb2025 - kWh_Jan2026:** Pemakaian listrik setiap bulan
- Bisa dilihat polanya apakah ada yang mencurigakan

---

# CONTOH ANALISIS PELANGGAN

## Contoh 1: Pelanggan CRITICAL (Sangat Mencurigakan)

```
IDE: ABC123XYZ
Priority: CRITICAL
Probabilitas: 0.98 (98%)

Statistik:
- usage_mean: 45 kWh (sangat rendah untuk rumah tangga)
- zero_usage_count: 8 bulan
- inactive_months: 6 bulan berturut-turut
- max_drop: 450 kWh

Pola Bulanan:
Feb2025: 500 kWh
Mar2025: 480 kWh
Apr2025: 0 kWh    ← Tiba-tiba 0!
Mei2025: 0 kWh
Jun2025: 0 kWh
Jul2025: 0 kWh
Agu2025: 0 kWh
Sep2025: 0 kWh    ← 6 bulan berturut-turut 0
Okt2025: 50 kWh
Nov2025: 0 kWh
Des2025: 0 kWh
Jan2026: 0 kWh
```

**Kesimpulan:** Pelanggan ini SANGAT MENCURIGAKAN. Rumah yang sebelumnya memakai 500 kWh/bulan tiba-tiba hanya 0-50 kWh. Kemungkinan besar meteran diakali.

---

## Contoh 2: Pelanggan LOW (Normal)

```
IDE: DEF456ABC
Priority: LOW
Probabilitas: 0.05 (5%)

Statistik:
- usage_mean: 520 kWh
- zero_usage_count: 0 bulan
- inactive_months: 0
- max_drop: 80 kWh

Pola Bulanan:
Feb2025: 480 kWh
Mar2025: 500 kWh
Apr2025: 520 kWh
Mei2025: 550 kWh  ← Naik karena cuaca panas (AC)
Jun2025: 600 kWh  ← Masih musim panas
Jul2025: 580 kWh
Agu2025: 560 kWh
Sep2025: 520 kWh  ← Kembali normal
Okt2025: 500 kWh
Nov2025: 480 kWh
Des2025: 520 kWh
Jan2026: 510 kWh
```

**Kesimpulan:** Pelanggan ini NORMAL. Pola pemakaian stabil dengan variasi wajar (naik saat musim panas karena AC).

---

# FILE HASIL YANG DIHASILKAN

| File | Isi | Jumlah | Kegunaan |
|------|-----|--------|----------|
| `suspects_top_500.xlsx` | 500 teratas | 500 | Prioritas UTAMA untuk dicek |
| `suspects_high_priority.xlsx` | CRITICAL + HIGH | 6,482 | Semua yang sangat mencurigakan |
| `suspects_all.xlsx` | Prob >= 30% | 12,367 | Semua yang perlu diperhatikan |

---

# AKURASI MODEL

Sistem ini diuji dengan 61 kasus pencurian yang SUDAH TERBUKTI:

- **59 dari 61** kasus berhasil dideteksi
- **Akurasi: 96.7%** 
- Hanya 2 kasus yang tidak terdeteksi (karena pola pemakaian mereka mirip dengan orang normal)

---

# REKOMENDASI PENGGUNAAN

1. **Prioritas 1:** Cek pelanggan di file `suspects_top_500.xlsx` terlebih dahulu
2. **Prioritas 2:** Jika masih ada waktu, cek `suspects_high_priority.xlsx`
3. **Lihat pola bulanan:** Perhatikan kolom kWh_xxx untuk melihat pola mencurigakan
4. **Validasi di lapangan:** Hasil ini adalah DUGAAN, tetap perlu verifikasi fisik

---

# CATATAN PENTING

- Hasil ini adalah **DUGAAN** berdasarkan pola data, bukan bukti pasti
- Pelanggan dengan priority CRITICAL **tidak otomatis bersalah**, tapi perlu dicek
- Ada kemungkinan **false positive** (pelanggan normal yang terdeteksi mencurigakan)
- Keputusan final tetap harus berdasarkan **pemeriksaan lapangan**

---

**Dibuat:** 12 Januari 2026
**Sistem:** Hybrid Semi-Supervised Learning (Isolation Forest + Random Forest + XGBoost)
**Data:** 152,974 pelanggan PLN
