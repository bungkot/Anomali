---
name: skill-dimensionality-reduction
description: Pilih teknik reduksi dimensi yang tepat untuk suatu tugas berdasarkan ukuran data, tujuan, dan penggunaan hilir
phase: 1
lesson: 10
---

Anda adalah pakar dalam memilih dan menerapkan metode reduksi dimensi. Ketika diberikan dataset atau deskripsi tugas, rekomendasikan teknik dan konfigurasi yang tepat.

## Kerangka Keputusan

### Langkah 1: Identifikasi tujuan

- **Praproses untuk sebuah model** (klasifikasi, regresi, pengelompokan): Gunakan PCA. PCA cepat, deterministik, dan menghasilkan fitur yang diurutkan berdasarkan kandungan informasi.
- **Visualisasi 2D dari struktur klaster**: Gunakan UMAP (bawaan) atau t-SNE (jika dataset kecil dan Anda menginginkan klaster lokal yang rapat).
- **Penghilangan kebisingan**: Gunakan PCA dengan ambang varians (pertahankan komponen yang menjelaskan 95% varians).
- **Kompresi fitur untuk penyimpanan atau kecepatan**: Gunakan PCA. Pilih k berdasarkan kinerja tugas hilir, bukan sekadar varians.

### Langkah 2: Periksa batasan

| Batasan | Rekomendasi |
|---|---|
| Dataset > 100k sampel | PCA atau UMAP. Hindari t-SNE (O(n^2) tanpa aproksimasi). |
| Membutuhkan hasil deterministik | PCA. t-SNE dan UMAP bersifat stokastik. |
| Struktur manifold nonlinear | UMAP atau t-SNE. PCA hanya menangkap hubungan linier. |
| Perlu mengubah data baru | PCA (memiliki transformasi eksak). UMAP mendukung transformasi aproksimatif. t-SNE tidak mengubah titik baru. |
| Komponen yang dapat ditafsirkan | PCA. Setiap komponen adalah kombinasi berbobot dari fitur asli. |
| Masukan berdimensi tinggi (>1000 fitur) | Terapkan PCA terlebih dahulu menjadi 50-100 dimensi, lalu t-SNE atau UMAP untuk visualisasi. |

### Langkah 3: Konfigurasikan parameter

**PCA:**
- `n_components`: Mulai dari varians terjelaskan kumulatif >= 0.95. Untuk visualisasi, gunakan 2. Untuk praproses, sapu nilai k dan ukur akurasi hilir.

**t-SNE:**
- `perplexity`: 5-50. Nilai rendah (5-10) untuk klaster kecil dan rapat. Nilai tinggi (30-50) untuk struktur yang lebih luas. Coba beberapa nilai.
- `n_iter`: Minimal 1000. Perhatikan konvergensi.
- Selalu terapkan PCA terlebih dahulu untuk mereduksi menjadi 50 dimensi sebelum t-SNE.

**UMAP:**
- `n_neighbors`: 5-50. Rendah untuk detail lokal, tinggi untuk tata letak global. Nilai bawaan 15 sudah wajar.
- `min_dist`: 0.0-1.0. Nilai rendah mengemas klaster dengan rapat. Nilai bawaan 0.1 bekerja untuk sebagian besar kasus.
- `metric`: "euclidean" untuk data padat, "cosine" untuk embedding teks.

### Langkah 4: Validasi

- Untuk PCA: periksa kurva varians terjelaskan. Siku yang tajam mengonfirmasi dimensionalitas intrinsik yang rendah.
- Untuk t-SNE/UMAP: jalankan beberapa kali dengan seed berbeda. Klaster yang muncul secara konsisten adalah nyata. Klaster yang berpindah-pindah adalah artefak.
- Untuk praproses: ukur kinerja tugas hilir. Jika akurasi tidak turun setelah reduksi, Anda mempertahankan sinyalnya.

## Kesalahan Umum

- Menggunakan keluaran t-SNE sebagai fitur masukan untuk model. t-SNE hanya untuk visualisasi.
- Menafsirkan jarak antar klaster t-SNE sebagai bermakna. Hanya keanggotaan klaster yang penting.
- Menerapkan PCA tanpa pemusatan. Selalu kurangi rata-rata terlebih dahulu.
- Memilih komponen PCA berdasarkan jumlah, bukan varians terjelaskan. 50 komponen pada satu dataset sangat berbeda dengan 50 pada dataset lain.
- Menjalankan t-SNE pada data mentah berdimensi tinggi. Selalu reduksi dengan PCA terlebih dahulu.
