---
name: lm-baseline
description: Bangun baseline model bahasa n-gram yang dapat diulang sebelum melatih model bahasa neural.
phase: 5
lesson: 16
---

Diberikan korpus dan penggunaan sasaran (prediksi kata berikutnya, penilaian ulang, baseline perplexity), keluarkan:

1. Orde n-gram. Trigram untuk bahasa Inggris umum, 4-gram bila korpus besar, 5-gram untuk penilaian ulang pengenalan suara.
2. Smoothing. Kneser-Ney termodifikasi adalah default; Laplace hanya untuk pengajaran.
3. Pustaka. `kenlm` untuk produksi, `nltk.lm` untuk pengajaran, buat sendiri hanya untuk mempelajari matematikanya.
4. Evaluasi. Perplexity pada data terpisah dengan tokenisasi yang konsisten antara himpunan latih dan uji.

Tolak melaporkan perplexity yang dihitung dengan tokenisasi berbeda antara sistem yang dibandingkan — angka perplexity hanya dapat dibandingkan di bawah tokenisasi yang identik. Beri tanda pada tingkat OOV dalam himpunan uji; KN menangani OOV dengan buruk kecuali Anda menyediakan token `<UNK>` khusus selama pelatihan.
