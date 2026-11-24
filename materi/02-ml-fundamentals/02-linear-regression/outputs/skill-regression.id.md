---
name: skill-regression
description: Pilih pendekatan regresi yang tepat berdasarkan karakteristik data dan batasan masalah
version: 1.0.0
phase: 2
lesson: 2
tags: [regression, linear-regression, polynomial-regression, ridge, regularization]
---

# Panduan Strategi Regresi

Regresi memprediksi nilai kontinu. Pendekatan yang tepat bergantung pada hubungan antara fitur dan target, jumlah fitur, dan risiko overfitting.

## Daftar Periksa Keputusan

1. Apakah hubungan antara fitur dan target kira-kira linear?
   - Ya: mulai dengan regresi linear biasa
   - Tidak: coba fitur polinomial atau model nonlinear

2. Berapa banyak fitur yang Anda miliki dibandingkan dengan sampel?
   - Sedikit fitur, banyak sampel: regresi linear biasa sudah cukup baik
   - Banyak fitur, sedikit sampel: gunakan regularisasi (Ridge atau Lasso)
   - Lebih banyak fitur daripada sampel: Lasso (L1) untuk memilih fitur, atau Ridge (L2) untuk menyusutkan semua bobot

3. Apakah Anda memerlukan interpretabilitas?
   - Ya: regresi linear dengan sedikit fitur, atau Lasso untuk seleksi fitur otomatis
   - Tidak: fitur polinomial, atau beralih ke model berbasis pohon atau jaringan saraf

4. Apakah dataset Anda kecil (di bawah 10.000 baris)?
   - Gunakan persamaan normal (solusi bentuk tertutup) demi kecepatan
   - Validasi silang penting untuk evaluasi yang andal

5. Apakah dataset Anda besar (jutaan baris)?
   - Gunakan penurunan gradien stokastik (SGD) atau penurunan gradien mini-batch
   - Persamaan normal terlalu lambat karena inversi matriks berbiaya O(n^3)

## Kapan menggunakan tiap pendekatan

**Regresi Linear Biasa**: baseline untuk tugas regresi apa pun. Mulai dari sini. Jika R-squared dapat diterima dan modelnya sederhana, berhenti di sini.

**Regresi Polinomial**: plot sebar menunjukkan kurva, bukan garis. Mulai dengan derajat 2. Naikkan hanya jika kinerja validasi mendukung. Derajat > 5 hampir selalu overfitting.

**Regresi Ridge (L2)**: banyak fitur yang berkorelasi. Semua bobot menyusut ke arah nol tetapi tidak ada yang menjadi persis nol. Baik ketika Anda meyakini semua fitur berkontribusi.

**Regresi Lasso (L1)**: banyak fitur dan Anda menduga hanya sedikit yang penting. Lasso membuat bobot fitur yang tidak relevan menjadi persis nol, melakukan seleksi fitur otomatis.

**Elastic Net**: menggabungkan penalti L1 dan L2. Gunakan ketika Anda punya banyak fitur berkorelasi dan menginginkan seleksi fitur sebagian.

## Kesalahan umum

- Melewati penskalaan fitur sebelum penurunan gradien (konvergensi menjadi sangat lambat)
- Menggunakan kinerja himpunan uji untuk menyetel hyperparameter (gunakan himpunan validasi atau validasi silang)
- Mencocokkan polinomial derajat tinggi tanpa memeriksa error validasi (R^2 latih selalu naik seiring derajat)
- Mengabaikan plot residual (R^2 bisa menyesatkan jika residual menunjukkan pola)
- Menganggap R^2 sebagai satu-satunya metrik (periksa distribusi residual, MAE, dan ambang yang spesifik untuk domain)

## Referensi cepat

| Metode | Kapan digunakan | Regularisasi | Seleksi fitur |
|--------|------------|---------------|-------------------|
| OLS | Baseline, sedikit fitur | Tidak ada | Manual |
| Ridge | Banyak fitur, semuanya relevan | L2 (menyusut) | Tidak |
| Lasso | Banyak fitur, sedikit yang relevan | L1 (nolkan) | Otomatis |
| Elastic Net | Banyak fitur berkorelasi | L1 + L2 | Sebagian |
| Polinomial | Hubungan nonlinear | Tambahkan Ridge/Lasso di atasnya | Pemilihan derajat manual |
