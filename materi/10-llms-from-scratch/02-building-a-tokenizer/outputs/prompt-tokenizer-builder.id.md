---
name: prompt-tokenizer-builder
description: Bangun dan debug tokenizer berkualitas produksi untuk proyek LLM
version: 1.0.0
phase: 10
lesson: 2
tags: [tokenizer, bpe, byte-level, special-tokens, chat-template, multilingual]
---

# Pembangun Tokenizer Produksi

Saat membangun atau men-debug tokenizer untuk proyek LLM, ikuti kerangka ini.

## Daftar Periksa Pipeline

Setiap tokenizer produksi memerlukan lima tahap ini. Jika ada yang hilang, Anda akan menemui kasus tepi di produksi.

1. **Normalisasi** — Terapkan normalisasi Unicode NFKC. Ini meruntuhkan ligatur ("fi" → "fi"), menormalkan karakter lebar penuh, dan menyeragamkan spasi. Lewati ini dan kata yang sama mendapat ID token berbeda tergantung cara diketik.

2. **Pratokenisasi** — Bagi teks menjadi potongan sebelum BPE. Gunakan pola regex GPT-2 untuk model berpusat pada bahasa Inggris. Gunakan pendekatan byte mentah SentencePiece untuk model multibahasa. Pilihan ini menentukan apakah BPE boleh menggabungkan lintas batas kata.

3. **Penggabungan BPE** — Terapkan tabel penggabungan hasil pembelajaran pada urutan byte di dalam setiap potongan. Tabel penggabungan *adalah* pengetahuan hasil pembelajaran tokenizer. Semua sisanya hanyalah perpipaan.

4. **Injeksi Token Khusus** — Cocokkan token khusus secara persis sebelum BPE berjalan. [BOS], [EOS], [PAD], dan penanda templat chat mendapat ID tetap. Token-token itu tidak pernah ikut penggabungan.

5. **Pemetaan ID** — Konversikan string token menjadi bilangan bulat. Model hanya melihat bilangan bulat.

## Men-debug Masalah Tokenizer

**Gejala: model menghasilkan omong kosong pada masukan chat**
- Periksa templat chat. Setiap model punya format berbeda. Llama 3 memakai penanda `<|start_header_id|>`. ChatGPT memakai penanda `<|im_start|>`. Templat yang salah membuat masukan berada di luar distribusi pelatihan.

**Gejala: teks non-Inggris memakai terlalu banyak token**
- Periksa fertilitas (token per kata). Nilai di atas 2.0 berarti tokenizer membuang jendela konteks untuk bahasa itu. Solusinya: latih ulang dengan lebih banyak data multibahasa, perbesar kosakata, atau gunakan SentencePiece dengan Unigram.

**Gejala: angka dan aritmetika gagal**
- Periksa bagaimana digit ditokenisasi. "1234" sebagai satu token berarti model tidak bisa melakukan operasi tingkat digit. Pisahkan digit satu per satu selama pratokenisasi.

**Gejala: token kode tidak efisien**
- Periksa bagaimana indentasi ditangani. Tokenizer GPT-2 membuang token pada spasi. Codex dan StarCoder memakai token indentasi khusus (4 spasi = 1 token).

## Keputusan Ukuran Kosakata

- 32K token: satu bahasa, model kecil, komputasi terbatas. Lapisan embedding berukuran 32K * d_model parameter.
- 50K–64K: multibahasa atau banyak kode. Keseimbangan yang baik untuk sebagian besar proyek.
- 100K+ (GPT-4, Llama 3): hanya dengan data pelatihan masif. Sekuens lebih pendek, tetapi 100K * d_model parameter embedding.

Untuk model berdimensi 4096: kosakata 32K = 131 juta parameter embedding. Kosakata 128K = 524 juta parameter embedding. Itu 400 juta parameter hanya di lapisan embedding.

## Persyaratan Kecepatan

- Tokenisasi data pelatihan: gunakan pustaka berbasis Rust (tiktoken, tokenizers HuggingFace). Python murni 10–100 kali lebih lambat.
- Tokenisasi inferensi: latensi kurang penting (satu sekuens), tetapi tetap gunakan implementasi terkompilasi.
- Benchmark: tokenisasi 1GB teks dan ukur waktu tempuh sebenarnya. Jika lebih dari 60 detik, beralih ke backend Rust.

## Validasi Templat Chat

Sebelum menerapkan model chat apa pun, verifikasi templatnya:

1. Enkode percakapan yang sudah diketahui dengan tokenizer
2. Dekode kembali menjadi teks
3. Bandingkan karakter demi karakter dengan format yang diharapkan dari dokumentasi model
4. Perhatikan: baris baru setelah penanda header, spasi sebelum konten, penanda akhir giliran
5. Uji kasus tepi: pesan sistem kosong, pesan pengguna sangat panjang, beberapa giliran asisten

Salah templat chat adalah sumber paling umum dari penurunan kinerja model chat.
