# Inferensi Async dan Hogwild!

> Dekode spekulatif (Fase 10 · 15) memparallelkan token di dalam satu urutan. Kerangka kerja multi-agen memparallelkan seluruh urutan, tetapi memaksa koordinasi eksplisit (pemungutan suara, pembagian sub-tugas). Inferensi Hogwild! (Rodionov et al., arXiv:2504.06261) melakukan sesuatu yang lain: menjalankan N instance LLM yang sama secara paralel terhadap KV cache BERSAMA. Setiap pekerja (worker) melihat token yang dihasilkan pekerja lain seketika. Model penalaran modern — QwQ, DeepSeek-R1 — dapat mengoordinasi diri lewat cache bersama itu tanpa fine-tuning apa pun. Pendekatan ini masih eksperimental, tetapi ia membuka sumbu paralelisme inferensi yang benar-benar baru dan ortogonal terhadap dekode spekulatif. Pelajaran ini mengimplementasikan simulator Hogwild! dua pekerja dalam Python stdlib dan menjelaskan mengapa kolaborasi lewat cache bersama muncul dari kemampuan penalaran model yang sudah ada.

**Jenis:** Bangun
**Bahasa:** Python (stdlib), Rust
**Prasyarat:** Fase 10 · 12 (optimasi inferensi), Fase 10 · 15 (dekode spekulatif)
**Waktu:** ~60 menit

## Tujuan Pembelajaran

- Mendeskripsikan tiga topologi LLM paralel yang umum (pemungutan suara, sub-tugas, Hogwild!) dan menyebut masalah yang ditargetkan masing-masing.
- Menyebutkan setup inti Hogwild!: beberapa pekerja, satu KV cache bersama, koordinasi yang muncul (emergent) lewat self-prompting.
- Menghitung percepatan wall-time Hogwild! sebagai fungsi dari jumlah pekerja `N`, paralelisme tingkat tugas `p`, dan overhead koordinasi `c`.
- Mengimplementasikan simulator Hogwild! dua pekerja pada problem mainan dan mengamati pembagian tugas yang muncul.

## Masalah

LLM modern memecahkan masalah sulit dengan menghasilkan rantai penalaran panjang — 5000 token logika langkah-demi-langkah itu umum, puluhan ribu token terjadi pada problem matematika mendalam. Pada dekode 35 token/detik untuk model 70B, 50k token adalah 24 menit. Model ini tidak interaktif.

Dekode spekulatif (Fase 10 · 15) memberi percepatan 3-5x dengan memparallelkan di dalam satu urutan. Lewat itu, ketergantungan berurutan dari dekode autoregresif adalah langit-langit keras. Setiap token baru bergantung pada setiap token sebelumnya.

Pertanyaan yang jelas: bisakah kita memparallelkan lintas urutan? Menjalankan beberapa salinan model yang sama pada problem yang sama, membiarkan mereka bekerja sama, meminta mereka membagi pekerjaan?

Pekerjaan sebelumnya: ensembel pemungutan suara (jalankan N model, pilih jawaban mayoritas), tree-of-thought (cabangkan jalur penalaran dan gabungkan kembali), dan kerangka kerja multi-agen (tugaskan sub-tugas pada setiap agen, pakai koordinator). Semua ini membantu pada domain tugas tertentu. Semua juga memperkenalkan mesin koordinasi eksplisit — aturan pemungutan suara, logika cabang-dan-pangkas, protokol pesan antar-agen.

Inferensi Hogwild! mengambil pendekatan berbeda. N pekerja berbagi satu KV cache. Setiap pekerja melihat token yang dihasilkan pekerja lain segera, seolah-olah itu konteksnya sendiri. Para pekerja — tanpa pelatihan atau fine-tuning apa pun — mencari tahu cara membagi pekerjaan. Model penalaran modern (QwQ, DeepSeek-R1, mode penalaran keluarga Claude) dapat membaca cache bersama dan berkata seperti "Saya melihat pekerja 2 sudah menangani kasus dasar, jadi saya akan mengerjakan langkah induksi."

Percepatannya bergantung beban kerja dan masih eksperimental per April 2026. Tetapi idenya layak diketahui karena ia membuka sumbu paralelisme inferensi baru.

## Konsep

### Setup-nya

Inisialisasi N proses pekerja, semuanya menjalankan LLM yang sama. Alih-alih KV cache per pekerja, pertahankan SATU cache bersama. Saat pekerja `i` menghasilkan token `t_j`, token ditulis ke cache bersama pada posisi berikutnya. Saat pekerja `k` mengambil langkah berikutnya, ia membaca keadaan cache saat ini (yang mencakup semua yang telah dihasilkan ke-N pekerja sejauh ini).

Pada saat langkah, pekerja berlomba menulis token. Tidak ada indeks posisi per pekerja — cache-nya adalah satu urutan yang terus bertambah. Urutan ditentukan oleh waktu kedatangan tulisan.

### Mengapa koordinasi muncul

Para pekerja berbagi prompt. Biasanya seperti "Anda adalah salah satu dari N instance yang bekerja sama pada problem ini. Setiap instance membaca memori bersama dan dapat melihat apa yang telah ditulis instance lain. Hindari pekerjaan redundan." Prompt plus cache bersama sudah cukup. Model penalaran membaca cache, mencatat bagian problem yang sudah dicoba, dan (sering, tetapi tidak selalu) beralih ke bagian yang belum dieksplorasi.

Makalah Hogwild! (Rodionov et al., 2025) melaporkan pengamatan seperti:

- Pekerja merumuskan rencana dan mengomunikasikannya ke pekerja lain lewat cache.
- Pekerja mencatat kesalahan dalam penalaran pekerja lain dan menunjukkannya.
- Pekerja beradaptasi saat rencana gagal dan mengusulkan alternatif.
- Saat diminta mengecek redundansi, pekerja mendeteksinya dan beralih.

Tidak ada yang memerlukan fine-tuning. Perilaku yang muncul itu berasal dari kemampuan penalaran yang sudah dimiliki model.

### Penamaannya

Nama makalah ini meledak-ledak dari Hogwild! SGD (Recht et al., 2011), pengoptimal pembaruan asinkron. Analoginya: pekerja asinkron SGD semuanya menulis ke vektor parameter bersama; pekerja Inferensi Hogwild! semuanya menulis ke KV cache bersama. Keduanya bergantung pada konvergensi empiris, bukan jaminan sinkronisasi.

### RoPE membuat ini memungkinkan

Rotary Position Embeddings (RoPE, Su et al. 2021) mengodekan informasi posisi lewat rotasi pada vektor Q dan K. Karena skor atensi antara kueri yang diputar dan kunci yang diputar hanya bergantung pada jarak relatifnya, seluruh blok entri cache dapat dipindah ke offset baru tanpa komputasi ulang: putar kuerinya, alih-alih memutar ulang setiap kunci yang tersimpan. Inferensi Hogwild! melakukan ini sekali per blok cache: untuk kueri pada posisi `i_q` dan blok yang dimulai pada `i_k` dalam pandangan pekerja tersebut, kueri diputar sejauh jarak relatif `i_q - i_k`. Dengan begitu, setiap pekerja melihat blok cache bersama sesuai urutannya sendiri. Trik ini bekerja untuk pergeseran seragam pada blok kontigu. Memindahkan token individual ke posisi berbeda akan mengubah rotasinya dan tetap membutuhkan komputasi ulang.

Pada model dengan posisi terpelajari atau posisi absolut, Hogwild! memerlukan invalidasi cache pada setiap penulisan konkuren. RoPE membuat cache tetap stabil.

### Matematika wall-time

Misalkan `T_serial` adalah waktu bagi satu pekerja menyelesaikan problem sendirian. Misalkan `p` adalah fraksi yang dapat diparallelkan pada tingkat tugas. Misalkan `c` adalah overhead koordinasi per langkah (membaca cache yang diperpanjang, memutuskan apa yang ditulis).

Waktu satu pekerja: `T_serial`.
Waktu Hogwild! N pekerja, jika koordinasi gratis: `T_serial * ((1 - p) + p / N)`. Amdahl klasik.
Dengan overhead koordinasi: `T_serial * ((1 - p) + p / N) + c * steps_per_worker`.

Agar pekerja produktif, `c` harus kecil relatif terhadap waktu dekode per langkah. Pada model penalaran yang menghasilkan 5k+ token, pekerja mampu menyisihkan ratusan token untuk overhead koordinasi dan tetap untung. Pada tugas obrolan pendek, koordinasi mendominasi dan Hogwild! lebih buruk daripada serial.

### Contoh konkret

Problem penalaran: 10k token chain-of-thought. Anggap problem memiliki `p = 0.7` konten yang dapat diparallelkan (strategi pembuktian berbeda, analisis kasus berbeda) dan `c = 200` token overhead koordinasi per pekerja. Dengan `N = 4` pekerja:

- Waktu serial: 10000 langkah dekode.
- Waktu Hogwild!: 10000 * (0.3 + 0.7 / 4) + 200 * 4 = 10000 * 0.475 + 800 = 5550 langkah dekode.
- Percepatan: 10000 / 5550 = 1.8x.

Itu moderat. Tetapi pada problem penalaran lebih panjang (50k token), overhead koordinasi teramortisasi dan percepatan mendorong 2,5-3x. Hogwild! adalah padanan inferensi dari paralelisme tingkat utas (thread-level) dalam bahasa yang memungkinkan Anda menulis kode multithread secara alami.

### Kapan memakai Hogwild!

- Problem penalaran panjang (ribuan token) yang dapat diparallelkan lintas sub-tujuan independen.
- Model penalaran yang telah dilatih untuk berpikir langkah demi langkah. Model non-penalaran tidak mengoordinasi diri dengan baik.
- Deploy satu simpul dengan VRAM cukup untuk menampung cache bersama plus N proses pekerja. Cache-nya bersama, tetapi setiap pekerja punya memori aktivasi sendiri.

### Kapan tidak

- Obrolan interaktif pendek. Overhead koordinasi mendominasi.
- Tugas yang tidak dapat diparallelkan (satu pembuktian linear, satu kompilasi). N=1 adalah maksimum.
- Model non-penalaran. Tidak ada koordinasi yang muncul.
- Deploy multi-simpul. Cache bersama memerlukan sinkronisasi antarpekerja yang sangat cepat. Di dalam simpul tidak masalah; lintas simpul adalah bencana latensi.

### Status eksperimental

Per April 2026, Hogwild! adalah metode riset dengan implementasi PyTorch sumber terbuka. Adopsi produksi belum terjadi. Tiga penghalang:

1. Manajemen KV cache bersama lintas proses konkuren adalah rekayasa yang tidak mudah.
2. Koordinasi yang muncul bergantung tugas; benchmark masih dibangun.
3. Percepatannya moderat dibandingkan apa yang sudah diberikan dekode spekulatif, dan keduanya dapat digabungkan, tetapi rekayasa gabungannya adalah lapisan lain.

Layak diketahui. Layak bereksperimen. Belum layak dipertaruhkan untuk produk.

```figure
continuous-batching
```

## Bangun

`code/main.py` mengimplementasikan simulator Hogwild! mainan:

- Dua proses pekerja, masing-masing "LLM" deterministik yang menghasilkan salah satu dari beberapa kategori token (work-token, observe-token, coordinate-token) dengan probabilitas diketahui.
- Cache bersama (hanya daftar token) yang dibaca dan ditulis kedua pekerja.
- Logika koordinasi sederhana: saat pekerja melihat pekerja lain sudah menghasilkan cukup work-token dalam satu kategori, ia memilih kategori lain.

Simulator berjalan pada anggaran langkah tetap dan melaporkan:

- Total work-token yang dihasilkan.
- Total wall time (jumlah langkah pekerja).
- Percepatan efektif terhadap satu pekerja.
- Jejak pekerja mana yang menulis token mana.

### Langkah 1: cache bersama

Daftar yang ditambahkan kedua pekerja. Penguncian sederhana (`threading.Lock` Python) dalam implementasi nyata; kita simulasikan dengan penghitung.

### Langkah 2: loop pekerja

Setiap pekerja, pada setiap langkah:

- Membaca cache bersama saat ini.
- Memutuskan kategori token apa yang ditulis berdasarkan apa yang sudah ada.
- Menulis satu token.

### Langkah 3: heuristik koordinasi

Jika kategori X sudah memiliki K token di cache dan kategori yang dimaksud pekerja adalah X, pekerja beralih ke kategori Y. Ini pengganti mainan untuk perilaku model penalaran "perhatikan ini sudah tercakup, lakukan hal lain saja."

### Langkah 4: percepatan terukur

Jalankan simulator dengan N=1 pekerja dan N=2 pekerja, anggaran langkah total sama. Hitung work-token yang dihasilkan. N=2 seharusnya menghasilkan sekitar 1,5-1,8x lebih banyak work-token berkat pembagian tugas yang digerakkan koordinasi.

### Langkah 5: uji koordinasi

Turunkan sensitivitas heuristik koordinasi. Jalankan lagi. Amati bahwa tanpa koordinasi baik, N=2 menghasilkan token yang sama secara redundan dan percepatan turun di bawah 1. Ini sesuai pengamatan makalah: trik ini hanya bekerja jika pekerja memiliki kapasitas penalaran untuk mengoordinasi diri.

## Gunakan

Integrasi Hogwild! dalam produksi per April 2026 bersifat setara-riset. Implementasi referensi dari Yandex/HSE/IST berbasis PyTorch dan menargetkan setup multiproses satu simpul pada model DeepSeek-R1 dan QwQ.

Jalur adopsi pragmatis:

1. Profilkan beban kerja tugas penalaran Anda. Ukur fraksi token yang bersifat eksploratif (beberapa strategi, analisis kasus, pencarian) vs linear.
2. Jika eksplorasi dominan, jalankan percobaan Hogwild! dua pekerja. Ukur perbaikan wall-time.
3. Jika perbaikannya di bawah 1,3x, Anda berada pada rezim dominan-koordinasi. Kembali ke satu pekerja.
4. Jika perbaikannya di atas 1,5x, naik ke N=4 dan ukur lagi. Pengembalian menurun biasanya mulai pada N=4-8.

Gabungkan dengan dekode spekulatif: setiap pekerja Hogwild! dapat memakai dekode spekulatif secara independen. Kedua percepatan saling mengalikan (kira-kira), sehingga dekode spekulatif 3x dan Hogwild! 1,8x menjadi 5,4x efektif terhadap dekode satu pekerja naif.

## Kirim

Pelajaran ini menghasilkan `outputs/skill-parallel-inference-router.md`. Diberikan profil beban kerja penalaran (anggaran token, profil paralelisme tugas, keluarga model, target deploy), berkas ini memilih antarstrategi pemungutan suara, tree-of-thought, multi-agen, Hogwild!, dan dekode spekulatif.

## Latihan

1. Jalankan `code/main.py` dengan pengaturan default. Konfirmasi konfigurasi Hogwild! N=2 menghasilkan lebih banyak work-token daripada baseline N=1 pada wall time yang sama.

2. Turunkan kekuatan heuristik koordinasi (setel `coordination_weight=0.1`). Jalankan ulang. Tunjukkan percepatan runtuh. Jelaskan mengapa: pekerja menduplikasi usaha ketika mereka tidak dapat berkoordinasi.

3. Hitung percepatan Hogwild! yang diharapkan untuk tugas penalaran 50k-token dengan `p=0.8, c=500` dan N=4 pekerja. Lakukan hal yang sama untuk tugas obrolan 1k-token dengan `p=0.3, c=200` dan N=4. Mengapa yang satu menang dan yang lain kalah?

4. Bacalah Bagian 4 makalah Hogwild! (evaluasi pendahuluan). Identifikasi dua mode kegagalan yang dilaporkan penulis. Jelaskan bagaimana prompt koordinasi yang lebih baik dapat meringankan masing-masing.

5. Gabungkan Hogwild! dengan dekode spekulatif pada simulator: setiap pekerja memakai dekode spekulatif 2-token secara internal. Laporkan percepatan multiplikatif. Masalah pembukuan apa yang muncul ketika dua pekerja sama-sama ingin memperpanjang prefiks cache bersama yang sama?

## Istilah Kunci

| Istilah | Yang orang katakan | Arti sebenarnya |
|---------|--------------------|-----------------|
| Hogwild! | "Pekerja paralel, cache bersama" | N instance LLM yang sama berjalan secara konkuren dengan satu KV cache bersama; koordinasi muncul lewat self-prompting |
| KV cache bersama | "Medium koordinasi" | Satu buffer KV yang terus bertambah dan dibaca serta ditulis semua pekerja; membuat token langsung terlihat lintas pekerja |
| Koordinasi yang muncul | "Tanpa pelatihan" | LLM yang mampu bernalar dapat membaca cache bersama dan membagi pekerjaan tanpa fine-tuning atau protokol eksplisit |
| Overhead koordinasi (c) | "Token yang dipakai untuk orientasi" | Biaya per pekerja untuk membaca cache yang diperpanjang dan memutuskan apa yang dilakukan; harus tetap kecil terhadap total waktu dekode |
| Fraksi yang dapat diparallelkan (p) | "Apa yang bisa jalan paralel" | Paralelisme tingkat tugas: fraksi dari total pekerjaan yang tidak bersifat intrinsik berurutan |
| RoPE memungkinkan Hogwild! | "Posisi rotary bersifat invarian-pergeseran" | Karena posisi adalah rotasi, menulis ke cache bersama tidak memerlukan komputasi ulang token sebelumnya |
| Ensembel pemungutan suara | "Jalankan N, pilih mayoritas" | Topologi inferensi paralel paling sederhana; berguna untuk klasifikasi, kurang untuk penalaran panjang |
| Tree of thought | "Cabang dan pangkas" | Strategi penalaran yang mengeksplorasi banyak cabang dan memangkasnya; logika koordinasi eksplisit |
| Kerangka kerja multi-agen | "Tugaskan sub-tugas" | Setiap agen mendapat peran; koordinator mengorkestrasi; overhead protokol berat |

## Bacaan Lanjutan

- [Rodionov et al. — Hogwild! Inference: Parallel LLM Generation via Concurrent Attention (arXiv:2504.06261)](https://arxiv.org/abs/2504.06261) — makalah Hogwild!, evaluasi pendahuluan pada QwQ dan DeepSeek-R1
- [Recht, Re, Wright, Niu — Hogwild!: A Lock-Free Approach to Parallelizing Stochastic Gradient Descent (arXiv:1106.5730, NeurIPS 2011)](https://arxiv.org/abs/1106.5730) — Hogwild! asli, asal-usul penamaannya
- [Su et al. — RoFormer: Enhanced Transformer with Rotary Position Embedding (arXiv:2104.09864)](https://arxiv.org/abs/2104.09864) — RoPE, sifat yang membuat inferensi cache bersama memungkinkan
- [Yao et al. — Tree of Thoughts: Deliberate Problem Solving with Large Language Models (arXiv:2305.10601)](https://arxiv.org/abs/2305.10601) — strategi penalaran tree-of-thought yang ortogonal dengan Hogwild!
- [Leviathan et al. — Fast Inference from Transformers via Speculative Decoding (arXiv:2211.17192)](https://arxiv.org/abs/2211.17192) — dekode spekulatif, paralelisme dalam urutan yang dikomposisikan dengan Hogwild!
- [Implementasi PyTorch referensi Hogwild!](https://github.com/eqimp/hogwild_llm) — satu-satunya sumber resmi eksperimen makalah
