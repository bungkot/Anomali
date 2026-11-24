---
name: skill-perceptron
description: Pahami pola perceptron dan kapan memakai arsitektur satu lapis versus banyak lapis
version: 1.0.0
phase: 3
lesson: 1
tags: [perceptron, neural-networks, classification, deep-learning]
---

# Pola Perceptron

Sebuah perceptron menghitung jumlah berbobot dari masukan ditambah bias, lalu menerapkan fungsi langkah untuk menghasilkan keluaran biner. Ia adalah unit dasar jaringan saraf.

```
output = step(w1*x1 + w2*x2 + ... + wn*xn + bias)
```

## Kapan satu perceptron sudah cukup

- Masalahnya terpisah secara linier: sebuah garis lurus (atau hiperplan) dapat membelah dua kelas
- Gerbang logika: AND, OR, NOT, NAND
- Keputusan ambang sederhana: "apakah skornya di atas X?"
- Klasifikator biner pada data yang mengelompok menjadi dua wilayah yang tidak tumpang tindih

## Kapan Anda memerlukan banyak lapis

- Masalahnya tidak terpisah secara linier: tidak ada satu garis pun yang dapat memisahkan kelas
- Masalah XOR dan paritas
- Tugas apa pun yang memerlukan penalaran "ini tetapi bukan itu" (kombinasi kondisi)
- Klasifikasi dunia nyata: gambar, teks, audio -- hampir selalu nonlinear

## Daftar periksa keputusan

1. Plot atau periksa data Anda. Dapatkah Anda menggambar satu batas lurus antara kelas?
   - Ya: satu perceptron cukup
   - Tidak: Anda memerlukan paling sedikit dua lapis
2. Dapatkah masalah diuraikan menjadi AND/OR dari keputusan linier yang lebih sederhana?
   - Penguraian ini menunjukkan struktur jaringan minimum
   - XOR = (A OR B) AND (NOT (A AND B)) = 3 perceptron dalam 2 lapis
3. Untuk masalah dengan lebih dari dua kelas, Anda memerlukan satu simpul keluaran per kelas

## Aturan pelatihan

```
error = expected - predicted
weight_new = weight_old + learning_rate * error * input
bias_new = bias_old + learning_rate * error
```

Jika prediksinya benar, tidak ada yang berubah. Jika salah, bobot bergeser untuk mengurangi galat. Ini hanya berlaku untuk perceptron satu lapis. Jaringan banyak lapis memerlukan propagasi balik.

## Kesalahan umum

- Mencoba mempelajari pola nonlinear dengan satu perceptron (ia tidak akan pernah konvergen)
- Menetapkan laju pembelajaran terlalu tinggi (bobot berosilasi) atau terlalu rendah (pelatihan berlangsung selamanya)
- Lupa menyertakan suku bias (tanpa itu, batas keputusan harus melalui titik asal)
- Mengira konvergensi perceptron (terjamin untuk data yang terpisah secara linier) sama dengan konvergensi jaringan saraf umum (tidak terjamin)
