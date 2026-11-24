---
name: skill-naive-bayes-chooser
description: Pilih varian Naive Bayes yang tepat untuk tugas klasifikasi Anda
phase: 2
lesson: 14
---

Anda adalah pakar dalam klasifikasi probabilistik. Ketika seseorang perlu memilih varian Naive Bayes, pandu mereka melalui proses pengambilan keputusan ini.

## Daftar Periksa Keputusan

### Langkah 1: Apa fitur Anda?

- **Hitungan kata atau nilai TF-IDF** -> MultinomialNB
- **Pengukuran kontinu (suhu, tinggi badan, pembacaan sensor)** -> GaussianNB
- **Indikator biner (kata hadir/tidak hadir, status kotak centang)** -> BernoulliNB
- **Jenis campuran** -> Pisahkan menjadi himpunan bagian, atau ubah semuanya menjadi satu jenis

### Langkah 2: Seberapa banyak data yang Anda miliki?

- **Di bawah 1.000 sampel**: Naive Bayes adalah pilihan yang kuat. Prior yang kuatnya (asumsi independensi) mencegah overfitting.
- **1.000 sampai 50.000 sampel**: NB masih kompetitif. Bandingkan dengan regresi logistik.
- **Di atas 50.000 sampel**: Regresi logistik atau gradient boosting kemungkinan akan mengungguli NB. Gunakan NB sebagai garis dasar.

### Langkah 3: Setel smoothing

- Mulai dengan alpha=1.0 (smoothing Laplace).
- Jika akurasi rendah dan Anda memiliki cukup data, coba alpha=0.1 atau 0.01.
- Jika model mengalami overfitting (akurasi latih >> akurasi uji), naikkan alpha menjadi 5.0 atau 10.0.
- Selalu validasi smoothing dengan validasi silang, bukan satu kali pembagian latih/uji.

### Langkah 4: Periksa asumsi

- **MultinomialNB**: Fitur harus tak-negatif. Jika Anda memiliki nilai negatif, geser nilainya atau gunakan GaussianNB.
- **GaussianNB**: Bekerja paling baik ketika fitur berbentuk kurva lonceng di dalam setiap kelas. Periksa dengan histogram.
- **BernoulliNB**: Binarisasi fitur Anda terlebih dahulu. Pilih ambang dengan saksama (untuk teks: hadir=1, tidak hadir=0).

## Kesalahan Umum

1. **Menggunakan GaussianNB pada data teks.** Hitungan kata tidak berdistribusi Gaussian. Gunakan MultinomialNB.
2. **Lupa melakukan smoothing Laplace.** Satu kata yang belum pernah terlihat membuat seluruh probabilitas menjadi nol. Selalu lakukan smoothing.
3. **Terlalu percaya pada keluaran probabilitas.** Probabilitas NB kurang terkalibrasi. Gunakan hanya untuk peringkat, bukan sebagai skor kepercayaan. Jika Anda memerlukan probabilitas yang terkalibrasi, gunakan CalibratedClassifierCV.
4. **Mengabaikan ketidakseimbangan kelas.** Prior NB mencerminkan frekuensi kelas. Dengan 99% negatif dan 1% positif, prior mengalahkan likelihood. Sesuaikan prior secara manual atau lakukan resampling.

## Rujukan Cepat

| Pertanyaan | MultinomialNB | GaussianNB | BernoulliNB |
|------------|:---:|:---:|:---:|
| Klasifikasi teks? | Ya | Tidak | Mungkin (teks pendek) |
| Fitur kontinu? | Tidak | Ya | Tidak |
| Fitur biner? | Tidak | Tidak | Ya |
| Perlu pelatihan sangat cepat? | Ya | Ya | Ya |
| Himpunan latih kecil? | Baik | Baik | Baik |
| Perlu probabilitas terkalibrasi? | Tidak | Tidak | Tidak |

## Kapan TIDAK Menggunakan Naive Bayes

- Fitur sangat berkorelasi dan Anda memiliki cukup data untuk model yang mampu menangani korelasi (regresi logistik, gradient boosting)
- Anda memerlukan akurasi setinggi mungkin dan memiliki banyak data
- Fitur Anda berupa citra, sekuens, atau graf (gunakan jaringan saraf)
- Anda memerlukan model yang menangkap interaksi fitur (gunakan metode berbasis pohon)
