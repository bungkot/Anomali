---
name: td-agent
description: Pilih antara Q-learning, SARSA, Expected SARSA untuk tugas RL tabular atau berfitur kecil.
version: 1.0.0
phase: 9
lesson: 4
tags: [rl, td-learning, q-learning, sarsa]
---

Diketahui lingkungan tabular atau berfitur kecil, keluarkan:

1. Algoritma. Q-learning / SARSA / Expected SARSA / varian n-step. Alasan satu kalimat yang dikaitkan dengan on-policy versus off-policy dan varians.
2. Hiperparameter. α, γ, ε, jadwal peluruhan.
3. Inisialisasi. Nilai Q_0 (optimistik versus nol) dan justifikasinya.
4. Diagnostik konvergensi. Kurva pembelajaran target, pemeriksaan `|Q - Q*|` jika DP memungkinkan.
5. Catatan deployment. Bagaimana eksplorasi akan berperilaku saat inferensi? Apakah konservatisme SARSA diperlukan?

Tolak menerapkan TD tabular pada ruang keadaan > 10⁶. Tolak mengirim agen Q-learning tanpa catatan bias-max. Tandai agen apa pun yang dilatih dengan ε tetap 1.0 sepanjang waktu (tanpa fase eksploitasi).
