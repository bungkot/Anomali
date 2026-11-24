---
name: positional-encoding-picker
description: Pilih penyandian posisi (RoPE, ALiBi, sinusoidal) + strategi penskalaan berdasarkan panjang konteks dan anggaran pelatihan.
version: 1.0.0
phase: 7
lesson: 4
tags: [transformers, positional-encoding, rope, alibi]
---

Dengan spesifikasi transformer (panjang konteks target saat inferensi, panjang konteks saat pelatihan, kebutuhan ekstrapolasi, anggaran fine-tuning dalam token), keluarkan:

1. Penyandian dasar. Salah satu dari: RoPE, ALiBi, sinusoidal, learned-absolute. Alasan satu kalimat.
2. Hiperparameter. Jika RoPE: nilai `base`, persyaratan `d_head` agar pembagian genap. Jika ALiBi: rumus slope. Jika sinusoidal: `max_len`.
3. Strategi perluasan. Jika target > panjang latih: faktor penskalaan NTK-aware, konfigurasi YaRN, spesifikasi LongRoPE, atau rasio interpolasi posisi. Nyatakan anggaran token untuk fine-tuning.
4. Rencana pengujian. Target tingkat lulus NIAH (needle-in-a-haystack) pada konteks maksimum, perplexity dalam rentang X dari baseline panjang latih.
5. Rencana cadangan. Apa yang dilakukan jika evaluasi konteks panjang gagal: latih ulang dengan `base` lebih besar, beralih ke ALiBi, atau batasi panjang konteks yang diterapkan.

Tolak merekomendasikan sinusoidal atau learned-absolute untuk model baru pada 2026 — keduanya tidak dapat melakukan ekstrapolasi dan setiap stack modern mengasumsikan RoPE atau ALiBi. Tolak menskala RoPE lebih dari 8× panjang latih tanpa tahap fine-tuning. Tolak merilis konfigurasi konteks panjang tanpa menjalankan NIAH pada panjang penerapan penuh.
