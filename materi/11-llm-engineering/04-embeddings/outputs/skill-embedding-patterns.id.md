---
name: skill-embedding-patterns
description: Pola produksi untuk embedding, pencarian vektor, dan kemiripan
version: 1.0.0
phase: 11
lesson: 4
tags: [embeddings, vectors, similarity, search, chunking, quantization]
---

# Pola Embedding

Setiap alur kerja embedding mengikuti kontrak ini:

```
text -> embed(text) -> vector (float array)
similarity(vector_a, vector_b) -> score (float)
```

Model embedding dan metrik kemiripan adalah dua keputusan yang benar-benar penting. Semua sisanya hanyalah perpipaan.

## Kapan menggunakan embedding

- Pencarian semantik lintas dokumen (menemukan makna, bukan kata kunci)
- Pengelompokan item serupa (tiket dukungan, ulasan produk, laporan bug)
- Klasifikasi berdasarkan tetangga terdekat (melabeli item baru berdasarkan kemiripan dengan contoh berlabel)
- Sistem rekomendasi (menemukan item yang mirip dengan yang disukai pengguna)
- Deduplikasi (menemukan konten hampir ganda menggunakan ambang kemiripan)

## Kapan TIDAK menggunakan embedding

- Pencocokan kata kunci eksak (gunakan pencarian teks lengkap)
- Kueri terstruktur (gunakan SQL, filter)
- Dataset kecil di mana pelabelan manual lebih cepat (<100 item)
- Tugas di mana penjelasan lebih penting daripada akurasi (embedding bersifat buram/tidak transparan)

## Pemilihan model

Pilih berdasarkan batasan Anda:

- **Perlu API, nilai terbaik**: OpenAI text-embedding-3-small (1536d, $0.02/1M token)
- **Perlu akurasi maksimum**: Voyage-3 (1024d, $0.06/1M token, skor MTEB tertinggi)
- **Perlu lokal/pribadi**: BGE-M3 (1024d, gratis, multibahasa, GPU disarankan)
- **Perlu prototyping lokal yang cepat**: all-MiniLM-L6-v2 (384d, gratis, berjalan di CPU)
- **Perlu multibahasa**: Cohere embed-v3 (1024d) atau BGE-M3 (keduanya kuat untuk multibahasa)

Aturan: jangan pernah mencampur model embedding antara pengindeksan dan kueri. Vektor dari model yang berbeda hidup di ruang yang tidak kompatibel.

## Aturan pemotongan (chunking)

1. Target 256-512 token per potongan dengan tumpang tindih 50 token
2. Jangan pernah memotong di tengah kalimat jika bisa dihindari
3. Sertakan metadata (berkas sumber, judul bagian, posisi) pada setiap potongan
4. Untuk dokumen terstruktur (Markdown, HTML), potong pada batas heading terlebih dahulu
5. Uji kualitas potongan dengan mencari jawaban yang sudah diketahui dan memeriksa hasil pengambilan

## Pemilihan metrik kemiripan

- **Kemiripan kosinus**: pilihan default, menangani teks dengan panjang bervariasi, ternormalisasi
- **Hasil titik (dot product)**: gunakan saat vektor sudah ternormalisasi satuan (model OpenAI sudah demikian), sedikit lebih cepat
- **Jarak Euclidean**: gunakan untuk pengelompokan, saat posisi absolut penting

Ketiganya memberikan peringkat yang sama saat vektor ternormalisasi. Pilihan hanya berpengaruh pada vektor yang tidak ternormalisasi.

## Optimasi penyimpanan

Tiga tingkat kompresi yang dapat ditumpuk:

1. **Pemotongan Matryoshka**: mengurangi dimensi (1536 -> 256 = penghematan 6x, kehilangan akurasi 3-5%)
2. **Kuantisasi Float16**: membagi dua penyimpanan per dimensi (penghematan 2x, kehilangan akurasi <1%)
3. **Kuantisasi biner**: 1 bit per dimensi (penghematan 32x, kehilangan akurasi 5-10%, gunakan dengan penskoran ulang)

Pola produksi: pencarian biner atas seluruh korpus, skor ulang 1000 teratas dengan vektor float32.

## Ambil lalu skor ulang

Pipeline dua tahap untuk akurasi terbaik:

1. Bi-encoder mengambil 100 kandidat teratas (cepat, menggunakan embedding yang dihitung sebelumnya)
2. Cross-encoder menykor ulang menjadi 10 teratas (lambat, memproses setiap pasangan kueri-dokumen)

Ini mengungguli pengambilan satu tahap sebesar 10-15% pada metrik presisi. Gunakan saat akurasi lebih penting daripada latensi.

## Kesalahan umum

- Menggunakan model embedding berbeda untuk pengindeksan dan kueri
- Mengembedding seluruh dokumen alih-alih potongan (embedding menjadi rata-rata dari semuanya)
- Tidak menormalisasi vektor sebelum kemiripan kosinus (kebanyakan model melakukan normalisasi awal, tetapi verifikasilah)
- Mengabaikan tumpang tindih potongan (kalimat yang terpotong pada batas kehilangan konteks)
- Menyimpan hanya vektor tanpa teks aslinya (Anda memerlukan keduanya untuk pengambilan)
- Tidak mengembedding ulang saat model berubah (vektor lama tidak kompatibel)
- Memilih dimensi hanya berdasarkan akurasi (penyimpanan dan latensi berskala linear dengan dimensi)

## Mendebug embedding

Jika hasil pencarian buruk:

1. Verifikasi embedding kueri tidak nol (masukan kosong atau hanya spasi menghasilkan vektor nol)
2. Periksa secara manual skor kemiripan dokumen yang diketahui relevan
3. Coba ubah kalimat kueri agar cocok dengan kosakata dokumen
4. Periksa batas potongan untuk memastikan konten relevan tidak terbagi dari satu potongan ke potongan lain
5. Bandingkan hasil top-k antarmetrik (kosinus, hasil titik, Euclidean) untuk mendeteksi masalah normalisasi
6. Uji dengan kueri yang pasti cocok (salin satu kalimat dari dokumen) untuk mengonfirmasi pipeline berfungsi

## Parameter produksi

- Ukuran potongan: 256-512 token
- Tumpang tindih potongan: 50 token (10-20% dari ukuran potongan)
- Pengambilan top-k: 5-10 untuk penggunaan langsung, 50-100 untuk penskoran ulang
- Ambang kemiripan: 0.7+ untuk kosinus (di bawah itu, hasil biasanya tidak relevan)
- Embedding batch: proses 100-500 teks per panggilan API untuk throughput
- Pembangunan ulang indeks: embedding ulang saat model berubah atau dokumen diperbarui secara signifikan
