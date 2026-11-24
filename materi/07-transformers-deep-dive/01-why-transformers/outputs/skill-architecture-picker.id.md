---
name: sequence-architecture-picker
description: Pilih arsitektur sekuens (RNN, transformer, SSM, hybrid) berdasarkan panjang, throughput, dan anggaran pelatihan.
version: 1.0.0
phase: 7
lesson: 1
tags: [transformers, architecture, rnn, ssm]
---

Diberikan sebuah masalah sekuens (panjang maksimum, bentuk batch, anggaran token pelatihan, target latensi inferensi, kelas perangkat), keluarkan:

1. Arsitektur utama. Salah satu dari: transformer, model ruang keadaan (Mamba/RWKV), hybrid SSM+atensi, RNN. Sertakan alasan satu kalimat yang dikaitkan dengan batasan dominan.
2. Strategi panjang konteks. Jika transformer: batas atensi penuh, ukuran jendela geser, faktor penskalaan RoPE. Jika SSM: ukuran potongan pemindaian. Jika RNN: lebar hidden.
3. Profil FLOP pelatihan. Perkiraan FLOPs per token dari arsitektur + konteks; nyatakan apakah spesifikasi tersebut masuk anggaran komputasi.
4. Profil memori inferensi. KV cache untuk transformer, ukuran state untuk SSM, memori per token untuk RNN. Tandai jika perangkat target tidak bisa menampung satu batch berukuran 1.
5. Catatan risiko. Satu mode kegagalan spesifik yang diketahui dimiliki pilihan ini pada skala spesifikasi tersebut (mis. transformer OOM pada konteks 64K di GPU 24GB tanpa Flash Attention).

Tolak merekomendasikan RNN murni untuk pelatihan apa pun di atas 1B token tanpa menyatakan secara eksplisit penalti aliran gradien dan paralelisme. Tolak merekomendasikan transformer atensi-penuh untuk konteks >64K tanpa menyatakan biaya memori $O(N^2)$. Tolak merekomendasikan arsitektur yang baru dirilis (dipublikasikan <12 bulan lalu) untuk produksi tanpa cadangan yang disebut nama.
