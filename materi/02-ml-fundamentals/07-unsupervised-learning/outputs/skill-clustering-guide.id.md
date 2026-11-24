---
name: skill-clustering-guide
description: Pilih algoritma clustering yang tepat berdasarkan bentuk data, noise, dan kendala
version: 1.0.0
phase: 2
lesson: 7
tags: [clustering, k-means, dbscan, hierarchical, gmm, unsupervised]
---

# Panduan Pemilihan Algoritma Clustering

Clustering tidak memiliki satu algoritma terbaik. Pilihan yang tepat bergantung pada bentuk klaster, apakah Anda mengetahui jumlah klaster, seberapa banyak noise dalam data, dan seberapa besar dataset-nya.

## Daftar Pemeriksaan Keputusan

1. Apakah Anda mengetahui jumlah klaster?
   - Ya: K-Means atau GMM
   - Tidak: DBSCAN (menemukan klaster secara otomatis), atau hierarchical (potong dendrogram pada berbagai tingkat)

2. Berapa bentuk klaster-nya?
   - Kurang lebih bulat (seperti gumpalan): K-Means
   - Elips dengan ukuran berbeda: GMM
   - Bentuk sembarang (sabit, cincin, rantai): DBSCAN
   - Bersarang atau hierarkis: clustering hierarkis

3. Apakah data mengandung noise atau outlier?
   - Ya: DBSCAN (menandai titik noise secara eksplisit) atau GMM (titik berprobabilitas rendah adalah outlier)
   - Tidak: K-Means sudah cukup

4. Apakah Anda memerlukan penugasan lunak (probabilitas)?
   - Ya: GMM memberikan P(klaster | titik data) untuk setiap klaster
   - Tidak: K-Means atau DBSCAN memberikan penugasan keras

5. Seberapa besar dataset-nya?
   - Di bawah 10.000: semua algoritma dapat digunakan
   - 10.000 sampai 1.000.000: K-Means (cepat), Mini-Batch K-Means (lebih cepat)
   - Di atas 1.000.000: Mini-Batch K-Means atau BIRCH. Hierarkis terlalu lambat.

## Kapan menggunakan setiap pendekatan

**K-Means**: titik awal default. Cepat (O(n * k * iterasi)), sederhana, dan cukup baik untuk banyak masalah. Gunakan metode elbow atau skor siluet untuk memilih K. Keterbatasan: mengasumsikan klaster bulat, sensitif terhadap inisialisasi (gunakan K-Means++ atau jalankan beberapa kali), dan kurang baik menangani klaster dengan ukuran yang berbeda-beda.

**DBSCAN**: terbaik untuk menemukan klaster berbentuk sembarang dan mendeteksi outlier secara otomatis. Dua parameter: eps (radius lingkungan) dan min_samples (kepadatan minimum). Tidak perlu menentukan K. Keterbatasan: kesulitan ketika klaster memiliki kepadatan yang sangat berbeda, dan menyetel eps bisa rumit. Gunakan plot k-distance untuk memperkirakan eps: hitung jarak ke tetangga ke-k dari setiap titik, urutkan, lalu carilah elbow.

**Hierarkis (Agglomerative)**: membangun pohon penggabungan. Berguna ketika Anda ingin menjelajahi struktur klaster pada beberapa tingkat butiran (potong dendrogram pada ketinggian berbeda). Ward's linkage paling baik untuk klaster kompak. Single linkage menemukan klaster memanjang tetapi sensitif terhadap noise. Keterbatasan: memori O(n^2) dan waktu O(n^3), sehingga tidak praktis untuk dataset besar.

**GMM (Gaussian Mixture Models)**: clustering lunak dengan penugasan probabilistik. Memodelkan setiap klaster sebagai distribusi Gaussian dengan rata-rata dan kovariansnya sendiri. Lebih baik daripada K-Means ketika klaster berbentuk elips atau tumpang tindih. Gunakan BIC (Bayesian Information Criterion) untuk memilih jumlah komponen. Keterbatasan: mengasumsikan distribusi Gaussian, dapat gagal pada bentuk non-konveks, dan sensitif terhadap inisialisasi.

## Evaluasi kualitas klaster (tanpa label)

| Metrik | Apa yang diukur | Rentang | Kapan digunakan |
|--------|-----------------|---------|-----------------|
| Skor siluet | Kohesi vs pemisahan | -1 sampai 1 (lebih tinggi lebih baik) | Membandingkan nilai K atau algoritma |
| Inersia (SS dalam klaster) | Kekompakan klaster | 0 sampai tak hingga (lebih rendah lebih baik) | Metode elbow untuk K-Means |
| BIC / AIC | Kecocokan model dengan penalti kompleksitas | Lebih rendah lebih baik | Memilih jumlah komponen GMM |
| Indeks Calinski-Harabasz | Rasio varians antar terhadap dalam klaster | Lebih tinggi lebih baik | Perbandingan cepat |
| Indeks Davies-Bouldin | Rata-rata kesamaan antarklaster | Lebih rendah lebih baik | Memberi penalti pada klaster yang tumpang tindih |

## Kesalahan umum

- Menjalankan K-Means tanpa menskalakan fitur (fitur dengan skala lebih besar mendominasi perhitungan jarak)
- Memilih K dengan mengamati data 2D secara kasar ketika data aktualnya berdimensi tinggi (gunakan skor siluet)
- Menggunakan K-Means pada klaster non-bulat (data berbentuk sabit atau cincin memerlukan DBSCAN)
- Menyetel eps DBSCAN terlalu besar (semuanya masuk satu klaster) atau terlalu kecil (semuanya jadi noise)
- Memperlakukan label klaster sebagai kebenaran (clustering bersifat eksploratif; validasilah dengan pengetahuan domain)
- Menjalankan clustering hierarkis pada dataset dengan lebih dari 20.000 titik (memori dan waktu meledak)

## Ringkasan cepat

| Algoritma | Bentuk klaster | Menemukan K | Menangani noise | Penugasan lunak | Skalabilitas |
|-----------|----------------|-------------|-----------------|-----------------|--------------|
| K-Means | Bulat | Tidak (Anda menetapkan K) | Tidak | Tidak | Jutaan |
| Mini-Batch K-Means | Bulat | Tidak | Tidak | Tidak | Puluhan juta |
| DBSCAN | Sembarang | Ya | Ya | Tidak | Ratusan ribu |
| Hierarkis | Apa saja (bergantung linkage) | Fleksibel (potong dendrogram) | Bergantung linkage | Tidak | Di bawah 20k |
| GMM | Elips | Tidak (Anda menetapkan K) | Sebagian (probabilitas rendah) | Ya | Di bawah 100k |
| HDBSCAN | Sembarang | Ya | Ya | Sebagian | Ratusan ribu |
