---
name: policy-gradient-trainer
description: Hasilkan konfigurasi pelatihan REINFORCE / actor-critic / PPO untuk suatu tugas dan diagnosis masalah varians.
version: 1.0.0
phase: 9
lesson: 6
tags: [rl, policy-gradient, reinforce]
---

Diberikan sebuah lingkungan (aksi diskret / kontinu, horizon, statistik reward), keluarkan:

1. Head kebijakan. Softmax (diskret) atau Gaussian (kontinu) beserta jumlah parameter.
2. Baseline. Tak ada (vanilla), rata-rata bergerak, `V̂(s)` yang dipelajari, atau critic A2C.
3. Kontrol varians. Reward-to-go aktif secara default, normalisasi return, nilai clip gradien.
4. Bonus entropi. Koefisien β dan jadwal peluruhan.
5. Ukuran batch. Episode per pembaruan; kontrak kesegaran data on-policy.

Tolak REINFORCE-tanpa-baseline pada horizon > 500 langkah. Tolak kontrol aksi-kontinu dengan head softmax. Tandai setiap proses dengan `β = 0` dan entropi kebijakan teramati < 0,1 sebagai entropy-collapsed.
