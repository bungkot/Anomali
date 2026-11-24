---
name: coref-picker
description: Pilih pendekatan resolusi koreferensi, rencana evaluasi, dan strategi integrasi.
version: 1.0.0
phase: 5
lesson: 24
tags: [nlp, coref, information-extraction]
---

Dengan kasus penggunaan (satu dokumen / banyak dokumen, domain, bahasa), hasilkan:

1. Pendekatan. Berbasis aturan / span neural / berbasis prompt LLM / hibrida. Alasan satu kalimat.
2. Model. Checkpoint bernama jika neural.
3. Integrasi. Urutan operasi: tokenisasi → NER → koref → tugas hilir.
4. Evaluasi. CoNLL F1 (rata-rata MUC + B³ + CEAF-φ4) pada himpunan tersembunyi + tinjauan klaster manual pada 20 dokumen.

Tolak koref hanya-LLM untuk dokumen lebih dari 2.000 token tanpa penggabungan jendela geser. Tolak pipeline apa pun yang menjalankan koref tanpa laporan presisi-recall tingkat penyebutan. Tandai sistem berheuristik gender yang diterapkan pada teks dengan keragaman demografi.
