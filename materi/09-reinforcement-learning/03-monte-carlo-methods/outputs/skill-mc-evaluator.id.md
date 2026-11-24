---
name: mc-evaluator
description: Evaluasi kebijakan melalui rollout Monte Carlo dan hasilkan laporan konvergensi dengan perbandingan DP jika tersedia.
version: 1.0.0
phase: 9
lesson: 3
tags: [rl, monte-carlo, evaluation]
---

Diketahui lingkungan (episodik, dengan API reset+step) dan sebuah kebijakan, keluarkan:

1. Metode. MC first-visit versus every-visit. Alasannya.
2. Anggaran episode. Jumlah target, diagnostik varians, kesalahan standar yang diharapkan.
3. Rencana eksplorasi. Jadwal ε (jika perlu) atau exploring starts.
4. Perbandingan standar emas. V* optimal-DP jika tabular; jika tidak, batas dari baseline Q-learning / PPO.
5. Pemeriksaan penghentian. Batas max-step, timeout, penanganan trajektori yang tidak berhenti.

Tolak menjalankan MC pada tugas non-episodik tanpa batas horizon hingga. Tolak melaporkan estimasi V^π dari kurang dari 100 episode per keadaan untuk tugas tabular. Tandai kebijakan apa pun dengan aksi varians-nol sebagai risiko eksplorasi.
