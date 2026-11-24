# Jamba — Transformer SSM Hibrida

> State space model (SSM) dan transformer menginginkan hal yang berbeda. Transformer membeli kualitas lewat atensi dengan biaya kuadratik. SSM membeli inferensi waktu-linear dan memori konstan lewat rekurensi, tetapi kualitasnya tertinggal. Jamba dari AI21 (Maret 2024) dan Jamba 1.5 (Agustus 2024) memadukan keduanya dalam satu model: 1 layer Transformer untuk setiap 7 layer Mamba, MoE pada setiap blok bergantian, dan jendela konteks 256k yang muat pada satu GPU 80GB. Mamba-3 (ICLR 2026) memperketat sisi SSM dengan ruang keadaan bernilai kompleks dan proyeksi MIMO. Pelajaran ini membaca kedua arsitektur dari awal sampai akhir dan menjelaskan mengapa resep hibrida bertahan tiga tahun penskalaan sementara upaya konteks panjang SSM-murni dan Transformer-murni tidak bertahan.

**Jenis:** Belajar
**Bahasa:** Python (stdlib, kalkulator campuran layer), Rust
**Prasyarat:** Fase 10 · 14 (arsitektur model terbuka), Fase 10 · 17 (native sparse attention)
**Waktu:** ~60 menit

## Tujuan Pembelajaran

- Menjelaskan tiga primitif dalam blok Jamba — layer Transformer, layer Mamba, MoE — dan resep penyelaan 1:7:genap.
- Menyebutkan bentuk rekurensi SSM secara garis besar dan mengapa ia memungkinkan inferensi memori konstan.
- Menghitung jejak KV cache model Jamba pada konteks 256k dan membandingkannya dengan kebutuhan model Transformer-murni.
- Menyebutkan tiga inovasi Mamba-3 (diskretisasi eksponensial-trapesium, pembaruan keadaan bernilai kompleks, MIMO) dan masalah yang ditargetkan masing-masing.

## Masalah

Atensi bersifat kuadratik terhadap panjang urutan. State space model bersifat linear. Selisih itu bertambah: pada 256k token, peta atensi Transformer memuat 65B entri per head; keadaan rekuren SSM berukuran tetap berapa pun panjang urutannya.

Model SSM-murni (Mamba, Mamba-2) menyamai perplexity Transformer pada skala kecil, tetapi tertinggal pada tugas pelacakan keadaan (state-tracking) dan gagal pada beberapa kategori pengambilan dalam konteks. Intuisinya: SSM mengompresi riwayat menjadi keadaan tetap, dan ketika riwayat panjang, informasi bocor. Atensi mengingat segalanya dengan tepat, tetapi membayar biaya kuadratik.

Perbaikan yang jelas: pakai keduanya. Tempatkan layer Transformer di tempat pengambilan tepat penting. Pakai layer SSM di tempat lain. Setel rasionya. Jamba adalah model pertama kelas produksi yang merilis resep hibrida ini pada skala (52B total, 12B aktif, konteks 256k, satu GPU 80GB). Jamba 1.5 memperluas keluarga menjadi 398B total / 94B aktif. Mamba-3 (ICLR 2026) adalah baseline SSM-murni terbaik saat ini yang dapat menjadi fondasi hibrida berikutnya.

Pelajaran ini membaca ketiga makalah dan membangun model mental untuk "memilih rasio yang tepat."

## Konsep

### SSM dalam satu halaman

State space model memproses urutan `x_1, ..., x_N` melalui keadaan berukuran tetap `h`:

```math
\begin{aligned}
h_t &= A h_{t-1} + B x_t \\
y_t &= C h_t
\end{aligned}
```

Pada setiap langkah, keadaan berevolusi melalui dinamika linear `A`, menerima masukan `B x_t`, dan mengeluarkan `C h_t`. `A, B, C` dapat dipelajari. Perhatikan sifat krusialnya: menghitung `y_t` hanya memerlukan `h_{t-1}` dan `x_t`, bukan `x` sebelumnya. Memori konstan. Inferensi $O(1)$ per token.

Trik untuk kualitas pemodelan ada pada struktur `A`. S4 (Gu 2021) memakai matriks terstruktur sangat khusus yang dapat dievaluasi secara efisien sebagai konvolusi panjang selama pelatihan. Mamba (Gu, Dao 2023) mengganti `A, B, C` tetap dengan yang bergantung data (bagian "selektif"). Mamba-2 (2024) menyederhanakan strukturnya lebih lanjut. Mamba-3 (2026) menambahkan kompleksitas kembali pada tempat-tempat tertentu.

Sifat kuncinya: untuk LLM decoder, layer SSM adalah pengganti langsung (drop-in) layer atensi, dengan keadaan per layer berukuran tetap alih-alih KV cache yang terus bertambah.

### Blok Jamba

Sebuah blok Jamba menyela layer berdasarkan dua angka:

- `l`: rasio atensi terhadap Mamba. Jamba memakai `l = 8`, artinya 1 layer Transformer untuk setiap 7 layer Mamba (7 Mamba + 1 Atensi = 8 layer per grup).
- `e`: frekuensi MoE. Jamba memakai `e = 2`, artinya setiap layer bergantian menerapkan MoE.

Urutan layer dalam satu blok:

```
M  M  M  M  M  M  M  A    (7 Mamba + 1 Atensi)
|  M  |  M  |  M  |  M    (di mana | menandakan MoE diterapkan)
```

Setiap blok Jamba berisi 8 layer. Pada kedalaman 4 blok (32 layer total), Anda mendapat 28 layer Mamba dan 4 layer Atensi. 16 di antaranya memakai MoE.

### Mengapa rasio 1:7

AI21 menjalankan ablasi: rasio atensi terhadap Mamba mana yang memberikan perplexity-per-parameter terbaik sekaligus pengambilan dalam konteks terbaik pada evaluasi konteks panjang mereka?

- Terlalu banyak atensi (1:1): kualitas naik, tetapi memori dan kecepatan memburuk.
- Terlalu sedikit atensi (1:15): memori bagus, tetapi pengambilan dalam konteks gagal.
- Titik manis: 1:7 atau 1:8.

Intuisinya: layer Transformer menangani pengambilan tepat dan pelacakan keadaan. Layer Mamba menangani sebagian besar pemrosesan yang murah.

### Encoding posisi

Layer Mamba sendiri sudah sadar posisi (lewat rekurensi). Layer atensi pada hibrida berbasis Mamba awalnya tidak memakai RoPE — layer SSM menyediakan informasi posisi. Jamba 1.5 menambahkan RoPE pada layer atensi untuk generalisasi konteks lebih panjang, penyempurnaan pasca-fakta berdasarkan evaluasi konteks panjang empiris.

### Anggaran memori

Untuk bentuk Jamba-1 (32 layer: 28 Mamba + 4 Atensi, hidden 4096, 32 head atensi):

- KV cache (hanya layer atensi): `2 * 4 * 32 * 128 * 256k * 2 = 16 GB` pada 256k BF16. Hanya 4 layer atensi yang menyumbang. Jamba yang dirilis memakai GQA dengan 8 KV head pada layer tersebut, yang menurunkannya menjadi 4 GB, angka dalam makalah Jamba.
- Keadaan SSM: `28 * hidden * state_size` per prefiks token, tetapi ini berukuran tetap per layer, tidak berskala dengan panjang urutan. Keadaan Mamba umumnya 16 per fitur, hidden 4096: `28 * 4096 * 16 * 2 = 3.7 MB` total.

Bandingkan dengan Transformer-murni 32 layer, hidden sama, MHA penuh 32 head: `2 * 32 * 32 * 128 * 256k * 2 = 128 GB` pada 256k BF16. Pengurangan 8x pada KV cache. Bahkan terhadap baseline GQA(8) yang dipakai kebanyakan model 2024 (`2 * 32 * 8 * 128 * 256k * 2 = 32 GB`), hibrida 1:7 dengan 32 KV head pada 16 GB tetap 2x lebih kecil. Jamba yang dirilis, pada 4 GB, 8x lebih kecil.

Itulah maksud AI21 dengan "konteks 256k pada satu GPU 80GB." KV cache Transformer-murni MHA penuh tidak akan muat; bahkan baseline GQA tidak menyisakan ruang untuk bobot dan aktivasi; Jamba menyisakannya.

### Mamba-3: baseline SSM-murni pada 2026

Mamba-3 (ICLR 2026, arXiv:2603.15569) memperkenalkan tiga inovasi pada sisi SSM-murni:

1. **Diskretisasi eksponensial-trapesium.** Menggantikan diskretisasi metode Euler pada Mamba-2 dengan rekurensi yang lebih ekspresif. Operasi mirip konvolusi diterapkan pada pasangan keadaan-masukan di dalam rekurensi inti, alih-alih sebagai konvolusi luar pada `x_t`.

2. **Pembaruan keadaan bernilai kompleks.** Mamba sebelumnya menyederhanakan matriks keadaan dari kompleks (S4) menjadi diagonal riil (Mamba) menjadi identitas berskala (Mamba-2). Mamba-3 menambahkan nilai kompleks kembali — setara dengan rotary embedding bergantung data pada keadaan. Ini memulihkan kemampuan pelacakan keadaan yang hilang akibat penyederhanaan bernilai riil sebelumnya.

3. **Proyeksi multi-masukan multi-keluaran (MIMO).** Alih-alih proyeksi skalar per fitur, pakai proyeksi bernilai matriks. Meningkatkan daya pemodelan dan pemanfaatan perangkat keras saat inferensi tanpa menambah latensi dekode.

Pada 1,5B parameter, Mamba-3 meningkatkan akurasi hilir rata-rata 0,6 poin atas Gated DeltaNet; varian MIMO menambah 1,2 poin lagi menjadi total 1,8 poin. Pada ukuran keadaan sama, Mamba-3 menyamai Mamba-2 dengan setengah keadaan.

Mamba-3 belum dikirim dalam hibrida produksi berskala besar — tetapi ia adalah kandidat yang jelas untuk sisi SSM pada model kelas Jamba berikutnya.

### Kapan memakai hibrida

Hibrida menang ketika:

- Konteks cukup panjang sehingga KV cache Transformer-murni menjadi masalah (64k+).
- Tugas mencampur struktur jarak pendek (cocok untuk SSM) dengan pengambilan jarak jauh (membutuhkan Transformer).
- Anda ingin deploy pada anggaran memori satu GPU, di mana KV cache Transformer saja tidak muat.

Hibrida kalah ketika:

- Konteks pendek (di bawah 16k). Overhead SSM terbuang; Transformer-murni sudah cukup.
- Tugas membutuhkan atensi semua-ke-semua (penalaran mendalam, referensi silang banyak dokumen). Jarangnya layer atensi pada hibrida merugikan.
- Anda menskala ke model frontier triliun parameter. Transformer-murni + MLA + MoE (gaya DeepSeek-V3) saat ini memenangkan perlombaan kemampuan.

### Lanskap kompetitif

| Model | Keluarga | Skala | Klaim unik |
|-------|--------|------|-------------|
| Mamba-2 | SSM murni | 3B | waktu linear, memori konstan |
| Jamba | hibrida | 52B/12B | 256k pada 80GB |
| Jamba 1.5 Large | hibrida | 398B/94B | konteks panjang kelas enterprise |
| Mamba-3 | SSM murni | 1,5B (makalah) | pelacakan keadaan dipulihkan |
| DeepSeek-V3 | Transformer murni + MoE | 671B/37B | kemampuan frontier |

Lanskap 2026: MoE Transformer-murni mendominasi frontier, tetapi hibrida menguasai ceruk konteks 256k-plus. Keunggulan pelacakan keadaan Mamba-3 mungkin mendorong rasio hibrida lebih rendah (lebih banyak SSM, lebih sedikit atensi) pada generasi berikutnya.

```figure
swiglu-ffn
```

## Gunakan

`code/main.py` adalah kalkulator memori untuk arsitektur hibrida. Diberikan rasio SSM-Transformer dan konfigurasi ukuran-tersembunyi/jumlah-layer, ia menghitung:

- KV cache pada konteks target.
- Memori keadaan SSM.
- Memori total pada konteks N untuk rentang bentuk model.

Kalkulator mendukung:

- Baseline Transformer-murni (KV cache bertambah dengan N).
- Hibrida gaya Jamba 1:7.
- SSM-murni (tanpa KV cache sama sekali).

Angkanya diambil langsung dari makalah Jamba-1 dan Jamba-1.5 untuk bentuk yang dipublikasikan dan diekstrapolasi untuk varian hipotetis.

Pertimbangan integrasi untuk deploy nyata:

- Kebanyakan server inferensi produksi (vLLM, SGLang) mendukung Jamba dan Mamba. Periksa versi spesifiknya.
- Pada konteks 256k, keunggulan memori Jamba muncul pada throughput permintaan konkuren. Pada VRAM sama, Anda muat lebih banyak urutan Jamba daripada urutan Transformer.
- Mamba-3 sebagai model mandiri belum dipakai di produksi — pratinjau riset pada 1,5B.

## Kirim

Pelajaran ini menghasilkan `outputs/skill-hybrid-picker.md`. Diberikan spesifikasi beban kerja (profil panjang konteks, campuran tugas, anggaran memori), berkas ini merekomendasikan antara Transformer-murni, hibrida gaya Jamba, dan SSM-murni, dengan penalaran eksplisit tentang trade-off memori dan kualitas.

## Latihan

1. Jalankan `code/main.py` untuk menghitung KV cache pada konteks 256k bagi Transformer-murni 32 layer (hidden 4096, 32 head) dan hibrida Jamba-1 berbentuk sama. Verifikasi pengurangan memori ~8x yang diklaim makalah AI21.

2. Ubah kalkulator untuk memodelkan hibrida 1:3 (4 Mamba : 1 Atensi) dan hibrida 1:15 (14 Mamba : 1 Atensi). Plot KV cache vs rasio. Pada rasio berapa KV cache sama dengan memori keadaan SSM?

3. Bacalah Bagian 3 makalah Jamba (arXiv:2403.19887). Jelaskan mengapa AI21 memakai Mamba-1 alih-alih Mamba-2 meskipun Mamba-2 lebih cepat. Petunjuk: bagian ablasi hibrida mendokumentasikan hal ini.

4. Hitung overhead parameter MoE-setiap-layer-bergantian pada Jamba 1.5 Large (398B total, 94B aktif). Bandingkan rasio aktif dengan DeepSeek-V3 (37B/671B) dan jelaskan mengapa arsitektur Jamba mendorong rasio aktif lebih tinggi.

5. Bacalah Bagian 3 makalah Mamba-3 (arXiv:2603.15569). Jelaskan dalam tiga kalimat mengapa pembaruan keadaan bernilai kompleks setara dengan rotary embedding bergantung data. Kaitkan jawaban Anda dengan turunan RoPE di Fase 7 · Pelajaran 04.

## Istilah Kunci

| Istilah | Yang orang katakan | Arti sebenarnya |
|---------|--------------------|-----------------|
| State space model (SSM) | "Rekurensi dengan keadaan tetap" | Layer dengan rekurensi terpelajari `h_t = A h_{t-1} + B x_t`; memori konstan per token |
| Selective SSM | "Triknya Mamba" | Parameter A, B, C bergantung data yang memberi model selektivitas mirip gerbang dalam waktu linear |
| Rasio atensi terhadap Mamba | "Berapa banyak layer atensi" | Pada Jamba, `l = 8` berarti 1 layer atensi per 7 layer Mamba |
| Blok Jamba | "Grup 8 layer" | Satu atensi + tujuh Mamba + MoE pada posisi bergantian |
| Keadaan SSM | "Buffer tersembunyi" | Keadaan per layer berukuran tetap yang menggantikan KV cache pada layer Mamba |
| Konteks 256k | "Angka andalan Jamba" | Panjang urutan yang muat pada satu GPU 80GB untuk Jamba-1; Transformer-murni tidak muat pada ukuran itu |
| Mamba-3 | "SSM murni 2026" | Arsitektur SSM-murni terbaik saat ini dengan keadaan kompleks + MIMO; baseline yang dijadikan fondasi hibrida |
| MIMO | "Multi-input multi-output" | Inovasi Mamba-3 yang memakai proyeksi bernilai matriks alih-alih skalar per fitur |
| Diskretisasi eksponensial-trapesium | "Rekurensinya Mamba-3" | Rekurensi lebih ekspresif yang mencakup diskretisasi metode Euler Mamba-2 |
| Arsitektur hibrida | "Campur atensi dan SSM" | Model apa pun yang menyela layer Transformer dan SSM; Jamba adalah arketipe produksinya |

## Bacaan Lanjutan

- [Lieber et al. — Jamba: A Hybrid Transformer-Mamba Language Model (arXiv:2403.19887)](https://arxiv.org/abs/2403.19887) — makalah Jamba asli, ablasi rasio, klaim konteks 256k
- [AI21 — Jamba 1.5: Hybrid Transformer-Mamba at Scale (arXiv:2408.12570)](https://arxiv.org/abs/2408.12570) — keluarga yang diskalakan, rilis publik 398B/94B dan 12B/52B
- [Gu, Dao — Mamba: Linear-Time Sequence Modeling with Selective State Spaces (arXiv:2312.00752)](https://arxiv.org/abs/2312.00752) — makalah selective SSM yang menjadi fondasi Jamba
- [Dao, Gu — Mamba-2 (arXiv:2405.21060)](https://arxiv.org/abs/2405.21060) — penerus ruang-keadaan-terstruktur yang disederhanakan
- [Lahoti et al. — Mamba-3 (arXiv:2603.15569, ICLR 2026)](https://arxiv.org/abs/2603.15569) — keadaan bernilai kompleks, MIMO, frontier SSM-murni 2026
- [Gu et al. — Efficiently Modeling Long Sequences with Structured State Spaces (arXiv:2111.00396)](https://arxiv.org/abs/2111.00396) — makalah S4, titik awal silsilah SSM untuk LLM
