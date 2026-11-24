---
name: skill-gradient-computation
description: Menghitung gradien fungsi rugi ML yang umum dan memilih pendekatan turunan yang tepat
version: 1.0.0
phase: 1
lesson: 4
tags: [calculus, gradients, backpropagation]
---

# Perhitungan Gradien untuk ML

Referensi praktis untuk menghitung gradien fungsi rugi, fungsi aktivasi, dan operasi lapisan yang digunakan dalam jaringan saraf.

## Daftar Periksa Keputusan

1. Apakah fungsi tersusun dari primitif sederhana (pangkat, exp, log, trig)? Gunakan turunan analitik dan aturan rantai.
2. Apakah fungsi merupakan operasi kustom atau kotak hitam? Gunakan diferensiasi numerik: `(f(x+h) - f(x-h)) / (2h)` dengan h = 1e-7.
3. Apakah fungsi dibangun dari operasi tensor di PyTorch/JAX? Serahkan ke autograd. Verifikasi dengan pemeriksaan numerik.
4. Apakah Anda membutuhkan gradien rugi skalar terhadap matriks bobot? Terapkan aturan rantai melalui graf komputasi, satu simpul pada satu waktu.
5. Apakah ada operasi yang tidak dapat diturunkan (argmax, pembulatan, pengambilan sampel)? Gunakan straight-through estimator atau trik reparameterisasi.

## Kapan menggunakan tiap pendekatan

| Pendekatan | Kapan digunakan | Biaya |
|---|---|---|
| Analitik (diturunkan manual) | Fungsi sederhana, memverifikasi keluaran autograd | Gratis saat runtime |
| Numerik (selisih hingga) | Debugging, pemeriksaan gradien, fungsi kotak hitam | 2n forward pass untuk n parameter |
| Diferensiasi otomatis | Graf komputasi yang dapat diturunkan (bawaan) | Satu backward pass |
| Simbolik (SymPy, Mathematica) | Menurunkan gradien bentuk tertutup untuk makalah | Hanya waktu kompilasi |

## Referensi cepat: turunan umum

| Fungsi | f(x) | f'(x) | Konteks ML |
|---|---|---|---|
| Rugi MSE | (1/n) sum(y_hat - y)^2 | (2/n)(y_hat - y) | Regresi |
| Cross-entropy (biner) | -(y log(p) + (1-y) log(1-p)) | p - y (setelah sigmoid) | Klasifikasi biner |
| Cross-entropy (multikelas) | -log(p_true_class) | p - one_hot(y) (setelah softmax) | Klasifikasi multikelas |
| Sigmoid | 1 / (1 + e^(-x)) | sigma(x) * (1 - sigma(x)) | Gerbang keluaran, keluaran biner |
| Tanh | (e^x - e^(-x)) / (e^x + e^(-x)) | 1 - tanh(x)^2 | Aktivasi tersembunyi (lawas) |
| ReLU | max(0, x) | 1 jika x > 0, 0 jika x < 0 | Aktivasi tersembunyi bawaan |
| Leaky ReLU | max(0.01x, x) | 1 jika x > 0, 0.01 jika x < 0 | Menghindari neuron mati |
| GELU | x * Phi(x) | Phi(x) + x * phi(x) | Transformers |
| Softmax_i | e^(x_i) / sum(e^(x_j)) | s_i(1 - s_i) untuk i=j, -s_i*s_j untuk i!=j | Lapisan keluaran (Jacobian) |
| Log-softmax | x_i - log(sum(e^(x_j))) | 1 - softmax(x_i) untuk entri ke-i | CE yang stabil secara numerik |
| Lapisan linear | y = Wx + b | dL/dW = dL/dy * x^T, dL/db = dL/dy | Setiap lapisan |
| Regularisasi L2 | lambda * sum(w^2) | 2 * lambda * w | Peluruhan bobot |
| Regularisasi L1 | lambda * sum(\|w\|) | lambda * sign(w) | Kejarangan |

## Kesalahan umum

- Lupa faktor 1/n pada rugi yang dirata-ratakan per batch (MSE, cross-entropy). Gradien diskalakan oleh ukuran batch.
- Menghitung gradien softmax sebagai vektor padahal sebenarnya matriks Jacobian. Untuk cross-entropy + softmax yang digabung, gradien menyederhana menjadi (p - y), sehingga menghindari Jacobian penuh.
- Menerapkan aturan rantai dengan urutan yang salah. Kerjakan mundur dari rugi: dL/dW = dL/dy * dy/dW.
- Memakai h yang terlalu besar (h = 0.1) atau terlalu kecil (h = 1e-15) untuk turunan numerik. Tetap gunakan h = 1e-7 untuk float64.
- Lupa bahwa ReLU memiliki gradien yang tidak terdefinisi tepat di x = 0. Dalam praktiknya, tetapkan menjadi 0 atau 0.5.

## Resep pemeriksaan gradien

```
Untuk setiap parameter w:
  numeric_grad = (loss(w + h) - loss(w - h)) / (2h)
  auto_grad = nilai dari backward pass
  relative_error = |numeric - auto| / max(|numeric|, |auto|, 1e-8)
  assert relative_error < 1e-5
```

Galat relatif di atas 1e-3 berarti ada yang salah. Antara 1e-5 dan 1e-3, selidiki lebih lanjut.
