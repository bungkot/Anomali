---
name: skill-rag-pipeline
description: Bangun dan debug pipeline RAG dari prinsip dasar
version: 1.0.0
phase: 11
lesson: 6
tags: [rag, retrieval, embeddings, vector-search, llm-engineering]
---

# Pola Pipeline RAG

Setiap sistem RAG mengikuti pola ini:

```
documents -> chunk -> embed -> store
query -> embed -> search(top_k) -> build_prompt -> generate
```

Pengindeksan terjadi sekali per dokumen. Kueri terjadi pada setiap permintaan pengguna.

## Kapan menggunakan RAG

- LLM memerlukan akses ke dokumen pribadi atau terkini
- Fine-tuning terlalu mahal atau terlalu lambat untuk diperbarui
- Anda perlu mengutip sumber untuk jawaban
- Basis pengetahuan berubah dengan cepat

## Kapan TIDAK menggunakan RAG

- Jawabannya adalah pengetahuan umum yang sudah dimiliki LLM
- Tugasnya bersifat kreatif (menulis, brainstorming), bukan faktual
- Anda perlu model mengadopsi gaya penalaran tertentu (gunakan fine-tuning)

## Daftar pemeriksaan implementasi

1. Pecah dokumen menjadi segmen 256-512 token dengan tumpang tindih 50 token
2. Embed setiap potongan menggunakan model embedding yang konsisten
3. Simpan embedding di basis data vektor bersama teks aslinya
4. Saat kueri, embed pertanyaan pengguna dengan model yang sama
5. Ambil potongan paling mirip teratas (top-k, 5-10) melalui kesamaan cosinus
6. Bangun prompt: instruksi sistem + konteks hasil pengambilan + pertanyaan pengguna
7. Hasilkan jawaban, berpijak pada konteks hasil pengambilan
8. Kembalikan jawaban beserta referensi sumbernya

## Kesalahan umum

- Menggunakan model embedding berbeda untuk pengindeksan dan kueri (vektor tidak kompatibel)
- Potongan terlalu kecil (kehilangan konteks) atau terlalu besar (mengencerkan relevansi)
- Tidak menyertakan tumpang tindih antar potongan (kalimat terpotong di batas)
- Lupa mengindeks ulang saat dokumen berubah
- Mengembalikan potongan hasil pengambilan kepada pengguna tanpa menghasilkan jawaban yang koheren
- Tidak menetapkan temperature=0 untuk kueri RAG faktual (temperature lebih tinggi = lebih banyak halusinasi)

## Debug pengambilan

Jika potongan yang tepat tidak diambil:
1. Cetak embedding kueri dan verifikasi bahwa nilainya bukan nol
2. Periksa kesamaan cosinus secara manual untuk potongan yang diketahui relevan
3. Coba ubah kalimat kueri agar sesuai dengan kosakata dokumen
4. Verifikasi model embedding sama antara saat indeks dan saat kueri
5. Periksa apakah konten relevan hilang saat pemotongan

## Parameter produksi

- Ukuran potongan: 256-512 token
- Tumpang tindih: 50 token (10-20% dari ukuran potongan)
- Top-k: 5-10 untuk sebagian besar kasus penggunaan
- Temperature: 0 untuk jawaban faktual
- Model embedding: text-embedding-3-small (hemat biaya) atau text-embedding-3-large (akurasi lebih tinggi)
