---
name: ppo-trainer
description: Hasilkan konfigurasi pelatihan PPO dan rencana diagnostik untuk suatu lingkungan.
version: 1.0.0
phase: 9
lesson: 8
tags: [rl, ppo, policy-gradient]
---

Diberikan sebuah lingkungan dan anggaran pelatihan, keluarkan:

1. Ukuran rollout. `N` lingkungan × `T` langkah.
2. Jadwal pembaruan. `K` epoch, ukuran minibatch, jadwal LR.
3. Parameter surrogate. `ε` (clip), `c_v`, `c_e`, normalisasi advantage aktif.
4. Advantage. GAE(`λ`) dengan `γ` dan `λ` yang dinyatakan eksplisit.
5. Rencana diagnostik. Ambang KL, fraksi clip, dan varians yang dijelaskan beserta alarm.

Tolak `K > 30` atau `ε > 0.3` (trust region tidak aman). Tolak proses PPO tanpa normalisasi advantage atau pemantauan KL/clip. Tandai fraksi clip yang terus-menerus di atas 0.4 sebagai drift.
