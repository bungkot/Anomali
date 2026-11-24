---
name: multilingual-picker
description: Pilih bahasa sumber, model target, dan rencana evaluasi untuk suatu tugas NLP multibahasa.
version: 1.0.0
phase: 5
lesson: 18
tags: [nlp, multilingual, cross-lingual]
---

Dengan persyaratan yang diberikan (bahasa target, jenis tugas, data berlabel yang tersedia per bahasa), keluarkan:

1. Bahasa sumber untuk fine-tuning. Default bahasa Inggris; periksa LANGRANK atau qWALS jika bahasa target memiliki kerabat sumber daya tinggi yang secara tipologis dekat.
2. Model dasar. XLM-R (klasifikasi), mT5 (generasi), NLLB (terjemahan), Aya-23 (LLM generatif).
3. Anggaran few-shot. Mulai dengan 100-500 contoh bahasa target jika tersedia. Zero-shot hanya jika pelabelan tidak memungkinkan.
4. Rencana evaluasi. Akurasi per bahasa (bukan agregat), konsistensi lintas bahasa, F1 tingkat entitas pada aksara non-Latin.

Tolak merilis model multibahasa tanpa evaluasi per bahasa — metrik agregat menyembunyikan kegagalan ekor panjang. Tandai aksara dengan cakupan tokenisasi rendah (Amhara, Tigrinya, banyak bahasa Afrika) sebagai memerlukan model dengan byte-fallback (SentencePiece dengan byte_fallback=True, atau tokenizer tingkat byte seperti GPT-2).
