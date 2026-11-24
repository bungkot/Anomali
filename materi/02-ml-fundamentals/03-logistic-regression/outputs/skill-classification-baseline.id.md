---
name: skill-classification-baseline
description: Bangun baseline klasifikasi yang kuat sebelum beralih ke model kompleks
version: 1.0.0
phase: 2
lesson: 3
tags: [classification, logistic-regression, baseline, preprocessing]
---

# Panduan Baseline Klasifikasi

Sebelum mencoba model kompleks, bangunlah baseline dengan regresi logistik. Model ini melatih dalam hitungan detik, menghasilkan probabilitas, dan sepenuhnya dapat diinterpretasi. Jumlah masalah dunia nyata yang tidak pernah memerlukan sesuatu yang lebih rumit ternyata mengejutkan.

## Daftar Periksa Keputusan

1. Apakah batas keputusan kemungkinan bersifat linear?
   - Ya: regresi logistik mungkin sudah cukup
   - Tidak: Anda tetap memerlukannya sebagai baseline untuk mengukur perbaikan

2. Berapa banyak fitur yang Anda miliki?
   - Di bawah 50: regresi logistik standar sudah memadai
   - 50 sampai 10.000: tambahkan regularisasi L2 (Ridge)
   - Di atas 10.000 (mis. fitur teks TF-IDF): gunakan regularisasi L1 (Lasso) atau LinearSVC

3. Apakah dataset tidak seimbang?
   - Rasio di bawah 5:1: kemungkinan baik tanpa penyesuaian
   - 5:1 sampai 50:1: gunakan `class_weight="balanced"` di sklearn
   - Di atas 50:1: kombinasikan pembobotan kelas dengan metrik yang tepat (presisi, recall, atau F1)

4. Apakah fitur berada pada skala yang berbeda?
   - Selalu lakukan standardisasi sebelum regresi logistik. Model ini menggunakan optimasi berbasis gradien, dan fitur yang tidak diskalakan memperlambat konvergensi atau mendistorsi batas keputusan.

5. Apakah ada nilai yang hilang?
   - Lakukan imputasi sebelum fitting. Regresi logistik tidak dapat menangani NaN.
   - Gunakan imputasi median untuk kolom numerik, modus untuk kolom kategorikal.

## Kapan regresi logistik sudah cukup baik

- Klasifikasi biner dengan hubungan fitur sebagian besar linear
- Anda memerlukan keluaran probabilitas (bukan hanya label kelas)
- Interpretabilitas diperlukan (koefisien menunjukkan arah kepentingan fitur dan besaran relatifnya setelah standardisasi)
- Data pelatihan kecil (ratusan hingga beberapa ribu sampel)
- Anda memerlukan model cepat untuk penyajian waktu nyata (satu produk titik pada inferensi)
- Persyaratan regulasi atau kepatuhan menuntut penjelasan

## Kapan harus meningkatkan model

- Akurasi mendatar jauh di bawah target dan Anda telah mencoba rekayasa fitur
- Hubungan antara fitur dan target jelas nonlinear (periksa plot residual)
- Anda memiliki data tabular besar (10k+ baris): coba gradient boosting (XGBoost atau LightGBM)
- Fitur memiliki interaksi kompleks yang tidak dapat ditangkap oleh fitur polinomial
- Anda memiliki data gambar, teks, atau sekuensial: regresi logistik pada masukan mentah tidak akan berhasil

## Langkah praproses untuk baseline klasifikasi

1. **Pemisahan train/test** terlebih dahulu, sebelum praproses apa pun. Ini mencegah kebocoran data.
2. **Tangani nilai yang hilang**: imputasi median untuk numerik, imputasi modus untuk kategorikal.
3. **Enkode data kategorikal**: one-hot untuk kardinalitas rendah (di bawah 10 nilai), target encoding untuk yang lebih tinggi. Lakukan fit target encoding hanya pada fold pelatihan (gunakan encoding out-of-fold untuk mencegah kebocoran).
4. **Skalakan data numerik**: StandardScaler (rata-rata nol, varians satuan). Lakukan fit pada data latih, lalu transformasi keduanya.
5. **Latih regresi logistik** dengan `C=1.0` (regularisasi bawaan).
6. **Evaluasi**: matriks konfusi, presisi, recall, F1. Bukan hanya akurasi.
7. **Setel ambang batas**: nilai bawaan 0.5 jarang optimal. Sapu rentang 0,1 hingga 0,9 dan pilih ambang yang sesuai dengan prioritas presisi/recall Anda.

## Kesalahan umum

- Hanya mengevaluasi akurasi pada data tidak seimbang (model yang memprediksi kelas mayoritas memperoleh skor tinggi tetapi tidak berguna)
- Lupa menskalakan fitur (regresi logistik dengan fitur tak terskala melatih lambat dan konvergen ke solusi yang lebih buruk)
- Menggunakan himpunan uji untuk menyetel ambang keputusan (gunakan validasi atau validasi silang)
- Melewati baseline dan langsung melompat ke XGBoost (Anda kehilangan interpretabilitas dan tidak memiliki titik acuan)
- Tidak memeriksa multikolinearitas (fitur yang sangat berkorelasi meningkatkan varians koefisien)

## Referensi cepat

| Skenario | Model | Regularisasi | Pengaturan utama |
|----------|-------|---------------|-------------|
| Sedikit fitur, dapat diinterpretasi | LogisticRegression | L2 (bawaan) | C=1.0 |
| Banyak fitur, sebagian tidak relevan | LogisticRegression | L1 | penalty="l1", solver="saga" |
| Sparse berdimensi tinggi (teks) | SGDClassifier | L1 atau ElasticNet | loss="log_loss" |
| Kelas tidak seimbang | LogisticRegression | L2 | class_weight="balanced" |
| Memerlukan probabilitas | LogisticRegression | L2 | predict_proba() |
| Hanya memerlukan label kelas | LinearSVC | L2 | Lebih cepat daripada LR untuk data besar |
