---
name: skill-tokenizer
description: Memilih dan membangun tokenizer untuk proyek LLM
version: 1.0.0
phase: 10
lesson: 1
tags: [tokenizer, bpe, wordpiece, sentencepiece, llm, nlp]
---

# Pemilihan dan Implementasi Tokenizer

Saat memulai proyek LLM, terapkan kerangka pengambilan keputusan ini untuk memilih tokenizer.

## Kapan memakai setiap tokenizer

**BPE tingkat byte (tiktoken):** Anda membangun di atas atau melakukan fine-tuning pada model keluarga GPT. Anda memerlukan penanganan yang dijamin untuk setiap urutan byte masukan. Anda tidak menginginkan token tak dikenal.

**WordPiece (Hugging Face):** Anda bekerja dengan model keluarga BERT untuk tugas klasifikasi, NER, atau embedding. Anda memerlukan awalan lanjutan `##` untuk tugas hilir yang bergantung pada sinyal batas kata.

**SentencePiece (BPE atau Unigram):** Anda melatih dari nol. Anda memerlukan tokenisasi yang agnostik terhadap bahasa. Data Anda mencakup bahasa CJK, Thai, atau aksara lain tanpa batas kata berbasis spasi. LLaMA, T5, dan sebagian besar model multibahasa memakai ini.

## Panduan ukuran kosakata

- 32K token: pilihan default yang baik untuk model satu bahasa, menjaga lapisan embedding tetap kecil
- 50K–64K token: lebih baik untuk model multibahasa atau yang banyak memuat kode
- 100K+ token: hanya jika Anda memiliki data pelatihan masif dan menginginkan sekuens pendek

Kosakata yang lebih besar berarti sekuens lebih pendek (inferensi lebih murah), tetapi lebih banyak parameter di matriks embedding. Untuk kosakata 100K dengan embedding berdimensi 4096, lapisan embedding saja sudah 400 juta parameter.

## Aturan pratokenisasi yang penting

1. Pisahkan pada spasi sebelum BPE agar penggabungan lintas kata tidak terjadi
2. Pisahkan setiap digit jika Anda ingin model mempelajari aritmetika
3. Normalisasi Unicode (NFC) sebelum tokenisasi agar perilakunya konsisten
4. Tambahkan token khusus untuk kasus Anda: `<pad>`, `<eos>`, `<bos>`, `<unk>`, dan penanda spesifik tugas apa pun

## Tanda bahaya pada perilaku tokenizer

- Fertilitas di atas 2.0 untuk bahasa target Anda: model membuang-buang jendela konteks
- Kata domain umum terbagi menjadi 3+ token: latih ulang dengan data domain
- Tokenisasi angka yang tidak konsisten: periksa aturan pemisahan digit
- Kosakata besar dengan banyak token yang dipakai sekali: kecilkan ukuran kosakata

## Membangun tokenizer kustom — daftar periksa

1. Kumpulkan data pelatihan yang representatif (minimal 1GB teks di domain target)
2. Pilih algoritme: BPE untuk penggunaan umum, Unigram untuk multibahasa
3. Tetapkan ukuran kosakata berdasarkan panduan di atas
4. Konfigurasi pratokenisasi: pemisahan spasi, penanganan digit, tanda baca
5. Tambahkan token khusus
6. Latih dengan pustaka tokenizers Hugging Face (backend Rust, cepat)
7. Validasi: periksa fertilitas pada teks yang disisihkan di semua bahasa target
8. Uji kasus tepi: string kosong, masukan sangat panjang, data biner, emoji, teks RTL
9. Simpan dan beri versi tokenizer bersama checkpoint model
