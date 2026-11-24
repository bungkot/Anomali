---
name: skill-advanced-rag
description: Bangun RAG kelas produksi dengan pencarian hibrida, peringkat ulang, dan evaluasi
version: 1.0.0
phase: 11
lesson: 7
tags: [rag, hybrid-search, bm25, reranking, hyde, evaluation]
---

# Pola RAG Lanjutan

RAG dasar: embed kueri -> pencarian vektor -> top-k -> generasi.
RAG lanjutan: embed kueri + BM25 -> gabungkan peringkat -> peringkat ulang -> top-k -> generasi.

```
query -> [vector search (top-50)] -+-> RRF fusion -> reranker (top-5) -> prompt -> LLM
                                   |
query -> [BM25 search (top-50)]  --+
```

## Kapan meningkat dari RAG dasar

- Kualitas pengambilan turun di bawah 70% Recall@5
- Pengguna melaporkan jawaban salah atau tidak relevan
- Korpus tumbuh melebihi 100K potongan
- Kueri menggunakan kosakata yang berbeda dari dokumen
- Pertanyaan multi-lompat gagal secara konsisten

## Daftar pemeriksaan implementasi

1. Tambahkan indeks BM25 di samping indeks vektor
2. Jalankan kedua pencarian secara paralel (masing-masing top-50)
3. Gabungkan dengan Reciprocal Rank Fusion (k=60)
4. Peringkat ulang kandidat teratas dengan cross-encoder
5. Ambil 5 teratas untuk prompt akhir
6. Tambahkan evaluasi faithfulness pada himpunan uji

## Panduan pemilihan teknik

- **Pencarian hibrida**: selalu gunakan dalam produksi. Tidak menambah biaya saat kueri.
- **Peringkat ulang**: gunakan ketika Recall@50 bagus tetapi Recall@5 buruk. Menambah latensi 50-200ms.
- **HyDE**: gunakan ketika kueri kabur atau menggunakan kosakata berbeda dari dokumen. Menambah satu panggilan LLM.
- **Potongan induk-anak**: gunakan ketika potongan kecil kekurangan konteks tetapi potongan besar mengencerkan relevansi.
- **Filter metadata**: gunakan ketika korpus memiliki kategori jelas (tanggal, jenis sumber, departemen).
- **Dekomposisi kueri**: gunakan untuk pertanyaan multi-lompat yang memerlukan informasi dari beberapa dokumen.

## Kesalahan umum

- Menjalankan BM25 dan pencarian vektor pada himpunan potongan berbeda (keduanya harus mencari korpus yang sama)
- Menggunakan kumpulan kandidat terlalu kecil untuk peringkat ulang (top-10 terlalu sedikit; gunakan top-50)
- Menambahkan HyDE untuk setiap kueri (hanya membantu ketika ketidaksesuaian kosakata menjadi hambatan)
- Tidak mengevaluasi perubahan (ukur Recall@k sebelum dan sesudah setiap teknik)
- Merekayasa pipeline berlebihan sebelum mengukur di mana ia gagal

## Alur kerja evaluasi

1. Buat 50+ pertanyaan uji dengan potongan jawaban yang diketahui
2. Ukur Recall@5 dan Recall@10 untuk setiap metode pengambilan
3. Untuk kueri yang pengambilannya berhasil, ukur faithfulness jawaban yang dihasilkan
4. Lacak metrik setiap minggu seiring pertumbuhan korpus
5. Selidiki kegagalan individu sebelum menambahkan teknik lain
