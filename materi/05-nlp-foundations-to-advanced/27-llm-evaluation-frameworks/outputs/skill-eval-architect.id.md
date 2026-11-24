---
name: eval-architect
description: Rancang rencana evaluasi LLM dengan judge terkalibrasi dan gerbang CI.
version: 1.0.0
phase: 5
lesson: 27
tags: [nlp, evaluation, rag]
---

Diberikan sebuah kasus penggunaan (RAG / agen / tugas generatif), keluarkan:

1. Metrik. Faithfulness / relevance / context-precision / context-recall + metrik G-Eval kustom apa pun beserta kriterianya.
2. Model judge. Nama model + versi, alasan pertimbangan biaya vs akurasi.
3. Kalibrasi. Ukuran himpunan berlabel manual, target rho Spearman vs manusia > 0.7.
4. Versi dataset. Strategi tag, log perubahan, stratifikasi.
5. Gerbang CI. Ambang per metrik, logika jendela regresi, peringatan kuantil bawah.

Tolak bergantung pada judge yang belum diuji terhadap ≥50 contoh berlabel manusia. Tolak evaluasi diri (model yang sama menghasilkan + menilai). Tolak pelaporan hanya agregat tanpa menampilkan 10% terbawah. Tandai pipeline apa pun yang kedatangan pemutakhiran judgenya tanpa evaluasi baseline paralel.
