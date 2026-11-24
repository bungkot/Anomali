---
name: skill-svm-kernel-chooser
description: Pilih kernel SVM yang tepat dan setel C serta gamma untuk masalah Anda
version: 1.0.0
phase: 2
lesson: 5
tags: [svm, kernel, classification, hyperparameter-tuning]
---

# Panduan Pemilihan Kernel SVM

SVM ditentukan oleh dua pilihan: kernel (yang menentukan bentuk batas keputusan) dan parameter regularisasi (yang mengontrol trade-off antara lebar margin dan kesalahan klasifikasi). Memilih keduanya dengan tepat adalah pembeda antara model yang tidak berguna dan model yang kuat.

## Daftar Pemeriksaan Keputusan

1. Apakah data dapat dipisahkan secara linear (atau hampir)?
   - Ya: gunakan kernel linear. Lebih cepat dan lebih mudah diinterpretasikan.
   - Tidak: lanjut ke langkah 2.

2. Berapa banyak fitur dibandingkan sampel?
   - Fitur >> sampel (mis. teks dengan TF-IDF): gunakan kernel linear. Data berdimensi tinggi sering kali dapat dipisahkan secara linear. RBF hanya menambah kompleksitas tanpa manfaat.
   - Sampel >> fitur (mis. data tabular dengan 10-50 fitur): kernel RBF adalah pilihan utama.

3. Apakah batas keputusan diperkirakan halus?
   - Batas halus dan kontinu: kernel RBF
   - Batas berbentuk polinomial: kernel polinomial (mulai dengan derajat 2 atau 3)
   - Pengetahuan domain menunjukkan interaksi tertentu: kernel polinomial dengan derajat yang sesuai

4. Seberapa besar dataset-nya?
   - Di bawah 10.000 sampel: semua kernel bisa digunakan, RBF adalah pilihan aman
   - 10.000 sampai 100.000: kernel linear atau LinearSVC (formulasi primal, O(n) per epoch)
   - Di atas 100.000: jangan gunakan kernel SVM. Beralihlah ke SVM linear, gradient boosting, atau jaringan saraf.

5. Apakah Anda sudah menskalakan fitur?
   - SVM memerlukan penskalaan fitur. Selalu lakukan standardisasi (rata-rata nol, varians satu) sebelum fit. Fitur yang tidak diskalakan mendistorsi geometri margin.

## Diagram alur pemilihan kernel

```
Mulai
  |
  v
Fitur > 1000 atau fitur >> sampel?
  Ya --> Kernel linear (LinearSVC untuk kecepatan)
  Tidak --> Dataset < 10k sampel?
              Ya --> Coba RBF terlebih dahulu (kernel serbaguna terbaik)
              Tidak --> Kernel linear (kernel SVM memerlukan O(n^2) sampai O(n^3))
```

Jika RBF tidak bekerja dengan baik, coba kernel polinomial derajat 2-3. Jika itu pun gagal, masalah ini mungkin tidak cocok untuk SVM.

## Menyetel C (regularisasi)

C mengontrol penalti atas kesalahan klasifikasi. Nilainya berbanding terbalik dengan kekuatan regularisasi.

| Nilai C | Efek | Kapan digunakan |
|---------|------|-----------------|
| 0,001 - 0,01 | Margin lebar, banyak pelanggaran diizinkan | Data bising, menginginkan generalisasi |
| 0,1 - 1,0 | Seimbang | Rentang awal yang baik |
| 10 - 1000 | Margin sempit, sedikit pelanggaran | Data bersih, memerlukan akurasi tinggi |

Strategi penyetelan:
- Mulai dengan C=1.0
- Cari pada skala logaritmik: [0.001, 0.01, 0.1, 1, 10, 100, 1000]
- Gunakan validasi silang untuk memilih nilai terbaik
- Jika C terbaik berada di tepi rentang Anda, perpanjang rentang ke arah tersebut

## Menyetel gamma (kernel RBF)

Gamma mengontrol seberapa jauh pengaruh satu titik latihan menjangkau. Gamma menentukan lebar Gaussian.

| Nilai gamma | Efek | Kapan digunakan |
|-------------|------|-----------------|
| Kecil (0,001) | Setiap titik memengaruhi area luas. Batas halus dan sederhana | Underfitting atau sedikit fitur |
| Sedang (auto: 1/n_features) | Default sklearn. Titik awal yang wajar | Penggunaan umum |
| Besar (10+) | Setiap titik hanya memengaruhi titik di sekitarnya. Batas kompleks dan berliku | Risiko overfitting |

Strategi penyetelan:
- Mulai dengan gamma="scale" (1 / (n_features * X.var()), default sklearn)
- Cari pada skala logaritmik: [0.001, 0.01, 0.1, 1, 10]
- Gamma rendah + C tinggi cenderung overfit
- Gamma tinggi + C rendah cenderung underfit

## Penyetelan C dan gamma secara bersamaan

C dan gamma saling berinteraksi. Selalu setel keduanya bersama-sama, bukan secara terpisah.

Pendekatan yang disarankan:
1. Pencarian grid kasar: C dalam [0.01, 0.1, 1, 10, 100], gamma dalam [0.001, 0.01, 0.1, 1, 10] (25 kombinasi)
2. Temukan wilayah terbaik
3. Pencarian grid halus di sekitar wilayah terbaik (mis. C dalam [5, 10, 20, 50], gamma dalam [0.05, 0.1, 0.2])
4. Gunakan validasi silang 5-fold di seluruh proses

## Kesalahan umum

- Menggunakan kernel RBF pada data jarang berdimensi tinggi (linear lebih baik dan 100 kali lebih cepat)
- Lupa menskalakan fitur (kesalahan SVM yang paling umum)
- Menyetel C terlalu tinggi pada data bising (menghafal noise alih-alih mempelajari batas)
- Menggunakan kernel SVM pada dataset di atas 50k sampel (waktu latihan menjadi tidak layak)
- Tidak menyetel C dan gamma bersama-sama (keduanya saling mengompensasi)
- Terlalu cepat memilih derajat polinomial 5+ (overfit secara agresif; coba 2 atau 3 terlebih dahulu)

## Ringkasan cepat

| Kernel | Kapan digunakan | Parameter utama | Kompleksitas latihan |
|--------|----------------|-----------------|---------------------|
| Linear | Teks/TF-IDF, banyak fitur, data besar | Hanya C | O(n) per epoch |
| RBF | Serbaguna, di bawah 10k sampel | C, gamma | O(n^2) sampai O(n^3) |
| Polinomial | Hubungan polinomial yang diketahui | C, degree, coef0 | O(n^2) sampai O(n^3) |
| Sigmoid | Jarang berguna (setara dengan jaringan saraf dua lapis) | C, gamma, coef0 | O(n^2) sampai O(n^3) |
