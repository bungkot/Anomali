---
name: embedding-picker
description: Pilih model embedding, dimensi, dan mode pengambilan untuk korpus dan target deploy tertentu.
version: 1.0.0
phase: 5
lesson: 22
tags: [nlp, embeddings, retrieval]
---

Dengan korpus (ukuran, bahasa, domain, panjang rata-rata), target deploy (cloud / edge / on-prem), anggaran latensi, dan anggaran penyimpanan, hasilkan:

1. Model. Checkpoint atau API bernama. Alasan satu kalimat.
2. Dimensi. Penuh / dipotong Matryoshka / dikuantisasi int8. Alasan yang terkait anggaran penyimpanan.
3. Mode. Padat / jarang / multi-vektor / hibrida. Alasan.
4. Awalan kueri / templat jika diperlukan oleh kartu model.
5. Rencana evaluasi. Tugas MTEB yang relevan dengan domain + evaluasi domain tersembunyi dengan nDCG@10.

Tolak rekomendasi yang memotong Matryoshka menjadi <64 dimensi tanpa validasi domain. Tolak ColBERTv2 untuk korpus di bawah 10k bagian teks (overhead tidak sepadan). Tandai korpus dokumen panjang (>8k token) yang diarahkan ke model dengan jendela 512 token.
