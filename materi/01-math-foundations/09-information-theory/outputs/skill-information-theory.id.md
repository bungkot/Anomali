---
name: skill-information-theory
description: Terapkan konsep teori informasi pada fungsi rugi model, evaluasi model, dan pemilihan fitur
version: 1.0.0
phase: 1
lesson: 9
tags: [information-theory, entropy, loss-functions]
---

# Teori Informasi untuk ML

Kapan menggunakan entropi, cross-entropy, divergensi KL, dan informasi bersama dalam sistem pembelajaran mesin.

## Daftar Periksa Keputusan

1. Mengukur ketidakpastian dalam satu distribusi? Gunakan **entropi**.
2. Mengukur seberapa baik model mendekati label sejati? Gunakan **cross-entropy** (ini adalah fungsi rugi klasifikasi Anda).
3. Mengukur jarak antara dua distribusi? Gunakan **divergensi KL**.
4. Memeriksa apakah dua variabel saling terkait? Gunakan **informasi bersama**.
5. Melaporkan kualitas model bahasa? Gunakan **perpleksitas** (eksponensial dari cross-entropy).
6. Mendistilasi satu model ke model lain? Minimalkan **divergensi KL** dari guru ke murid.

## Kapan menggunakan setiap ukuran

| Ukuran | Rumus | Kasus penggunaan | Aplikasi ML |
|---|---|---|---|
| Entropi H(P) | -sum(p log p) | Seberapa tidak pasti distribusi ini? | Kompleksitas data, model entropi maksimum |
| Cross-entropy H(P,Q) | -sum(p log q) | Seberapa baik model Q memprediksi P yang sejati? | Fungsi rugi klasifikasi, fungsi rugi model bahasa |
| Divergensi KL D(P\|\|Q) | sum(p log(p/q)) | Seberapa berbeda P dan Q? | Fungsi rugi VAE (ELBO), distilasi pengetahuan, RLHF |
| Informasi bersama I(X;Y) | H(X) - H(X\|Y) | Seberapa banyak Y memberi tahu kita tentang X? | Pemilihan fitur, pembelajaran representasi |
| Perpleksitas | exp(H(P,Q)) atau 2^H | Seberapa bingung modelnya? | Evaluasi model bahasa |
| Entropi bersyarat H(X\|Y) | -sum(p(x,y) log p(x\|y)) | Ketidakpastian yang tersisa pada X setelah mengetahui Y | Informativitas fitur |

## Hubungan utama

```
Cross-entropy  = Entropy + KL divergence
H(P, Q)        = H(P)   + D_KL(P || Q)

Karena H(P) konstan selama pelatihan:
  Meminimalkan cross-entropy = Meminimalkan divergensi KL

Informasi bersama = Entropi - Entropi bersyarat
I(X; Y) = H(X) - H(X|Y) = H(Y) - H(Y|X)

Perpleksitas = exp(cross-entropy dalam nats)
             = 2^(cross-entropy dalam bits)
```

## Referensi cepat: rumus dan satuan

| Rumus | Bits (log dasar 2) | Nats (log dasar e) |
|---|---|---|
| Informasi: -log(p) | -log2(p) | -ln(p) |
| Entropi: -sum(p log p) | bits | nats |
| 1 nat = | 1.4427 bits | 1 nat |
| Bawaan PyTorch | -- | nats |
| Makalah teori informasi | bits | -- |

## Menafsirkan nilai

| Nilai entropi | Artinya |
|---|---|
| 0 | Deterministik. Satu hasil memiliki probabilitas 1. |
| log(n) | Ketidakpastian maksimum. Distribusi seragam atas n hasil. |
| Rendah | Distribusi memuncak. Model percaya diri. |
| Tinggi | Distribusi landai. Model tidak pasti. |

| Nilai perpleksitas | Kualitas model bahasa |
|---|---|
| 1 | Prediksi sempurna (tidak pernah terjadi dalam praktik) |
| 10 | Rata-rata memilih di antara ~10 token yang sama kemungkinannya |
| 50 | Setara GPT-2 pada benchmark standar |
| < 10 | State-of-the-art untuk domain yang terwakili dengan baik |

## Kesalahan umum

- Menghitung divergensi KL dan menganggapnya simetris. D_KL(P\|\|Q) != D_KL(Q\|\|P). Untuk ukuran yang simetris, gunakan divergensi Jensen-Shannon: JS = 0.5 * KL(P\|\|M) + 0.5 * KL(Q\|\|M) dengan M = 0.5*(P+Q).
- Lupa bahwa cross-entropy dengan label one-hot menyederhana menjadi -log(p_true_class). Anda tidak perlu menjumlahkan semua kelas ketika distribusi sejatinya adalah one-hot.
- Menggunakan log dasar 2 dalam kode tetapi melaporkan nats (atau sebaliknya). PyTorch menggunakan log alami secara bawaan. Kalikan dengan log2(e) = 1.4427 untuk mengonversi nats ke bits.
- Menghitung entropi dari peristiwa kosong atau berprobabilitas nol. Konvensi: 0 * log(0) = 0, karena lim(p->0) p*log(p) = 0.
- Membandingkan perpleksitas antar kosakata yang berbeda. Model dengan ukuran kosakata 50k dan perpleksitas 30 tidak dapat dibandingkan langsung dengan model berukuran kosakata 10k dan perpleksitas 30.

## Di mana setiap konsep muncul dalam ML produksi

| Konsep | Tempat Anda melihatnya |
|---|---|
| Fungsi rugi cross-entropy | Setiap model klasifikasi (nn.CrossEntropyLoss) |
| Divergensi KL | ELBO VAE, pemotongan PPO, distilasi pengetahuan |
| Regularisasi entropi | Bonus eksplorasi dalam RL (entropi lebih tinggi = lebih banyak eksplorasi) |
| Informasi bersama | Pemilihan fitur, fungsi rugi InfoNCE (pembelajaran kontrastif) |
| Perpleksitas | Benchmark model bahasa (lebih rendah = lebih baik) |
| Pelurusan label (label smoothing) | Mengganti one-hot dengan target lunak, mengurangi kepercayaan berlebih cross-entropy |
| Penskalaan suhu | Membagi logits dengan T sebelum softmax, mengontrol entropi keluaran |
