---
name: nsa-integrator
description: Rencana integrasi untuk Native Sparse Attention dalam proses pra-pelatihan konteks panjang.
version: 1.0.0
phase: 10
lesson: 17
tags: [nsa, sparse-attention, long-context, pre-training, kernel-aligned, deepseek]
---

Diberikan spesifikasi proses pra-pelatihan konteks panjang (target konteks, arsitektur dasar, token pelatihan yang tersedia, topologi GPU, target deployment), hasilkan rencana integrasi NSA.

Hasilkan:

1. Ukuran blok kompresi `l`. Pilih 32, 64, atau 128. Justifikasi terhadap target konteks: `l = 32` untuk 16k-32k, `l = 64` untuk 64k-128k, `l = 128` untuk 256k ke atas. `l` yang lebih besar berarti lebih sedikit kunci terkompresi, tetapi sinyal perutean yang lebih kasar.
2. Jumlah pemilihan top-k. Pilih antara 8 dan 32. Default makalah adalah 16. Justifikasi terhadap campuran tugas target: tugas yang berat penalaran (matematika, kode) mendapat manfaat dari `k` yang lebih tinggi karena presisi seleksi lebih penting. Tugas yang berat pengambilan bekerja baik pada `k` yang lebih rendah.
3. Jendela geser `W`. Pilih 256, 512, atau 1024. Default 512. Lebih pendek untuk konten yang sangat terstruktur (kode) yang konteks lokalnya sudah cukup; lebih panjang untuk prosa.
4. Gate MLP. Tentukan lebar dan inisialisasi. Default: lapisan linear dari `hidden` ke 3, dengan aktivasi `sigmoid` atau `softplus`. Beri peringatan jika bobot gate runtuh dan lebih memilih satu cabang — ini menandakan `l`, `k`, atau `W` tidak tersetel dengan baik.
5. Pilihan kernel. Konfirmasi ketersediaan kernel Triton atau CUDA untuk akselerator target. Tolak fallback ke atensi padat saat inferensi (seluruh tujuan NSA adalah menghemat komputasi dekode). Jika hanya kernel forward yang ada dan tidak ada kernel backward, tolak pra-pelatihan dan rekomendasikan pelatihan lanjutan pada checkpoint padat yang sudah ada.

Penolakan keras:
- NSA pada model yang pra-dilatih dengan atensi padat tanpa pra-pelatihan lanjutan. Tidak bisa dipasang begitu saja saat inferensi.
- Target konteks di bawah 16k. Overhead tiga cabang mendominasi.
- Deployment hanya-inferensi pada stack tanpa dukungan kernel NSA. Rekomendasikan MLA atau atensi jendela geser sebagai gantinya.

Aturan penolakan:
- Jika data evaluasi konteks panjang (RULER, LongBench, needle-in-haystack) tidak tersedia, tolak dan minta data kalibrasi terlebih dahulu.
- Jika distribusi konteks data pelatihan didominasi oleh urutan pendek, tolak dan rekomendasikan penimbangan ulang data sebelum mengintegrasikan NSA.
- Jika akselerator lebih tua dari A100, tolak — keunggulan kernel NSA mengasumsikan hierarki memori H100/H200/MI300.

Output: rencana integrasi satu halaman yang mencantumkan `l`, `k`, `W`, konfigurasi gate, jalur kernel, dan penghematan komputasi yang diharapkan pada target konteks. Akhiri dengan paragraf "kriteria keberhasilan": angka RULER atau LongBench spesifik (poin persentase terhadap baseline atensi padat yang sepadan) yang membenarkan mempertahankan NSA. Sertakan pemicu rollback — ambang metrik di bawah mana arsitektur harus dikembalikan ke MLA atau GQA padat.
