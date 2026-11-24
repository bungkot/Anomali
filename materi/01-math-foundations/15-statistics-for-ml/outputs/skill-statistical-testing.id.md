---
name: skill-statistical-testing
description: Memilih uji statistik yang tepat untuk membandingkan model ML dan mengevaluasi eksperimen
version: 1.0.0
phase: 1
lesson: 15
tags: [statistics, hypothesis-testing, model-comparison]
---

# Uji Statistik untuk ML

Cara memilih uji yang tepat ketika membandingkan model, menjalankan eksperimen A/B, atau memvalidasi hasil.

## Daftar Pemeriksaan Keputusan

1. Apa yang Anda bandingkan? Rata-rata, proporsi, distribusi, atau korelasi?
2. Berapa banyak kelompok? Satu sampel terhadap referensi, dua kelompok, atau beberapa kelompok?
3. Apakah pengamatan berpasangan (himpunan uji yang sama, fold yang sama) atau independen?
4. Apakah data berdistribusi normal? Jika n < 30 dan jelas tidak normal, gunakan metode nonparametrik.
5. Apakah data kontinu, ordinal, atau kategorikal?
6. Berapa banyak uji yang Anda jalankan? Terapkan koreksi jika lebih dari satu.

## Pohon keputusan

```text
Membandingkan rata-rata?
  Dua kelompok?
    Berpasangan (pembagian data yang sama)? --> Paired t-test (atau Wilcoxon signed-rank jika tidak normal)
    Independen? --> Welch's t-test (atau Mann-Whitney U jika tidak normal)
  Beberapa kelompok?
    Berpasangan? --> ANOVA ukuran berulang (atau uji Friedman)
    Independen? --> One-way ANOVA (atau Kruskal-Wallis)

Membandingkan proporsi?
  Dua kelompok? --> Uji chi-squared atau uji eksak Fisher (n kecil)
  Beberapa kelompok? --> Uji chi-squared

Membandingkan distribusi?
  Apakah salah satu distribusi merupakan referensi? --> Uji Kolmogorov-Smirnov
  Keduanya empiris? --> Uji KS dua sampel

Mengukur asosiasi?
  Keduanya kontinu, kira-kira normal? --> Korelasi Pearson
  Ordinal atau tidak normal? --> Korelasi peringkat Spearman
  Kategorikal x Kategorikal? --> Uji independensi chi-squared

Menjalankan banyak uji?
  Terapkan koreksi Bonferroni: alpha_adjusted = alpha / number_of_tests
  Atau gunakan Holm-Bonferroni (kurang konservatif, tetap mengendalikan family-wise error)
```

## Kapan menggunakan tiap uji

| Uji | Jenis data | Asumsi | Kasus penggunaan ML |
|---|---|---|---|
| Paired t-test | Kontinu, berpasangan | Selisih berdistribusi normal | Membandingkan 2 model pada pembagian k-fold yang sama |
| Wilcoxon signed-rank | Kontinu/ordinal, berpasangan | Tanpa asumsi (nonparametrik) | Membandingkan 2 model, k kecil (5-10 fold) |
| Welch's t-test | Kontinu, independen | Kira-kira normal | Membandingkan model pada dua dataset terpisah |
| Mann-Whitney U | Kontinu/ordinal, independen | Tanpa asumsi | Membandingkan distribusi latensi |
| ANOVA | Kontinu, 3+ kelompok | Normal, varians sama | Membandingkan beberapa arsitektur model |
| Kruskal-Wallis | Kontinu/ordinal, 3+ kelompok | Tanpa asumsi | Membandingkan beberapa model, metrik tidak normal |
| Chi-squared | Hitungan kategorikal | Hitungan harapan >= 5 | Membandingkan distribusi kelas, matriks kebingungan |
| Fisher's exact | Hitungan kategorikal | Sampel kecil | Perbandingan kejadian langka |
| Uji KS | Kontinu | Tanpa asumsi | Memeriksa apakah prediksi mengikuti distribusi yang diharapkan |
| CI Bootstrap | Statistik apa pun | Tanpa asumsi | Selang kepercayaan untuk AUC, F1, metrik apa pun |
| Uji McNemar | Biner berpasangan | Tanpa asumsi | Membandingkan dua klasifier pada himpunan uji yang sama |

## Resep perbandingan model

1. Tentukan metrik dan tingkat signifikansi (alpha = 0.05) sebelum menjalankan eksperimen.
2. Jalankan kedua model pada pembagian validasi silang k-fold yang sama (k = 5 atau 10).
3. Kumpulkan skor berpasangan: (a_1, b_1), (a_2, b_2), ..., (a_k, b_k).
4. Hitung selisih: d_i = b_i - a_i.
5. Jalankan uji berpasangan (Wilcoxon untuk k <= 10, paired t-test untuk k > 10 atau selisih normal).
6. Laporkan: nilai p, selisih rata-rata, selang kepercayaan 95%, ukuran efek (Cohen's d).
7. Jika p < alpha DAN ukuran efek bermakna, selisih itu nyata dan layak ditindaklanjuti.

## Kesalahan umum

- Menggunakan uji independen ketika data berpasangan. Jika kedua model dievaluasi pada fold uji yang sama, Anda wajib menggunakan uji berpasangan. Uji independen membuang informasi pemasangan dan kehilangan daya statistik.
- Melaporkan p < 0.05 tanpa ukuran efek. Peningkatan akurasi 0.1% yang signifikan secara statistik tidak layak untuk diterapkan. Selalu hitung Cohen's d atau selisih rata-rata mentah.
- Membandingkan model lintas himpunan uji yang berbeda. Himpunan uji HARUS identik untuk kedua model. Himpunan uji yang berbeda membuat perbandingan tidak bermakna.
- Menjalankan 20 perbandingan dan melaporkan yang terbaik tanpa koreksi Bonferroni. Dengan 20 uji pada alpha = 0.05, Anda mengharapkan 1 positif palsu secara kebetulan.
- Menggunakan akurasi pada data tidak seimbang. Pada kelas mayoritas 99%, klasifier trivial mencapai 99%. Gunakan F1, AUC presisi-recall, atau koefisien korelasi Matthews.
- Memperlakukan fold validasi silang sebagai sampel independen. Fold tersebut berbagi data pelatihan, yang melanggar asumsi independensi. Uji t resampling terkoreksi (corrected resampled t-test) memperhitungkan hal ini.

## Referensi cepat: interpretasi ukuran efek

| Cohen's d | Interpretasi |
|---|---|
| 0.2 | Efek kecil |
| 0.5 | Efek sedang |
| 0.8 | Efek besar |
| > 1.0 | Efek sangat besar |

| Yang dilaporkan | Mengapa |
|---|---|
| Nilai p | Apakah selisih itu nyata? |
| Selang kepercayaan | Seberapa besar selisihnya bisa? |
| Ukuran efek (Cohen's d) | Apakah selisihnya bermakna? |
| Ukuran sampel (n atau k fold) | Bisakah kita mempercayai hasilnya? |
