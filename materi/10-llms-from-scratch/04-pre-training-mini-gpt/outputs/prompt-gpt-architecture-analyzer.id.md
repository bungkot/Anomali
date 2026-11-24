---
name: prompt-gpt-architecture-analyzer
description: Menganalisis pilihan arsitektur pada model transformer bergaya GPT mana pun
version: 1.0.0
phase: 10
lesson: 4
tags: [gpt, transformer, architecture, attention, kv-cache, scaling, pre-training]
---

# Analis Arsitektur GPT

Saat mengevaluasi model bergaya GPT dari laporan teknis, kartu model, atau log pelatihan, gunakan kerangka kerja ini untuk membedah arsitektur dan mengidentifikasi trade-off desain.

## Protokol Analisis

### 1. Rincian Alokasi Parameter

Hitung jumlah parameter yang tepat untuk setiap komponen:

- **Embedding token**: vocab_size x embed_dim
- **Embedding posisi**: max_seq_len x embed_dim
- **Atensi per blok**: 4 x embed_dim x embed_dim (proyeksi Q, K, V, dan output)
- **FFN per blok**: 2 x embed_dim x ff_dim + embed_dim + ff_dim (dua lapisan linear + bias)
- **LayerNorm per blok**: 4 x embed_dim (dua norm, masing-masing dengan skala + bias)
- **LayerNorm akhir**: 2 x embed_dim
- **Head output**: vocab_size x embed_dim (atau 0 jika bobot diikat dengan embedding token)

Tandai jika ada satu komponen yang melebihi 40% dari total parameter. Matriks embedding mendominasi pada model kecil. Atensi dan FFN mendominasi pada model besar.

### 2. Analisis Desain Atensi

Evaluasi konfigurasi atensi:

- **Dimensi head**: embed_dim / num_heads. Nilai standar adalah 64 (GPT-2) atau 128 (Llama 3). Di bawah 32 membatasi daya ungkap per head. Di atas 128 membuang-buang komputasi dengan manfaat yang kecil.
- **Jumlah head per lapisan**: Lebih banyak head berarti pola atensi lebih beragam, tetapi memori KV cache lebih besar.
- **Grouped Query Attention (GQA)**: Apakah model berbagi head K/V ke beberapa head Q? Llama 3 menggunakan GQA dengan 8 head KV untuk 32 head Q. Ini mengurangi KV cache sebesar 4x.
- **Panjang konteks**: Embedding posisi maksimum. RoPE memungkinkan ekstrapolasi melampaui panjang pelatihan. Embedding posisi absolut tidak.

### 3. Anggaran Memori

Untuk inferensi pada panjang konteks maksimum model:

- **Bobot (FP16)**: total_params x 2 byte
- **KV Cache (FP16)**: 2 x num_layers x num_kv_heads x head_dim x max_seq_len x 2 byte
- **Aktivasi**: batch_size x seq_len x embed_dim x 2 byte x num_layers (perkiraan)

Tandai jika KV cache melebihi memori bobot. Ini terjadi pada model konteks panjang (128K+) dan menunjukkan model terikat memori (memory-bound) saat dekode.

### 4. Profil Komputasi

- **FLOPS prefill per token**: sekitar 2 x total_params (satu perkalian matriks per parameter, pada forward pass)
- **FLOPS dekode per token**: sama seperti prefill, tetapi pada satu token
- **Bottleneck prefill**: terikat komputasi (GPU TFLOPS)
- **Bottleneck dekode**: terikat memori (bandwidth memori GPU)
- **Intensitas aritmetika**: FLOPS per byte memori yang diakses. Di bawah 100 = terikat memori.

### 5. Keputusan Penskalaan

Evaluasi terhadap hukum penskalaan yang diketahui:

- **Optimal Chinchilla**: Untuk anggaran komputasi C tertentu, ukuran model optimal N dan jumlah token D memenuhi N ~ D (skala yang kurang lebih sama). Model 7B membutuhkan ~140B token.
- **Llama 3 overtrained**: Meta melatih Llama 3 8B pada 15T token (100x optimal Chinchilla). Melatih model kecil secara berlebih dengan lebih banyak data menghasilkan biaya inferensi per token yang lebih baik.
- **Lebar vs kedalaman**: Model yang lebih dalam (lebih banyak lapisan) umumnya lebih efisien dalam sampel daripada model yang lebih lebar (embed_dim lebih besar) pada jumlah parameter yang sama.

## Tanda Bahaya

- **Rasio FFN bukan 4x**: Standar adalah ff_dim = 4 x embed_dim. Llama menggunakan 8/3 x embed_dim dengan SwiGLU. Penyimpangan harus dibenarkan.
- **Tanpa pengikatan bobot (weight tying)**: Head output sebaiknya berbagi bobot dengan embedding token, kecuali vocab_size sangat besar relatif terhadap embed_dim.
- **Tanpa GQA di atas 13B**: Model di atas 13B tanpa grouped-query attention akan memiliki KV cache yang terlalu besar.
- **Tanpa RoPE untuk konteks panjang**: Embedding posisi absolut tidak dapat diekstrapolasi melampaui panjang pelatihan. Model yang menargetkan konteks 32K+ sebaiknya menggunakan embedding rotasi.
- **Laju pembelajaran terlalu tinggi untuk ukuran model**: Model besar memerlukan laju pembelajaran puncak yang lebih rendah. GPT-2 Small menggunakan 6e-4. Llama 3 405B menggunakan 8e-5.

## Format Keluaran

1. **Tabel parameter**: jumlah parameter per komponen beserta persentasenya
2. **Anggaran memori**: bobot, KV cache, dan memori aktivasi pada panjang konteks maksimum
3. **Profil komputasi**: estimasi throughput prefill dan dekode untuk A100/H100
4. **Penilaian desain**: hal yang sudah tepat pada model dan hal yang tidak standar
5. **Kesimpulan penskalaan**: apakah ukuran model sudah sesuai untuk data pelatihannya
