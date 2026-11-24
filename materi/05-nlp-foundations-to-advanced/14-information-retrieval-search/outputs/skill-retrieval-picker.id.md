---
name: retrieval-picker
description: Pilih tumpukan retrieval untuk korpus dan pola kueri tertentu.
version: 1.0.0
phase: 5
lesson: 14
tags: [nlp, retrieval, rag, search]
---

Diberikan persyaratan (ukuran korpus, pola kueri, anggaran latensi, standar kualitas, batasan infrastruktur), keluarkan:

1. Tumpukan. Hanya BM25, hanya dense, hibrida (BM25 + dense + RRF), hibrida + rerank cross-encoder, atau tiga arah (BM25 + dense + sparse terpelajar).
2. Encoder dense. Sebutkan model spesifiknya (`all-MiniLM-L6-v2`, `bge-large-en-v1.5`, `e5-large-v2`, `paraphrase-multilingual-MiniLM-L12-v2`). Sesuaikan dengan bahasa, domain, dan panjang konteks.
3. Reranker. Sebutkan model cross-encoder bila dipakai (`cross-encoder/ms-marco-MiniLM-L-6-v2`, `BAAI/bge-reranker-large`). Beri tanda pada latensi tambahan sekitar 30-100 ms pada 30 teratas.
4. Rencana evaluasi. Recall@10 adalah metrik utama retriever. MRR untuk jawaban ganda. Buat baseline dulu, lalu ukur peningkatan inkremental terhadapnya.

Tolak merekomendasikan dense-only untuk korpus yang memuat named entity, kode galat, atau SKU produk kecuali pengguna memiliki bukti bahwa dense menangani pencocokan persis. Tolak melewati reranking untuk retrieval berisiko tinggi (hukum, medis) yang 5 teratasnya menentukan jawaban pengguna.
