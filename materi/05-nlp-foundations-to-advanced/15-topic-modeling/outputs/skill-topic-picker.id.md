---
name: topic-picker
description: Pilih LDA atau BERTopic untuk sebuah korpus. Sebutkan pustaka, parameter, dan evaluasi.
version: 1.0.0
phase: 5
lesson: 15
tags: [nlp, topic-modeling]
---

Diberikan deskripsi korpus (jumlah dokumen, panjang rata-rata, domain, bahasa, anggaran komputasi), keluarkan:

1. Algoritme. LDA / NMF / BERTopic / Top2Vec / FASTopic. Alasan satu kalimat.
2. Konfigurasi. Jumlah topik (mulai dari sekitar sqrt(n_docs)), filter `min_df` / `max_df`, dan model embedding untuk pendekatan neural.
3. Evaluasi. Koherensi topik (c_v) via `gensim.models.CoherenceModel`, keragaman topik, serta bacaan manusia atas 20 sampel.
4. Mode kegagalan untuk diuji. Pada LDA, "topik sampah" yang menyerap kata henti dan istilah frekuen. Pada BERTopic, kelompok pencilan -1 yang menelan dokumen ambigu.

Tolak BERTopic pada dokumen yang lebih panjang dari jendela konteks model embedding tanpa strategi pemotongan. Tolak LDA pada teks sangat pendek (tweet, ulasan di bawah 10 token) karena koherensi hancur. Beri tanda pada pilihan n_topics di bawah 5 atau di atas 200 sebagai kemungkinan salah untuk data nyata.
