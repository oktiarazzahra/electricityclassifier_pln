# Electricity Theft Detection - Machine Learning Project

Project untuk mendeteksi pencurian listrik menggunakan Machine Learning dengan menganalisis pola pemakaian historis pelanggan PLN.

## Struktur Project

```
ElectricityClassifier/
├── data/
│   ├── Temuan 2425.xlsx           # Data temuan fraud (52 records)
│   └── Histori Pemakaian Pelanggan.xlsx  # Data historis (152,974 records)
├── models/                         # Model yang tersimpan (akan dibuat saat training)
├── notebooks/
│   └── electricity_theft_detection.ipynb  # Main analysis notebook
├── results/                        # Output hasil prediksi (akan dibuat)
├── PENJELASAN_KONSEP.md           # Dokumentasi lengkap konsep project
└── README.md                       # File ini
```

## Dataset

### Data Temuan 2425.xlsx
- Jumlah: 52 records
- Periode: 2024-2025
- Status: Pelanggan yang sudah terbukti mencuri listrik (investigasi lapangan)

### Data Histori Pemakaian Pelanggan.xlsx
- Jumlah: 152,974 records
- Periode: 2021-2026 (59 bulan)
- Isi: Data pemakaian listrik bulanan semua pelanggan

## Metodologi

### 1. Data Preparation
- Load dan match data temuan dengan data historis
- Data cleaning dan handling missing values
- Labelling strategy: 3 kategori (Fraud, Safe Normal, Unknown)

### 2. Feature Engineering
13 features diekstrak dari pola pemakaian:
- Statistical features (mean, std, CV)
- Pattern features (drop, spike, trend)
- Disconnect features (inactive, zero usage)
- Range features (max, min, range)

### 3. Machine Learning (Tahap Berikutnya)
- Algorithms: Random Forest & XGBoost
- Techniques: SMOTE untuk balance dataset
- Metrics: F1-Score, Precision, Recall, ROC-AUC

### 4. Output
- Ranking pelanggan suspicious
- Risk level classification
- Top 100 prioritas investigasi

## Requirements

```
pandas >= 2.3.0
numpy >= 2.4.0
scikit-learn >= 1.8.0
xgboost >= 3.1.0
matplotlib >= 3.9.0
seaborn >= 0.13.0
openpyxl >= 3.1.0
imbalanced-learn >= 0.12.0
```

## Installation

```bash
cd ElectricityClassifier
pip install pandas numpy scikit-learn imbalanced-learn xgboost matplotlib seaborn openpyxl
```

## Usage

1. Buka notebook: `notebooks/electricity_theft_detection.ipynb`
2. Execute cells secara berurutan (Cell 1-14)
3. Lihat hasil EDA dan feature engineering
4. (Nanti) Lanjutkan dengan model training dan prediction

## Dokumentasi

Untuk penjelasan lengkap konsep, strategi labelling, dan alur kerja, baca:
**[PENJELASAN_KONSEP.md](PENJELASAN_KONSEP.md)**

## Status Project

### Completed
- [x] Data loading dan matching
- [x] Data cleaning
- [x] Smart labelling strategy
- [x] Feature engineering (13 features)
- [x] Exploratory Data Analysis

### Pending
- [ ] Model training (Random Forest & XGBoost)
- [ ] Temporal anomaly detection
- [ ] Model evaluation
- [ ] Prediction untuk unknown customers
- [ ] Ranking dan export hasil
- [ ] Model deployment

## License

Project untuk internal PLN - Electricity theft detection analysis

---

**Note:** Untuk pertanyaan atau penjelasan lebih lanjut, lihat file [PENJELASAN_KONSEP.md](PENJELASAN_KONSEP.md)
