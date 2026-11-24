---
name: checkpointing-planner
description: Pilih kebijakan komputasi ulang aktivasi per lapisan (none / selective / full / offload) dengan konfigurasi pelatihan dan anggaran HBM.
version: 1.0.0
phase: 10
lesson: 34
tags: [gradient-checkpointing, activation-recomputation, selective-checkpoint, fsdp-offload, training-memory]
---

Dengan konfigurasi pelatihan (jumlah lapisan L, ukuran tersembunyi d, panjang sekuens S, mikrobatch B, byte dtype per nilai, kernel atensi, derajat paralelisme tensor TP, derajat paralelisme pipeline PP, derajat paralelisme pakar EP jika MoE) dan anggaran HBM per rank setelah bobot dan keadaan pengoptimal, keluarkan:

1. Kebijakan per lapisan. Untuk setiap keluarga lapisan dalam tumpukan (embedding, atensi, FFN, pakar MoE, norm, head keluaran) pilih none, selective, full, atau offload. Default selective untuk atensi saat S melebihi 4_096; default none pada aliran residual dan norm; default offload pada FFN hanya ketika waktu transfer PCIe terukur untuk aktivasi lapisan tersebut lebih pendek daripada waktu komputasi ulang terukurnya.
2. Ukuran segmen k. Jika checkpointing penuh aktif, pilih k = round(sqrt(L)) untuk biaya lapisan seragam, k lebih kecil jika memori aktivasi mendominasi anggaran. Laporkan persentase FLOP tambahan sebagai (1/k) dari FLOP forward.
3. Interaksi FlashAttention. Konfirmasi apakah kernel atensi sudah menghitung ulang softmax. Jika ya, checkpointing atensi selektif memberi manfaat kecil; turunkan ke none. Sebut kernel berdasarkan namanya (FlashAttention-2/3, xFormers memory-efficient, vanilla).
4. Rencana TP / PP. Untuk TP, sebut aktivasi yang perlu digather atau direscatter saat komputasi ulang dan byte komunikasi tambahan per langkah. Untuk PP, konfirmasi tahap pipeline mana yang di-checkpoint dari awal hingga akhir sehingga mikrobatch balik membebaskan memori aktivasi sebelum mengalir kembali.
5. Kalkulasi anggaran. Prediksi memori aktivasi sebelum dan sesudah kebijakan (dalam MB per rank). Prediksi overhead FLOP sebagai persentase dari fwd+bwd. Tolak rencana apa pun yang tidak muat dalam anggaran HBM dengan ruang kosong 10 persen.

Tolak checkpointing penuh pada setiap lapisan ketika selective pada atensi saja sudah menutup anggaran; profil menunjukkan overhead FLOP berkali-kali lebih tinggi daripada selective untuk penghematan memori yang sama, dan rasio pastinya bergantung pada beban kerja. Tolak offload ketika waktu transfer aktivasi terukur lapisan tersebut pada tautan PCIe target melebihi waktu komputasi ulang terukurnya; komputasi ulang menang. Tolak "checkpoint di mana-mana" untuk pelatihan FP8 ketika kerangka yang dipilih tidak menyimpan riwayat amax; komputasi ulang akan menggeser skala dan merusak gradien secara diam-diam.

Contoh input: "L=64, d=8192, S=8192, B=1, bf16, FlashAttention-3, TP=8, PP=4, anggaran HBM per rank 32 GB setelah bobot, MoE dengan 8 pakar dan EP=8."

Contoh output:
- Kebijakan per lapisan: atensi selective, FFN none, pakar MoE full, embedding none, head keluaran offload.
- Ukuran segmen: full diterapkan hanya pada MoE dengan k=8; overhead FLOP 12 persen pada jalur pakar, 0 di tempat lain.
- Interaksi FlashAttention: FA-3 sudah menghitung ulang softmax; selective pada pembungkus lapisan, bukan di dalam kernel.
- Rencana TP / PP: gather TP pada input atensi saat komputasi ulang, 0.3 GB komunikasi tambahan per langkah; setiap tahap PP di-checkpoint pada forward penuhnya; tahap PP 3 menyimpan aktivasi untuk backward terakhir.
- Kalkulasi anggaran: aktivasi 38 GB tanpa kebijakan, 11 GB dengan kebijakan. Total overhead FLOP 7.5 persen fwd+bwd.
