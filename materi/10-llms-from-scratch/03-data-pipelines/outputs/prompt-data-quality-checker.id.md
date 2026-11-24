---
name: prompt-data-quality-checker
description: Validasi dan debug kualitas data dalam pipeline pra-pelatihan LLM
version: 1.0.0
phase: 10
lesson: 3
tags: [data-pipeline, deduplication, quality-filter, pre-training, llm, data-cleaning]
---

# Pemeriksa Kualitas Data untuk Pra-pelatihan LLM

Saat membangun atau mengaudit pipeline data untuk pra-pelatihan LLM, gunakan kerangka kerja ini untuk menangkap masalah sebelum masalah itu mencapai model.

## Tanda Bahaya dalam Keluaran Pipeline

**Deduplikasi menghapus kurang dari 20% data web.** Common Crawl biasanya berisi 30-40% duplikat. Jika langkah dedup Anda menghapus kurang dari 20%, parameter MinHash Anda terlalu konservatif atau ambang batas Anda terlalu tinggi. Periksa: ukuran shingle k, jumlah fungsi hash, jumlah band LSH, ambang batas Jaccard.

**Rasio kompresi di bawah 2,0 karakter/token.** Ini berarti tokenizer Anda memecah terlalu agresif. Entah latih ulang dengan lebih banyak penggabungan, perbesar ukuran kosakata, atau periksa apakah pratokenisasi memecah teks secara tidak perlu.

**Rasio kompresi di atas 6,0 karakter/token.** Tokenizer Anda telah mempelajari penggabungan yang sangat spesifik untuk domain tertentu yang mungkin tidak umum. Ini tidak masalah untuk model spesifik-domain tetapi merupakan tanda peringatan untuk model serbaguna.

**Sequence utilization di bawah 90%.** Terlalu banyak padding. Entah dokumen Anda sangat pendek (saring mereka atau naikkan panjang dokumen minimum) atau pengemasan urutan Anda tidak efisien (beralih dari padding naif ke pengemasan multi-dokumen).

**Vocab utilization di bawah 50%.** Lebih dari separuh kosakata Anda tidak digunakan pada korpus ini. Entah kosakatanya terlalu besar untuk domain Anda atau tokenizer dilatih pada data yang sangat berbeda.

## Kalibrasi Filter Kualitas

Jalankan pemeriksaan berikut pada sampel acak 1.000 dokumen di setiap tahap pipeline:

1. **Baca 20 dokumen acak setelah pembersihan.** Apakah mereka masih mengandung HTML, JavaScript, teks navigasi, atau boilerplate? Jika ya, penghapusan HTML Anda belum lengkap.

2. **Baca 20 dokumen acak yang LULUS filter kualitas.** Apakah ada di antaranya yang berupa spam, daftar kata kunci, atau hasil mesin? Jika ya, perketat ambang batas filter.

3. **Baca 20 dokumen acak yang GAGAL pada filter kualitas.** Apakah ada di antaranya yang benar-benar berisi konten bagus? Jika ya, filter Anda terlalu agresif. Longgarkan ambang batas atau tambahkan pengecualian untuk pola tertentu.

4. **Baca 20 pasangan hampir-duplikat acak dari hasil dedup.** Apakah mereka memang benar-benar mirip? Jika tidak, turunkan ambang batas Jaccard atau tambahkan jumlah fungsi hash.

## Rasio Pencampuran Data

Tidak ada rumus universal. Mulailah dari baseline ini dan sesuaikan berdasarkan evaluasi:

| Kategori | Rasio Llama 3 | Titik Awal |
|----------|---------------|------------|
| Teks web | 50% | 50% |
| Kode | 25% | 15-25% |
| Buku/akademik | 13% | 10-15% |
| Matematika | 8% | 5-10% |
| Web multibahasa | 4% | 5-10% |

Tambah rasio kode jika model diharapkan kuat dalam pemrograman. Tambah rasio matematika jika penalaran penting. Kurangi rasio web jika Anda ingin lebih sedikit derau. Selalu evaluasi pada benchmark setelah mengubah rasio.

## Estimasi Skala

Untuk jumlah token target tertentu:

- 1T token dari web: perkirakan ~3-5TB teks mentah, ~1,5-2TB setelah pembersihan dan dedup
- Kecepatan tokenisasi (Rust): ~100M token/detik per core
- Kecepatan tokenisasi (Python): ~1-10M token/detik per core
- Dedup MinHash pada 128 hash, 16 band: ~10K dokumen/detik per core
- Sequence packing: terikat I/O, gunakan berkas memory-mapped untuk korpus di atas 10GB

Untuk 15T token (skala Llama 3), rencanakan ~30-50TB data masukan mentah, 1-2 minggu praproses pada mesin 64-core, serta 100TB+ ruang disk untuk berkas antara.

## Daftar Periksa Sebelum Pelatihan

1. Jumlah token total sesuai dengan anggaran komputasi Anda (gunakan skala Chinchilla atau rasio overtrain Llama 3 sebagai panduan)
2. Dedup menghapus 30-40% data web
3. Filter kualitas menghapus 10-20% data yang tersisa
4. Rasio kompresi 3-5 karakter/token untuk bahasa Inggris
5. Sequence utilization di atas 95%
6. Pemeriksaan acak menunjukkan teks yang bersih dan koheren di setiap tahap pipeline
7. Rasio campuran data telah divalidasi pada pelatihan berskala kecil
8. Penghapusan PII telah diverifikasi pada sampel
9. Semua format biner (urutan hasil kemasan, larik ID token) lolos uji encoding/decoding putar-balik
10. Pipeline dapat diulang: masukan yang sama menghasilkan keluaran yang identik dengan seed acak yang tetap
