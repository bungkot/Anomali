---
name: summary-picker
description: Pilih ekstraktif atau abstraktif, sebutkan pustakanya, tambahkan pemeriksaan fakta.
version: 1.0.0
phase: 5
lesson: 12
tags: [nlp, summarization]
---

Diberikan sebuah tugas (jenis dokumen, persyaratan kepatuhan, panjang, anggaran komputasi), keluarkan:

1. Pendekatan. Ekstraktif atau abstraktif. Jelaskan dalam satu kalimat mengapa.
2. Model/pustaka awal. Sebutkan namanya. `sumy.TextRankSummarizer`, `facebook/bart-large-cnn`, `google/pegasus-pubmed`, atau sebuah prompt LLM.
3. Rencana evaluasi. ROUGE-1, ROUGE-2, ROUGE-L (pakai `rouge-score` dengan stemming). Tambahkan pemeriksaan fakta jika abstraktif.
4. Satu mode kegagalan untuk diuji. Pertukaran entitas adalah yang paling umum pada peringkasan berita abstraktif; beri tanda pada sampel yang entitas sumbernya tidak muncul di ringkasan.

Tolak peringkasan abstraktif untuk konten medis, hukum, keuangan, atau yang diatur tanpa gerbang fakta. Beri tanda pada input yang melebihi jendela konteks model sebagai perlu peringkasan map-reduce bertahap, bukan sekadar pemotongan.
