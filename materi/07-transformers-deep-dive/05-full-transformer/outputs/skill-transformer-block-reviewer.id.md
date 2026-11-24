---
name: transformer-block-reviewer
description: Tinjau implementasi blok transformer terhadap bawaan 2026 dan tandai pergeseran (drift).
version: 1.0.0
phase: 7
lesson: 5
tags: [transformers, architecture, review]
---

Dengan kode sumber blok transformer (PyTorch / JAX / numpy / pseudocode) dan peran yang dimaksud (encoder / decoder / encoder-decoder), keluarkan:

1. Pemeriksaan rangkaian. Pre-norm atau post-norm. Koneksi residual di sekitar setiap sublapisan. Tandai post-norm sebagai bukan bawaan untuk 2026 kecuali penulis menyatakan alasannya.
2. Normalisasi. LayerNorm vs RMSNorm. RMSNorm lebih disukai. Tandai jika suku bias ada pada proyeksi Q/K/V/O — sebagian besar model 2026 menghilangkannya.
3. Bentuk atensi. MHA / GQA / MQA / MLA. Untuk blok decoder: pastikan penyamaran kausal diterapkan. Untuk cross-attention: pastikan Q berasal dari decoder, K/V dari encoder.
4. FFN. Fungsi aktivasi (ReLU / GELU / SwiGLU / GeGLU). Rasio ekspansi. SwiGLU dengan ~2,67× adalah bawaan modern; 4× ReLU/GELU adalah yang klasik.
5. Sinyal posisi. Pastikan RoPE / ALiBi / absolut diterapkan di tempat yang diharapkan (biasanya proyeksi Q,K untuk RoPE).

Tolak menyetujui blok yang menumpuk lebih dari 12 lapisan dengan post-norm tanpa jadwal warmup — pelatihan akan divergen. Tolak blok decoder tanpa penyamaran kausal. Tandai blok mana pun yang ekspansi FFN-nya turun di bawah 2× sebagai kemungkinan kekurangan kapasitas. Beri peringatan jika blok meng-hardcode `d_model` tanpa kolom konfigurasi untuk penggantian ukuran.
