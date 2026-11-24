---
name: skill-imbalanced-data
description: Daftar periksa keputusan untuk menangani masalah klasifikasi tidak seimbang
version: 1.0.0
phase: 2
lesson: 17
tags: [imbalanced-data, smote, class-weights, threshold-tuning, evaluation]
---

# Strategi Data Tidak Seimbang

Sebuah daftar periksa keputusan untuk menangani klasifikasi tidak seimbang. Ikuti urutan ini untuk memilih pendekatan yang tepat bagi masalah Anda.

## Langkah 1: Ukur ketidakseimbangan

- Hitung jumlah sampel per kelas
- Hitung rasio ketidakseimbangan (mayoritas / minoritas)
- Ringan: rasio < 3:1 (mis. 70/30)
- Sedang: rasio 3:1 sampai 20:1 (mis. 95/5)
- Parah: rasio > 20:1 (mis. 99/1)

## Langkah 2: Pilih metrik yang tepat

Utamakan presisi/recall/F1 daripada akurasi untuk dataset tidak seimbang. Pilih berdasarkan masalah Anda:

| Situasi | Metrik Utama | Metrik Sekunder |
|---------|--------------|-----------------|
| Melewatkan sampel positif sangat mahal (penipuan, penyakit) | Recall | Skor F2 |
| Alarm palsu mahal (filter spam, rekomendasi) | Presisi | Skor F0.5 |
| Keduanya kurang lebih sama pentingnya | Skor F1 | MCC |
| Membutuhkan satu metrik peringkat | AUPRC | AUC-ROC |
| Membutuhkan perbandingan antar dataset | MCC | AUPRC |

## Langkah 3: Pilih strategi penyeimbangan kembali

### Berdasarkan tingkat ketidakseimbangan

| Ketidakseimbangan | Coba Pertama | Coba Kedua | Hindari |
|-------------------|--------------|------------|---------|
| Ringan (< 3:1) | Bobot kelas | Penyetelan ambang | Oversampling (tidak perlu) |
| Sedang (3:1 sampai 20:1) | SMOTE + bobot kelas | Penyetelan ambang di atasnya | Undersampling (terlalu banyak kehilangan data) |
| Parah (> 20:1) | SMOTE + bobot kelas + ambang | Ensemble dengan balanced bagging | Undersampling saja |

### Berdasarkan ukuran dataset

| Ukuran Dataset | Strategi yang Disukai | Alasan |
|----------------|----------------------|--------|
| < 1.000 sampel | Oversampling atau SMOTE | Tidak mampu kehilangan data mayoritas |
| 1.000 - 10.000 | SMOTE + penyetelan ambang | Cukup sampel minoritas untuk k-NN |
| > 10.000 | Bobot kelas atau undersampling | Cepat, data minoritas memadai |

## Langkah 4: Terapkan tekniknya

### Bobot kelas (selalu coba lebih dulu)
- Di sklearn: `class_weight='balanced'`
- Tidak perlu memodifikasi data
- Berfungsi dengan model berbasis loss apa pun
- Setara dengan oversampling secara ekspektasi

### SMOTE
- Terapkan hanya pada data latih (jangan pernah pada data uji/validasi)
- Gunakan k=5 tetangga (nilai bawaan)
- Gabungkan dengan bobot kelas untuk hasil terbaik
- Waspadai titik sintetis yang bising di dekat batas

### Penyetelan ambang
- Latih model, lalu dapatkan probabilitas prediksi pada himpunan validasi
- Sapu ambang dari 0,05 sampai 0,95
- Pilih ambang yang memaksimalkan metrik pilihan Anda
- Selalu setel pada data validasi, jangan pernah pada data uji

## Langkah 5: Validasi dengan benar

- Gunakan validasi silang terstratifikasi (mempertahankan rasio kelas di setiap lipatan)
- Laporkan metrik pada himpunan uji asli (yang tidak di-resample)
- Jangan pernah menerapkan SMOTE sebelum pembagian data -- hanya pada lipatan latih
- Bandingkan dengan baseline "selalu prediksi kelas mayoritas"

## Langkah 6: Kesalahan umum yang harus dihindari

- Menerapkan SMOTE pada seluruh dataset sebelum pembagian latih/uji (kebocoran data)
- Memakai akurasi sebagai metrik evaluasi
- Tidak mencoba bobot kelas lebih dulu (pendekatan paling sederhana, sering cukup)
- Melakukan oversampling lalu validasi silang (titik sintetis bocor lintas lipatan)
- Mengabaikan penyetelan ambang (peningkatan gratis, tanpa perlu latih ulang)
- Memakai undersampling acak pada dataset kecil (membuang terlalu banyak data)

## Pohon Keputusan Singkat

1. Apakah rasio ketidakseimbangan < 3:1? -> Coba bobot kelas saja
2. Apakah dataset > 10.000 sampel? -> Bobot kelas + penyetelan ambang
3. Apakah dataset < 1.000 sampel? -> SMOTE + bobot kelas
4. Selain itu -> SMOTE + bobot kelas + penyetelan ambang
5. Masih belum cukup baik? -> Ensemble balanced bagging
