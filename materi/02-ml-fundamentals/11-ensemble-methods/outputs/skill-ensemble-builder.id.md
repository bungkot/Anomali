---
name: skill-ensemble-builder
description: Pilih metode ansambel yang tepat dan konfigurasikan untuk masalah Anda
version: 1.0.0
phase: 2
lesson: 11
tags: [ensemble, bagging, boosting, random-forest, xgboost, stacking]
---

# Panduan Pemilihan Metode Ansambel

Ansambel menggabungkan beberapa model untuk menghasilkan prediksi yang lebih baik daripada model tunggal mana pun. Pertanyaannya selalu: jenis ansambel apa, dan kapan?

## Daftar Periksa Keputusan

1. Apa masalah utama model Anda saat ini?
   - Varians tinggi (overfitting): gunakan bagging (Random Forest)
   - Bias tinggi (underfitting): gunakan boosting (Gradient Boosting, XGBoost)
   - Keduanya, atau Anda menginginkan akurasi maksimal: gunakan stacking

2. Berapa banyak data yang Anda miliki?
   - Di bawah 1.000 baris: Random Forest (tangguh, sulit salah dikonfigurasi)
   - 1.000 hingga 100.000: XGBoost atau LightGBM (terbaik secara keseluruhan untuk data tabular)
   - Lebih dari 100.000: LightGBM (gradient boosting tercepat, menangani data besar dengan baik)

3. Berapa waktu penyetelan yang dapat Anda investasikan?
   - Minimal: Random Forest dengan nilai bawaan (hampir selalu bekerja)
   - Sedang: XGBoost dengan learning_rate=0.1, setel n_estimators dengan early stopping
   - Maksimal: LightGBM atau XGBoost dengan pencarian hiperparameter Bayes

4. Apakah Anda memerlukan interpretabilitas?
   - Ya: satu pohon keputusan atau Random Forest kecil dengan pentingnya fitur
   - Sebagian: gradient boosting dengan nilai SHAP
   - Tidak: stacking atau ansambel dalam

5. Apakah data bernoise dengan banyak pencilan?
   - Ya: Random Forest (bagging tangguh terhadap noise)
   - Tidak: gradient boosting (dapat mendorong akurasi lebih jauh pada data bersih)

## Kapan menggunakan setiap metode

**Random Forest (Bagging)**: pilihan pertama Anda yang aman. Melatih banyak pohon pada sampel bootstrap dan merata-ratakannya. Menurunkan varians tanpa menaikkan bias. Hampir mustahil overfitting pada data sedang. Penyetelan minimal yang dibutuhkan: tetapkan n_estimators=100-500 dan biarkan nilai bawaan.

**AdaBoost**: boosting berurutan dengan pemberat ulang sampel. Bekerja baik dengan pembelajar dasar sederhana (decision stump). Sensitif terhadap pencilan dan label bernoise karena memberikan bobot lebih besar pada titik yang salah diklasifikasikan. Sebagian besar telah digantikan oleh gradient boosting dalam praktiknya.

**Gradient Boosting**: memasangkan setiap pohon baru pada residual ansambel sejauh ini. Menurunkan bias. Metode paling kuat untuk data tabular. Memerlukan penyetelan: learning_rate, n_estimators, max_depth, min_child_weight, subsample.

**XGBoost**: gradient boosting dengan regularisasi, optimasi orde kedua, dan akselerasi tingkat sistem. Menangani nilai hilang secara bawaan. Pilihan bawaan untuk kompetisi Kaggle dan ML produksi pada data tabular.

**LightGBM**: gradient boosting dengan pertumbuhan leaf-wise (alih-alih level-wise). Lebih cepat daripada XGBoost pada dataset besar. Menggunakan pemisahan berbasis histogram. Terbaik untuk dataset lebih dari 50k baris.

**CatBoost**: gradient boosting dengan penanganan fitur kategorikal secara bawaan. Tidak perlu one-hot encoding. Baik ketika Anda memiliki banyak fitur kategorikal.

**Stacking**: melatih meta-learner pada prediksi dari beberapa model dasar yang beragam. Gunakan ketika Anda memerlukan akurasi absolut terbaik dan memiliki komputasi berlebih. Selalu hasilkan prediksi model dasar melalui validasi silang untuk menghindari kebocoran.

**Voting**: ansambel paling sederhana. Hard voting (kelas mayoritas) atau soft voting (rata-rata probabilitas). Cara cepat menggabungkan 2-3 model beragam tanpa meta-learner.

## Kesalahan umum

- Menggunakan gradient boosting tanpa early stopping (ia akan overfitting jika dibiarkan berjalan terlalu banyak putaran)
- Menetapkan learning_rate terlalu tinggi (di atas 0.3 biasanya menyebabkan ketidakstabilan)
- Tidak menyetel max_depth untuk gradient boosting (bawaan tak terbatas atau pohon sangat dalam menyebabkan overfitting)
- Stacking dengan model yang semuanya sejenis (keberagaman adalah inti dari stacking)
- Menggunakan AdaBoost pada data bernoise (pencilan mendapat bobot semakin besar setiap putaran)
- Mengharapkan Random Forest memperbaiki underfitting (ia menurunkan varians, bukan bias)

## Prioritas penyetelan per metode

**Random Forest:**
1. n_estimators: 100-500 (lebih banyak jarang lebih buruk, hanya lebih lambat)
2. max_depth: None (biarkan pohon tumbuh penuh) atau batasi 10-20 untuk kecepatan
3. max_features: "sqrt" untuk klasifikasi, "log2" atau n/3 untuk regresi

**XGBoost / LightGBM:**
1. learning_rate: 0.01-0.3 (lebih rendah lebih baik jika Anda memiliki komputasi untuk lebih banyak pohon)
2. n_estimators: gunakan early stopping pada himpunan validasi alih-alih menebak
3. max_depth: 3-8 (mulai dengan 6)
4. min_child_weight / min_data_in_leaf: 1-20 (nilai lebih tinggi mencegah overfitting)
5. subsample: 0.7-1.0
6. colsample_bytree: 0.7-1.0
7. reg_alpha (L1) dan reg_lambda (L2): 0-10

## Referensi cepat

| Metode | Menurunkan | Kecepatan | Upaya penyetelan | Terbaik untuk |
|--------|-----------|-----------|-----------------|---------------|
| Random Forest | Varians | Cepat | Rendah | Data bernoise, baseline cepat |
| AdaBoost | Bias | Cepat | Rendah | Pembelajar dasar sederhana, data bersih |
| Gradient Boosting | Bias | Sedang | Tinggi | Data tabular, kompetisi |
| XGBoost | Keduanya | Cepat | Tinggi | ML tabular produksi |
| LightGBM | Keduanya | Paling cepat | Tinggi | Dataset besar (50k+ baris) |
| CatBoost | Keduanya | Sedang | Sedang | Banyak fitur kategorikal |
| Stacking | Keduanya | Lambat | Tinggi | Akurasi maksimal, model beragam |
| Voting | Varians | Cepat | Tidak ada | Kombinasi cepat 2-3 model |
