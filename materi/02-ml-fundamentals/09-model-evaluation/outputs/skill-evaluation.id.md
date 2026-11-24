---
name: skill-evaluation
description: Daftar periksa strategi evaluasi untuk model klasifikasi dan regresi
version: 1.0.0
phase: 2
lesson: 9
tags: [evaluation, metrics, cross-validation, model-selection]
---

# Strategi Evaluasi Model

Daftar periksa untuk mengevaluasi model ML apa pun dengan benar. Ikuti urutan ini untuk menghindari kesalahan evaluasi yang paling umum.

## Langkah 1: Bagi data dengan benar

- Bagikan sebelum praproses apa pun (penskalaan, imputasi, encoding)
- Gunakan pembagian stratifikasi untuk tugas klasifikasi
- Sisihkan himpunan uji yang hanya Anda sentuh tepat sekali di akhir
- Untuk dataset kecil, gunakan validasi silang 5-lipat atau 10-lipat alih-alih satu pembagian
- Untuk deret waktu, gunakan pembagian berbasis waktu (jangan pernah mengacak)

## Langkah 2: Pilih metrik yang tepat

### Klasifikasi

| Situasi | Gunakan metrik ini | Alasan |
|---------|-------------------|--------|
| Kelas seimbang, perbandingan sederhana | Akurasi | Mudah diinterpretasi, bermakna ketika kelas sama |
| Positif palsu mahal (filter spam, peringatan penipuan) | Presisi | Mengukur berapa banyak item yang ditandai benar-benar positif |
| Negatif palsu mahal (skrining kanker, keamanan) | Recall | Mengukur berapa banyak positif sebenarnya yang Anda tangkap |
| Perlu menyeimbangkan presisi dan recall | Skor F1 | Rata-rata harmonik, menghukum ketimpangan ekstrem |
| Membandingkan model lintas ambang | AUC-ROC | Kualitas peringkat yang independen terhadap ambang |
| Data tidak seimbang | F1, AUC-ROC, atau PR-AUC | Akurasi menyesatkan pada kelas yang tidak seimbang |

### Regresi

| Situasi | Gunakan metrik ini | Alasan |
|---------|-------------------|--------|
| Regresi standar, pencilan dapat diterima | RMSE | Satuan sama dengan target, menghukum error besar |
| Evaluasi tangguh terhadap pencilan | MAE | Memperlakukan semua error secara setara, tidak didominasi pencilan |
| Membandingkan model pada skala berbeda | R-squared | Skala ternormalisasi 0-1 (fraksi varians yang dijelaskan) |
| Bisnis memerlukan nilai dalam dolar | MAE atau RMSE | Dapat diinterpretasi langsung sebagai besar error |

## Langkah 3: Tetapkan baseline

Sebelum mengevaluasi model Anda, hitung kinerja baseline:
- Klasifikasi: prediktor kelas mayoritas (selalu memprediksi kelas yang paling umum)
- Regresi: selalu memprediksi rata-rata target pelatihan
- Model apa pun yang tidak dapat mengalahkan baseline ini tidak sedang belajar

## Langkah 4: Lakukan validasi silang

- Gunakan K-fold (K=5 atau K=10) untuk estimasi yang stabil
- Gunakan K-fold stratifikasi untuk klasifikasi
- Laporkan rata-rata dan simpangan baku lintas lipatan
- Model dengan mean=0.85 dan std=0.02 lebih dapat dipercaya daripada mean=0.87 dan std=0.10

## Langkah 5: Bandingkan model secara statistik

- Jangan memilih model dengan skor rata-rata tertinggi tanpa memeriksa signifikansi
- Gunakan uji-t berpasangan lintas lipatan validasi silang
- Jika |t| < 2.78 (untuk K=5, df=4, p<0.05), perbedaan tersebut mungkin karena kebetulan
- Pertimbangkan model yang lebih sederhana ketika perbedaan kinerja tidak signifikan

## Langkah 6: Periksa kesalahan umum

- Kebocoran data: apakah ada informasi data uji yang masuk ke pelatihan? (penskalaan sebelum pembagian, fitur yang diturunkan dari target)
- Ketimpangan kelas: apakah akurasi menyembunyikan kinerja buruk pada kelas minoritas?
- Overfitting: apakah kesenjangan antara kinerja pelatihan dan validasi besar?
- Terlalu banyak evaluasi: apakah Anda telah melihat himpunan uji lebih dari sekali?

## Langkah 7: Laporkan kinerja akhir

- Latih pada gabungan data latih + validasi
- Evaluasi pada himpunan uji yang disisihkan tepat sekali
- Laporkan metrik yang dipilih dengan selang kepercayaan jika memungkinkan
- Nyatakan perbandingan baseline (seberapa jauh lebih baik daripada acak/rata-rata)
