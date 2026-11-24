---
name: skill-convexity-checker
description: Menentukan apakah suatu masalah optimasi bersifat konveks dan memilih penyelesai yang tepat
version: 1.0.0
phase: 1
lesson: 18
tags: [optimization, convexity, solvers]
---

# Pemeriksa Konveksitas

Cara memverifikasi apakah suatu masalah optimasi bersifat konveks, dan apa yang harus dilakukan dengan jawaban tersebut.

## Daftar Periksa Keputusan

1. Apakah fungsi objektif bersifat konveks? (Periksa definit semi-positif Hessian atau gunakan aturan komposisi.)
2. Apakah semua kendala pertidaksamaan berbentuk g_i(x) <= 0 dengan setiap g_i konveks?
3. Apakah semua kendala persamaan bersifat afin (linear)?
4. Jika ketiganya ya, masalah tersebut konveks. Gunakan penyelesai konveks dengan jaminan konvergensi.
5. Jika ada yang tidak, masalah tersebut non-konveks. Gunakan SGD/Adam dan terima optimum lokal.

## Cara menguji konveksitas suatu fungsi

| Uji | Berlaku untuk | Metode |
|---|---|---|
| Turunan kedua >= 0 | Fungsi skalar f(x) | Hitung f''(x). Jika f''(x) >= 0 untuk semua x, konveks. |
| Hessian bersifat PSD | Fungsi multivariat f(x) | Hitung H(x). Jika semua nilai eigen >= 0 di mana pun, konveks. |
| Uji definisi | Semua fungsi | Periksa f(tx + (1-t)y) <= t*f(x) + (1-t)*f(y) untuk x, y, t yang diambil sampelnya. |
| Aturan komposisi | Fungsi hasil komposisi | Lihat tabel komposisi di bawah. |
| Pembatasan pada satu garis | f multivariat | f konveks jika dan hanya jika g(t) = f(x + tv) konveks dalam t untuk semua x, v. |

## Aturan komposisi (yang mempertahankan konveksitas)

| Operasi | Hasil |
|---|---|
| f + g (keduanya konveks) | Konveks |
| c * f (c > 0, f konveks) | Konveks |
| max(f, g) (keduanya konveks) | Konveks |
| f(Ax + b) dengan f konveks | Konveks |
| g(f(x)) dengan g konveks non-menurun dan f konveks | Konveks |
| g(f(x)) dengan g konveks non-naik dan f konkaf | Konveks |
| jumlah fungsi-fungsi konveks | Konveks |
| supremum pointwise dari fungsi-fungsi konveks | Konveks |

## Objektif ML umum: konveks atau tidak?

| Objektif | Konveks? | Alasan |
|---|---|---|
| MSE: (1/n) sum(y - Xw)^2 | Ya | Kuadratik dalam w, Hessian = (2/n) X^T X bersifat PSD |
| Logistic loss: sum(log(1 + exp(-y_i * w^T x_i))) | Ya | Jumlah fungsi konveks (keluarga log-sum-exp) |
| Hinge loss: sum(max(0, 1 - y_i * w^T x_i)) | Ya | Maksimum dari fungsi (linear) konveks |
| Regularisasi L2: lambda * \|\|w\|\|^2 | Ya | Kuadratik, Hessian = 2*lambda*I |
| Regularisasi L1: lambda * \|\|w\|\|_1 | Ya | Jumlah nilai mutlak (konveks tetapi tidak terdiferensiasi) |
| Regresi ridge: MSE + L2 | Ya | Jumlah dua fungsi konveks |
| LASSO: MSE + L1 | Ya | Jumlah dua fungsi konveks |
| Elastic net: MSE + L1 + L2 | Ya | Jumlah fungsi konveks |
| SVM (primal): hinge + L2 | Ya | Jumlah fungsi konveks |
| Cross-entropy dengan softmax | Ya (dalam logit) | Log-sum-exp bersifat konveks |
| Jaringan saraf (loss apa pun) | Tidak | Aktivasi nonlinear menciptakan komposisi non-konveks |
| Objektif k-means | Tidak | Langkah penugasan diskrit |
| Faktorisasi matriks: \|\|X - UV^T\|\|^2 | Tidak | Bilinear dalam U dan V |
| Loss GAN | Tidak | Minimaks, non-konveks dalam generator |
| Contrastive loss (InfoNCE) | Tidak | Log dari rasio eksponensial dengan sampel negatif |

## Pemilihan penyelesai berdasarkan konveksitas

| Jenis masalah | Penyelesai | Jaminan konvergensi |
|---|---|---|
| Konveks, mulus, tanpa kendala | Gradient descent | O(1/k) menuju minimum global |
| Konveks, mulus, tanpa kendala | L-BFGS | Superlinear menuju minimum global |
| Konveks, mulus, tanpa kendala | Metode Newton | Kuadratik di dekat minimum (jika Hessian dapat dihitung) |
| Konveks, mulus, berkendala | Metode interior point | Waktu polinomial |
| Konveks, tidak mulus (L1) | Proximal gradient / ISTA | O(1/k) menuju minimum global |
| Konveks, tidak mulus (L1) | ADMM | Fleksibel, menangani kendala |
| Konveks, kuadratik | Conjugate gradient | Tepat dalam n langkah |
| Non-konveks, mulus | SGD / Adam | Konvergen ke minimum lokal |
| Non-konveks, mulus | SGD + restart | Minimum lokal yang lebih baik secara rata-rata |
| Non-konveks, mulus | Overparameterisasi + SGD | Minimum datar, generalisasi baik |

## Kesalahan umum

- Mengira suatu masalah konveks hanya karena fungsi loss-nya konveks. Loss harus konveks dalam parameter yang Anda optimasi. Cross-entropy konveks dalam logit, tetapi pemetaan jaringan saraf lengkap dari masukan ke logit bersifat non-konveks.
- Menggunakan metode Newton pada masalah non-konveks. Hessian mungkin memiliki nilai eigen negatif, menyebabkan Newton bergerak menuju titik pelana atau maksimum alih-alih minimum.
- Lupa bahwa regularisasi L1 membuat objektif tidak terdiferensiasi di nol. Gradient descent standar tidak bekerja dengan baik. Gunakan proximal gradient descent atau metode subgradient.
- Menguadratkan angka kondisi dengan membentuk A^T A. Jika Anda perlu menyelesaikan masalah least squares dan A terkondisi buruk, gunakan QR atau SVD alih-alih persamaan normal.
- Menyatakan suatu masalah non-konveks tanpa memeriksanya. Banyak masalah ML (model linear, SVM, regresi logistik) bersifat konveks dan mendapat manfaat dari penyelesai yang lebih kuat.

## Uji cepat: apakah masalah saya konveks?

```
1. Tulis objektifnya: minimalkan f(w) dengan kendala
2. Untuk setiap suku dalam f(w):
   - Apakah kuadratik dengan matriks PSD? -> Konveks
   - Apakah berupa norma? -> Konveks
   - Apakah log-sum-exp? -> Konveks
   - Apakah melibatkan w secara nonlinear (sigmoid(w), w1*w2)? -> Kemungkinan non-konveks
3. Apakah semua kendala berupa pertidaksamaan linear atau konveks?
4. Jika SEMUA suku konveks dan kendala konveks/linear -> masalah bersifat konveks
5. Jika ADA suku non-konveks -> masalah bersifat non-konveks
```
