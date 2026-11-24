---
name: skill-sampling-strategy
description: Memilih metode pengambilan sampel yang tepat untuk generasi, estimasi, atau inferensi
version: 1.0.0
phase: 1
lesson: 16
tags: [sampling, mcmc, generation]
---

# Pemilihan Strategi Pengambilan Sampel

Cara memilih metode pengambilan sampel yang tepat untuk generasi teks, inferensi Bayes, estimasi Monte Carlo, dan pelatihan.

## Daftar Pemeriksaan Keputusan

1. Apakah Anda menghasilkan keluaran (teks, gambar) atau mengestimasi suatu besaran (integral, ekspektasi)?
2. Dapatkah Anda mengambil sampel langsung dari distribusi target, atau hanya mengevaluasi densitasnya?
3. Apakah distribusi target diskret atau kontinu?
4. Berapa dimensi ruang sampelnya? Rendah (< 5), sedang (5-100), atau tinggi (> 100)?
5. Apakah Anda memerlukan sampel eksak atau perkiraan?
6. Apakah Anda memerlukan gradien melalui operasi pengambilan sampel?

## Kapan menggunakan tiap metode

| Metode | Kapan digunakan | Kompleksitas | Eksak? |
|---|---|---|---|
| Direct sampling | Anda memiliki CDF atau dapat memakai fungsi pustaka | O(1) per sampel | Ya |
| Inverse CDF | Invers CDF bentuk tertutup yang diketahui (eksponensial, Cauchy) | O(1) per sampel | Ya |
| Box-Muller | Memerlukan sampel normal tanpa pustaka | O(1) per sampel | Ya |
| Rejection sampling | Dapat mengevaluasi PDF target, dimensi rendah (1-3) | O(1/acceptance) per sampel | Ya |
| Importance sampling | Memerlukan ekspektasi, bukan sampel individual | O(n) untuk n sampel | Perkiraan |
| Stratified sampling | Estimasi Monte Carlo, menginginkan varians lebih rendah | O(n) untuk n sampel | Perkiraan |
| Metropolis-Hastings | Dimensi tinggi, dapat mengevaluasi densitas takternormalisasi | O(1) per langkah + burn-in | Secara asimtotik |
| Gibbs sampling | Dapat mengambil sampel dari tiap distribusi kondisional | O(d) per sapuan penuh | Secara asimtotik |
| HMC/NUTS | Kontinu dimensi tinggi, densitas halus | O(L * d) per langkah | Secara asimtotik |
| Temperature sampling | Generasi teks LLM, mengendalikan kreativitas | O(V) untuk ukuran kosakata V | N/A |
| Top-k sampling | Generasi LLM, membuang token yang tidak mungkin | O(V log k) | N/A |
| Top-p (nucleus) | Generasi LLM, himpunan kandidat adaptif | O(V log V) | N/A |
| Reparameterization | Memerlukan gradien melalui sampling Gaussian (VAE) | O(d) | Ya |
| Gumbel-Softmax | Memerlukan gradien melalui sampling kategorikal | O(k) untuk k kelas | Perkiraan |

## Pengaturan generasi LLM

| Kasus penggunaan | Temperature | Top-p | Top-k | Catatan |
|---|---|---|---|---|
| Tanya-jawab faktual | 0.0 (greedy) | -- | -- | Deterministik, tanpa acakan |
| Generasi kode | 0.2-0.5 | 0.9 | -- | Kreativitas rendah, koherensi tinggi |
| Obrolan umum | 0.7 | 0.9 | -- | Seimbang |
| Penulisan kreatif | 0.9-1.2 | 0.95 | -- | Keanekaragaman lebih tinggi |
| Brainstorming | 1.0-1.5 | 0.95 | -- | Keanekaragaman maksimal, koherensi bisa turun |

Temperature dan top-p dapat digabungkan. Terapkan temperature terlebih dahulu (skalakan logit), lalu terapkan pemfilteran top-p.

## Pemilihan metode MCMC

| Properti | Metropolis-Hastings | Gibbs | HMC/NUTS |
|---|---|---|---|
| Dimensi | Apa pun | Apa pun (terbaik < 100) | Tinggi (100+) |
| Membutuhkan kondisional | Tidak | Ya | Tidak |
| Membutuhkan gradien | Tidak | Tidak | Ya |
| Tingkat penerimaan | Setel ke ~23% | Selalu 100% | Setel ke ~65% |
| Korelasi | Tinggi (random walk) | Sedang | Rendah |
| Burn-in | Panjang | Sedang | Pendek |
| Terbaik untuk | Eksplorasi, model sederhana | Model konjugat, jaringan Bayes | Posterior kontinu, model probabilistik dalam |

## Kesalahan umum

- Menggunakan rejection sampling pada dimensi tinggi. Tingkat penerimaan turun secara eksponensial seiring dimensi. Di atas 5 dimensi, beralihlah ke MCMC.
- Menetapkan varians proposal MCMC terlalu tinggi atau terlalu rendah. Terlalu tinggi: sebagian besar proposal ditolak, rantai mandek. Terlalu rendah: semua proposal diterima, rantai bergerak lambat. Targetkan penerimaan ~23% untuk MH random walk.
- Lupa melakukan burn-in. Sampel awal dari MCMC bias terhadap titik awal. Buang setidaknya 1000 langkah (atau lebih untuk distribusi kompleks).
- Menggunakan importance sampling dengan proposal yang sangat berbeda dari target. Beberapa sampel mendapat bobot sangat besar sehingga estimasi menjadi tidak andal. Pantau ukuran sampel efektif: ESS = (sum w_i)^2 / sum(w_i^2).
- Menggunakan temperature > 0 untuk tugas yang memerlukan keluaran deterministik (misalnya klasifikasi, ekstraksi terstruktur). Gunakan greedy (T=0) atau beam search sebagai gantinya.
- Tidak menggabungkan top-p dengan temperature. Temperature saja tidak membuang token sampah dari ekor panjang. Top-p melakukannya.
- Melakukan backpropagasi melalui operasi pengambilan sampel standar. Gunakan trik reparameterisasi untuk kasus kontinu (Gaussian) dan Gumbel-Softmax untuk kasus diskret (kategorikal).

## Referensi cepat: teknik reduksi varians

| Teknik | Cara kerja | Reduksi varians |
|---|---|---|
| Stratified sampling | Bagi ruang menjadi strata, ambil sampel dari masing-masing | Selalu <= MC standar |
| Antithetic variates | Gunakan U dan 1-U | Bekerja untuk fungsi monoton |
| Control variates | Kurangi variabel dengan rata-rata yang diketahui | Proporsional dengan korelasi |
| Importance sampling | Berbobot ulang sampel dari proposal yang lebih baik | Bergantung pada kualitas proposal |
| Latin hypercube | Stratifikasi setiap dimensi secara independen | Lebih baik daripada stratified pada dimensi tinggi |
