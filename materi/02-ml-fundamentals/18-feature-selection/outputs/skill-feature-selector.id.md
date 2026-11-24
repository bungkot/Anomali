---
name: skill-feature-selector
description: Pohon keputusan referensi cepat untuk memilih metode pemilihan fitur yang tepat
version: 1.0.0
phase: 2
lesson: 18
tags: [feature-selection, mutual-information, rfe, lasso, tree-importance]
---

# Strategi Pemilihan Fitur

Referensi cepat untuk memilih dan menerapkan metode pemilihan fitur yang tepat.

## Langkah 1: Mulai dengan pembersihan

Sebelum menerapkan metode apa pun, hapus fitur yang jelas tidak berguna:

- **Fitur konstan**: varians = 0. Hapus.
- **Fitur nyaris konstan**: varians < 0,01 (atau ambang Anda sendiri). Hapus.
- **Fitur duplikat**: kolom identik. Simpan satu, buang sisanya.
- **Kolom ID**: unik per baris, tidak membawa informasi yang dapat digeneralisasi. Hapus.

Ini hanya memakan waktu beberapa detik dan dapat menghilangkan 10-30% fitur pada dataset dunia nyata yang berantakan.

## Langkah 2: Pilih metode sesuai situasi Anda

### Pohon Keputusan Singkat

1. **< 50 fitur?** Mulai dengan peringkat informasi mutual. Simpan K teratas.
2. **50 - 500 fitur?** Gunakan ambang varians lebih dulu, lalu L1 (Lasso) jika memakai model linear, atau kepentingan pohon jika memakai pohon.
3. **> 500 fitur?** Rangkai metode: ambang varians -> filter informasi mutual (50% teratas) -> RFE pada fitur yang tersisa.
4. **Butuh interpretabilitas?** Regularisasi L1 memberi Anda nol/non-nol yang persis. Kepentingan pohon memberi skor berperingkat.
5. **Perlu menangkap hubungan nonlinear?** Informasi mutual atau kepentingan berbasis pohon. Hindari L1 (linear saja).
6. **Perlu interaksi antarfitur?** RFE atau kepentingan berbasis pohon. Metode filter tidak menangkap interaksi.

### Referensi Metode

| Metode | Kapan Dipakai | Kapan Dihindari |
|--------|---------------|-----------------|
| Ambang varians | Selalu, sebagai langkah pertama | Jangan pernah dilompati |
| Informasi mutual | Peringkat cepat, hubungan nonlinear | Saat Anda perlu deteksi interaksi antarfitur |
| RFE | Seleksi menyeluruh, jumlah fitur sedang | Model sangat mahal, > 1000 fitur |
| L1 / Lasso | Model linear, seleksi embedded yang cepat | Masalah nonlinear, fitur sangat berkorelasi |
| Kepentingan pohon | Hubungan nonlinear, interaksi antarfitur | Bias oleh fitur dengan kardinalitas tinggi |
| Kepentingan permutasi | Validasi agnostik model, pemeriksaan akhir | Terlalu lambat untuk penyaringan awal |

## Langkah 3: Validasi seleksi Anda

- Bandingkan kinerja model dengan fitur terpilih versus semua fitur
- Gunakan validasi silang, bukan satu pembagian latih/uji
- Jika kinerja turun lebih dari 1-2%, mungkin Anda menghapus fitur yang berguna
- Jika kinerja membaik, Anda berhasil menghilangkan kebisingan

## Langkah 4: Tangani jebakan umum

### Fitur berkorelasi
- L1 secara sembarangan memilih satu dari kelompok yang berkorelasi dan mennolkan sisanya
- Hitung matriks korelasi terlebih dahulu dan putuskan fitur berkorelasi mana yang disimpan
- Kepentingan pohon menyebarkan kepentingan ke fitur-fitur yang berkorelasi

### Kebocoran data
- Lakukan fit pemilihan fitur hanya pada data latih
- Terapkan seleksi yang sama pada data uji
- Dalam validasi silang, pemilihan fitur harus terjadi di dalam setiap lipatan

### Overfitting pada pemilihan fitur
- RFE dengan terlalu banyak iterasi dapat overfit pada himpunan latih
- Validasi pada data yang disisihkan, bukan pada data yang dipakai untuk seleksi
- Gunakan stability selection (ulang pada subsampel) untuk hasil yang lebih tangguh

## Langkah 5: Daftar periksa produksi

- [ ] Ambang varians diterapkan sebagai filter pertama
- [ ] Pemilihan fitur di-fit hanya pada data latih
- [ ] Fitur terpilih terdokumentasi (nama, metode yang dipakai, skor)
- [ ] Kinerja dibandingkan: fitur terpilih versus semua fitur
- [ ] Divalidasi silang, bukan evaluasi satu pembagian
- [ ] Pemilihan fitur terintegrasi ke dalam pipeline pelatihan (tidak dilakukan manual)
- [ ] Pemantauan pergeseran fitur sudah ada (fitur terpilih bisa menjadi usang)
