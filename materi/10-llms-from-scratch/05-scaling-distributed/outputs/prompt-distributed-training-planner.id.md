---
name: prompt-distributed-training-planner
description: Rencanakan pelatihan terdistribusi dengan ukuran model dan perangkat keras yang tersedia
version: 1.0.0
phase: 10
lesson: 5
tags: [distributed-training, fsdp, deepspeed, tensor-parallelism, pipeline-parallelism, scaling]
---

# Perencana Pelatihan Terdistribusi

Saat merencanakan pelatihan terdistribusi untuk model bahasa besar, gunakan kerangka kerja ini untuk menentukan strategi paralelisme, anggaran memori, overhead komunikasi, dan throughput yang diharapkan.

## Persyaratan Masukan

Berikan:
- **Ukuran model** (parameter dalam miliaran)
- **Token pelatihan target** (dalam triliunan)
- **GPU yang tersedia** (jenis: A100/H100/H200, jumlah, interkoneksi: NVLink/InfiniBand)
- **Memori GPU** (80GB untuk A100/H100, 141GB untuk H200)
- **Node** (GPU per node, jumlah node)
- **Batasan anggaran** (biaya maksimum dalam dolar, waktu wall-clock maksimum)

## Langkah 1: Anggaran Memori

Hitung memori per-GPU untuk setiap komponen:

| Komponen | Rumus | FP16 | FP32 |
|----------|-------|------|------|
| Bobot | params x bytes_per_param | params x 2 | params x 4 |
| Pengoptimal Adam (m + v) | params x 4 x 2 | selalu 8 byte/param | 8 byte/param |
| Gradien | params x bytes_per_param | params x 2 | params x 4 |
| Aktivasi (estimasi) | seq_len x batch x hidden x layers x 2 | bervariasi | bervariasi |

Jika total melebihi memori GPU, sharding diperlukan. Coba secara berurutan:
1. ZeRO-1 (sharding pengoptimal saja) -- komunikasi termurah
2. ZeRO-2 (+ gradien) -- komunikasi sedang
3. FSDP/ZeRO-3 (+ bobot) -- komunikasi tertinggi tetapi penghematan memori maksimal
4. Tambahkan activation checkpointing jika aktivasi masih terlalu besar
5. Tambahkan paralelisme tensor jika satu lapisan tidak muat pada satu GPU

## Langkah 2: Strategi Paralelisme

### Pohon Keputusan

1. **Apakah satu lapisan muat pada satu GPU?**
   - Tidak: Anda membutuhkan paralelisme tensor. Tetapkan TP = 2, 4, atau 8 (di dalam satu node).
   - Ya: Lewati paralelisme tensor.

2. **Apakah model penuh (dengan sharding) muat pada GPU dalam satu node?**
   - Tidak: Anda membutuhkan paralelisme pipeline. Tetapkan PP = jumlah node / grup.
   - Ya: Lewati paralelisme pipeline.

3. **Berapa GPU yang tersisa untuk paralelisme data?**
   - DP = total_gpus / (TP x PP)

4. **Level sharding apa di dalam grup data-parallel?**
   - Mulai dengan FSDP (ZeRO-3). Turunkan ke ZeRO-2 atau ZeRO-1 jika komunikasi menjadi hambatan.

### Konfigurasi Khas

| Ukuran Model | Total GPU | TP | PP | DP | Sharding |
|--------------|-----------|----|----|----|----------|
| 7B | 8 | 1 | 1 | 8 | FSDP |
| 13B | 16 | 2 | 1 | 8 | FSDP |
| 70B | 64 | 8 | 1 | 8 | FSDP |
| 70B | 128 | 8 | 2 | 8 | FSDP |
| 405B | 16.384 | 8 | 16 | 128 | FSDP |

## Langkah 3: Analisis Komunikasi

Estimasi volume komunikasi per langkah pelatihan:

- **Paralelisme data (all-reduce)**: 2 x gradient_size x (N-1)/N per langkah
- **FSDP (all-gather + reduce-scatter)**: ~3 x weight_size x (N-1)/N per langkah (lebih tinggi daripada DP)
- **Paralelisme tensor (all-reduce per lapisan)**: 2 x activation_size x num_layers per langkah (membutuhkan NVLink)
- **Paralelisme pipeline (point-to-point)**: activation_size per batas stage (minimal)

Jika waktu komunikasi melebihi 20% dari waktu komputasi, strategi tersebut terikat komunikasi. Solusinya:
- Akumulasi gradien (kurangi frekuensi all-reduce)
- Buat komunikasi tumpang tindih dengan komputasi (FSDP melakukan ini secara default)
- Perbesar ukuran micro-batch (rasio komputasi-terhadap-komunikasi lebih baik)
- Beralih ke stage sharding yang lebih ringan komunikasinya

## Langkah 4: Estimasi Throughput dan Biaya

**FLOPS per langkah pelatihan:**
- Forward: ~2 x params x tokens_per_batch
- Backward: ~4 x params x tokens_per_batch (2x forward)
- Total: ~6 x params x tokens_per_batch

**Waktu pelatihan:**
- total_flops = 6 x params x total_tokens
- time_seconds = total_flops / (num_gpus x gpu_tflops x 1e12 x utilization)
- Utilisasi khas: 35-45% (dengan mempertimbangkan komunikasi, bubble pipeline, overhead memori)

**Biaya:**
- total_gpu_hours = num_gpus x time_seconds / 3600
- cost = total_gpu_hours x cost_per_gpu_hour

## Langkah 5: Daftar Periksa Validasi

Sebelum meluncurkan:

1. Memori per-GPU muat dalam batas perangkat keras (dengan headroom 10%)
2. Effective batch size sesuai target (per_gpu_batch x DP x gradient_accumulation_steps)
3. Rasio komunikasi-terhadap-komputasi di bawah 20%
4. Fraksi bubble pipeline di bawah 15% (micro-batch cukup banyak)
5. Laju pembelajaran diskalakan untuk effective batch size
6. Frekuensi checkpoint mempertimbangkan probabilitas kegagalan (simpan setiap 1-2 jam untuk pelatihan besar)
7. Gradient clipping sudah diatur (biasanya 1,0 untuk model besar)
8. Langkah warmup proporsional dengan total langkah (biasanya 0,1-1% dari total)

## Tanda Bahaya

- **TP > 8**: Paralelisme tensor antar node (melalui InfiniBand) hampir selalu lebih lambat daripada paralelisme pipeline
- **Stage pipeline > 32**: Overhead bubble menjadi signifikan bahkan dengan banyak micro-batch
- **Effective batch size > 10M token**: Manfaat yang semakin berkurang; dapat merusak konvergensi
- **Utilisasi di bawah 30%**: Terikat komunikasi -- evaluasi ulang strategi paralelisme
- **Tanpa activation checkpointing di atas 13B**: Anda akan kehabisan memori selama backward pass
- **Tanpa akumulasi gradien dengan batch per-GPU kecil**: Derau gradien meningkat; akumulasikan sampai effective batch 256+ sampel
