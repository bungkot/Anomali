# Stabilitas Numerik

> Titik mengambang adalah abstraksi yang bocor. Hal itu akan menggigit Anda saat pelatihan, dan Anda tidak akan melihatnya datang.

**Jenis:** Build
**Bahasa:** Python, TypeScript
**Prasyarat:** Fase 1, Pelajaran 01-04
**Waktu:** ~120 menit

## Tujuan Pembelajaran

- Mengimplementasikan softmax dan log-sum-exp yang stabil secara numerik menggunakan trik pengurangan nilai maksimum
- Mengidentifikasi overflow, underflow, dan pembatalan katastropik dalam komputasi titik mengambang
- Memverifikasi gradien analitik terhadap gradien numerik menggunakan selisih hingga terpusat
- Menjelaskan mengapa bfloat16 lebih disukai daripada float16 untuk pelatihan dan bagaimana penskalaan loss mencegah underflow gradien

## Masalahnya

Model Anda berlatih selama tiga jam, lalu loss menjadi NaN. Anda menambahkan pernyataan cetak. Logit masih baik pada langkah 9.000. Pada langkah 9.001 nilainya `inf`. Pada langkah 9.002 setiap gradien adalah `nan` dan pelatihan mati.

Atau: model Anda selesai dilatih, tetapi akurasi 2% lebih rendah daripada yang diklaim makalah. Anda memeriksa semuanya. Arsitektur cocok. Hyperparameter cocok. Data cocok. Masalahnya adalah makalah itu menggunakan float32 dan Anda menggunakan float16 tanpa penskalaan yang tepat. Tiga puluh dua bit galat pembulatan yang terakumulasi diam-diam memakan akurasi Anda.

Atau: Anda mengimplementasikan loss cross-entropy dari awal. Fungsi itu bekerja pada logit kecil. Ketika logit melebihi 100, fungsi mengembalikan `inf`. Softmax mengalami overflow karena `exp(100)` lebih besar daripada yang dapat direpresentasikan float32. Setiap kerangka kerja ML menangani hal ini dengan trik dua baris. Anda tidak tahu trik itu ada.

Stabilitas numerik bukan sekadar kekhawatiran teoretis. Hal itulah yang membedakan antara proses pelatihan yang berhasil dan yang gagal secara senyap. Setiap bug ML serius yang akan Anda debug pada akhirnya bermuara pada titik mengambang.

## Konsepnya

### IEEE 754: Bagaimana Komputer Menyimpan Bilangan Riil

Komputer menyimpan bilangan riil sebagai nilai titik mengambang mengikuti standar IEEE 754. Sebuah float memiliki tiga bagian: bit tanda, eksponen, dan mantisa (significand).

```
Tata letak Float32 (total 32 bit):
[1 tanda] [8 eksponen] [23 mantisa]

Nilai = (-1)^tanda * 2^(eksponen - 127) * 1.mantisa
```

Mantisa menentukan presisi (berapa banyak digit signifikan). Eksponen menentukan rentang (seberapa besar atau kecil sebuah bilangan bisa).

```
Format     Bits   Exponent  Mantissa  Digit desimal  Rentang (perkiraan)
float64    64     11        52        ~15-16          +/- 1.8e308
float32    32     8         23        ~7-8            +/- 3.4e38
float16    16     5         10        ~3-4            +/- 65,504
bfloat16   16     8         7         ~2-3            +/- 3.4e38
```

float32 memberi Anda sekitar 7 digit desimal presisi. Artinya, ia dapat membedakan 1.0000001 dan 1.0000002, tetapi tidak 1.00000001 dan 1.00000002. Setelah 7 digit, semuanya adalah derau pembulatan.

float16 memberi Anda sekitar 3 digit. Bilangan terbesar yang dapat direpresentasikannya adalah 65,504. Itu mengkhawatirkan kecilnya untuk ML, di mana logit, gradien, dan aktivasi secara rutin melebihi nilai itu.

bfloat16 adalah jawaban Google atas masalah rentang float16. Ia memiliki eksponen 8 bit yang sama dengan float32 (rentang sama, hingga 3.4e38) tetapi hanya 7 bit mantisa (presisi lebih rendah daripada float16). Untuk pelatihan jaringan saraf, rentang lebih penting daripada presisi, jadi bfloat16 biasanya menang.

### Mengapa 0.1 + 0.2 != 0.3

Bilangan 0.1 tidak dapat direpresentasikan secara eksak dalam titik mengambang biner. Dalam basis 2, bilangan itu adalah pecahan berulang:

```
0.1 dalam biner = 0.0001100110011001100110011... (berulang selamanya)
```

Float32 memotong ini menjadi 23 bit mantisa. Nilai yang tersimpan kurang lebih 0.100000001490116. Demikian pula, 0.2 disimpan sebagai kurang lebih 0.200000002980232. Jumlah keduanya adalah 0.300000004470348, bukan 0.3.

```
Di Python:
>>> 0.1 + 0.2
0.30000000000000004

>>> 0.1 + 0.2 == 0.3
False
```

Ini penting untuk ML karena:

1. Perbandingan loss seperti `if loss < threshold` dapat memberi jawaban yang salah
2. Akumulasi banyak nilai kecil (pembaruan gradien selama ribuan langkah) menyimpang dari jumlah sejati
3. Checksum dan uji keterulangan gagal jika Anda membandingkan float dengan `==`

Perbaikannya: jangan pernah membandingkan float dengan `==`. Gunakan `abs(a - b) < epsilon` atau `math.isclose()`.

### Pembatalan Katastropik

Ketika Anda mengurangi dua bilangan titik mengambang yang hampir sama, digit signifikan saling meniadakan dan yang tersisa hanyalah derau pembulatan yang dipromosikan menjadi digit terdepan.

```
a = 1.0000001    (disimpan sebagai 1.00000011920929 pada float32)
b = 1.0000000    (disimpan sebagai 1.00000000000000 pada float32)

Selisih sejati:  0.0000001
Hasil hitungan:  0.00000011920929

Galat relatif: 19.2%
```

Itu galat relatif 19% dari satu operasi pengurangan. Dalam ML, hal ini terjadi setiap kali Anda:

- Menghitung varians data dengan rata-rata besar: $\mathbb{E}[x^2] - \mathbb{E}[x]^2$ ketika $\mathbb{E}[x]$ besar
- Mengurangi log-probabilitas yang hampir sama
- Menghitung gradien selisih hingga dengan epsilon yang terlalu kecil

Perbaikannya: susun ulang rumus agar tidak mengurangi bilangan besar yang hampir sama. Untuk varians, gunakan algoritma Welford atau pusatkan data terlebih dahulu. Untuk log-probabilitas, bekerjalah dalam ruang log sepanjang waktu.

### Overflow dan Underflow

Overflow terjadi ketika hasil terlalu besar untuk direpresentasikan. Underflow terjadi ketika hasil terlalu kecil (lebih dekat ke nol daripada bilangan positif terkecil yang dapat direpresentasikan).

```
Batas float32:
  Maksimum:  3.4028235e+38
  Minimum positif (normal): 1.175e-38
  Minimum positif (denorm): 1.401e-45
  Overflow:  apa pun > 3.4e38 menjadi inf
  Underflow: apa pun < 1.4e-45 menjadi 0.0
```

Fungsi `exp()` adalah sumber utama overflow dalam ML:

```
exp(88.7)  = 3.40e+38   (pas-pasan masuk float32)
exp(89.0)  = inf         (overflow)
exp(-87.3) = 1.18e-38   (pas-pasan di atas underflow)
exp(-104)  = 0.0         (underflow menjadi nol)
```

Fungsi `log()` menyerang arah sebaliknya:

```
log(0.0)   = -inf
log(-1.0)  = nan
log(1e-45) = -103.3      (baik-baik saja)
log(1e-46) = -inf        (input underflow menjadi 0, lalu log(0) = -inf)
```

Dalam ML, `exp()` muncul pada softmax, sigmoid, dan komputasi probabilitas. `log()` muncul pada cross-entropy, log-likelihood, dan divergensi KL. Kombinasi `log(exp(x))` adalah ladang ranjau tanpa trik yang tepat.

### Trik Log-Sum-Exp

Menghitung `log(sum(exp(x_i)))` secara langsung berbahaya secara numerik. Jika ada satu `x_i` yang besar, `exp(x_i)` mengalami overflow. Jika semua `x_i` sangat negatif, setiap `exp(x_i)` underflow menjadi nol dan `log(0)` adalah `-inf`.

Triknya: kurangi nilai maksimum sebelum melakukan eksponensiasi.

```math
\log\left(\sum_i \exp(x_i)\right) = \max(x) + \log\left(\sum_i \exp(x_i - \max(x))\right)
```

Mengapa ini bekerja: setelah mengurangi `max(x)`, eksponen terbesar adalah `exp(0) = 1`. Tidak mungkin terjadi overflow. Minimal satu suku dalam jumlahan bernilai 1, sehingga jumlahnya minimal 1, dan `log(1) = 0`. Underflow menjadi `-inf` tidak mungkin terjadi.

Bukti:

```math
\begin{aligned}
& \log\left(\sum_i \exp(x_i)\right) \\
&= \log\left(\sum_i \exp(x_i - c + c)\right) && \text{(tambah dan kurangi } c\text{)} \\
&= \log\left(\sum_i \exp(x_i - c) \cdot \exp(c)\right) && (\exp(a+b) = \exp(a) \cdot \exp(b)) \\
&= \log\left(\exp(c) \cdot \sum_i \exp(x_i - c)\right) && \text{(faktorkan } \exp(c)\text{)} \\
&= c + \log\left(\sum_i \exp(x_i - c)\right) && (\log(a \cdot b) = \log(a) + \log(b))
\end{aligned}
```

Tetapkan `c = max(x)` dan overflow tersingkir.

Trik ini muncul di mana-mana dalam ML:
- Normalisasi softmax
- Komputasi loss cross-entropy
- Penjumlahan log-probabilitas pada model sekuens
- Campuran Gaussian (mixture of Gaussians)
- Inferensi variasional

### Mengapa Softmax Memerlukan Trik Pengurangan Nilai Maksimum

Softmax mengubah logit menjadi probabilitas:

```math
\mathop{\mathrm{softmax}}(x_i) = \frac{\exp(x_i)}{\sum_j \exp(x_j)}
```

Tanpa trik itu, logit [100, 101, 102] menyebabkan overflow:

```
exp(100) = 2.69e43
exp(101) = 7.31e43
exp(102) = 1.99e44
jumlah   = 2.99e44

2.69e43 > 3.4e38, batas maksimum float32, sehingga semua suku overflow.
exp(88.7) sudah berada di batas float32.
exp(100) = inf pada float32.
```

Dengan trik itu, kurangi max(x) = 102:

```
exp(100 - 102) = exp(-2) = 0.135
exp(101 - 102) = exp(-1) = 0.368
exp(102 - 102) = exp(0)  = 1.000
jumlah = 1.503

softmax = [0.090, 0.245, 0.665]
```

Probabilitasnya identik. Komputasinya aman. Ini bukan sekadar optimasi. Ini adalah syarat kebenaran.

### NaN dan Inf: Deteksi dan Pencegahan

`nan` (Not a Number) dan `inf` (infinity) menyebar seperti virus melalui komputasi. Satu `nan` dalam pembaruan gradien membuat bobot menjadi `nan`, yang membuat setiap keluaran berikutnya menjadi `nan`. Pelatihan mati dalam satu langkah.

Bagaimana `inf` muncul:
- `exp()` dari bilangan positif besar
- Pembagian dengan nol: `1.0 / 0.0`
- Overflow float32 pada akumulasi

Bagaimana `nan` muncul:
- `0.0 / 0.0`
- `inf - inf`
- `inf * 0`
- `sqrt()` dari bilangan negatif
- `log()` dari bilangan negatif
- Operasi aritmetika apa pun yang melibatkan `nan` yang sudah ada

Deteksi:

```python
import math

math.isnan(x)       # True jika x adalah nan
math.isinf(x)       # True jika x adalah +inf atau -inf
math.isfinite(x)    # True jika x bukan nan maupun inf
```

Strategi pencegahan:

1. Batasi input untuk `exp()`: `exp(clamp(x, -80, 80))`
2. Tambahkan epsilon pada penyebut: `x / (y + 1e-8)`
3. Tambahkan epsilon di dalam `log()`: `log(x + 1e-8)`
4. Gunakan implementasi yang stabil (log-sum-exp, softmax stabil)
5. Pemotongan gradien untuk mencegah bobot meledak
6. Periksa `nan`/`inf` setelah setiap forward pass saat melakukan debugging

### Pemeriksaan Gradien Numerik

Gradien analitik (dari propagasi balik) bisa memiliki bug. Pemeriksaan gradien numerik memverifikasinya dengan menghitung gradien menggunakan selisih hingga.

Rumus selisih terpusat:

```math
\frac{df}{dx} \approx \frac{f(x + h) - f(x - h)}{2h}
```

Ini akurat hingga $O(h^2)$, jauh lebih baik daripada selisih maju $(f(x+h) - f(x)) / h$ yang hanya $O(h)$.

Memilih $h$: terlalu besar dan perkiraannya salah. Terlalu kecil dan pembatalan katastropik merusak jawaban. `h = 1e-5` sampai `1e-7` adalah nilai umum.

Pemeriksaannya: hitung selisih relatif antara gradien analitik dan numerik.

```math
\text{relative\_error} = \frac{\lvert \text{grad}_{\text{analytical}} - \text{grad}_{\text{numerical}} \rvert}{\max(\lvert \text{grad}_{\text{analytical}} \rvert, \lvert \text{grad}_{\text{numerical}} \rvert, 10^{-8})}
```

Aturan praktis:
- relative_error < 1e-7: sempurna, gradien benar
- relative_error < 1e-5: bisa diterima, kemungkinan benar
- relative_error > 1e-3: ada yang salah
- relative_error > 1: gradien sepenuhnya salah

Selalu periksa gradien saat mengimplementasikan lapisan atau fungsi loss baru. PyTorch menyediakan `torch.autograd.gradcheck()` untuk ini.

### Pelatihan Presisi Campuran

GPU modern memiliki perangkat keras khusus (Tensor Cores) yang menghitung perkalian matriks float16 2-8x lebih cepat daripada float32. Pelatihan presisi campuran memanfaatkannya:

```
1. Simpan salinan utama bobot dalam float32
2. Forward pass dalam float16 (cepat)
3. Hitung loss dalam float32 (mencegah overflow)
4. Backward pass dalam float16 (cepat)
5. Skalakan gradien ke float32
6. Perbarui bobot utama float32
```

Masalah pelatihan float16 murni: gradien sering kali sangat kecil (1e-8 atau lebih kecil). Float16 mengunderflow apa pun di bawah ~6e-8 menjadi nol. Model Anda berhenti belajar karena semua pembaruan gradien bernilai nol.

Perbaikannya adalah penskalaan loss:

```
1. Kalikan loss dengan faktor skala besar (misalnya 1024)
2. Backward pass menghitung gradien dari (loss * 1024)
3. Semua gradien 1024x lebih besar (didorong di atas underflow float16)
4. Bagi gradien dengan 1024 sebelum memperbarui bobot
5. Hasil bersih: pembaruan sama, tetapi tanpa underflow
```

Penskalaan loss dinamis menyesuaikan faktor skala secara otomatis. Mulai dengan nilai besar (65536). Jika gradien overflow menjadi `inf`, bagi dua. Jika $N$ langkah berlalu tanpa overflow, kalikan dua.

### bfloat16 vs float16: Mengapa bfloat16 Menang untuk Pelatihan

```
float16:   [1 tanda] [5 eksponen]  [10 mantisa]
bfloat16:  [1 tanda] [8 eksponen]  [7 mantisa]
```

float16 memiliki presisi lebih tinggi (10 bit mantisa vs 7) tetapi rentang terbatas (maks ~65,504). bfloat16 presisinya lebih rendah tetapi rentangnya sama dengan float32 (maks ~3.4e38).

Untuk pelatihan jaringan saraf:

- Aktivasi dan logit secara rutin melebihi 65,504 saat lonjakan dalam pelatihan. float16 overflow; bfloat16 menanganinya.
- Penskalaan loss diperlukan pada float16 tetapi biasanya tidak perlu pada bfloat16 karena rentangnya mencakup spektrum besar gradien.
- bfloat16 adalah pemotongan sederhana dari float32: buang 16 bit bawah mantisa. Konversi menjadi trivial dan tanpa kehilangan pada eksponen.

float16 lebih disukai untuk inferensi, di mana nilai terbatas dan presisi lebih penting. bfloat16 lebih disukai untuk pelatihan, di mana rentang lebih penting. Inilah sebabnya TPU dan GPU NVIDIA modern (A100, H100) memiliki dukungan bfloat16 native.

### Pemotongan Gradien

Gradien meledak ketika gradien tumbuh secara eksponensial melalui banyak lapisan (umum pada RNN, jaringan dalam, dan transformer). Satu gradien besar dapat merusak semua bobot dalam satu langkah.

Ada dua jenis pemotongan:

**Potong berdasarkan nilai:** batasi setiap elemen gradien secara independen.

```
grad = clamp(grad, -max_val, max_val)
```

Sederhana, tetapi dapat mengubah arah vektor gradien.

**Potong berdasarkan norma:** skalakan seluruh vektor gradien agar normanya tidak melebihi ambang batas.

```
if ||grad|| > max_norm:
    grad = grad * (max_norm / ||grad||)
```

Menjaga arah gradien. Inilah yang dilakukan `torch.nn.utils.clip_grad_norm_()`. Ini adalah pilihan standar.

Nilai umum: `max_norm=1.0` untuk transformer, `max_norm=0.5` untuk RL, `max_norm=5.0` untuk jaringan yang lebih sederhana.

Pemotongan gradien bukan sekadar jalan pintas. Ini adalah mekanisme pengaman. Tanpa itu, satu batch pencilan dapat menghasilkan gradien yang cukup besar untuk merusak berminggu-minggu pelatihan.

### Lapisan Normalisasi sebagai Penstabil Numerik

Batch normalization, layer normalization, dan RMS normalization biasanya diperkenalkan sebagai regularizer yang membantu pelatihan converge. Keduanya juga merupakan penstabil numerik.

Tanpa normalisasi, aktivasi dapat tumbuh atau menyusut secara eksponensial melintasi lapisan:

```
Lapisan 1:  nilai dalam [0, 1]
Lapisan 5:  nilai dalam [0, 100]
Lapisan 10: nilai dalam [0, 10,000]
Lapisan 50: nilai dalam [0, inf]
```

Normalisasi memusatkan dan menskalakan ulang aktivasi di setiap lapisan:

```math
\mathop{\mathrm{LayerNorm}}(x) = \frac{x - \mathop{\mathrm{mean}}(x)}{\mathop{\mathrm{std}}(x) + \epsilon} \cdot \gamma + \beta
```

`epsilon` (umumnya 1e-5) mencegah pembagian dengan nol ketika semua aktivasi identik. Parameter yang dipelajari `gamma` dan `beta` memungkinkan jaringan memulihkan skala apa pun yang dibutuhkannya.

Ini menjaga nilai tetap dalam rentang yang aman secara numerik di seluruh jaringan, mencegah overflow pada forward pass maupun ledakan gradien pada backward pass.

### Bug Numerik ML Umum

**Bug: Loss menjadi NaN setelah beberapa epoch.**
Penyebab: logit tumbuh terlalu besar, softmax overflow. Atau laju pembelajaran terlalu tinggi dan bobot menyimpang.
Perbaikan: gunakan softmax stabil (pengurangan nilai maksimum), turunkan laju pembelajaran, tambahkan pemotongan gradien.

**Bug: Loss mandek di log(num_classes).**
Penyebab: keluaran model berupa probabilitas hampir seragam. Sering kali berarti gradien menghilang atau model sama sekali tidak belajar.
Perbaikan: periksa kebenaran label data, verifikasi fungsi loss, periksa ReLU mati.

**Bug: Akurasi validasi lebih rendah dari perkiraan 1-3%.**
Penyebab: presisi campuran tanpa penskalaan loss yang tepat. Underflow gradien diam-diam mengnolkan pembaruan kecil.
Perbaikan: aktifkan penskalaan loss dinamis, atau beralih ke bfloat16.

**Bug: Norma gradien 0.0 pada beberapa lapisan.**
Penyebab: neuron ReLU mati (semua input negatif), atau underflow float16.
Perbaikan: gunakan LeakyReLU atau GELU, gunakan penskalaan gradien, periksa inisialisasi bobot.

**Bug: Model bekerja pada satu GPU tetapi memberi hasil berbeda pada GPU lain.**
Penyebab: urutan akumulasi titik mengambang non-deterministik. Reduksi paralel GPU menjumlahkan dalam urutan berbeda pada perangkat keras berbeda, dan penjumlahan titik mengambang tidak asosiatif.
Perbaikan: terima perbedaan kecil (1e-6), atau tetapkan `torch.use_deterministic_algorithms(True)` dan terima penalti kecepatan.

**Bug: `exp()` mengembalikan `inf` dalam komputasi loss.**
Penyebab: logit mentah diteruskan ke `exp()` tanpa trik pengurangan nilai maksimum.
Perbaikan: gunakan `torch.nn.functional.log_softmax()` yang menerapkan log-sum-exp secara internal.

**Bug: Pelatihan menyimpang setelah beralih dari float32 ke float16.**
Penyebab: float16 tidak dapat merepresentasikan besar gradien di bawah 6e-8 atau aktivasi di atas 65,504.
Perbaikan: gunakan presisi campuran dengan penskalaan loss (AMP), atau gunakan bfloat16 sebagai gantinya.

```figure
logsumexp-stability
```

## Bangun Sendiri

### Langkah 1: Demonstrasi batas presisi titik mengambang

```python
print("=== Floating Point Precision ===")
print(f"0.1 + 0.2 = {0.1 + 0.2}")
print(f"0.1 + 0.2 == 0.3? {0.1 + 0.2 == 0.3}")
print(f"Difference: {(0.1 + 0.2) - 0.3:.2e}")
```

### Langkah 2: Implementasi softmax naif vs stabil

```python
import math

def softmax_naive(logits):
    exps = [math.exp(z) for z in logits]
    total = sum(exps)
    return [e / total for e in exps]

def softmax_stable(logits):
    max_logit = max(logits)
    exps = [math.exp(z - max_logit) for z in logits]
    total = sum(exps)
    return [e / total for e in exps]

safe_logits = [2.0, 1.0, 0.1]
print(f"Naive:  {softmax_naive(safe_logits)}")
print(f"Stable: {softmax_stable(safe_logits)}")

dangerous_logits = [100.0, 101.0, 102.0]
print(f"Stable: {softmax_stable(dangerous_logits)}")
# softmax_naive(dangerous_logits) akan mengembalikan [nan, nan, nan]
```

### Langkah 3: Implementasi log-sum-exp yang stabil

```python
def logsumexp_naive(values):
    return math.log(sum(math.exp(v) for v in values))

def logsumexp_stable(values):
    c = max(values)
    return c + math.log(sum(math.exp(v - c) for v in values))

safe = [1.0, 2.0, 3.0]
print(f"Naive:  {logsumexp_naive(safe):.6f}")
print(f"Stable: {logsumexp_stable(safe):.6f}")

large = [500.0, 501.0, 502.0]
print(f"Stable: {logsumexp_stable(large):.6f}")
# logsumexp_naive(large) mengembalikan inf
```

### Langkah 4: Implementasi cross-entropy yang stabil

```python
def cross_entropy_naive(true_class, logits):
    probs = softmax_naive(logits)
    return -math.log(probs[true_class])

def cross_entropy_stable(true_class, logits):
    max_logit = max(logits)
    shifted = [z - max_logit for z in logits]
    log_sum_exp = math.log(sum(math.exp(s) for s in shifted))
    log_prob = shifted[true_class] - log_sum_exp
    return -log_prob

logits = [2.0, 5.0, 1.0]
true_class = 1
print(f"Naive:  {cross_entropy_naive(true_class, logits):.6f}")
print(f"Stable: {cross_entropy_stable(true_class, logits):.6f}")
```

### Langkah 5: Pemeriksaan gradien

```python
def numerical_gradient(f, x, h=1e-5):
    grad = []
    for i in range(len(x)):
        x_plus = x[:]
        x_minus = x[:]
        x_plus[i] += h
        x_minus[i] -= h
        grad.append((f(x_plus) - f(x_minus)) / (2 * h))
    return grad

def check_gradient(analytical, numerical, tolerance=1e-5):
    for i, (a, n) in enumerate(zip(analytical, numerical)):
        denom = max(abs(a), abs(n), 1e-8)
        rel_error = abs(a - n) / denom
        status = "OK" if rel_error < tolerance else "FAIL"
        print(f"  param {i}: analytical={a:.8f} numerical={n:.8f} "
              f"rel_error={rel_error:.2e} [{status}]")

def f(params):
    x, y = params
    return x**2 + 3*x*y + y**3

def f_grad(params):
    x, y = params
    return [2*x + 3*y, 3*x + 3*y**2]

point = [2.0, 1.0]
analytical = f_grad(point)
numerical = numerical_gradient(f, point)
check_gradient(analytical, numerical)
```

## Gunakan

### Simulasi presisi campuran

```python
import struct

def float32_to_float16_round(x):
    packed = struct.pack('f', x)
    f32 = struct.unpack('f', packed)[0]
    packed16 = struct.pack('e', f32)
    return struct.unpack('e', packed16)[0]

def simulate_bfloat16(x):
    packed = struct.pack('f', x)
    as_int = int.from_bytes(packed, 'little')
    truncated = as_int & 0xFFFF0000
    repacked = truncated.to_bytes(4, 'little')
    return struct.unpack('f', repacked)[0]
```

### Pemotongan gradien

```python
def clip_by_norm(gradients, max_norm):
    total_norm = math.sqrt(sum(g**2 for g in gradients))
    if total_norm > max_norm:
        scale = max_norm / total_norm
        return [g * scale for g in gradients]
    return gradients

grads = [10.0, 20.0, 30.0]
clipped = clip_by_norm(grads, max_norm=5.0)
print(f"Original norm: {math.sqrt(sum(g**2 for g in grads)):.2f}")
print(f"Clipped norm:  {math.sqrt(sum(g**2 for g in clipped)):.2f}")
print(f"Direction preserved: {[c/clipped[0] for c in clipped]} == {[g/grads[0] for g in grads]}")
```

### Deteksi NaN/Inf

```python
def check_tensor(name, values):
    has_nan = any(math.isnan(v) for v in values)
    has_inf = any(math.isinf(v) for v in values)
    if has_nan or has_inf:
        print(f"WARNING {name}: nan={has_nan} inf={has_inf}")
        return False
    return True

check_tensor("good", [1.0, 2.0, 3.0])
check_tensor("bad",  [1.0, float('nan'), 3.0])
check_tensor("ugly", [1.0, float('inf'), 3.0])
```

Lihat `code/numerical.py` untuk implementasi lengkap dengan semua kasus tepi yang didemonstrasikan. Berkas `code/numerical.ts` adalah port TypeScript dengan fungsi dan demo yang sama.

## Kirimkan

Pelajaran ini menghasilkan:
- `code/numerical.py` dengan softmax stabil, log-sum-exp, cross-entropy, pemeriksaan gradien, dan simulasi presisi campuran
- `outputs/prompt-numerical-debugger.id.md` untuk mendiagnosis NaN/Inf dan masalah numerik dalam pelatihan

Implementasi stabil ini muncul kembali pada Fase 3 saat membangun loop pelatihan dan pada Fase 4 saat mengimplementasikan mekanisme atensi.

## Latihan

1. **Pembatalan katastropik.** Hitung varians dari [1000000.0, 1000001.0, 1000002.0] menggunakan rumus naif $\mathbb{E}[x^2] - \mathbb{E}[x]^2$ dalam float32. Lalu hitunglah menggunakan algoritma online Welford. Bandingkan galatnya terhadap varians sejati (0.6667).

2. **Pemecahan presisi.** Temukan nilai float32 positif terkecil `x` sehingga `1.0 + x == 1.0` di Python. Itulah epsilon mesin. Verifikasi bahwa nilainya cocok dengan `numpy.finfo(numpy.float32).eps`.

3. **Kasus tepi log-sum-exp.** Uji fungsi `logsumexp_stable` Anda dengan: (a) semua nilai sama, (b) satu nilai jauh lebih besar daripada sisanya, (c) semua nilai sangat negatif (-1000). Verifikasi bahwa fungsi memberi hasil benar di tempat versi naif gagal.

4. **Pemeriksaan gradien sebuah lapisan jaringan saraf.** Implementasikan satu lapisan linier `y = Wx + b` dan backward pass analitiknya. Gunakan `numerical_gradient` untuk memverifikasi kebenaran pada matriks bobot 3x2.

5. **Eksperimen penskalaan loss.** Simulasikan pelatihan dengan float16: buat gradien acak dalam rentang [1e-9, 1e-3], konversi ke float16, dan ukur berapa bagian yang menjadi nol. Lalu terapkan penskalaan loss (kalikan dengan 1024), konversi ke float16, skalakan kembali, dan ukur lagi bagian yang menjadi nol.

### Petunjuk

**Latihan 1.** Konversi ketiga nilai dengan `np.float32` sebelum mengkuadratkannya, lalu panggil `variance_naive` dan `welford_variance`. Nilai di dekat $10^{12}$ pada float32 memiliki jarak antarnilai jauh lebih besar dari 1, jadi harapkan hasil naif jauh dari 0.6667 dan hasil Welford dekat dengannya.

**Latihan 2.** Mulai dengan `x = np.float32(1.0)` dan bagi dua terus selama `np.float32(1.0) + x != np.float32(1.0)`. Jaga setiap nilai tetap dalam float32, lalu bandingkan nilai terakhir yang masih mengubah jumlah dengan `numpy.finfo(numpy.float32).eps`, yaitu $2^{-23}$.

**Latihan 3.** Prediksi dulu setiap jawabannya. Untuk $n$ nilai yang sama $c$, hasilnya adalah $c + \ln n$. Untuk satu nilai besar, hasilnya mendekati nilai itu. Untuk nilai di dekat -1000, hasilnya mendekati $-1000 + \ln n$, sedangkan `logsumexp_naive` mengembalikan `-inf` karena setiap eksponensial underflow menjadi 0.

**Latihan 4.** Untuk $y = Wx + b$ dengan loss $L$, gradien terhadap $W$ adalah hasil kali luar (outer product) antara gradien hulu dan $x$. Bungkus loss sebagai fungsi dari $W$ yang diratakan, panggil `numerical_gradient`, lalu teruskan kedua hasil ke `check_gradient`.

**Latihan 5.** Ambil sampel besaran pada skala log dengan `10 ** np.random.uniform(-9, -3, n)` dan konversi dengan `np.float16`. Float16 tidak dapat mempertahankan nilai di bawah sekitar $6 \times 10^{-8}$, jadi harapkan sekitar seperempat nilai menjadi nol sebelum penskalaan dan hampir tidak ada setelah Anda mengalikannya dengan 1024.

## Istilah Kunci

| Istilah | Apa yang orang katakan | Arti sebenarnya |
|---|---|---|
| IEEE 754 | "Standar float" | Standar internasional yang mendefinisikan format titik mengambang biner, aturan pembulatan, dan nilai khusus (inf, nan). Setiap CPU dan GPU modern menerapkannya. |
| Epsilon mesin | "Batas presisi" | Nilai terkecil $e$ sehingga $1.0 + e \ne 1.0$ pada format float tertentu. Untuk float32, nilainya sekitar 1.19e-7. |
| Pembatalan katastropik | "Kehilangan presisi akibat pengurangan" | Saat mengurangi bilangan titik mengambang yang hampir sama, digit signifikan saling meniadakan dan derau pembulatan mendominasi hasil. |
| Overflow | "Bilangan terlalu besar" | Hasil melebihi nilai maksimum yang dapat direpresentasikan dan menjadi inf. exp(89) overflow pada float32. |
| Underflow | "Bilangan terlalu kecil" | Hasil lebih dekat ke nol daripada bilangan positif terkecil yang dapat direpresentasikan dan menjadi 0.0. exp(-104) underflow pada float32. |
| Trik log-sum-exp | "Kurangi nilai maksimum dulu" | Menghitung $\log(\sum \exp(x))$ dengan memfaktorkan $\exp(\max(x))$ untuk mencegah overflow dan underflow. Dipakai pada softmax, cross-entropy, dan matematika log-probabilitas. |
| Softmax stabil | "Softmax yang tidak meledak" | Mengurangi $\max(\text{logits})$ sebelum eksponensiasi. Hasil secara numerik identik, overflow tidak mungkin terjadi. |
| Pemeriksaan gradien | "Verifikasi propagasi balik Anda" | Membandingkan gradien analitik dari propagasi balik dengan gradien numerik dari selisih hingga untuk menangkap bug implementasi. |
| Presisi campuran | "Forward float16, backward float32" | Memakai float untuk operasi yang kritis pada kecepatan dan float presisi tinggi untuk operasi yang sensitif secara numerik. Percepatan khas 2-3x. |
| Penskalaan loss | "Mencegah underflow gradien" | Mengalikan loss dengan konstanta besar sebelum backprop agar gradien tetap dalam rentang yang dapat direpresentasikan float16, lalu membagi dengan konstanta yang sama sebelum pembaruan bobot. |
| bfloat16 | "Brain floating point" | Format 16 bit dari Google dengan 8 bit eksponen (rentang sama seperti float32) dan 7 bit mantisa (presisi lebih rendah daripada float16). Lebih disukai untuk pelatihan. |
| Pemotongan gradien | "Batasi norma gradien" | Menskalakan vektor gradien agar normanya tidak melebihi ambang batas. Mencegah gradien meledak merusak bobot. |
| NaN | "Not a Number" | Nilai float khusus dari operasi takterdefinisi (0/0, inf-inf, sqrt(-1)). Menyebar ke semua aritmetika berikutnya. |
| Inf | "Infinity" | Nilai float khusus dari overflow atau pembagian dengan nol. Dapat dikombinasikan menghasilkan NaN (inf - inf, inf * 0). |
| Gradien numerik | "Turunan brute force" | Mendekati turunan dengan mengevaluasi $f(x+h)$ dan $f(x-h)$ lalu membagi dengan $2h$. Lambat tetapi andal untuk verifikasi. |

## Bacaan Lanjutan

- [What Every Computer Scientist Should Know About Floating-Point Arithmetic (Goldberg 1991)](https://docs.oracle.com/cd/E19957-01/806-3568/ncg_goldberg.html) -- referensi definitif, padat tetapi lengkap
- [Mixed Precision Training (Micikevicius et al., 2018)](https://arxiv.org/abs/1710.03740) -- makalah NVIDIA yang memperkenalkan penskalaan loss untuk pelatihan float16
- [AMP: Automatic Mixed Precision (dokumentasi PyTorch)](https://pytorch.org/docs/stable/amp.html) -- panduan praktis presisi campuran di PyTorch
- [Format bfloat16 (dokumentasi Google Cloud TPU)](https://cloud.google.com/tpu/docs/bfloat16) -- mengapa Google memilih format ini untuk TPU
- [Kahan Summation (Wikipedia)](https://en.wikipedia.org/wiki/Kahan_summation_algorithm) -- algoritma untuk mengurangi galat pembulatan pada penjumlahan titik mengambang
