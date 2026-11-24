---
name: dqn-trainer
description: Hasilkan konfigurasi pelatihan DQN (buffer, sinkronisasi target, jadwal ε, pemotongan hadiah) untuk tugas RL beraksi diskrit.
version: 1.0.0
phase: 9
lesson: 5
tags: [rl, dqn, deep-rl]
---

Diketahui lingkungan beraksi diskrit (bentuk pengamatan, jumlah aksi, horizon, skala hadiah), keluarkan:

1. Jaringan. Arsitektur (MLP / CNN / Transformer), dimensi fitur, kedalaman.
2. Buffer replay. Kapasitas, ukuran minibatch, ukuran pemanasan (warmup).
3. Jaringan target. Strategi sinkronisasi (hard setiap C langkah atau lunak τ).
4. Eksplorasi. ε awal / akhir / panjang jadwal.
5. Loss. Huber versus MSE, nilai pemotongan gradien, aturan pemotongan hadiah.
6. Double DQN. Aktif secara default kecuali ada alasan eksplisit untuk menonaktifkan.

Tolak mengirim DQN tanpa jaringan target, tanpa buffer replay, atau dengan ε yang tetap berada di 1. Tolak tugas beraksi kontinu (arahkan ke SAC / TD3). Tandai rentang hadiah > 10× rerata per langkah sebagai yang memerlukan pemotongan atau normalisasi skala.
