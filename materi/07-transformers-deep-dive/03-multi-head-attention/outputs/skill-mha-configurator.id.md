---
name: mha-configurator
description: Rekomendasikan jumlah head, jumlah KV-head, dan strategi proyeksi (MHA / MQA / GQA / MLA) untuk transformer baru.
version: 1.0.0
phase: 7
lesson: 3
tags: [transformers, attention, mha, gqa]
---

Dengan spesifikasi transformer (anggaran parameter, ukuran tersembunyi `d_model`, panjang konteks target, memori perangkat inferensi, prioritas pelatihan vs inferensi), keluarkan:

1. Varian proyeksi. Salah satu dari: MHA, GQA, MQA, MLA. Alasan satu kalimat yang terkait dengan batasan KV-cache.
2. Geometri head. `n_heads`, `n_kv_heads`, `d_head`. Nilai harus memenuhi `d_model = n_heads * d_head` dan `n_heads % n_kv_heads == 0`.
3. Estimasi KV cache. Byte per token per lapisan (fp16) untuk varian yang dipilih pada panjang konteks target. Tandai jika satu batch melebihi memori perangkat target.
4. Inisialisasi. Skala Xavier / Kaiming untuk matriks Q, K, V, O. Catat apakah suku bias disertakan (sebagian besar model 2026 menghilangkannya).
5. Pengait pengujian. Satu tugas sintetis (mis. pola head induksi `A B A ? → B`) yang harus dapat diselesaikan hingga ≥95% oleh versi dua lapisan dari konfigurasi ini setelah dilatih.

Tolak merekomendasikan `d_head < 32` — dinamika atensi rusak. Tolak merekomendasikan MHA dengan `n_heads > 16` untuk panjang konteks di atas 32K tanpa secara eksplisit menghitung biaya KV cache dan menyarankan GQA atau MLA sebagai gantinya. Tolak menyarankan MLA untuk model di bawah 1B parameter kecuali pengguna secara eksplisit melakukan benchmark terhadapnya.
