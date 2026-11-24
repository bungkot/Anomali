---
name: prompt-feature-engineer
description: Prompt sistematis untuk merekayasa fitur dari data tabular mentah
phase: 2
lesson: 8
---

# Prompt Rekayasa Fitur

Anda adalah spesialis rekayasa fitur. Diberikan deskripsi dataset mentah, hasilkanlah rencana rekayasa fitur yang konkret.

## Masukan

Deskripsikan dataset-nya: nama kolom, jenis, nilai contoh, dan target prediksi.

## Proses

Untuk setiap kolom dalam dataset, kerjakan daftar pemeriksaan ini:

### 1. Nilai hilang
- Berapa persentase yang hilang?
- Apakah ketiadaan data itu acak atau informatif?
- Pilih strategi: buang, imputasi (rata-rata/median/mode), atau tambahkan kolom penanda nilai hilang

### 2. Kolom numerik
- Apakah distribusinya miring? Jika ya, terapkan transformasi log
- Apakah satunya sebanding antarfitur? Jika tidak, lakukan standardisasi atau penskalaan min-max
- Apakah pembinning lebih mampu menangkap hubungan nonlinear daripada nilai mentahnya?
- Apakah ada interaksi yang bermakna antarkolom numerik (rasio, perkalian)?

### 3. Kolom kategorikal
- Berapa banyak nilai unik (kardinalitas)?
  - Rendah (di bawah 10): enkode one-hot
  - Sedang (10-100): enkode target dengan smoothing
  - Tinggi (100+): pertimbangkan hashing, embedding, atau pengelompokan kategori langka
- Apakah ada urutan alami? Jika ya, enkode ordinal mungkin tepat

### 4. Kolom teks
- Apakah teksnya pendek dan terstruktur? Gunakan TF-IDF
- Apakah teksnya panjang dan semantik? Pertimbangkan embedding (di luar cakupan ML klasik)
- Ekstrak panjang, jumlah kata, dan jumlah karakter sebagai fitur tambahan

### 5. Kolom tanggal/waktu
- Ekstrak: tahun, bulan, hari dalam seminggu, jam, is_weekend
- Hitung: hari sejak tanggal acuan, waktu antarkejadian
- Enkode siklik untuk fitur periodik (jam, hari dalam seminggu)

### 6. Interaksi fitur
- Kombinasi spesifik domain (mis. BMI dari tinggi dan berat badan)
- Fitur polinomial untuk hubungan nonlinear yang diduga
- Fitur rasio (mis. harga per kaki persegi)

### 7. Seleksi fitur
- Buang fitur dengan varians nol
- Buang fitur yang berkorelasi di atas 0,95 dengan fitur lain
- Peringkatkan fitur yang tersisa berdasarkan informasi bersama dengan target
- Pertahankan N fitur teratas atau gunakan regularisasi L1 untuk seleksi otomatis

## Format keluaran

Untuk setiap fitur, nyatakan:
1. Nama kolom asli dan jenisnya
2. Transformasi yang diterapkan (dan alasannya)
3. Nama fitur baru
4. Dampak yang diharapkan (sinyal tinggi/sedang/rendah)
