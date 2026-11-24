---
name: mdp-modeler
description: Diketahui deskripsi tugas, hasilkan spesifikasi Proses Keputusan Markov dan tandai risiko formulasi sebelum pelatihan.
version: 1.0.0
phase: 9
lesson: 1
tags: [rl, mdp, modeling]
---

Diketahui sebuah tugas (kontrol / permainan / rekomendasi / fine-tuning LLM), keluarkan:

1. Keadaan. Spesifikasi vektor fitur atau tensor yang eksak. Justifikasi sifat Markov.
2. Aksi. Himpunan diskrit atau rentang kontinu. Dimensinya.
3. Transisi. Deterministik, stokastik dengan model diketahui, atau hanya sampel.
4. Hadiah. Fungsi dan sumbernya. Jarang vs dibentuk. Terminal vs per langkah.
5. Diskonto. Nilainya dan justifikasi horizon.

Tolak mengirim MDP apa pun yang keadaannya non-Markov tanpa penyebutan eksplisit penumpukan frame atau keadaan rekuren. Tolak hadiah apa pun yang tidak didefinisikan dalam hasil target. Tandai setiap `γ ≥ 1.0` pada tugas ber-horizon tak hingga. Tandai rentang hadiah >100x hadiah langkah tipikal sebagai sumber ledakan gradien yang mungkin.
