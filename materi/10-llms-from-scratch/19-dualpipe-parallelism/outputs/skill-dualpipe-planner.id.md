---
name: dualpipe-planner
description: Merencanakan strategi paralelisme pipeline (1F1B, Zero Bubble, DualPipe, DualPipeV) untuk sebuah klaster pelatihan.
version: 1.0.0
phase: 10
lesson: 19
tags: [pipeline-parallelism, dualpipe, dualpipev, zero-bubble, expert-parallelism, distributed-training]
---

Diberikan spesifikasi klaster pelatihan (total GPU, topologi interkoneksi, model akselerator, memori per GPU), bentuk model (total parameter, parameter aktif, MoE atau dense, perkiraan jumlah layer), dan volume data pelatihan target, rekomendasikan strategi paralelisme pipeline dan konfirmasi perkiraan fraksi bubble.

Hasilkan:

1. Kedalaman pipeline P. Pilih berdasarkan anggaran memori GPU (satu tahap pipeline harus muat per rank), MoE vs dense, dan bandwidth interkoneksi. Rentang: 4 untuk klaster kecil, 16-32 untuk pelatihan MoE frontier.
2. Jumlah micro-batch M. Harus habis dibagi 2 untuk DualPipe dan DualPipeV. Rasio M/P yang umum antara 8 dan 16. Justifikasi terhadap target akumulasi gradien dan memori aktivasi pada panjang urutan target.
3. Pilihan jadwal. Pilih dari 1F1B, Zero Bubble, DualPipe, DualPipeV. Tabel keputusan: pelatihan dense di bawah 500 GPU -> Zero Bubble. MoE dengan paralelisme pakar -> DualPipe. Pelatihan dense di atas 500 GPU tanpa all-to-all berat -> DualPipeV. Proses kecil di bawah 100 GPU -> 1F1B sudah cukup.
4. Perkiraan fraksi bubble. Hitung untuk jadwal yang dipilih pada P dan M target. Laporkan sebagai persentase dan sebagai jam-GPU absolut yang dihemat dibandingkan 1F1B pada total anggaran pelatihan.
5. Rencana replikasi parameter (khusus DualPipe). Konfirmasi replikasi parameter 2x muat di VRAM yang tersedia. Laporkan kepadatan parameter efektif per GPU dengan P yang dipilih.

Penolakan keras:
- DualPipe tanpa Expert Parallelism. Replikasi 2x tidak dibenarkan tanpa komunikasi EP-heavy yang perlu disembunyikan.
- P > 64 pada proses pelatihan apa pun. Fraksi bubble tumbuh linier dengan P terlepas dari jadwal.
- Jumlah micro-batch tidak habis dibagi 2 untuk DualPipe/DualPipeV. Jadwal tidak akan menutup.
- Paralelisme pipeline sama sekali ketika model muat dalam memori satu GPU. Pakai hanya paralelisme data.

Aturan penolakan:
- Jika interkoneksi berkecepatan 200Gbps atau lebih lambat per GPU, tolak DualPipe dan rekomendasikan DualPipeV. Jendela tumpang tindih all-to-all terlalu sempit untuk membenarkan replikasi.
- Jika pengguna tidak dapat menyediakan kernel all-to-all kustom yang cocok untuk topologi klaster mereka, rekomendasikan Zero Bubble daripada DualPipe.
- Jika proses pelatihan di bawah 1B token, tolak perencanaan paralelisme pipeline seluruhnya dan rekomendasikan paralelisme data ditambah paralelisme tensor.

Output: rencana satu halaman yang mencantumkan P, M, jadwal, perkiraan fraksi bubble, biaya replikasi parameter (jika DualPipe), dan rekomendasi kernel all-to-all. Akhiri dengan paragraf "pemicu rollback" yang menyebut metrik utilisasi spesifik (persentase utilisasi GPU agregat, diukur atas 1000 langkah pertama) yang akan membenarkan beralih ke jadwal yang lebih sederhana jika angka target tidak tercapai.
