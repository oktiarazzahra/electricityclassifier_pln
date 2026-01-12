# DOKUMENTASI NOTEBOOK: DETEKSI PENCURIAN LISTRIK PLN
## Penjelasan Detail Setiap Cell di Notebook

---

# DAFTAR ISI

1. [Overview Notebook](#overview)
2. [BAGIAN 1: Setup & Import Libraries](#bagian-1)
3. [BAGIAN 2: Load Data](#bagian-2)
4. [BAGIAN 3: Data Analysis & Labeling](#bagian-3)
5. [BAGIAN 4: Data Cleaning](#bagian-4)
6. [BAGIAN 5: Feature Engineering](#bagian-5)
7. [BAGIAN 6: Model Training](#bagian-6)
8. [BAGIAN 7: Model Evaluation](#bagian-7)
9. [BAGIAN 8: Suspect Ranking](#bagian-8)
10. [BAGIAN 9: Visualization](#bagian-9)
11. [BAGIAN 10: Export Results](#bagian-10)
12. [BAGIAN 11: Save Models](#bagian-11)
13. [Alur Lengkap Pipeline](#alur-pipeline)

---

# OVERVIEW NOTEBOOK {#overview}

Notebook ini berisi **35 cells** total:
- **14 Markdown Cells**: Penjelasan dan dokumentasi
- **16 Code Cells**: Kode Python yang dieksekusi
- **5 Section Headers**: Pembagi bagian

**Tujuan Notebook:**
Mendeteksi pelanggan PLN yang dicurigai melakukan pencurian listrik menggunakan pendekatan Hybrid Semi-Supervised Learning.

**Data yang Digunakan:**
- 152,974 pelanggan dengan histori pemakaian 59 bulan
- 61 kasus fraud yang sudah terkonfirmasi (data temuan)

**ELIGIBILITY FILTERING:**
Model hanya memprediksi pelanggan yang memenuhi syarat:
- **ELIGIBLE:** 132,738 pelanggan (86.77%) - minimal 12 bulan data aktif dan masih berlangganan
- **INSUFFICIENT_HISTORY:** 5,916 pelanggan (3.87%) - riwayat < 12 bulan, tidak bisa diprediksi
- **INACTIVE_STOPPED:** 14,320 pelanggan (9.36%) - sudah berhenti berlangganan

**Hasil (hanya pelanggan ELIGIBLE):**
- 40 dari 41 fraud eligible terdeteksi (Recall 97.6%)
- 5,151 pelanggan diprediksi sebagai suspect (3.88%)
- 20 known fraud tidak eligible karena riwayat < 12 bulan

---

# BAGIAN 1: SETUP & IMPORT LIBRARIES {#bagian-1}

## Cell 1 (Markdown) - Judul Notebook
**Isi:** Header notebook dengan judul "DETEKSI PENCURIAN LISTRIK PLN"

**Penjelasan:**
Cell ini menampilkan judul proyek dan ringkasan pipeline yang akan dijalankan:
1. Load & Explore Data
2. Data Cleaning
3. Feature Engineering
4. Model Training
5. Evaluation & Prediction
6. Export Results
7. Save Models

---

## Cell 2 (Markdown) - Section Header
**Isi:** "## 1. SETUP & IMPORT LIBRARIES"

---

## Cell 3 (Code) - Import Libraries
**Kode:**
```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns
import warnings
import os
import json
import joblib
from datetime import datetime

from sklearn.ensemble import IsolationForest, RandomForestClassifier
from sklearn.preprocessing import StandardScaler, LabelEncoder
from imblearn.over_sampling import SMOTE
from xgboost import XGBClassifier

warnings.filterwarnings('ignore')
pd.set_option('display.max_columns', None)
print("[OK] Libraries imported successfully!")
```

**Output:**
```
[OK] Libraries imported successfully!
```

**Penjelasan Library yang Digunakan:**

| Library | Fungsi |
|---------|--------|
| `pandas` | Manipulasi data dalam bentuk DataFrame |
| `numpy` | Operasi matematika dan array |
| `matplotlib` | Membuat visualisasi/grafik |
| `seaborn` | Visualisasi statistik (tidak digunakan langsung) |
| `sklearn` | Library machine learning |
| `imblearn` | Library untuk handling imbalanced data (SMOTE) |
| `xgboost` | Algoritma XGBoost untuk klasifikasi |
| `joblib` | Menyimpan model ke file |

**Mengapa Library Ini Diperlukan:**
- `IsolationForest`: Mendeteksi anomali/outlier
- `RandomForestClassifier`: Model ensemble berbasis decision tree
- `StandardScaler`: Normalisasi fitur agar rentang sama
- `LabelEncoder`: Mengubah kategori ke angka
- `SMOTE`: Menyeimbangkan data training yang tidak seimbang
- `XGBClassifier`: Model gradient boosting yang powerful

---

# BAGIAN 2: LOAD DATA {#bagian-2}

## Cell 4 (Markdown) - Section Header
**Isi:** "## 2. LOAD DATA"

---

## Cell 5 (Code) - Load Data
**Kode:**
```python
df_temuan = pd.read_excel('../data/Temuan 2425.xlsx')
df_history = pd.read_excel('../data/Histori Pemakaian Pelanggan.xlsx')
display(df_temuan.head())
display(df_history.head())
```

**Output:**
```
Data Temuan: 52 rows
Data History: 152,974 rows

--- Data Temuan (Confirmed Fraud) ---
[Tabel 5 baris pertama data temuan: TGL TEMUAN, P & K, IDE, UE]

--- Data History (All Customers) ---
[Tabel 5 baris pertama data history: IDE, UE, 59 kolom tanggal pemakaian]
```

**Penjelasan Data:**

### Data Temuan (df_temuan)
| Kolom | Deskripsi |
|-------|-----------|
| TGL TEMUAN | Tanggal kasus ditemukan |
| P & K | Kode pelanggaran |
| IDE | ID unik pelanggan (terenkripsi) |
| UE | Unit Elektrik (wilayah) |

**Jumlah:** 52 baris = 52 kasus temuan (51 pelanggan unik, 1 pelanggan ada 2 kasus)

### Data History (df_history)
| Kolom | Deskripsi |
|-------|-----------|
| IDE | ID unik pelanggan |
| UE | Unit Elektrik |
| Kolom tanggal (59) | Pemakaian listrik per bulan (kWh) |

**Jumlah:** 152,974 baris = 152,974 pelanggan

**Insight:**
- Data history mencakup 59 bulan pemakaian listrik
- Setiap kolom tanggal berisi nilai kWh yang digunakan bulan tersebut
- Contoh: Kolom "2024-01-01" berisi pemakaian bulan Januari 2024

---

# BAGIAN 3: DATA ANALYSIS & LABELING {#bagian-3}

## Cell 6 (Markdown) - Section Header
**Isi:** "## 3. DATA ANALYSIS & LABELING"

---

## Cell 7 (Code) - Labeling
**Kode:**
```python
ide_temuan = df_temuan['IDE'].unique()
df_matched = df_history[df_history['IDE'].isin(ide_temuan)]
df_history['label'] = df_history['IDE'].isin(ide_temuan).astype(int)
```

**Output:**
```
Unique IDE in temuan: 51
Matched records in history: 61

Label Distribution:
  Fraud (1):   61
  Unknown (0): 152,913
```

**Penjelasan:**
Cell ini mencocokkan data temuan dengan data history untuk membuat label:
- **Label 1 (Fraud):** Pelanggan yang IDE-nya ada di data temuan = 61 record
- **Label 0 (Unknown):** Pelanggan yang belum diketahui statusnya = 152,913 record

**Mengapa 51 IDE menghasilkan 61 record?**
- Beberapa IDE memiliki lebih dari 1 record di data history
- Misalnya: 1 pelanggan punya 2 meteran/tarif berbeda

**Insight Penting:**
- Kita HANYA punya 61 data fraud yang terkonfirmasi
- 152,913 data lainnya berstatus "unknown" (bisa fraud atau normal)
- Ini adalah masalah **semi-supervised learning** karena tidak ada label "normal" yang pasti

---

# BAGIAN 4: DATA CLEANING {#bagian-4}

## Cell 8 (Markdown) - Section Header
**Isi:** "## 4. DATA CLEANING"

---

## Cell 9 (Code) - Data Cleaning dengan Strategi TRIMMING
**Kode:**
```python
date_columns = [col for col in df_history.columns if col not in ['IDE', 'UE', 'label']]

# STRATEGI TRIMMING KE PERIODE AKTIF
# N/A hanya muncul di awal (belum berlangganan) atau akhir (berhenti berlangganan)
# N/A BUKAN missing value - ini adalah periode tidak aktif
# Nilai 0 di tengah periode aktif adalah data REAL (tidak ada pemakaian)

def find_active_period(row):
    not_na_mask = ~pd.isna(row)
    if not_na_mask.sum() == 0:
        return None, None
    indices = np.where(not_na_mask)[0]
    return indices[0], indices[-1]

# Simpan periode aktif untuk setiap pelanggan
for idx, row in df_history[date_columns].iterrows():
    start_idx, end_idx = find_active_period(row.values)
    # Simpan ke kolom baru

df_history['active_start_idx'] = ...
df_history['active_end_idx'] = ...
df_history['active_months_count'] = ...

# JANGAN isi N/A dengan 0!
for col in date_columns:
    df_history[col] = pd.to_numeric(df_history[col], errors='coerce')
```

**Output:**
```
Date columns (monthly usage): 59
Kolom bulan pertama: 2021-03-01
Kolom bulan terakhir: 2026-01-01

Menghitung periode aktif setiap pelanggan...

============================================================
ELIGIBILITY FILTERING
============================================================
  ELIGIBLE: 132,738 (86.77%)
  INACTIVE_STOPPED: 14,320 (9.36%)
  INSUFFICIENT_HISTORY: 5,916 (3.87%)

[INFO] Statistik Periode Aktif (ELIGIBLE only):
  Min bulan aktif: 12
  Max bulan aktif: 59
  Rata-rata bulan aktif: 58.5

[WARNING] 20 known fraud TIDAK ELIGIBLE untuk prediksi!
  - INACTIVE_STOPPED: 0
  - INSUFFICIENT_HISTORY: 20

[INFO] Pelanggan yang TIDAK akan diprediksi:
  - INSUFFICIENT_HISTORY (< 12 bulan): 5,916
  - INACTIVE_STOPPED (sudah berhenti): 14,320

[OK] Data cleaned dengan TRIMMING + ELIGIBILITY strategy
     Total records: 152,974
     ELIGIBLE untuk prediksi: 132,738
     TIDAK eligible: 20,236
```

**Penjelasan Strategi TRIMMING + ELIGIBILITY:**

### Aturan Eligibility
| Status | Kondisi | Jumlah | Perlakuan |
|--------|---------|--------|-----------|
| **ELIGIBLE** | >= 12 bulan aktif DAN masih berlangganan | 132,738 (86.77%) | Diprediksi oleh model |
| **INSUFFICIENT_HISTORY** | < 12 bulan aktif | 5,916 (3.87%) | Tidak diprediksi - data belum cukup |
| **INACTIVE_STOPPED** | Kolom bulan terakhir = N/A | 14,320 (9.36%) | Tidak diprediksi - sudah berhenti |

### Mengapa Minimal 12 Bulan?
1. **Pola pencurian membutuhkan observasi jangka panjang** (minimal 1 tahun)
2. **Pola seperti "penurunan drastis lalu naik lagi" butuh waktu untuk terlihat**
3. **Pelanggan baru dengan 2-11 bulan data berisiko tinggi false positive/negative**
4. **Memprediksi dengan data < 12 bulan tidak akurat**

### Perbedaan N/A vs 0
| Nilai | Lokasi | Artinya | Perlakuan |
|-------|--------|---------|----------|
| **N/A** | Di awal | Pelanggan belum berlangganan | BUKAN data - jangan dihitung |
| **N/A** | Di akhir | Pelanggan sudah berhenti | INACTIVE_STOPPED - tidak diprediksi |
| **0** | Di tengah periode aktif | Tidak ada pemakaian bulan itu | DATA REAL - hitung sebagai pola |

### Mengapa TIDAK Mengganti N/A dengan 0?
1. **N/A di awal/akhir bukan missing value** - ini menandakan pelanggan tidak aktif
2. **Mengganti N/A dengan 0 akan mencampur periode tidak aktif dengan periode aktif**
3. **Nilai 0 di tengah periode aktif adalah sinyal penting** - bisa menandakan:
   - Rumah kosong (pemilik pergi)
   - Meteran diakali (pencurian listrik)
   - Gangguan teknis
4. **Feature engineering harus fokus pada periode aktif saja**

### Cara Kerja Trimming + Eligibility
1. Cari indeks bulan pertama yang BUKAN N/A (start)
2. Cari indeks bulan terakhir yang BUKAN N/A (end)
3. Hitung active_months = end - start + 1
4. Cek apakah bulan terakhir N/A (sudah berhenti?)
5. Tentukan eligibility:
   - Jika bulan terakhir N/A → INACTIVE_STOPPED
   - Jika active_months < 12 → INSUFFICIENT_HISTORY
   - Sisanya → ELIGIBLE

**Catatan Penting:**
- 20 dari 61 known fraud tidak eligible karena riwayat < 12 bulan
- Ini mungkin pelanggan baru yang tertangkap sebelum punya riwayat panjang
- Model hanya mengevaluasi 41 fraud yang eligible

---

# BAGIAN 5: FEATURE ENGINEERING {#bagian-5}

## Cell 10 (Markdown) - Section Header
**Isi:** "## 5. FEATURE ENGINEERING"

---

## Cell 11 (Code) - Feature Engineering dari PERIODE AKTIF
**Kode:**
```python
# FEATURE ENGINEERING DENGAN TRIMMING KE PERIODE AKTIF
# Fitur diekstraksi HANYA dari periode aktif setiap pelanggan
# Nilai 0 di tengah periode aktif = data real (tidak ada pemakaian)
# N/A di luar periode aktif = tidak diikutkan dalam perhitungan

def extract_active_period_data(row, date_cols):
    start_idx = int(row['active_start_idx'])
    end_idx = int(row['active_end_idx'])
    active_data = row[date_cols].values[start_idx:end_idx+1]
    return active_data

# Untuk setiap pelanggan, ekstrak data periode aktif
for idx, row in df_features.iterrows():
    active_data = extract_active_period_data(row, date_columns)
    
    # Hitung fitur dari PERIODE AKTIF saja
    usage_mean = np.mean(active_data)
    usage_std = np.std(active_data)
    zero_count = (active_data == 0).sum()  # Zero di periode aktif = data real!
    # ... dst
```

**Output:**
```
Mengekstrak fitur dari PERIODE AKTIF setiap pelanggan...
(Nilai 0 di tengah periode aktif = data real, bukan missing value)

[OK] Created 13 features dari PERIODE AKTIF:
  - usage_mean
  - usage_std
  - usage_min
  - usage_max
  - usage_range
  - coefficient_variation
  - zero_usage_count
  - max_drop
  - trend_slope
  - recent_disconnect
  - inactive_months
  - ue_encoded
  - ue_fraud_risk

[INFO] Statistik Zero Usage (dalam periode aktif):
  Pelanggan dengan 0 bulan zero usage: 111,090
  Pelanggan dengan >= 3 bulan zero usage: 33,399
  Pelanggan dengan >= 6 bulan zero usage: 29,847
  Max zero usage count: 59

[INFO] Rata-rata Zero Usage Count:
  Known Fraud (61): 20.16 bulan
  Unknown (152,913): 7.20 bulan
```

**INSIGHT PENTING dari Zero Usage:**
- **Known Fraud rata-rata 20.16 bulan dengan zero usage**
- **Unknown (non-fraud) rata-rata 7.20 bulan dengan zero usage**
- Fraud cenderung memiliki LEBIH BANYAK bulan tanpa pemakaian!
- Ini memperkuat hipotesis bahwa zero usage di tengah periode aktif = indikator fraud

**Penjelasan 13 Fitur yang Dibuat:**

### Fitur Statistik Dasar
| Fitur | Rumus | Artinya |
|-------|-------|---------|
| `usage_mean` | Rata-rata 59 bulan | Konsumsi rata-rata pelanggan |
| `usage_std` | Standar deviasi | Seberapa bervariasi pemakaian |
| `usage_min` | Nilai minimum | Pemakaian terendah |
| `usage_max` | Nilai maksimum | Pemakaian tertinggi |
| `usage_range` | max - min | Rentang pemakaian |

### Fitur Variabilitas
| Fitur | Rumus | Artinya |
|-------|-------|---------|
| `coefficient_variation` | std / mean | Tingkat ketidakstabilan (%) |

### Fitur Mencurigakan
| Fitur | Cara Hitung | Mengapa Mencurigakan |
|-------|-------------|---------------------|
| `zero_usage_count` | Hitung bulan = 0 | Banyak bulan 0 = meteran diakali? |
| `max_drop` | Penurunan terbesar | Drop drastis = ada manipulasi? |
| `trend_slope` | Kemiringan tren | Tren turun terus = mencurigakan |
| `recent_disconnect` | 3 bulan terakhir < 10 | Baru-baru ini tidak aktif |
| `inactive_months` | Bulan < 5 kWh | Total bulan tidak aktif |

### Fitur Lokasi
| Fitur | Cara Hitung | Artinya |
|-------|-------------|---------|
| `ue_encoded` | LabelEncoder(UE) | Kode wilayah dalam angka |
| `ue_fraud_risk` | fraud_count / total per UE | Risiko fraud di wilayah tersebut |

**Insight:**
- Fitur `ue_fraud_risk` paling penting (30% importance)
- Wilayah dengan banyak kasus fraud historis = lebih berisiko
- `zero_usage_count` tinggi sering menandakan kecurangan

---

# BAGIAN 6: MODEL TRAINING {#bagian-6}

## Cell 12 (Markdown) - Pipeline Overview
**Isi:** Penjelasan pipeline 5 langkah:
1. Isolation Forest → Deteksi anomali
2. Pseudo-labeling → Buat data training
3. SMOTE → Balance data
4. RF + XGBoost → Train model
5. Ensemble → Gabungkan prediksi

---

## Cell 13 (Markdown) - Subsection Header
**Isi:** "### 6.1 ISOLATION FOREST - Detect Anomaly"

---

## Cell 14 (Code) - Isolation Forest
**Kode:**
```python
X_all = df_features[feature_cols].values
scaler = StandardScaler()
X_all_scaled = scaler.fit_transform(X_all)

iso_forest = IsolationForest(
    n_estimators=200,
    contamination=0.01,
    random_state=42,
    n_jobs=-1
)
iso_forest.fit(X_all_scaled)
iso_predictions = iso_forest.predict(X_all_scaled)
```

**Output:**
```
============================================================
STEP 1: ISOLATION FOREST
============================================================
Total data: 152,974 pelanggan

Training Isolation Forest...

Results:
  Likely Normal: 151,444 (99.0%)
  Anomaly:       1,530 (1.0%)

Validation: 0/61 known fraud detected as anomaly
```

**Penjelasan:**
Isolation Forest adalah algoritma **unsupervised** untuk mendeteksi anomali.

**Cara Kerja:**
1. Membuat banyak decision tree secara random
2. Data yang "mudah diisolasi" (perlu sedikit split) = anomali
3. Data yang "sulit diisolasi" (perlu banyak split) = normal

**Parameter:**
- `n_estimators=200`: Jumlah tree
- `contamination=0.01`: Estimasi 1% data adalah anomali

**Hasil:**
- 99% pelanggan dianggap "likely normal"
- 1% pelanggan dianggap "anomaly"

**Catatan Penting:**
- Validasi menunjukkan 0/61 fraud terdeteksi sebagai anomali
- Ini karena fraud yang sudah tertangkap pola pemakaiannya "mirip normal"
- Isolation Forest digunakan untuk mencari "normal samples", bukan fraud

---

## Cell 15 (Markdown) - Subsection Header
**Isi:** "### 6.2 PSEUDO-LABELING - Create Training Data"

---

## Cell 16 (Code) - Pseudo-Labeling
**Kode:**
```python
likely_normal_mask = (df_features['iso_prediction'] == 1) & (df_features['label'] != 1)
df_likely_normal = df_features[likely_normal_mask]
df_pseudo_normal = df_likely_normal.sample(n=5000, random_state=42)

df_confirmed_fraud = df_features[df_features['label'] == 1].copy()
df_train = pd.concat([df_pseudo_normal, df_confirmed_fraud])
```

**Output:**
```
============================================================
STEP 2: PSEUDO-LABELING
============================================================
Likely Normal pool: 151,383
Sampled pseudo-normal: 5000
Confirmed fraud: 61

Training Dataset:
  Normal: 5000
  Fraud:  61
  Ratio:  1:81
```

**Penjelasan:**
Pseudo-labeling adalah teknik untuk membuat label "sementara" pada data yang tidak berlabel.

**Logika:**
1. Pelanggan yang **bukan anomali** dan **bukan fraud** = diasumsikan "normal"
2. Ambil 5,000 sample dari mereka sebagai "pseudo-normal"
3. Gabungkan dengan 61 confirmed fraud

**Hasil:**
- 5,000 data dengan label "normal" (pseudo)
- 61 data dengan label "fraud" (confirmed)
- Rasio 1:81 (sangat tidak seimbang!)

**Mengapa Pseudo-Labeling?**
- Kita tidak punya data "normal" yang pasti
- Pelanggan yang tidak pernah tertangkap diasumsikan normal
- Ini adalah pendekatan **semi-supervised learning**

---

## Cell 17 (Markdown) - Subsection Header
**Isi:** "### 6.3 SMOTE - Balance Training Data"

---

## Cell 18 (Code) - SMOTE Balancing
**Kode:**
```python
X_train_scaled = scaler.transform(X_train)
smote = SMOTE(sampling_strategy=1.0, random_state=42, k_neighbors=4)
X_train_balanced, y_train_balanced = smote.fit_resample(X_train_scaled, y_train)
```

**Output:**
```
============================================================
STEP 3: SMOTE BALANCING
============================================================
Before SMOTE: Normal=5000, Fraud=61
After SMOTE:  Normal=5000, Fraud=5000
[OK] Data balanced 1:1
```

**Penjelasan:**
SMOTE (Synthetic Minority Over-sampling Technique) membuat data sintetis untuk menyeimbangkan kelas.

**Cara Kerja:**
1. Pilih sample dari kelas minoritas (fraud)
2. Cari K tetangga terdekat (k_neighbors=4)
3. Buat data sintetis di antara sample dan tetangganya
4. Ulangi sampai rasio 1:1

**Hasil:**
- Before: 5,000 normal vs 61 fraud (1:81)
- After: 5,000 normal vs 5,000 fraud (1:1)

**Mengapa SMOTE Diperlukan?**
- Model cenderung "malas" jika data tidak seimbang
- Model akan prediksi semua sebagai "normal" karena akurasi sudah 99%
- SMOTE memaksa model belajar pola fraud dengan lebih baik

---

## Cell 19 (Markdown) - Subsection Header
**Isi:** "### 6.4 TRAIN MODELS - Random Forest & XGBoost"

---

## Cell 20 (Code) - Train Models
**Kode:**
```python
# Random Forest
rf_model = RandomForestClassifier(
    n_estimators=300,
    max_depth=15,
    min_samples_split=5,
    min_samples_leaf=2,
    random_state=42,
    n_jobs=-1
)
rf_model.fit(X_train_balanced, y_train_balanced)

# XGBoost
xgb_model = XGBClassifier(
    n_estimators=200,
    max_depth=8,
    learning_rate=0.1,
    subsample=0.8,
    colsample_bytree=0.8,
    random_state=42
)
xgb_model.fit(X_train_balanced, y_train_balanced)
```

**Output:**
```
============================================================
STEP 4: TRAIN MODELS
============================================================

Training Random Forest (300 trees)...
[OK] Random Forest trained!

Training XGBoost (200 estimators)...
[OK] XGBoost trained!
```

**Penjelasan Model:**

### Random Forest
| Parameter | Nilai | Artinya |
|-----------|-------|---------|
| n_estimators | 300 | 300 decision trees |
| max_depth | 15 | Maksimum kedalaman tree |
| min_samples_split | 5 | Minimal 5 sample untuk split |
| min_samples_leaf | 2 | Minimal 2 sample di leaf |

**Cara Kerja Random Forest:**
1. Buat 300 decision tree dengan data random
2. Setiap tree memberikan "vote" untuk prediksi
3. Hasil akhir = mayoritas vote

### XGBoost
| Parameter | Nilai | Artinya |
|-----------|-------|---------|
| n_estimators | 200 | 200 boosting iterations |
| max_depth | 8 | Kedalaman tree |
| learning_rate | 0.1 | Kecepatan belajar |
| subsample | 0.8 | Gunakan 80% data per tree |

**Cara Kerja XGBoost:**
1. Buat tree pertama, hitung error
2. Tree berikutnya fokus memperbaiki error
3. Ulangi 200 kali dengan bobot error

---

## Cell 21 (Markdown) - Subsection Header
**Isi:** "### 6.5 PREDICT - Ensemble Averaging"

---

## Cell 22 (Code) - Prediction
**Kode:**
```python
rf_proba = rf_model.predict_proba(X_all_scaled)[:, 1]
xgb_proba = xgb_model.predict_proba(X_all_scaled)[:, 1]
ensemble_proba = (rf_proba + xgb_proba) / 2
df_features['prediction'] = (ensemble_proba >= 0.5).astype(int)
```

**Output:**
```
============================================================
STEP 5: PREDICTION
============================================================

Generating predictions on all data...
[OK] Predictions complete!

Prediction Summary:
  Predicted Fraud:  6,541 (4.28%)
  Predicted Normal: 146,433 (95.72%)
```

**Penjelasan:**
Ensemble averaging menggabungkan prediksi dari 2 model.

**Cara Kerja:**
1. Random Forest → Probabilitas fraud (0-1)
2. XGBoost → Probabilitas fraud (0-1)
3. Ensemble = (RF + XGB) / 2
4. Jika ensemble >= 0.5 → Prediksi "Fraud"

**Hasil:**
- 6,541 pelanggan diprediksi sebagai fraud (4.28%)
- 146,433 pelanggan diprediksi normal (95.72%)

**Mengapa Ensemble?**
- Mengurangi overfitting dari satu model
- Meningkatkan akurasi dan stabilitas
- Jika kedua model setuju → prediksi lebih reliable

---

# BAGIAN 7: MODEL EVALUATION {#bagian-7}

## Cell 23 (Markdown) - Section Header
**Isi:** "## 7. MODEL EVALUATION"

---

## Cell 24 (Code) - Basic Evaluation
**Kode:**
```python
known_fraud = df_features[df_features['label'] == 1]
detected = (known_fraud['prediction'] == 1).sum()
recall = detected / len(known_fraud)
```

**Output:**
```
============================================================
MODEL EVALUATION
============================================================

Known Fraud Detection:
  Detected: 59/61
  Recall:   96.7%

Fraud Probability Distribution (known fraud):
  > 0.8:    50
  0.5-0.8:  9
  < 0.5:    2 (missed)
```

**Penjelasan:**
Evaluasi dilakukan pada 61 fraud yang sudah terkonfirmasi.

**Hasil:**
- 59 dari 61 fraud berhasil dideteksi
- Recall = 96.7%
- Hanya 2 fraud yang tidak terdeteksi (probability < 0.5)

**Distribusi Probabilitas:**
- 50 fraud memiliki prob > 80% (sangat yakin)
- 9 fraud memiliki prob 50-80% (cukup yakin)
- 2 fraud memiliki prob < 50% (missed)

---

## Cell 25 (Markdown) - Subsection Header
**Isi:** "### 7.1 Detailed Metrics & Model Comparison"

---

## Cell 26 (Code) - Detailed Metrics
**Kode:**
```python
from sklearn.metrics import precision_score, recall_score, f1_score, accuracy_score, confusion_matrix
# ... perhitungan metrics lengkap
```

**Output:**
```
======================================================================
DETAILED METRICS & MODEL COMPARISON
======================================================================

======================================================================
1. EVALUATION ON KNOWN FRAUD (61 records)
======================================================================

RECALL COMPARISON (on 61 known fraud):
  Random Forest : 56/61 → 91.8%
  XGBoost       : 59/61 → 96.7%
  Ensemble      : 59/61 → 96.7%

======================================================================
2. EVALUATION ON TRAINING DATA (10,000 records)
======================================================================

[RF] RANDOM FOREST:
  Accuracy  : 98.02%
  Precision : 96.30%
  Recall    : 99.88%
  F1-Score  : 98.06%

[XGB] XGBOOST:
  Accuracy  : 99.95%
  Precision : 99.96%
  Recall    : 99.94%
  F1-Score  : 99.95%

[ENS] ENSEMBLE (RF + XGBoost Average):
  Accuracy  : 99.71%
  Precision : 99.48%
  Recall    : 99.94%
  F1-Score  : 99.71%

======================================================================
3. CONFUSION MATRIX (Training Data)
======================================================================

[RF] RANDOM FOREST:
  True Negative : 4,808  |  False Positive:   192
  False Negative:     6  |  True Positive : 4,994

[XGB] XGBOOST:
  True Negative : 4,998  |  False Positive:     2
  False Negative:     3  |  True Positive : 4,997

[ENS] ENSEMBLE:
  True Negative : 4,974  |  False Positive:    26
  False Negative:     3  |  True Positive : 4,997

======================================================================
4. MODEL AGREEMENT
======================================================================

RF vs XGBoost Agreement:
  Training Data : 9,807/10,000 (98.1%)
  Known Fraud   :    58/    61 (95.1%)

======================================================================
5. SUMMARY TABLE
======================================================================

        Model Accuracy Precision Recall F1-Score Known Fraud Recall
Random Forest   98.02%    96.30% 99.88%   98.06%              91.8%
      XGBoost   99.95%    99.96% 99.94%   99.95%              96.7%
     Ensemble   99.71%    99.48% 99.94%   99.71%              96.7%
```

**Penjelasan Metrics:**

### Accuracy, Precision, Recall, F1-Score
| Metric | Artinya | Nilai Ensemble |
|--------|---------|----------------|
| Accuracy | % prediksi yang benar | 99.71% |
| Precision | % prediksi fraud yang benar | 99.48% |
| Recall | % fraud yang terdeteksi | 99.94% |
| F1-Score | Harmonic mean precision & recall | 99.71% |

### Confusion Matrix
```
                    Predicted Normal    Predicted Fraud
Actual Normal          TN (4,974)         FP (26)
Actual Fraud           FN (3)             TP (4,997)
```

- **True Negative (TN):** Normal, prediksi Normal ✓
- **True Positive (TP):** Fraud, prediksi Fraud ✓
- **False Positive (FP):** Normal, salah prediksi Fraud
- **False Negative (FN):** Fraud, tidak terdeteksi

### Perbandingan Model
- XGBoost sedikit lebih baik dari Random Forest
- Ensemble mengambil keuntungan dari keduanya
- Agreement 98.1% artinya kedua model hampir selalu setuju

---

# BAGIAN 8: SUSPECT RANKING {#bagian-8}

## Cell 27 (Markdown) - Section Header
**Isi:** "## 8. SUSPECT RANKING"

---

## Cell 28 (Code) - Suspect Ranking
**Kode:**
```python
df_suspects = df_features[df_features['label'] != 1].copy()
df_suspects = df_suspects.sort_values('ensemble_fraud_prob', ascending=False)

df_suspects['priority'] = pd.cut(
    df_suspects['ensemble_fraud_prob'],
    bins=[0, 0.3, 0.5, 0.7, 1.0],
    labels=['LOW', 'MEDIUM', 'HIGH', 'CRITICAL']
)
```

**Output:**
```
============================================================
SUSPECT RANKING
============================================================

Priority Distribution:
  CRITICAL  :   3,308 ( 2.16%)
  HIGH      :   3,174 ( 2.08%)
  MEDIUM    :   5,885 ( 3.85%)
  LOW       : 140,546 (91.91%)

------------------------------------------------------------
TOP 20 SUSPECTS:
------------------------------------------------------------
[Tabel 20 suspect teratas dengan IDE, UE, probability, priority]
```

**Penjelasan Priority Tiers:**

| Priority | Probabilitas | Jumlah | Artinya |
|----------|--------------|--------|---------|
| CRITICAL | >= 70% | 3,308 (2.16%) | Sangat mencurigakan |
| HIGH | 50% - 69.9% | 3,174 (2.08%) | Cukup mencurigakan |
| MEDIUM | 30% - 49.9% | 5,885 (3.85%) | Agak mencurigakan |
| LOW | < 30% | 140,546 (91.91%) | Kemungkinan normal |

**Note:** Hanya pelanggan yang BUKAN known fraud yang dimasukkan ke ranking.

---

# BAGIAN 9: VISUALIZATION {#bagian-9}

## Cell 29 (Markdown) - Section Header
**Isi:** "## 9. VISUALIZATION"

---

## Cell 30 (Code) - Visualization
**Output:** 4 grafik dalam satu figure

### Grafik 1: Distribution of Fraud Probability (Kiri Atas)
**Tipe:** Histogram

**Penjelasan:**
- Sumbu X: Probabilitas fraud (0-1)
- Sumbu Y: Jumlah pelanggan
- Garis merah: Threshold 0.5
- Mayoritas pelanggan memiliki prob rendah (< 0.1)
- Sedikit pelanggan memiliki prob tinggi (> 0.5)

**Insight:**
- Distribusi sangat skewed ke kiri (kebanyakan normal)
- Pelanggan dengan prob > 0.5 adalah suspect

### Grafik 2: Known Fraud Probability Distribution (Kanan Atas)
**Tipe:** Histogram

**Penjelasan:**
- Hanya menampilkan 61 known fraud
- Garis hijau: Threshold 0.5
- Mayoritas di area > 0.8

**Insight:**
- 50 fraud memiliki prob > 0.8 (berhasil dideteksi dengan yakin)
- 9 fraud memiliki prob 0.5-0.8
- 2 fraud memiliki prob < 0.5 (missed)

### Grafik 3: Model Agreement RF vs XGBoost (Kiri Bawah)
**Tipe:** Scatter Plot

**Penjelasan:**
- Sumbu X: Probabilitas dari Random Forest
- Sumbu Y: Probabilitas dari XGBoost
- Garis merah: Garis diagonal (perfect agreement)

**Insight:**
- Titik-titik dekat garis diagonal = kedua model setuju
- Titik jauh dari diagonal = model tidak setuju
- Mayoritas titik dekat diagonal = agreement tinggi (98.1%)

### Grafik 4: Top 10 Feature Importance (Kanan Bawah)
**Tipe:** Horizontal Bar Chart

**Penjelasan:**
- Menampilkan 10 fitur paling berpengaruh
- Rata-rata importance dari RF dan XGBoost

**Hasil Feature Importance:**
| Rank | Feature | Importance |
|------|---------|------------|
| 1 | ue_fraud_risk | ~30% |
| 2 | ue_encoded | ~17% |
| 3 | inactive_months | ~9% |
| 4 | recent_disconnect | ~8% |
| 5 | zero_usage_count | ~7% |
| 6 | usage_min | ~6% |
| 7 | coefficient_variation | ~6% |
| 8 | trend_slope | ~5% |
| 9 | usage_std | ~5% |
| 10 | max_drop | ~4% |

**Insight:**
- `ue_fraud_risk` paling dominan (30%)
- Lokasi/wilayah sangat berpengaruh terhadap risiko fraud
- Fitur terkait "tidak aktif" (inactive_months, recent_disconnect, zero_usage_count) penting

---

# BAGIAN 10: EXPORT RESULTS {#bagian-10}

## Cell 31 (Markdown) - Section Header
**Isi:** "## 10. EXPORT RESULTS"

---

## Cell 32 (Code) - Export Results
**Output:**
```
============================================================
EXPORT RESULTS
============================================================
[OK] Saved: results/suspects_top_500.xlsx (with usage pattern)
[OK] Saved: results/suspects_high_priority.xlsx (6,482 records)
[OK] Saved: results/suspects_all.xlsx (12,367 records)

============================================================
KOLOM YANG DIEKSPOR:
============================================================
1. IDE, UE: Identitas pelanggan
2. priority: CRITICAL/HIGH/MEDIUM/LOW
3. ensemble_fraud_prob: Probabilitas fraud (0-1)
4. usage_mean, usage_std, dll: Fitur statistik pemakaian
5. kWh_Feb2025 - kWh_Jan2026: Pola pemakaian listrik 12 bulan terakhir (kWh)
```

**File yang Dihasilkan:**

| File | Isi | Jumlah |
|------|-----|--------|
| suspects_top_500.xlsx | 500 suspect teratas | 500 |
| suspects_high_priority.xlsx | CRITICAL + HIGH | 6,482 |
| suspects_all.xlsx | Prob >= 30% | 12,367 |

**Kolom dalam File Excel:**
- IDE, UE: Identitas pelanggan
- priority: Tingkat kecurigaan
- ensemble_fraud_prob: Probabilitas
- Fitur statistik: usage_mean, usage_std, dll
- **Pola bulanan: kWh_Feb2025 - kWh_Jan2026** (12 bulan terakhir)

---

# BAGIAN 11: SAVE MODELS {#bagian-11}

## Cell 33 (Markdown) - Section Header
**Isi:** "## 11. SAVE MODELS"

---

## Cell 34 (Code) - Save Models
**Output:**
```
============================================================
SAVE MODELS
============================================================
[OK] Saved: models/isolation_forest.joblib
[OK] Saved: models/random_forest.joblib
[OK] Saved: models/xgboost.joblib
[OK] Saved: models/scaler.joblib
[OK] Saved: models/label_encoder_ue.joblib
[OK] Saved: models/metadata.json
```

**File Model yang Disimpan:**

| File | Ukuran | Fungsi |
|------|--------|--------|
| isolation_forest.joblib | ~5-10 MB | Deteksi anomali |
| random_forest.joblib | ~50-100 MB | Klasifikasi fraud |
| xgboost.joblib | ~10-20 MB | Klasifikasi fraud |
| scaler.joblib | ~1-5 KB | Normalisasi fitur |
| label_encoder_ue.joblib | ~1-10 KB | Encoding UE |
| metadata.json | ~1 KB | Info training |

---

## Cell 35 (Markdown) - Summary
**Isi:** Ringkasan struktur folder dan cara menggunakan model

---

# ALUR LENGKAP PIPELINE {#alur-pipeline}

```
                            ALUR DETEKSI FRAUD
                            ==================

    ┌─────────────────────────────────────────────────────────────┐
    │                    1. DATA PREPARATION                       │
    ├─────────────────────────────────────────────────────────────┤
    │                                                             │
    │  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐   │
    │  │ Load Data   │────>│ Labeling    │────>│ Cleaning    │   │
    │  │ (152,974)   │     │ (61 fraud)  │     │ (fillna)    │   │
    │  └─────────────┘     └─────────────┘     └─────────────┘   │
    │                                                   │         │
    │                                                   v         │
    │                              ┌─────────────────────────┐    │
    │                              │  Feature Engineering    │    │
    │                              │  (13 fitur)             │    │
    │                              └─────────────────────────┘    │
    └─────────────────────────────────────────────────────────────┘
                                          │
                                          v
    ┌─────────────────────────────────────────────────────────────┐
    │                    2. MODEL TRAINING                         │
    ├─────────────────────────────────────────────────────────────┤
    │                                                             │
    │  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐   │
    │  │ Isolation   │────>│ Pseudo-     │────>│ SMOTE       │   │
    │  │ Forest      │     │ Labeling    │     │ Balancing   │   │
    │  │ (99% normal)│     │ (5000+61)   │     │ (1:1)       │   │
    │  └─────────────┘     └─────────────┘     └─────────────┘   │
    │                                                   │         │
    │                                                   v         │
    │            ┌─────────────┐     ┌─────────────┐             │
    │            │ Random      │     │ XGBoost     │             │
    │            │ Forest      │     │ Classifier  │             │
    │            │ (300 trees) │     │ (200 est.)  │             │
    │            └─────────────┘     └─────────────┘             │
    │                    │                   │                    │
    │                    └─────────┬─────────┘                    │
    │                              v                              │
    │                    ┌─────────────────┐                      │
    │                    │    ENSEMBLE     │                      │
    │                    │    AVERAGE      │                      │
    │                    └─────────────────┘                      │
    └─────────────────────────────────────────────────────────────┘
                                          │
                                          v
    ┌─────────────────────────────────────────────────────────────┐
    │                    3. EVALUATION & OUTPUT                    │
    ├─────────────────────────────────────────────────────────────┤
    │                                                             │
    │  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐   │
    │  │ Prediction  │────>│ Evaluation  │────>│ Ranking     │   │
    │  │ (152,974)   │     │ (96.7%      │     │ (CRITICAL/  │   │
    │  │             │     │  recall)    │     │  HIGH/...)  │   │
    │  └─────────────┘     └─────────────┘     └─────────────┘   │
    │                                                   │         │
    │                    ┌──────────────────────────────┘         │
    │                    v                                        │
    │  ┌─────────────────────────────────────────────────────┐    │
    │  │                    OUTPUT                            │    │
    │  ├─────────────────────────────────────────────────────┤    │
    │  │  Excel Files:                                        │    │
    │  │   - suspects_top_500.xlsx                           │    │
    │  │   - suspects_high_priority.xlsx (6,482)             │    │
    │  │   - suspects_all.xlsx (12,367)                      │    │
    │  │                                                      │    │
    │  │  Model Files:                                        │    │
    │  │   - random_forest.joblib                            │    │
    │  │   - xgboost.joblib                                  │    │
    │  │   - scaler.joblib                                   │    │
    │  │   - metadata.json                                   │    │
    │  └─────────────────────────────────────────────────────┘    │
    └─────────────────────────────────────────────────────────────┘
```

---

# RINGKASAN HASIL

## Eligibility Summary
| Kategori | Jumlah | Persentase | Keterangan |
|----------|--------|------------|------------|
| **ELIGIBLE** | 132,738 | 86.77% | Dapat diprediksi (>= 12 bulan aktif) |
| INSUFFICIENT_HISTORY | 5,916 | 3.87% | Riwayat < 12 bulan |
| INACTIVE_STOPPED | 14,320 | 9.36% | Sudah berhenti berlangganan |
| **Total** | 152,974 | 100% | |

## Model Performance (ELIGIBLE only)
| Metrik | Nilai |
|--------|-------|
| Total Pelanggan ELIGIBLE | 132,738 |
| Known Fraud ELIGIBLE | 41 |
| Fraud Terdeteksi | 40/41 (97.6%) |
| Predicted Fraud | 5,151 (3.88%) |
| Predicted Normal | 127,587 (96.12%) |

## Priority Distribution (ELIGIBLE only)
| Priority | Jumlah | Persentase |
|----------|--------|------------|
| CRITICAL | 2,428 | 1.83% |
| HIGH | 2,683 | 2.02% |
| MEDIUM | 5,028 | 3.79% |
| LOW | 122,558 | 92.36% |

## File Output

### Pelanggan ELIGIBLE (dapat diprediksi)
| File | Isi | Jumlah |
|------|-----|--------|
| suspects_top_500.xlsx | Top 500 suspect | 500 |
| suspects_high_priority.xlsx | CRITICAL + HIGH | 5,111 |
| suspects_all.xlsx | prob >= 30% | 10,139 |

### Pelanggan TIDAK ELIGIBLE
| File | Isi | Jumlah |
|------|-----|--------|
| customers_not_eligible.xlsx | Semua tidak eligible | 20,236 |
| customers_insufficient_history.xlsx | Riwayat < 12 bulan | 5,916 |
| customers_inactive_stopped.xlsx | Sudah berhenti | 14,320 |

---

**Dibuat:** 12 Januari 2026
**Notebook:** electricity_theft_detection.ipynb
**Total Cells:** 35 (14 Markdown + 16 Code)
