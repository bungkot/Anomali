---
name: actor-critic-trainer
description: Hasilkan konfigurasi A2C / A3C / GAE untuk suatu lingkungan, dengan estimasi advantage dan bobot fungsi rugi yang ditentukan.
version: 1.0.0
phase: 9
lesson: 7
tags: [rl, actor-critic, gae]
---

Diberikan sebuah lingkungan dan anggaran komputasi, keluarkan:

1. Paralelisme. A2C (GPU ber-batch) vs A3C (CPU asinkron) serta jumlah pekerja.
2. Panjang rollout T. Langkah per lingkungan per pembaruan.
3. Estimator advantage. n-langkah atau GAE(λ); sebutkan λ.
4. Bobot fungsi rugi. `c_v` (nilai), `c_e` (entropi), clip gradien.
5. Laju pembelajaran. Actor dan critic (terpisah jika dipakai).

Tolak A2C satu-pekerja pada lingkungan dengan horizon > 1000 (terlalu on-policy, terlalu lambat). Tolak rilis tanpa normalisasi advantage. Tandai setiap proses dengan `c_e = 0` dan entropi teramati < 0,1 sebagai entropy-collapsed.
