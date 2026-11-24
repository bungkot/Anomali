# Penelusuran Arsitektur DeepSeek-V3

> Fase 10 · Pelajaran 14 menyebutkan enam tombol arsitektur yang dimiliki setiap model terbuka. DeepSeek-V3 (Desember 2024, 671B parameter total, 37B aktif) memutar keenamnya dan menambah empat lagi: Multi-Head Latent Attention, penyeimbangan beban tanpa loss bantu, Multi-Token Prediction, dan pelatihan DualPipe. Pelajaran ini membaca arsitektur DeepSeek-V3 dari atas ke bawah dan menurunkan setiap jumlah parameter dari konfigurasi yang dipublikasikan. Pada akhirnya, Anda dapat menjelaskan mengapa rasio 671B/37B adalah taruhan yang tepat dan mengapa MLA + MoE bersama-sama mengungguli masing-masing secara tunggal di frontier.

**Jenis:** Belajar
**Bahasa:** Python (stdlib, kalkulator parameter), Rust
**Prasyarat:** Fase 10 · 14 (penelusuran model terbuka), Fase 10 · 17 (NSA), Fase 10 · 18 (MTP), Fase 10 · 19 (DualPipe)
**Waktu:** ~75 menit

## Tujuan Pembelajaran

- Membaca konfigurasi DeepSeek-V3 dari atas ke bawah dan menjelaskan setiap kolom dalam hal enam tombol GPT-2 ditambah empat tambahan spesifik DeepSeek.
- Menurunkan jumlah parameter total (671B), jumlah parameter aktif (37B), dan komponen yang berkontribusi pada masing-masing.
- Menghitung jejak KV cache MLA pada konteks 128k dan membandingkannya dengan biaya model dense dengan parameter aktif sama yang memakai GQA.
- Menyebutkan empat inovasi spesifik DeepSeek (MLA, MTP, routing tanpa loss bantu, DualPipe) dan nama bagian arsitektur/tumpukan pelatihan yang ditargetkan masing-masing.

## Masalah

DeepSeek-V3 adalah model frontier terbuka pertama yang arsitekturnya secara bermakna berbeda dari keluarga Llama. Llama 3 405B adalah "GPT-2 dengan enam tombol diputar." DeepSeek-V3 adalah GPT-2 dengan keenam tombol itu plus empat lagi. Membaca konfigurasi Llama 3 adalah pemanasan untuk membaca konfigurasi DeepSeek, tetapi struktur dalam — bentuk blok atensi, logika routing, objektif saat pelatihan — cukup berbeda sehingga Anda memerlukan penelusuran tersendiri.

Manfaat mempelajarinya: rilis bobot terbuka DeepSeek-V3 menggeser arti "kemampuan frontier" pada model terbuka. Arsitekturnya adalah cetak biru yang ditiru banyak proses pelatihan tahun 2026. Memahaminya adalah syarat wajib untuk peran apa pun yang menyentuh pelatihan atau inferensi LLM frontier.

## Konsep

### Inti yang tetap, lagi

DeepSeek-V3 tetap autoregresif. Ia masih menumpuk blok decoder. Setiap blok masih memiliki atensi plus MLP plus dua RMSNorm. Ia masih memakai SwiGLU di MLP. Ia masih memakai RoPE. Pre-norm. Embedding dengan bobot terikat (weight-tied). Baseline yang sama dengan setiap Llama atau Mistral.

### Putarannya: MLA alih-alih GQA

Dari Fase 10 · 14 Anda tahu bahwa GQA menyusutkan KV cache dengan berbagi K dan V antargrup head Q. Multi-Head Latent Attention (MLA) melangkah lebih jauh: K dan V dikompresi menjadi representasi latent peringkat rendah bersama (`kv_lora_rank`), lalu didekompresi per head secara langsung (on the fly). KV cache hanya menyimpan latent tersebut — biasanya 512 float per token per layer, bukan 8 x 128 = 1024 float.

Pada konteks 128k, DeepSeek-V3 dengan MLA (satu latent bersama `c^{KV}` per token per layer; K dan V keduanya diturunkan dari latent ini melalui proyeksi-naik yang dapat diserap ke dalam matmul berikutnya):

```
kv_cache = num_layers * kv_lora_rank * max_seq_len * bytes_per_element
         = 61 * 512 * 131072 * 2
         = 7.6 GB
```

Baseline GQA hipotetis (bentuk Llama 3 70B, 8 KV head, head dim 128) akan membayar:

```
kv_cache = 2 * 61 * 8 * 128 * 131072 * 2
         = 30.5 GB
```

MLA 4x lebih kecil daripada cache bergaya GQA Llama-3-70B pada konteks 128k.

Trade-off-nya: MLA menambah langkah dekompresi per komputasi atensi (per head). Komputasi ekstra itu kecil dibandingkan bandwidth yang dihemat. Menang bersih untuk inferensi konteks panjang.

### Routing: penyeimbangan beban tanpa loss bantu

Router MoE memutuskan pakar mana (top-k) yang memproses setiap token. Router naif memusatkan terlalu banyak pekerjaan pada beberapa pakar, membuat yang lain menganggur. Perbaikan standar: tambahkan suku loss bantu yang menghukum ketidakseimbangan beban. Ini bekerja, tetapi sedikit menurunkan kinerja tugas utama.

DeepSeek-V3 memperkenalkan skema tanpa loss bantu. Suku bias per pakar ditambahkan ke logit router, disesuaikan selama pelatihan dengan aturan sederhana: jika pakar `e` kelebihan beban, turunkan `bias_e`; jika kekurangan beban, naikkan. Tidak ada suku loss tambahan. Pelatihan tetap bersih. Beban pakar tetap seimbang.

Efek pada loss utama: tidak terukur. Efek pada arsitektur MoE: lebih bersih, tidak ada hiperparameter loss bantu yang perlu disetel.

### MTP: pelatihan lebih padat + draft gratis

Dari Fase 10 · 18 Anda tahu DeepSeek-V3 menambahkan modul MTP D=1 yang memprediksi token dua posisi ke depan. Saat inferensi, modul yang telah dilatih dipakai ulang sebagai draft dekode spekulatif dengan tingkat penerimaan 80%+. Saat pelatihan, setiap keadaan tersembunyi diawasi pada D+1 = 2 target, memberikan sinyal yang lebih padat.

Parameter: 14B di atas 671B utama. Overhead: 2,1%.

### Pelatihan: DualPipe

Dari Fase 10 · 19 Anda tahu DualPipe adalah pipeline bidireksional yang tumpang tindihkan potongan forward dan backward dengan komunikasi all-to-all lintas simpul. Pada skala 2.048 H800 DeepSeek-V3, ia memulihkan sekitar 245 ribu jam-GPU yang akan hilang pada 1F1B akibat gelembung pipeline.

### Konfigurasi, kolom per kolom

Berikut konfigurasi DeepSeek-V3 (disederhanakan):

```
hidden_size: 7168
intermediate_size: 18432   (ukuran tersembunyi MLP dense, dipakai pada beberapa layer awal)
moe_intermediate_size: 2048 (ukuran tersembunyi MLP pakar)
num_hidden_layers: 61
first_k_dense_layers: 3    (3 layer pertama memakai MLP dense)
num_attention_heads: 128
num_key_value_heads: 128   (secara formal sama dengan num_heads pada MLA, tetapi
                           kompresi sesungguhnya ada di kv_lora_rank)
kv_lora_rank: 512          (dimensi latent MLA)
num_experts: 256            (jumlah pakar MoE per blok)
num_experts_per_tok: 8      (routing top-8)
shared_experts: 1           (satu pakar bersama yang selalu aktif per blok)
max_position_embeddings: 163840
rope_theta: 10000.0
vocab_size: 129280
mtp_module: 1               (1 modul MTP pada kedalaman 1)
```

Mari kita bedah:

- `hidden_size=7168`: dimensi embedding.
- `num_hidden_layers=61`: kedalaman blok total.
- `first_k_dense_layers=3`: 3 blok pertama memakai MLP dense berukuran 18432. 58 sisanya memakai MoE.
- `num_attention_heads=128`: 128 head kueri.
- `kv_lora_rank=512`: K dan V dikompresi ke dimensi latent ini dan didekompresi per head.
- `num_experts=256, num_experts_per_tok=8`: setiap blok MoE memiliki 256 pakar, routing top-8.
- `shared_experts=1`: di atas 256 pakar yang di-routing, 1 pakar yang selalu aktif menyumbang pada setiap token. Anggap sebagai "lantai dense" yang memastikan setiap token mendapat sesuatu yang andal.
- `moe_intermediate_size=2048`: ukuran tersembunyi MLP setiap pakar. Lebih kecil daripada MLP dense karena jumlahnya 256.

### Pembukuan parameter

Perhitungan lengkap ada di `code/main.py`. Ringkasannya:

- Embedding: `vocab * hidden = 129280 * 7168 = ~0.93B`.
- 3 blok dense pertama: atensi dengan MLA (~84M per blok pada hitungan sederhana kalkulator) + MLP SwiGLU dense (`3 * 7168 * 18432 = ~396M` per blok) + norm. Sekitar 1,4B total.
- 58 blok MoE: atensi dengan MLA (~84M) + 256 pakar yang di-routing (`3 * 7168 * 2048 = ~44M` masing-masing) + 1 pakar bersama (~44M) + router + norm. Sekitar 11,4B per blok, termasuk semua pakar. Sekitar 661B untuk 58 blok MoE.
- Modul MTP: modul yang dipublikasikan berukuran 14B. Kalkulator memakai pengganti dense ~0,8B.

Total keseluruhan: kalkulator menghasilkan ~664B terhadap 671B yang dipublikasikan untuk model utama. Checkpoint Hugging Face adalah 685B karena menambahkan modul MTP 14B di atas 671B. Selisih ~1% sisanya berasal dari penyederhanaan: kalkulator melewati dimensi RoPE terpisah (decoupled) pada MLA, head keluaran yang tidak terikat, dan beberapa tensor kecil.

Parameter aktif per forward:

- Atensi: 84M per layer * 61 = ~5,1B (semua layer aktif).
- MLP aktif: 3 layer pertama dense (`3 * 396M = ~1,2B`), 58 layer MoE masing-masing aktif dengan 8 pakar routed + 1 bersama (`9 * 44M = ~396M` per layer, ~23B total). Sekitar 24B semuanya.
- Embedding + norm: ~0,9B.
- Total aktif: ~30B dari kalkulator terhadap 37B yang dipublikasikan. Sebagian besar selisih ada pada atensi: MLA penuh, dengan dimensi RoPE dan head kueri/kunci yang lebih lebar, lebih besar daripada hitungan sederhana. Modul MTP tidak dihitung dalam 37B.

### Rasio 671B / 37B

Rasio kejarangan 18x (parameter aktif adalah 5,5% dari total). DeepSeek-V3 adalah model MoE frontier paling jarang yang merilis bobot terbuka. Mixtral 8x7B dengan rasio 13/47 (28%) jauh lebih padat. Llama 4 Maverick dengan rasio 17B/400B (4,25%) sebanding. Taruhan DeepSeek: pada skala frontier, lebih banyak pakar dengan rasio aktivasi lebih rendah menghasilkan kualitas lebih baik per FLOP aktif.

### Posisi DeepSeek-V3

| Model | Total | Aktif | Rasio | Atensi | Ide baru |
|-------|------|-------|-------|-----------|-------------|
| Llama 3 70B | 70B | 70B | 100% | GQA 64/8 | — |
| Llama 4 Maverick | 400B | 17B | 4.25% | GQA | — |
| Mixtral 8x22B | 141B | 39B | 27% | GQA | — |
| DeepSeek V3 | 671B | 37B | 5.5% | MLA 512 | MLA + MTP + aux-free + DualPipe |
| Qwen 2.5 72B | 72B | 72B | 100% | GQA 64/8 | YaRN extension |

### Kelanjutan: R1, V4

DeepSeek-R1 (2025) adalah proses pelatihan penalaran di atas tulang punggung V3. R1 memakai arsitektur yang sama. Yang berubah adalah resep pascapelatihan (RL skala besar pada tugas yang dapat diverifikasi), bukan arsitektur pra-pelatihan.

DeepSeek-V4 (jika dirilis) diperkirakan tetap memakai MLA + MoE + MTP dan menambahkan DSA (DeepSeek Sparse Attention), penerus NSA dari Fase 10 · 17. Garis keturunannya stabil: inovasi tingkat arsitektur bertumpuk; setiap versi memutar tombol tambahan.

```figure
moe-routing
```

## Gunakan

`code/main.py` adalah kalkulator parameter yang dikhususkan untuk bentuk DeepSeek-V3. Jalankan, bandingkan keluarannya dengan angka makalah, dan pakai pada varian hipotetis (256 pakar vs 512, top-8 vs top-16, MLA rank 512 vs 1024).

Yang perlu dilihat:

- Jumlah parameter total vs 671B yang dipublikasikan.
- Jumlah parameter aktif vs 37B yang dipublikasikan.
- KV cache pada konteks 128k — perbandingan MLA vs GQA.
- Rincian per layer untuk melihat ke mana anggaran parameter benar-benar pergi.

## Kirim

Pelajaran ini menghasilkan `outputs/skill-deepseek-v3-reader.md`. Diberikan model keluarga DeepSeek (V3, R1, atau varian masa depan), berkas ini menghasilkan pembacaan arsitektur komponen per komponen yang menyebutkan setiap kolom konfigurasi, menurunkan jumlah parameter per komponen, dan mengidentifikasi mana dari empat inovasi spesifik DeepSeek yang dipakai model tersebut.

## Latihan

1. Jalankan `code/main.py`. Bandingkan estimasi total parameter kalkulator dengan 671B yang dipublikasikan dan identifikasi sumber selisihnya. Bagian 2 makalah memuat rincian lengkapnya.

2. Ubah konfigurasi agar memakai MLA rank 256 alih-alih 512. Hitung ukuran KV cache yang dihasilkan pada konteks 128k. Berapa persen penghematan yang didapat, dan dengan biaya apa terhadap ekspresivitas per head?

3. Bandingkan routing DeepSeek-V3 (256 pakar, top-8) dengan varian hipotetis (512 pakar, top-8). Total parameter bertambah; parameter aktif tetap sama. Apa yang dibeli kapasitas pakar ekstra secara teori, dan apa biayanya saat inferensi?

4. Bacalah Bagian 2.1 laporan teknis DeepSeek-V3 (arXiv:2412.19437) tentang MLA. Jelaskan dalam tiga kalimat mengapa matriks dekompresi K dan V dapat "diserap" ke dalam matmul berikutnya demi efisiensi saat inferensi.

5. DeepSeek-V3 memakai pelatihan FP8 untuk sebagian besar operasi. Hitung penghematan memori FP8 vs BF16 untuk menyimpan bobot 671B. Bagaimana hal ini beririsan dengan anggaran pelatihan 14,8T token?

## Istilah Kunci

| Istilah | Yang orang katakan | Arti sebenarnya |
|---------|--------------------|-----------------|
| MLA | "Multi-Head Latent Attention" | Kompresi K dan V menjadi latent peringkat rendah bersama (kv_lora_rank, biasanya 512), dekompresi per head secara langsung; KV cache hanya menyimpan latent |
| kv_lora_rank | "Dimensi kompresi MLA" | Ukuran latent bersama untuk K dan V; DeepSeek-V3 memakai 512 |
| First k dense layers | "Layer awal tetap dense" | Beberapa layer awal model MoE melewati router MoE dan menjalankan MLP dense demi stabilitas |
| num_experts_per_tok | "Routing top-k" | Berapa banyak pakar yang di-routing yang aktif per token; DeepSeek-V3 memakai 8 |
| Shared experts | "Pakar yang selalu aktif" | Pakar yang memproses setiap token apa pun hasil routing; DeepSeek-V3 memakai 1 |
| Routing tanpa loss bantu | "Keseimbangan beban dengan penyesuaian bias" | Suku bias per pakar disesuaikan selama pelatihan menjaga beban pakar tetap seimbang tanpa menambah suku loss |
| Modul MTP | "Head prediksi ekstra" | Blok transformer yang memprediksi $t+2$ dari $h^{(1)}$ dan $E(t+1)$; pelatihan lebih padat, draft dekode spekulatif gratis |
| DualPipe | "Pipeline bidireksional" | Jadwal pelatihan yang tumpang tindihkan komputasi forward/backward dengan all-to-all lintas simpul |
| Rasio parameter aktif | "Kejarangan" | active_params / total_params; DeepSeek-V3 mencapai 5,5% |
| Pelatihan FP8 | "Pelatihan 8-bit" | Penyimpanan dan banyak operasi komputasi dalam FP8; kira-kira setengah memori vs BF16 dengan biaya kualitas kecil |

## Bacaan Lanjutan

- [DeepSeek-AI — Laporan Teknis DeepSeek-V3 (arXiv:2412.19437)](https://arxiv.org/abs/2412.19437) — dokumen lengkap arsitektur, pelatihan, dan hasil
- [Kartu model DeepSeek-V3 di Hugging Face](https://huggingface.co/deepseek-ai/DeepSeek-V3) — berkas konfigurasi dan catatan deploy
- [Makalah DeepSeek-V2 (arXiv:2405.04434)](https://arxiv.org/abs/2405.04434) — pendahulu yang memperkenalkan MLA
- [Makalah DeepSeek-R1 (arXiv:2501.12948)](https://arxiv.org/abs/2501.12948) — penerus pelatihan penalaran pada arsitektur V3
- [Native Sparse Attention (arXiv:2502.11089)](https://arxiv.org/abs/2502.11089) — arah masa depan atensi keluarga DeepSeek
- [Repositori DualPipe](https://github.com/deepseek-ai/DualPipe) — referensi jadwal pelatihan
