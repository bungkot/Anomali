---
name: skill-embeddings-picker
description: Pilih pendekatan tokenisasi untuk model bahasa atau pipeline teks baru.
version: 1.0.0
phase: 5
lesson: 04
tags: [nlp, tokenization, embeddings]
---

Diberi deskripsi tugas dan dataset, keluarkan:

1. Strategi tokenisasi (level kata, BPE, WordPiece, SentencePiece, BPE level byte). Alasan dalam satu kalimat.
2. Target ukuran kosakata. LM bahasa Inggris saja: 32k. Multibahasa: 64k-100k. Kode: 50k-100k.
3. Panggilan pustaka dengan perintah pelatihan yang persis. Sebutkan pustakanya (Hugging Face `tokenizers`, `sentencepiece`). Kutip argumennya.
4. Satu jebakan keterulangan. Ketidakcocokan tokenizer-model adalah bug produksi senyap yang paling umum. Sebutkan tokenizer mana yang berpasangan dengan checkpoint pra-terlatih mana, dan peringatkan agar tidak menggantinya.

Tolak merekomendasikan pelatihan tokenizer kustom ketika pengguna melakukan fine-tuning LLM pra-terlatih (fine-tuning wajib memakai tokenizer pra-terlatih). Tolak merekomendasikan tokenisasi level kata untuk jalur inferensi produksi apa pun. Tandai korpus non-Inggris atau multi-aksara sebagai memerlukan SentencePiece dengan fallback byte.
