---
name: skill-svd
description: Terapkan SVD pada masalah nyata termasuk kompresi, penghilangan kebisingan, rekomendasi, dan penyelesaian least-squares
phase: 1
lesson: 11
---

Anda adalah pakar dalam menerapkan Dekomposisi Nilai Singular (SVD) pada masalah rekayasa praktis. Ketika diberikan tugas yang melibatkan matriks, kompresi data, kebisingan, data yang hilang, atau sistem linear, tentukan apakah SVD adalah alat yang tepat dan cara menerapkannya.

## Kerangka Keputusan

### Langkah 1: Identifikasi jenis masalah

- **Kompresi data / reduksi dimensi**: Gunakan SVD terpotong. Pertahankan k nilai singular teratas. Pilih k berdasarkan ambang energi (95% adalah target umum) atau kinerja tugas hilir.
- **Penghilangan kebisingan**: Hitung SVD penuh. Cari celah pada spektrum nilai singular. Potong di bawah celah tersebut. Celah itu memisahkan sinyal dari kebisingan.
- **Data hilang / rekomendasi**: Isi entri yang hilang (rata-rata baris atau nol), hitung SVD, rekonstruksi dengan pangkat rendah. Dalam produksi, gunakan ALS atau SVD inkremental yang menangani data hilang secara langsung.
- **Least-squares / pseudoinverse**: Hitung SVD. Inverskan nilai singular yang bukan nol. Kalikan V Sigma+ U^T dengan vektor target. Lebih stabil daripada persamaan normal.
- **Kesamaan teks / pemodelan topik**: Bangun matriks istilah-dokumen. Terapkan SVD (ini adalah LSA/LSI). Proyeksikan dokumen dan istilah ke ruang pangkat rendah. Gunakan kesamaan kosinus untuk perbandingan.
- **Penentuan pangkat numerik**: Hitung SVD. Hitung nilai singular di atas ambang (relatif terhadap nilai terbesar). Ini lebih andal daripada reduksi baris.
- **Perhitungan norma matriks**: Norma spektral = nilai singular terbesar. Norma Frobenius = sqrt(jumlah kuadrat nilai singular). Norma nuklir = jumlah nilai singular.
- **Bilangan kondisi**: sigma_max / sigma_min. Menunjukkan seberapa sensitif sistem terhadap gangguan.

### Langkah 2: Pilih varian yang tepat

| Situasi | Metode | Alasan |
|---|---|---|
| Matriks padat, dekomposisi penuh diperlukan | `np.linalg.svd(A)` / `svd(A)` di Julia | Algoritma standar, stabil secara numerik |
| Hanya k komponen teratas yang diperlukan | `scipy.sparse.linalg.svds(A, k)` | Lebih cepat daripada SVD penuh saat k kecil |
| Matriks jarang | `scipy.sparse.linalg.svds` | Menangani penyimpanan jarang secara efisien |
| Data streaming | SVD inkremental / SVD online | Memperbarui dekomposisi tanpa menghitung ulang dari awal |
| Data hilang (rekomendasi) | ALS, Funk SVD, atau NMF | SVD standar memerlukan matriks lengkap |
| Matriks sangat besar (jutaan baris) | SVD acak (`sklearn.utils.extmath.randomized_svd`) | O(mn log k) alih-alih O(mn min(m,n)) |
| PCA pada data yang dipusatkan | SVD dari matriks data yang dipusatkan | Setara dengan dekomposisi eigen pada kovarians, tetapi lebih stabil |

### Langkah 3: Pilih pangkat k

- **Ambang energi**: Hitung energi kumulatif = sum(sigma_1^2 ... sigma_k^2) / sum(all sigma^2). Berhenti saat energi melebihi 0.95 (atau 0.99 untuk tugas fidelitas tinggi).
- **Deteksi celah**: Plot nilai singular. Cari penurunan tajam. Celah menandakan batas antara sinyal dan kebisingan.
- **Validasi silang**: Untuk tugas hilir, sapu k dan ukur kinerja pada data yang disisihkan.
- **Metode siku**: Plot error rekonstruksi terhadap k. Siku adalah titik di mana penambahan komponen berhenti membantu.
- **Pengetahuan domain**: Jika Anda tahu data memiliki d faktor mendasar, gunakan k = d.

### Langkah 4: Validasi hasil

- **Error rekonstruksi**: Hitung ||A - A_k|| / ||A||. Seharusnya kecil jika pemotongan bermakna.
- **Varians terjelaskan**: Untuk PCA/kompresi, laporkan fraksi varians (energi) total yang tertangkap.
- **Kinerja tugas hilir**: Jika SVD adalah langkah praproses, ukur metrik menyeluruh.
- **Inspeksi visual**: Untuk gambar, bandingkan secara visual gambar asli dan hasil rekonstruksi. Untuk rekomendasi, periksa prediksi terhadap peringkat yang diketahui.

## Kesalahan Umum

- Menghitung SVD melalui dekomposisi eigen dari A^T A. Ini mengkuadratkan bilangan kondisi dan kehilangan presisi numerik. Gunakan rutin SVD khusus.
- Menggunakan SVD penuh ketika hanya k komponen teratas yang diperlukan. Untuk matriks besar, gunakan SVD terpotong atau SVD acak (randomized).
- Menerapkan SVD langsung pada matriks dengan entri yang hilang. SVD standar memerlukan matriks lengkap. Gunakan metode pelengkapan matriks (ALS, Funk SVD) sebagai gantinya.
- Mengabaikan pemusatan. Untuk PCA, data harus dipusatkan (rata-rata dikurangkan) sebelum SVD. Tanpa pemusatan, komponen pertama menangkap rata-rata, bukan varians.
- Memotong terlalu jauh. Jika terlalu sedikit nilai singular yang dipertahankan, sinyal hilang. Jika terlalu banyak, kebisingan ikut dipertahankan. Gunakan ambang energi atau validasi silang.
- Mencampuradukkan SVD dengan dekomposisi eigen. SVD bekerja pada matriks apa pun (bentuk apa pun, pangkat apa pun). Dekomposisi eigen memerlukan matriks persegi dengan himpunan lengkap vektor eigen. Untuk matriks simetris semi-definit positif, keduanya sama.

## Pola Kode

### Kompresi cepat
```python
U, S, Vt = np.linalg.svd(A, full_matrices=False)
k = np.searchsorted(np.cumsum(S**2) / np.sum(S**2), 0.95) + 1
A_compressed = U[:, :k] @ np.diag(S[:k]) @ Vt[:k, :]
```

### Pseudoinverse untuk least squares
```python
U, S, Vt = np.linalg.svd(A, full_matrices=False)
S_inv = np.array([1/s if s > 1e-10 else 0 for s in S])
x = Vt.T @ np.diag(S_inv) @ U.T @ b
```

### Penghilangan kebisingan
```python
U, S, Vt = np.linalg.svd(noisy_data, full_matrices=False)
k = find_gap(S)
clean_data = U[:, :k] @ np.diag(S[:k]) @ Vt[:k, :]
```

### PCA skala besar
```python
from sklearn.utils.extmath import randomized_svd
U, S, Vt = randomized_svd(X_centered, n_components=50, random_state=42)
explained_variance = S**2 / (n_samples - 1)
```

## Kapan TIDAK menggunakan SVD

- Matriks sangat jarang dan Anda hanya memerlukan beberapa komponen. Gunakan penyelesai eigen jarang secara langsung.
- Anda memerlukan faktor bukan negatif (pemodelan topik, unmixing spektral). Gunakan NMF sebagai gantinya.
- Data memiliki struktur nonlinear kuat yang tidak dapat ditangkap metode linier. Gunakan autoencoder atau pembelajaran manifold.
- Anda memerlukan pembaruan waktu nyata pada data streaming dan matriks berubah terus-menerus. Gunakan SVD inkremental/online atau metode aproksimatif.
- Matriks muat dalam memori, tetapi sangat besar sehingga bahkan SVD acak pun terlalu lambat. Pertimbangkan metode sketching atau pendekatan berbasis sampling.

## Biaya Komputasi

| Metode | Waktu | Ruang |
|---|---|---|
| SVD penuh dari matriks m x n | O(mn min(m,n)) | O(mn) |
| SVD terpotong (k teratas) | O(mnk) | O((m+n)k) |
| SVD acak (k teratas) | O(mn log k) | O((m+n)k) |
| Iterasi daya (1 vektor) | O(mn * iters) | O(m+n) |

Untuk matriks 10000 x 5000:
- SVD penuh: ~250 miliar operasi
- SVD terpotong (k=50): ~2,5 miliar operasi
- SVD acak (k=50): ~500 juta operasi

Pilih metode yang sesuai dengan skala dan kebutuhan akurasi Anda.
