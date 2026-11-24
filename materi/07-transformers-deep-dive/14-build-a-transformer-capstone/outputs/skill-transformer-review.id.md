---
name: transformer-review
description: Tinjau implementasi transformer-dari-awal terhadap 13 pelajaran Fase 7.
version: 1.0.0
phase: 7
lesson: 14
tags: [transformers, review, capstone]
---

Diberikan basis kode transformer-dari-awal (PyTorch / JAX), tinjau terhadap default 2026 dan tandai bagian yang hilang atau salah:

1. Atensi. Mask kausal ada. Skalakan dengan `sqrt(d_head)`. Pembagian multi-head bekerja. Flash Attention digunakan jika tersedia. GQA disebut jika d_model ≥ 1024.
2. Penyandian posisi. RoPE (pilihan utama 2026) atau absolut yang dipelajari (boleh untuk model kecil). Tandai sinusoidal sebagai historis.
3. Rangkaian blok. Pre-norm (bukan post-norm). RMSNorm (bukan LayerNorm). SwiGLU FFN (bukan ReLU/GELU). Residual mengitari setiap sublapisan. Bias dihapus pada lapisan linear (default modern).
4. Pelatihan. AdamW (atau Muon untuk 2026+), jadwal LR cosine dengan warmup linier, pemotongan gradien di 1,0, autocast bf16. Pengikatan bobot antara embedding token dan lm_head.
5. Rugi. Cross-entropy geser-satu di setiap posisi. Topengi padding jika ada. Catat rugi train dan val pada interval tetap.

Tolak menyetujui basis kode yang memiliki salah satu dari: post-norm tanpa alasan eksplisit, LayerNorm dalam kode produksi 2026 tanpa justifikasi, mask kausal hilang pada self-attention decoder, embedding tidak terikat pada LM kecil. Tandai: tidak ada pemisahan validasi, tidak ada pemotongan gradien, LR > 1e-3 tanpa warmup, atau block_size yang melebihi jangkauan embedding posisi tanpa fallback. Sarankan menjalankan `python code/main.py` dari awal hingga akhir dan memeriksa rugi val akhir berada di bawah 2,5 pada tinyshakespeare dengan konfigurasi nano.
