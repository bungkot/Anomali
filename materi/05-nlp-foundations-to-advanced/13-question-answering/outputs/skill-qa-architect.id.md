---
name: qa-architect
description: Pilih arsitektur QA, strategi retrieval, dan rencana evaluasi.
version: 1.0.0
phase: 5
lesson: 13
tags: [nlp, qa, rag]
---

Diberikan persyaratan (ukuran korpus, jenis pertanyaan, batasan fakta, anggaran latensi), keluarkan:

1. Arsitektur. Ekstraktif, RAG dengan reader ekstraktif, RAG dengan reader generatif, atau LLM buku tertutup. Alasan satu kalimat.
2. Retriever. Tidak ada, BM25, padat (sebutkan encodernya seperti `all-MiniLM-L6-v2`), atau hibrida.
3. Reader. Model yang disetel pada SQuAD (`deepset/roberta-base-squad2`), LLM yang disebut nama, atau DistilBERT yang di-fine-tune untuk domain tertentu.
4. Evaluasi. EM + F1 untuk benchmark ekstraktif; akurasi jawaban + akurasi sitasi + kalibrasi penolakan untuk produksi. Sebutkan apa yang Anda ukur dan bagaimana caranya.

Tolak jawaban LLM buku tertutup untuk pertanyaan yang sensitif terhadap regulasi atau kepatuhan. Tolak sistem QA apa pun tanpa baseline recall retrieval (Anda tidak bisa mengevaluasi reader tanpa mengetahui bahwa retriever memunculkan bagian teks yang tepat). Beri tanda pada pertanyaan yang memerlukan penalaran multi-hop sebagai perlu retriever multi-hop khusus seperti sistem yang dilatih pada HotpotQA.
