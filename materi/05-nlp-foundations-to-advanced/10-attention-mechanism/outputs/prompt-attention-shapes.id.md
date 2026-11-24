---
name: attention-shapes
description: Debug bug bentuk pada implementasi atensi.
phase: 5
lesson: 10
---

Diberikan implementasi atensi yang rusak, Anda mengidentifikasi ketidakcocokan bentuknya. Hasilkan:

1. Matriks mana yang salah bentuk. Sebutkan tensor-nya.
2. Apa bentuk seharusnya, diturunkan dari `(d_s, d_h, d_attn, T_enc, T_dec, batch_size)`.
3. Perbaikan satu baris. Transpos, bentuk ulang, atau proyeksikan.
4. Tes untuk menangkap regresi. Biasanya tegaskan `output.shape == (batch, T_dec, d_h)` dan `weights.shape == (batch, T_dec, T_enc)` serta `weights.sum(dim=-1)` mendekati 1.

Tolak merekomendasikan perbaikan yang diam-diam melakukan broadcast. Bug yang disembunyikan broadcast muncul kemudian sebagai penurunan akurasi senyap.

Untuk kebingungan Bahdanau, tegaskan bahwa masukan decoder adalah `s_{t-1}` (keadaan sebelum langkah). Untuk Luong, `s_t` (keadaan sesudah langkah). Kesalahan pemula yang paling umum pada dot-product attention adalah ketidakcocokan dimensi query/key — tandai secara eksplisit.
