# ⚡ Electricity Theft Detection System

Sistem deteksi pencurian listrik menggunakan **21 fitur optimized** dengan Machine Learning (Random Forest + XGBoost Ensemble)

## 📊 Project Overview

Sistem ini mendeteksi **3 pola pencurian listrik utama**:
1. **Zero Fraud** - Bypass meter / meter mati
2. **Gradual Decline** - Manipulasi meter bertahap  
3. **Sudden Spike** - Tapping jaringan listrik

**Performance Target:**
- 🎯 Target: **94-97% accuracy**, **<12% false positive**
- ✅ Achieved: **86% accuracy**, **20% recall**, **13% FPR**

---

## 📁 Project Structure

```
ElectricityClassifier/
│
├── data/
│   ├── Histori Pemakaian Pelanggan_rev0A..xlsx  # 152,948 customers × 59 months
│   ├── Temuan_rev0A.xlsx                        # 168 known fraud cases
│   └── Normal.xlsx                              # 9,810 confirmed normal customers
│
├── notebooks/
│   └── electricity_theft_detection_clean.ipynb  # Main pipeline (27 cells)
│
├── results/
│   └── Prediction_Results_21Features.xlsx        # Output predictions
│
├── DOKUMENTASI_21_FITUR.md                       # Detailed feature documentation
├── README.md                                      # This file
└── requirements.txt
```

---

## 🚀 Quick Start

### 1. Installation

```bash
# Create virtual environment
python -m venv venv
venv\Scripts\activate  # Windows
# source venv/bin/activate  # Linux/Mac

# Install dependencies
pip install pandas numpy scikit-learn xgboost imbalanced-learn openpyxl scipy
```

### 2. Run Pipeline

Open `notebooks/electricity_theft_detection_clean.ipynb` and run all cells:

```python
# Total runtime: ~15 minutes
# Output: results/Prediction_Results_21Features.xlsx
```

### 3. Review Results

```python
# Load high priority cases
import pandas as pd
df = pd.read_excel('results/Prediction_Results_21Features.xlsx', 
                   sheet_name='High Priority')

# Show top suspects
print(df[df['priority'] == 'CRITICAL'].head(10))
```

---

## 🎯 Key Results

### Data Summary
- **Total customers analyzed:** 139,044 (after removing 13,904 unsubscribed)
- **Training data:** 9,978 (168 Fraud + 9,810 Normal)
- **Prediction data:** 129,066 unknown customers

### Prediction Results
- **Predicted fraud:** 9,640 customers (7.47%)
- **High priority cases:** 2,935 (ROI ≥ 60)
  - CRITICAL: 266 cases (ROI ≥ 80)
  - HIGH: 2,669 cases (ROI 60-80)

### Financial Impact
- **Total estimated monthly loss:** Rp 14.68 billion
- **High priority loss:** Rp 1.27 billion (8.7%)

---

## 🔧 Features (21 Total)

### TIER S: Universal (5 features)
Relevan untuk semua pelanggan

1. **tariff_zscore** - Z-score vs tarif group
2. **usage_vs_expected** - Ratio actual/expected usage
3. **neighborhood_percentile** - Percentile in UE group
4. **cv** - Coefficient of variation (stability)
5. **zero_fraud_score** - Composite zero detection score

### TIER A: Pattern Detection (10 features)
Deteksi pola temporal mencurigakan

6. **plateau_months** - Months stuck at same value
7. **gradual_decline** - Gradual 20%+ decline detected
8. **max_consecutive_zero** - Longest zero streak
9. **zero_clusters** - Count of zero clusters (2+ months)
10. **extreme_spike_count** - Spikes >200% baseline
11. **sudden_drop_count** - Drops >40% month-to-month
12. **consecutive_below_threshold** - Months below 70% baseline
13. **drop_recovery_pattern** - Drop → recovery cycles
14. **variance_spike_count** - Variance increases >2x
15. **max_drop_from_baseline** - Maximum % drop

### TIER B: Tapping & ROI (3 features)
Financial impact & prioritization

16. **sudden_spike_sustained** - Spike >50% sustained >6 months
17. **estimated_monthly_loss** - Estimated loss in Rupiah
18. **investigation_roi** - Priority score (0-100)

### TIER C: Enhanced Zero (3 features)
Advanced zero pattern analysis

19. **zero_pattern_legitimacy** - Legitimacy score (0-1)
20. **zero_to_nonzero_transition** - Count of 0→non-zero transitions
21. **post_zero_spike_severity** - Max recovery ratio after zero

📖 **[Full Feature Documentation →](DOKUMENTASI_21_FITUR.md)**

---

## 📈 Model Architecture

### Models Used
1. **Random Forest** (200 estimators, max_depth=15)
2. **XGBoost** (200 estimators, max_depth=8, lr=0.1)
3. **Ensemble** - Average probability from both models

### Training Process
1. ✅ SMOTE balancing (58:1 → 1:1 ratio)
2. ✅ StandardScaler normalization
3. ✅ Ensemble voting (RF + XGBoost)

### Performance Metrics
```
              precision    recall  f1-score   support
      Normal       0.98      0.87      0.93      9,810
       Fraud       0.03      0.20      0.05        168

    accuracy                           0.86      9,978
```

**Interpretation:**
- High accuracy (86%) on majority class
- 20% recall on fraud (catches 1 in 5 actual frauds)
- Low precision (3%) acceptable for **screening system**
- Field validation needed for final confirmation

---

## 🔍 Usage Example

### Filter High Priority Cases

```python
import pandas as pd

# Load results
df = pd.read_excel('results/Prediction_Results_21Features.xlsx', 
                   sheet_name='All Predictions')

# Get CRITICAL cases (ROI ≥ 80)
critical = df[df['priority'] == 'CRITICAL'].sort_values('roi_score', ascending=False)

print(f"CRITICAL cases to investigate: {len(critical)}")
print(critical[['IE', 'UE', 'TARIF', 'fraud_probability', 'roi_score', 'estimated_monthly_loss']].head(10))
```

### Analyze Fraud Patterns

```python
# Count by pattern type
print("Pattern Analysis:")
print(f"Zero fraud (score > 60): {(critical['zero_fraud_score'] > 60).sum()}")
print(f"Gradual decline: {(critical['gradual_decline'] == 1).sum()}")
print(f"Sudden spike: {(critical['sudden_spike_sustained'] == 1).sum()}")

# Financial impact
print(f"\nTotal monthly loss: Rp {critical['estimated_monthly_loss'].sum():,.0f}")
```

### Export for Field Team

```python
# Select columns for field inspection
export_cols = ['IE', 'UE', 'TARIF', 'priority', 'roi_score', 
               'fraud_probability', 'estimated_monthly_loss',
               'zero_fraud_score', 'max_consecutive_zero']

field_list = critical[export_cols].copy()
field_list.to_excel('field_inspection_list.xlsx', index=False)
```

---

## 📊 Output Files

### Prediction_Results_21Features.xlsx

Contains 3 sheets:

1. **All Predictions** (129,066 rows)
   - All customers with predictions and 21 features
   - Columns: IE, UE, TARIF, fraud_probability, roi_score, priority, etc.

2. **High Priority** (2,935 rows)
   - Filtered: ROI ≥ 60 (CRITICAL + HIGH)
   - Sorted by roi_score descending
   - Ready for immediate investigation

3. **Summary**
   - Aggregate statistics
   - Priority breakdown
   - Total financial impact

---

## 🔄 Pipeline Workflow

```
[1-9]   Data Loading & Cleaning
         ├─ Load 3 datasets
         ├─ Remove duplicates
         ├─ Assign labels (Fraud/Normal/Unknown)
         └─ Filter unsubscribed customers (13,904 removed)

[10-14] Data Visualization
         ├─ Label distribution
         ├─ Usage patterns
         ├─ Zero distribution
         └─ Temporal trends

[15-18] Feature Engineering (21 features)
         ├─ Build benchmarks from Normal customers
         ├─ Extract TIER S (Universal)
         ├─ Extract TIER A (Pattern Detection)
         ├─ Extract TIER B (Tapping & ROI)
         └─ Extract TIER C (Enhanced Zero)

[19-23] Data Preparation
         ├─ Train/Predict split
         ├─ SMOTE balancing (58:1 → 1:1)
         └─ StandardScaler normalization

[24-25] Model Training & Prediction
         ├─ Train Random Forest
         ├─ Train XGBoost
         ├─ Ensemble prediction
         └─ Prioritize by ROI

[26-27] Export & Summary
         ├─ Save to Excel (3 sheets)
         └─ Display final summary
```

**Total Runtime:** ~15 minutes on standard laptop

---

## 🎯 Priority Classification

| Priority | ROI Range | Count | % | Action Required |
|----------|-----------|-------|---|-----------------|
| **CRITICAL** | ≥ 80 | 266 | 0.21% | **Immediate field inspection** |
| **HIGH** | 60-79 | 2,669 | 2.07% | Priority investigation within 1 week |
| **MEDIUM** | 40-59 | 13,684 | 10.60% | Scheduled monitoring |
| **LOW** | < 40 | 112,447 | 87.12% | Routine monitoring |

---

## 📖 Documentation

- **[DOKUMENTASI_21_FITUR.md](DOKUMENTASI_21_FITUR.md)** - Detailed feature explanation
- **[PANDUAN_AWAM.md](PANDUAN_AWAM.md)** - User guide (if exists)
- **[notebooks/](notebooks/)** - Jupyter notebooks with code

---

## 🔧 Maintenance

### When to Update Model

1. **New fraud cases confirmed** (minimum 50 cases)
2. **After field validation** - update labels
3. **Quarterly retraining** - with latest data

### Feature Monitoring

- Track feature drift (distribution changes)
- Validate thresholds with field results
- Adjust ROI weights based on field success rate

### Model Improvement

Current limitations:
- Low precision (3%) - many false positives
- Needs field validation for final confirmation
- Performance may vary across different regions/tariffs

Future improvements:
- Collect more fraud cases for training
- Add external features (demographics, building type)
- Implement semi-supervised learning
- Build region-specific models

---

## 📝 Requirements

```
pandas>=2.0.0
numpy>=1.24.0
scikit-learn>=1.3.0
xgboost>=2.0.0
imbalanced-learn>=0.11.0
openpyxl>=3.1.0
scipy>=1.11.0
matplotlib>=3.7.0
seaborn>=0.12.0
```

Install with:
```bash
pip install -r requirements.txt
```

---

## 🤝 Contributing

This is an internal project for electricity theft detection.

For questions or issues:
1. Check [DOKUMENTASI_21_FITUR.md](DOKUMENTASI_21_FITUR.md) for feature details
2. Review notebook cells for implementation
3. Contact project maintainer

---

## 📜 License

Internal use only. Confidential data.

---

## 📞 Contact

**Project:** Electricity Theft Detection System  
**Version:** 1.0  
**Last Updated:** January 2025  
**Python:** 3.11+

---

## 🎉 Quick Summary

✅ **139,044 customers analyzed**  
✅ **21 optimized features extracted**  
✅ **9,640 potential frauds detected (7.47%)**  
✅ **2,935 high-priority cases for investigation**  
✅ **Rp 14.68B total estimated monthly loss**  
✅ **86% accuracy, 20% recall**

🚀 **Next Step:** Review [High Priority sheet](results/Prediction_Results_21Features.xlsx) and start field investigation!

---

Made with ❤️ using Python, scikit-learn, XGBoost, and a lot of ☕
