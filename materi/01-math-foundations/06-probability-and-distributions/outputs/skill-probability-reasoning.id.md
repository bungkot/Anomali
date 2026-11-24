---
name: skill-probability-reasoning
description: Memilih distribusi probabilitas yang tepat untuk suatu masalah ML
version: 1.0.0
phase: 1
lesson: 6
tags: [probability, distributions, modeling]
---

# Pemilihan Distribusi Probabilitas

Cara memilih distribusi yang tepat saat memodelkan data, merancang fungsi rugi, atau menetapkan prior.

## Daftar Periksa Keputusan

1. Apakah hasil diskret (kategori, jumlah) atau kontinu (pengukuran, skor)?
2. Apakah hasil terbatas (mis. [0, 1]) atau tidak terbatas?
3. Berapa banyak hasil yang mungkin? Dua? k? Tak hingga?
4. Apakah data simetris atau miring?
5. Apakah peristiwa saling merdeka atau berkorelasi?
6. Apakah Anda memodelkan laju, jumlah, proporsi, atau pengukuran?

## Pohon keputusan distribusi

```
Apakah variabel diskret?
  Ya --> Hanya 2 hasil? --> Bernoulli (p)
     |    k hasil, satu percobaan? --> Categorical (p1...pk)
     |    k hasil, n percobaan? --> Multinomial (n, p1...pk)
     |    Jumlah keberhasilan dalam n percobaan? --> Binomial (n, p)
     |    Jumlah peristiwa per interval? --> Poisson (lambda)
     |    Jumlah percobaan hingga keberhasilan pertama? --> Geometric (p)
     |    Jumlah percobaan hingga r keberhasilan? --> Negative Binomial (r, p)
  Tidak --> Simetris, berbentuk lonceng? --> Normal (mu, sigma)
     |   Nilai positif, miring ke kanan? --> Log-normal atau Exponential
     |   Terbatas dalam [0, 1]? --> Beta (alpha, beta)
     |   Nilai positif, bentuk fleksibel? --> Gamma (alpha, beta)
     |   Waktu antarperistiwa? --> Exponential (lambda)
     |   Perlu ekor berat? --> Student's t (nu) atau Cauchy
     |   Multivariabel, berbentuk lonceng? --> Multivariate Normal
     |   Di atas simpleks (berjumlah 1)? --> Dirichlet (alpha)
```

## Memetakan skenario ML dunia nyata ke distribusi

| Skenario | Distribusi | Parameter |
|---|---|---|
| Keluaran klasifikasi biner | Bernoulli | p = sigmoid(logit) |
| Keluaran klasifikasi multikelas | Categorical | p = softmax(logits) |
| Prediksi token dalam model bahasa | Categorical atas kosakata | p dari softmax |
| Intensitas piksel (ternormalisasi) | Beta atau Uniform [0, 1] | Bergantung pada statistik citra |
| Jumlah kata dalam dokumen | Poisson | lambda = rata-rata jumlah kata |
| Waktu antarpermintaan pengguna | Exponential | lambda = laju permintaan |
| Galat pengukuran | Normal | mu = 0, sigma dari data |
| Inisialisasi bobot | Normal atau Uniform | Aturan Kaiming/Xavier |
| Prior ruang laten VAE | Normal Standar | mu = 0, sigma = 1 |
| Prior Bayes pada proporsi | Beta | alpha, beta dari keyakinan |
| Prior Bayes pada bobot kategori | Dirichlet | vektor alpha |
| Derau pada target regresi | Normal | mu = 0, sigma diestimasi |
| Regresi tangguh terhadap pencilan | Student's t | derajat kebebasan rendah |
| Pemodelan durasi/umur | Weibull atau Gamma | bentuk dan skala |
| Distribusi topik per dokumen (LDA) | Dirichlet | alpha < 1 untuk kejarangan |

## Saat distribusi menjadi salah

- Memakai Normal ketika data memiliki batas bawah yang tegas (mis. harga, jarak). Distribusi normal memberikan probabilitas bukan nol pada nilai negatif. Gunakan log-normal atau gamma sebagai gantinya.
- Memakai Poisson ketika variansi berbeda dari rata-rata. Poisson mengasumsikan rata-rata = variansi. Jika variansi > rata-rata, gunakan binomial negatif.
- Memakai Bernoulli untuk masalah multikelas. Bernoulli bersifat ketat biner. Gunakan kategorikal untuk k > 2.
- Mengasumsikan kemerdekaan ketika pengamatan berkorelasi. Deret waktu, data spasial, dan data berkelompok melanggar kemerdekaan. Gunakan model autoregresif atau hierarkis.

## Kesalahan umum

- Mengira nilai PDF sebagai probabilitas. PDF dapat melebihi 1. Probabilitas diperoleh dari integrasi PDF atas suatu interval.
- Lupa bahwa keluaran softmax adalah probabilitas kategorikal, bukan probabilitas Bernoulli yang merdeka. Keluarannya berjumlah 1 secara konstruksi.
- Memakai prior seragam ketika Anda memiliki pengetahuan domain. Prior informatif mengurangi variansi tanpa memihak hasil jika dipilih dengan baik.
- Memperlakukan log-probabilitas sebagai probabilitas. Log-prob selalu negatif (atau nol). Log-prob tidak berjumlah 1.

## Referensi cepat: sifat-sifat distribusi

| Distribusi | Support | Rata-rata | Variansi | Sifat kunci |
|---|---|---|---|---|
| Bernoulli(p) | {0, 1} | p | p(1-p) | Diskret paling sederhana |
| Binomial(n, p) | {0..n} | np | np(1-p) | Jumlah n Bernoulli |
| Poisson(lam) | {0, 1, 2, ...} | lam | lam | Rata-rata = variansi |
| Normal(mu, s^2) | (-inf, inf) | mu | s^2 | Entropi maksimum untuk rata-rata/variansi tertentu |
| Exponential(lam) | [0, inf) | 1/lam | 1/lam^2 | Tanpa memori |
| Beta(a, b) | [0, 1] | a/(a+b) | ab/((a+b)^2(a+b+1)) | Konjugat dari Binomial |
| Gamma(a, b) | (0, inf) | a/b | a/b^2 | Konjugat dari Poisson |
| Dirichlet(alpha) | Simpleks | alpha_i/sum | (lihat rumus) | Konjugat dari Categorical |
