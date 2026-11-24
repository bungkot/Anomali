---
name: re-designer
description: Rancang pipeline ekstraksi relasi dengan provenansi dan kanonisasi.
version: 1.0.0
phase: 5
lesson: 26
tags: [nlp, relation-extraction, knowledge-graph]
---

Dengan korpus (domain, bahasa, volume) dan penggunaan hilir (KG-RAG, analitik, kepatuhan), hasilkan:

1. Ekstraktor. Berbasis pola / terselia / LLM / hibrida AEVS. Alasan yang terkait target presisi versus recall.
2. Ontologi. Daftar properti tertutup (Wikidata / domain) atau Open IE dengan lintasan kanonisasi.
3. Provenansi. Setiap triple membawa span karakter sumber + id dokumen. Tidak bisa ditawar untuk audit.
4. Strategi penggabungan. Id entitas kanonik + id relasi + qualifier temporal; kebijakan deduplikasi.
5. Evaluasi. Presisi / recall pada 200 triple berlabel manual + laju halusinasi pada sampel hasil ekstraksi LLM.

Tolak pipeline RE berbasis LLM apa pun tanpa verifikasi span (provenansi sumber). Tolak keluaran Open IE yang masuk ke graf produksi tanpa kanonisasi. Tandai pipeline tanpa qualifier temporal pada relasi terikat waktu (pemberi kerja, pasangan, jabatan).
