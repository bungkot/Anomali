# Atensi Jarang Native (DeepSeek NSA)

> Pada 64k token, atensi mengonsumsi 70-80% latensi dekode. Setiap laboratorium model terbuka punya rencana untuk memperbaikinya. NSA dari DeepSeek (makalah terbaik ACL 2025) adalah yang bertahan: tiga cabang atensi paralel — token kasar terkompresi, token butir halus yang dipertahankan secara selektif, dan jendela geser untuk konteks lokal — digabungkan melalui gerbang yang dipelajari. Teknik ini selaras dengan perangkat keras (ramah kernel), dapat dilatih secara native (bekerja dalam pra-pelatihan, bukan dipasang tambahan saat inferensi), dan pada dekode 64k ia berjalan lebih cepat daripada FlashAttention sambil menyamai atau mengungguli kualitas atensi penuh. Pelajaran ini membangun ketiga cabang dari awal hingga akhir dan menunjukkan mengapa kejarangannya dapat terdiferensiasi secara end-to-end.

**Jenis:** Membangun
**Bahasa:** Python (stdlib), Rust
**Prasyarat:** Fase 7 · 12 (KV cache, flash-attention), Fase 7 · 15 (varian atensi), Fase 10 · 16 (atensi diferensial)
**Waktu:** ~60 menit

## Tujuan Pembelajaran

- Menyebut ketiga cabang atensi NSA dan apa yang ditangkap masing-masing.
- Menjelaskan mengapa NSA "dapat dilatih secara native" sementara metode atensi jarang sebelumnya hanya berlaku saat inferensi.
- Menghitung penghematan komputasi atensi NSA dibanding atensi penuh pada konteks 64k sebagai fungsi dari ukuran blok kompresi dan top-k seleksi.
- Mengimplementasikan kombinasi tiga cabang dalam Python stdlib pada urutan sintetik pendek dan memverifikasi perilaku bobot gerbang.

## Masalah

Atensi penuh pada panjang urutan N memerlukan waktu `O(N^2)` dan `O(N)` KV cache per lapisan. Pada 64k token, angka komputasi dan bandwidth memori sangat besar. Estimasi teoretis terukur dari makalah NSA: atensi menyumbang 70-80% dari total latensi dekode pada 64k. Semua yang di hilir — TTFT, token/detik, biaya per juta token — didominasi oleh biaya atensi.

Atensi jarang adalah jawaban yang jelas. Upaya sebelumnya terbagi dalam dua kelompok. Kejarangan pola tetap (jendela geser, bergerak melangkah, blok-lokal) membuang informasi dan gagal pada tugas pengambilan informasi jarak jauh. Kejarangan saat inferensi (pemangkasan KV cache, H2O, StreamingLLM) diterapkan pada model yang pra-dilatih dengan atensi padat dan hanya memulihkan sebagian dari potensi percepatan karena model tidak pernah diminta merutekan informasi melalui pola jarang itu.

Native Sparse Attention (Yuan et al., DeepSeek + PKU + UW, makalah terbaik ACL 2025, arXiv:2502.11089) melakukan keduanya: pola kejarangan yang dipelajari model selama pra-pelatihan, diterapkan sebagai algoritme yang selaras dengan kernel dan benar-benar memberikan penghematan komputasi saat inferensi. Dua tahun dari sekarang, NSA atau turunan langsungnya adalah atensi bawaan pada setiap model konteks panjang terdepan.

## Konsep

### Tiga cabang paralel

Untuk setiap query, NSA menjalankan atensi tiga kali, terhadap tiga tampilan berbeda dari KV cache:

1. **Cabang terkompresi.** Token dikelompokkan menjadi blok berukuran `l` (biasanya 32 atau 64). Setiap blok dikompresi menjadi satu token ringkasan melalui MLP kecil yang dipelajari. Query menghadiri token-token terkompresi ini, memperoleh tampilan butir kasar dari seluruh urutan.

2. **Cabang terpilih.** Menggunakan skor atensi dari cabang terkompresi, blok-blok top-k yang paling relevan dengan query saat ini diidentifikasi. Token butir halus (tidak terkompresi) dari blok-blok itu dibaca dan query menghadiri semuanya. Anggap atensi cabang terkompresi sebagai sinyal perutean untuk seleksi.

3. **Cabang jendela geser.** Query menghadiri `W` token terakhir (biasanya 512) untuk konteks lokal. Cabang ini menangkap pola jarak pendek yang kaya struktur (sintaks, coreferensi lokal) yang mungkin terlewat oleh dua cabang lainnya.

Keluaran ketiga cabang digabungkan melalui gerbang per posisi yang dipelajari:

```math
\text{out} = g_{\text{cmp}} \cdot \text{out}_{\text{cmp}} + g_{\text{sel}} \cdot \text{out}_{\text{sel}} + g_{\text{win}} \cdot \text{out}_{\text{win}}
```

`g_cmp, g_sel, g_win` adalah bobot gerbang dari MLP kecil pada query. Bobot itu tidak harus berjumlah 1 — mereka dapat membobot cabang secara independen.

### Mengapa ini "dapat dilatih secara native"

Langkah seleksi (blok top-k) bersifat diskrit. Operasi diskrit memutus aliran gradien. Penelitian atensi jarang sebelumnya baik melewati propagasi balik melalui seleksi (membatasi pelatihan) maupun menggunakan relaksasi kontinu yang tidak menghasilkan kejarangan nyata saat inferensi.

NSA mengakalinya: atensi cabang terkompresi ADALAH atensi butir kasar yang dapat terdiferensiasi pada seluruh urutan. Operasi top-k hanya menggunakan kembali skor atensi teratas dari cabang terkompresi untuk memilih blok butir halus mana yang akan dimuat. Gradien mencapai skor cabang terkompresi melalui keluaran cabang terkompresi itu sendiri, bukan melalui pilihan blok. Di dalam blok yang dipilih, atensi atas kunci dan nilai terpilih adalah atensi biasa, sehingga kontribusinya terhadap keluaran akhir juga dapat terdiferensiasi. Pilihan indeks `top_k` sendiri tidak mendapat gradien: ia hanya mengontrol blok mana yang dimuat dari memori.

Inilah sebabnya NSA dapat digunakan dalam pra-pelatihan secara end-to-end. Model belajar merutekan informasi melalui ketiga cabang secara bersama-sama, menghasilkan pola jarang yang saat inferensi benar-benar memberikan percepatan yang dijanjikan.

### Kernel yang selaras dengan perangkat keras

Kernel NSA dirancang untuk hierarki memori GPU modern. Kernel memuat query per kelompok GQA (loop luar), mengambil blok KV jarang yang bersesuaian per kelompok (loop dalam), dan menjalankan atensi pada SRAM. Karena setiap kelompok query melihat blok terpilih yang sama (seleksi bersifat per-kelompok-query, bukan per-head-query), pemuatan KV diamortisasi di seluruh kelompok. Intensitas aritmetika tetap tinggi.

Makalah melaporkan kernel Triton yang berjalan 9x lebih cepat daripada FlashAttention pada dekode 64k, dengan rasio percepatan yang bertambah seiring panjang urutan. Kernel forward dan backward keduanya disediakan.

### Anggaran komputasi

Misalkan `N` adalah panjang urutan, `l` ukuran blok kompresi, `k` jumlah seleksi top-k, `w` jendela geser, dan `b` ukuran blok terpilih (biasanya sama dengan `l`).

- Cabang terkompresi: `O(N/l)` kunci per query, sehingga `O(N * N / l)` total.
- Cabang terpilih: `O(k * b)` kunci per query, sehingga `O(N * k * b)`.
- Cabang geser: `O(w)` kunci per query, sehingga `O(N * w)`.

Total: `O(N * (N/l + k*b + w))`.

Dengan `N = 64k, l = 64, k = 16, b = 64, w = 512`: biaya per query adalah `1000 + 1024 + 512 = 2536 kunci`. Atensi penuh adalah `64000 kunci`. Pengurangan komputasi 25x.

Dengan `N = 128k, l = 64, k = 16, b = 64, w = 512`: biaya per query adalah `2000 + 1024 + 512 = 3536 kunci`. Atensi penuh adalah `128000 kunci`. Pengurangan 36x. Manfaatnya bertambah seiring panjang urutan — itulah inti persoalannya.

### Bagaimana perbandingannya

| Metode | Dapat terdiferensiasi | Percepatan inferensi nyata | Pengambilan jarak jauh |
|--------|---------------|----------------------|-------------------|
| Jendela geser saja | ya | ya | gagal |
| Bergerak melangkah / blok-jarang | ya | ya | sebagian |
| Pemangkasan KV (H2O, StreamingLLM) | N/A (saat inferensi) | ya | sebagian |
| MoBA (Moonshot) | sebagian | ya | baik |
| NSA | ya (secara native) | ya (9x pada 64k) | menyamai atensi penuh |

MoBA (Moonshot, arXiv:2502.13189) diterbitkan bersamaan dan mengambil pendekatan serupa bahwa tiga lebih baik dari satu, menerapkan prinsip MoE pada blok atensi. NSA dan MoBA adalah dua arsitektur yang perlu Anda ketahui untuk pra-pelatihan konteks panjang 2026.

```figure
sliding-window-attention
```

## Membangunnya

`code/main.py` mengimplementasikan ketiga cabang pada urutan sintetik pendek dan menunjukkan:

- MLP kompresi (baseline mean-pool sederhana digunakan demi kejelasan pedagogis; NSA yang asli menggunakan MLP yang dipelajari).
- Seleksi blok top-k yang digerakkan oleh skor cabang terkompresi.
- Atensi jendela geser pada `w` token terakhir.
- Kombinasi bergerbang.
- Cetakan jumlah komputasi yang dibandingkan dengan atensi penuh.

### Langkah 1: kompres token menjadi blok

```python
def compress(K, l):
    n = len(K)
    n_blocks = (n + l - 1) // l
    out = []
    for b in range(n_blocks):
        start, end = b * l, min((b + 1) * l, n)
        block = K[start:end]
        summary = [sum(row[d] for row in block) / len(block) for d in range(len(K[0]))]
        out.append(summary)
    return out
```

### Langkah 2: atensi cabang terkompresi

Jalankan atensi softmax query terhadap kunci terkompresi. Skor cabang terkompresi berfungsi ganda sebagai sinyal untuk seleksi top-k.

### Langkah 3: seleksi blok top-k

Pilih indeks `k` blok terkompresi dengan skor tertinggi. Muat token asli yang tidak terkompresi dari blok-blok itu dan jalankan atensi atas mereka.

### Langkah 4: atensi jendela geser

Ambil `w` token terakhir dan jalankan atensi standar terhadap mereka.

### Langkah 5: gerbang + kombinasi

MLP kecil pada query menghasilkan tiga bobot gerbang. Keluaran akhir adalah jumlah berbobot dari ketiga keluaran cabang.

### Langkah 6: penghitungan komputasi

Cetak jumlah kunci yang dihadiri per query untuk setiap cabang dan totalnya. Bandingkan dengan `N` (atensi penuh). Pada sintetik 1024 token dengan `l = 32, k = 4, w = 128`, NSA melihat `32 + 128 + 128 = 288` kunci per query versus 1024 pada atensi penuh — 3,5x lebih sedikit.

## Menggunakannya

NSA sedang dikirimkan dalam pipeline pra-pelatihan konteks panjang internal DeepSeek. Status integrasi pada tumpukan inferensi publik per April 2026:

- **Internal DeepSeek**: native; bobot yang dipublikasikan menggunakan NSA atau penerusnya, DSA (Deepseek Sparse Attention).
- **vLLM**: dukungan NSA eksperimental sedang dikembangkan untuk bobot DeepSeek-V3.x.
- **SGLang**: benchmark NSA telah dipublikasikan; jalur produksi mengikuti vLLM.
- **llama.cpp / CPU**: tidak didukung; overhead dekomposisi kernel tidak sepadan pada throughput CPU.

Kapan menggunakan NSA:

- Proses pra-pelatihan atau pelatihan lanjutan yang menargetkan konteks 64k ke atas dengan anggaran komputasi yang memadai.
- Inferensi checkpoint konteks panjang milik DeepSeek sendiri. Bobotnya bersifat NSA-native.

Kapan tidak menggunakannya:

- Melayani model pra-terlatih atensi padat yang sudah ada. NSA tidak dapat dipasang ulang tanpa pelatihan lanjutan.
- Konteks di bawah 16k. Overhead tiga cabang mendominasi penghematannya.
- Chat interaktif batch 1. Dekode yang sensitif latensi mendapat manfaat, tetapi hanya pada konteks panjang.

## Merilisnya

Pelajaran ini menghasilkan `outputs/skill-nsa-integrator.md`. Dengan spesifikasi proses pra-pelatihan konteks panjang, skill ini menghasilkan rencana integrasi NSA: ukuran blok kompresi, top-k, jendela geser, lebar MLP gerbang, pilihan kernel, serta evaluasi konteks panjang spesifik yang membenarkan perubahan arsitektur tersebut.

## Latihan

1. Jalankan `code/main.py` pada sintetik 1024 token. Sapu `(l, k, w)` dalam tiga preset dan cetak jumlah komputasi. Identifikasi preset yang mencapai jumlah kunci terendah per query sambil mempertahankan 95% recall terhadap atensi penuh pada uji needle-in-haystack.

2. Ganti kompresor mean-pool dengan MLP kecil yang dipelajari (2 lapisan, hidden 32). Latih pada tugas sintetik ketika sinyalnya adalah rata-rata sebuah blok. Ukur selisih perplexity terhadap baseline mean-pool pada data yang tidak terlihat.

3. Implementasikan MLP gerbang. MLP ini menerima query sebagai masukan dan menghasilkan tiga skalar. Tunjukkan bahwa gerbang berperilaku wajar: pembobotan hampir seragam pada query acak, bobot besar pada cabang terpilih ketika query menyentuh blok jauh di belakang.

4. Hitung anggaran memori KV cache untuk model 70B berkemampuan NSA pada konteks 128k. Head KV berjumlah 8, head dim 128, BF16. Bandingkan dengan atensi penuh dan dengan MLA (Fase 10 · 14 menampilkan angka MLA). Identifikasi panjang urutan ketika KV cache cabang butir halus NSA sama dengan atensi penuh.

5. Bacalah Bagian 4 makalah NSA (arXiv:2502.11089) dan jelaskan dalam tiga kalimat mengapa skor atensi cabang terkompresi digunakan kembali untuk seleksi top-k alih-alih menghitung skor perutean terpisah. Hubungkan jawaban Anda dengan aliran gradien.

## Istilah Kunci

| Istilah | Yang orang katakan | Yang sebenarnya dimaksud |
|------|----------------|------------------------|
| Cabang terkompresi | "Tampilan kasar" | Atensi atas kunci hasil rata-rata blok yang menyediakan konteks global dalam $O(N/l)$ kunci per query |
| Cabang terpilih | "Blok top-k" | Atensi butir halus atas `k` blok dengan skor cabang terkompresi tertinggi |
| Jendela geser | "Konteks lokal" | Atensi atas `W` token terakhir untuk pola jarak pendek |
| Kemampuan latih native | "Pra-latih dengan kejarangan aktif" | Pola kejarangan dipelajari selama pra-pelatihan, bukan dipasang tambahan saat inferensi |
| Ukuran blok kompresi l | "Ukuran kelompok untuk tampilan kasar" | Berapa token yang digabungkan menjadi satu ringkasan; 32-64 lazim |
| Top-k | "Blok yang dipertahankan" | Jumlah blok terkompresi yang token tidak terkompresinya dibaca; 16 lazim |
| Jendela geser W | "Radius atensi lokal" | Biasanya 512; lebih pendek merusak koherensi lokal, lebih panjang membuang komputasi |
| Gerbang cabang | "Cara mencampur ketiganya" | Keluaran MLP per posisi yang membobot kontribusi ketiga cabang |
| Keselarasan perangkat keras | "Kejarangan ramah kernel" | Pola jarang dipilih sehingga kernel GPU aktual mencapai percepatan teoretis |
| DSA | "Penerus NSA" | Deepseek Sparse Attention, arsitektur yang mengikuti NSA dalam garis keturunan DeepSeek |

## Bacaan Lanjutan

- [Yuan et al. — Native Sparse Attention: Hardware-Aligned and Natively Trainable Sparse Attention (arXiv:2502.11089, ACL 2025 Best Paper)](https://arxiv.org/abs/2502.11089) — makalahnya
- [DeepSeek-V3 Technical Report (arXiv:2412.19437)](https://arxiv.org/abs/2412.19437) — keluarga arsitektur yang menjadi sasaran NSA
- [Moonshot AI — MoBA: Mixture of Block Attention for Long-Context LLMs (arXiv:2502.13189)](https://arxiv.org/abs/2502.13189) — karya bersamaan, atensi bergaya MoE atas blok
- [Beltagy et al. — Longformer: The Long-Document Transformer (arXiv:2004.05150)](https://arxiv.org/abs/2004.05150) — asal-usul jendela geser
- [Xiao et al. — StreamingLLM: Efficient Streaming Language Models with Attention Sinks (arXiv:2309.17453)](https://arxiv.org/abs/2309.17453) — baseline kejarangan saat inferensi yang ditingkatkan NSA
- [Dao et al. — FlashAttention-2 (arXiv:2307.08691)](https://arxiv.org/abs/2307.08691) — baseline atensi penuh yang dikalahkan kernel NSA pada 64k
