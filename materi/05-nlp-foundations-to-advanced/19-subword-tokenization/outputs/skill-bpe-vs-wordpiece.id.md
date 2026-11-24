---
name: skill-bpe-vs-wordpiece
description: Pilih algoritma tokenizer, ukuran kosakata, dan pustaka untuk suatu korpus dan target deployment tertentu.
version: 1.0.0
phase: 5
lesson: 19
tags: [nlp, tokenization]
---

Dengan suatu korpus (ukuran, bahasa, domain) dan target deployment (pelatihan dari awal / fine-tuning / inferensi kompatibel API), keluarkan:

1. Algoritma. BPE, Unigram, atau WordPiece. Alasan satu kalimat.
2. Pustaka. SentencePiece, HF Tokenizers, atau tiktoken. Alasan.
3. Ukuran kosakata. Dibulatkan ke kelipatan 1 ribu terdekat. Alasan yang terkait dengan ukuran model dan cakupan bahasa.
4. Pengaturan cakupan. `character_coverage`, `byte_fallback`, daftar token khusus.
5. Rencana validasi. Rata-rata token per kata pada himpunan yang disisihkan, laju OOV, rasio kompresi, kesamaan dekode pulang-pergi.

Tolak melatih tokenizer dengan character-coverage <0,995 pada korpus yang memuat konten aksara langka. Tolak merilis kosakata tanpa pemeriksaan hash `tokenizer.json` yang dibekukan di CI. Tandai tokenizer monolingua mana pun dengan kosakata di bawah 16k sebagai kemungkinan kurang spesifikasi.
