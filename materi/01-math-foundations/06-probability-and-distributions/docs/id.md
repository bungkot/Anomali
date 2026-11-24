# Probabilitas dan Distribusi

> Probabilitas adalah bahasa yang digunakan AI untuk menyatakan ketidakpastian.

**Jenis:** Belajar
**Bahasa:** Python
**Prasyarat:** Fase 1, Pelajaran 01-04
**Waktu:** ~75 menit

## Tujuan Pembelajaran

- Mengimplementasikan PMF dan PDF dari awal untuk distribusi Bernoulli, kategorikal, Poisson, seragam, dan normal
- Menghitung nilai harapan, varians, dan menggunakan Teorema Limit Pusat untuk menjelaskan mengapa distribusi Gauss mendominasi
- Membangun fungsi softmax dan log-softmax dengan trik stabilitas numerik (kurangi logit maksimum)
- Menghitung rugi cross-entropy dari logit dan mengaitkannya dengan negatif log-likelihood

## Masalahnya

Sebuah klasifikator menghasilkan `[0.03, 0.91, 0.06]`. Sebuah model bahasa memilih kata berikutnya dari 50.000 kandidat. Sebuah model difusi menghasilkan gambar dengan mengambil sampel dari distribusi yang dipelajarinya. Semuanya adalah probabilitas yang sedang bekerja.

Setiap prediksi yang dibuat model adalah sebuah distribusi probabilitas. Setiap fungsi rugi mengukur seberapa jauh distribusi hasil prediksi dari distribusi yang sebenarnya. Setiap langkah pelatihan menyesuaikan parameter agar satu distribusi tampak lebih mirip distribusi lainnya. Tanpa probabilitas, Anda tidak dapat membaca satu pun makalah ML, men-debug satu pun model, atau memahami mengapa rugi pelatihan Anda bernilai NaN.

## Konsepnya

### Kejadian, Ruang Sampel, dan Probabilitas

Ruang sampel $S$ adalah himpunan semua hasil yang mungkin. Sebuah kejadian adalah himpunan bagian dari ruang sampel. Probabilitas memetakan kejadian ke bilangan antara 0 dan 1.

```
Lemparan koin:
  S = {H, T}
  P(H) = 0.5,  P(T) = 0.5

Satu kali lemparan dadu:
  S = {1, 2, 3, 4, 5, 6}
  P(genap) = P({2, 4, 6}) = 3/6 = 0.5
```

Tiga aksioma mendefinisikan seluruh probabilitas:
1. $P(A) \ge 0$ untuk setiap kejadian $A$
2. $P(S) = 1$ (sesuatu selalu terjadi)
3. $P(A \text{ atau } B) = P(A) + P(B)$ ketika $A$ dan $B$ tidak mungkin terjadi bersamaan

Semua hal lainnya (teorema Bayes, nilai harapan, distribusi) mengikuti dari ketiga aturan ini.

### Probabilitas Bersyarat dan Kemerdekaan

$P(A \mid B)$ adalah probabilitas $A$ ketika $B$ telah terjadi.

```
P(A|B) = P(A dan B) / P(B)

Contoh: satu set kartu
  P(Raja | Kartu wajah) = P(Raja dan Kartu wajah) / P(Kartu wajah)
                        = (4/52) / (12/52)
                        = 4/12 = 1/3
```

Dua kejadian saling merdeka ketika mengetahui salah satunya tidak memberi tahu apa pun tentang kejadian lainnya:

```math
\begin{aligned}
\text{Merdeka:} \quad & P(A \mid B) = P(A) \\
\text{Setara dengan:} \quad & P(A \text{ dan } B) = P(A) \cdot P(B)
\end{aligned}
```

Lemparan koin saling merdeka. Mengambil kartu tanpa penggantian tidak demikian.

### Fungsi Massa Probabilitas versus Fungsi Kepadatan Probabilitas

Peubah acak diskrit memiliki fungsi massa probabilitas (PMF). Setiap hasil memiliki probabilitas tertentu yang dapat Anda baca langsung.

```
PMF: P(X = k)

Dadu adil:
  P(X = 1) = 1/6
  P(X = 2) = 1/6
  ...
  P(X = 6) = 1/6

  Jumlah semua probabilitas = 1
```

Peubah acak kontinu memiliki fungsi kepadatan probabilitas (PDF). Kepadatan di satu titik bukanlah probabilitas. Probabilitas diperoleh dengan mengintegralkan kepadatan pada suatu interval.

```
PDF: f(x)

P(a <= X <= b) = integral f(x) dari a sampai b

f(x) dapat lebih besar dari 1 (kepadatan, bukan probabilitas)
integral dari -inf sampai +inf atas f(x) dx = 1
```

Pembedaan ini penting dalam ML. Keluaran klasifikasi adalah PMF (pilihan diskrit). Ruang latent VAE menggunakan PDF (kontinu).

### Distribusi Umum

**Bernoulli:** satu percobaan, dua hasil. Memodelkan klasifikasi biner.

```math
\begin{aligned}
P(X = 1) &= p \\
P(X = 0) &= 1 - p \\
\text{Rata-rata} = p, &\quad \text{Varians} = p(1-p)
\end{aligned}
```

**Kategorikal:** satu percobaan, $k$ hasil. Memodelkan klasifikasi multiclass (keluaran softmax).

```math
\begin{aligned}
& P(X = i) = p_i, \quad \text{dengan } \textstyle\sum_i p_i = 1 \\
& \text{Contoh: } P(\text{kucing}) = 0.7, \quad P(\text{anjing}) = 0.2, \quad P(\text{burung}) = 0.1
\end{aligned}
```

**Seragam:** semua hasil sama mungkin. Digunakan untuk inisialisasi acak.

```math
\begin{aligned}
\text{Diskrit:} \quad & P(X = k) = 1/n \text{ untuk } k \in \lbrace 1, \ldots, n \rbrace \\
\text{Kontinu:} \quad & f(x) = 1/(b-a) \text{ untuk } x \in [a, b]
\end{aligned}
```

**Normal (Gauss):** kurva lonceng. Diparameterisasi oleh rata-rata ($\mu$) dan varians ($\sigma^2$).

```
f(x) = (1 / sqrt(2*pi*sigma^2)) * exp(-(x - mu)^2 / (2*sigma^2))

Normal standar: mu = 0, sigma = 1
  68% data berada dalam 1 sigma
  95% dalam 2 sigma
  99,7% dalam 3 sigma
```

**Poisson:** jumlah kejadian langka dalam interval tetap. Memodelkan laju kejadian.

```math
\begin{aligned}
P(X = k) &= \frac{\lambda^k e^{-\lambda}}{k!} \\
\text{Rata-rata} = \lambda, &\quad \text{Varians} = \lambda
\end{aligned}
```

### Nilai Harapan dan Varians

Nilai harapan adalah rata-rata hasil yang dibobot.

```math
\begin{aligned}
\text{Diskrit:} \quad & \mathbb{E}[X] = \sum_i x_i \, P(X = x_i) \\
\text{Kontinu:} \quad & \mathbb{E}[X] = \int x \, f(x) \, dx
\end{aligned}
```

Varians mengukur penyebaran di sekitar rata-rata.

```math
\begin{aligned}
\mathrm{Var}(X) &= \mathbb{E}[(X - \mathbb{E}[X])^2] = \mathbb{E}[X^2] - (\mathbb{E}[X])^2 \\
\text{Simpangan baku} &= \sqrt{\mathrm{Var}(X)}
\end{aligned}
```

Dalam ML, nilai harapan muncul sebagai fungsi rugi (rata-rata rugi atas distribusi data). Varians memberi tahu Anda tentang kestabilan model. Varians gradien yang tinggi berarti pelatihan yang berisik.

### Distribusi Gabungan dan Marginal

Distribusi gabungan $P(X, Y)$ mendeskripsikan dua peubah acak secara bersamaan.

Contoh PMF gabungan ($X$ = cuaca, $Y$ = payung):

| | $Y=0$ (tanpa payung) | $Y=1$ (berpayung) | Marginal $P(X)$ |
|---|---|---|---|
| $X=0$ (cerah) | 0.40 | 0.10 | $P(X=0) = 0.50$ |
| $X=1$ (hujan) | 0.05 | 0.45 | $P(X=1) = 0.50$ |
| **Marginal $P(Y)$** | $P(Y=0) = 0.45$ | $P(Y=1) = 0.55$ | 1.00 |

Distribusi marginal menjumlahkan peubah yang lain:

```math
P(X = x) = \sum_{y} P(X = x, Y = y)
```

Jumlah baris dan kolom pada tabel di atas adalah marginalnya.

### Mengapa Distribusi Normal Muncul di Mana-Mana

Teorema Limit Pusat: rata-rata dari banyak peubah acak independen yang berasal dari distribusi yang sama, setelah dipusatkan dan diskalakan dengan $\sqrt{n}$, mendekati distribusi normal, apa pun bentuk distribusi awalnya, selama variansnya berhingga. Hal yang sama berlaku untuk jumlah yang dipusatkan dan diskalakan.

```
Lempar 1 dadu:  distribusi seragam (datar)
Rata-rata 2 dadu:  segitiga (memuncak)
Rata-rata 30 dadu: hampir kurva lonceng sempurna

Ini berlaku untuk SEMUA distribusi awal dengan varians berhingga.
```

Inilah sebabnya:
- Galat pengukuran kira-kira berdistribusi normal (banyak sumber kecil yang independen)
- Inisialisasi bobot dalam jaringan saraf menggunakan distribusi normal
- Derakan gradien dalam SGD kira-kira normal (jumlah dari banyak gradien sampel)
- Distribusi normal adalah distribusi entropi maksimum untuk rata-rata dan varians tertentu

### Log Probabilitas

Probabilitas mentah menimbulkan masalah numerik. Mengalikan banyak probabilitas kecil dengan cepat menghasilkan underflow menjadi nol.

```math
\begin{aligned}
P(\text{kalimat}) &= P(\text{kata}_1) \cdot P(\text{kata}_2) \cdot \ldots \cdot P(\text{kata}_n) \\
&= 0.01 \cdot 0.003 \cdot 0.02 \cdot \ldots \\
&\to 0.0 \quad \text{(underflow setelah } \sim 30 \text{ suku)}
\end{aligned}
```

Log probabilitas mengatasi ini. Perkalian berubah menjadi penjumlahan.

```math
\begin{aligned}
\log P(\text{kalimat}) &= \log P(\text{kata}_1) + \log P(\text{kata}_2) + \ldots + \log P(\text{kata}_n) \\
&= -4.6 + -5.8 + -3.9 + \ldots \\
&\to \text{bilangan berhingga (tanpa underflow)}
\end{aligned}
```

Aturannya:
- $\log(a \cdot b) = \log(a) + \log(b)$
- Log probabilitas selalu $\le 0$ (karena $0 < P \le 1$)
- Semakin negatif = semakin tidak mungkin
- Rugi cross-entropy adalah log probabilitas negatif dari kelas yang benar

### Softmax sebagai Distribusi Probabilitas

Jaringan saraf menghasilkan skor mentah (logit). Softmax mengubahnya menjadi distribusi probabilitas yang valid.

```
softmax(z_i) = exp(z_i) / sum(exp(z_j) for all j)

Sifat-sifat:
  - Semua keluaran berada di (0, 1)
  - Semua keluaran berjumlah 1
  - Mempertahankan urutan relatif masukan
  - exp() memperbesar perbedaan antarlogit
```

Trik softmax: kurangi logit maksimum sebelum eksponensiasi untuk mencegah overflow.

```
z = [100, 101, 102]
exp(102) = overflow

z_shifted = z - max(z) = [-2, -1, 0]
exp(0) = 1  (aman)

Hasil sama, tanpa overflow.
```

Log-softmax menggabungkan softmax dan log untuk stabilitas numerik. PyTorch menggunakannya secara internal untuk rugi cross-entropy.

### Pengambilan Sampel

Pengambilan sampel berarti menarik nilai acak dari suatu distribusi. Dalam ML:
- Dropout secara acak memilih neuron yang akan dinolkan
- Augmentasi data mengambil sampel transformasi acak
- Model bahasa mengambil sampel token berikutnya dari distribusi hasil prediksi
- Model difusi mengambil sampel derau dan secara bertahap menghilangkannya

Pengambilan sampel dari distribusi sembarang memerlukan teknik seperti invers transformasi, rejection sampling, atau trik reparameterisasi (digunakan dalam VAE).

```figure
gaussian-pdf
```

## Bangunlah

### Langkah 1: Dasar-dasar probabilitas

```python
import math
import random

def factorial(n):
    result = 1
    for i in range(2, n + 1):
        result *= i
    return result

def combinations(n, k):
    return factorial(n) // (factorial(k) * factorial(n - k))

def conditional_probability(p_a_and_b, p_b):
    return p_a_and_b / p_b

p_king_given_face = conditional_probability(4/52, 12/52)
print(f"P(King | Face card) = {p_king_given_face:.4f}")
```

### Langkah 2: PMF dan PDF dari awal

```python
def bernoulli_pmf(k, p):
    return p if k == 1 else (1 - p)

def categorical_pmf(k, probs):
    return probs[k]

def poisson_pmf(k, lam):
    return (lam ** k) * math.exp(-lam) / factorial(k)

def uniform_pdf(x, a, b):
    if a <= x <= b:
        return 1.0 / (b - a)
    return 0.0

def normal_pdf(x, mu, sigma):
    coeff = 1.0 / (sigma * math.sqrt(2 * math.pi))
    exponent = -0.5 * ((x - mu) / sigma) ** 2
    return coeff * math.exp(exponent)
```

### Langkah 3: Nilai harapan dan varians

```python
def expected_value(values, probabilities):
    return sum(v * p for v, p in zip(values, probabilities))

def variance(values, probabilities):
    mu = expected_value(values, probabilities)
    return sum(p * (v - mu) ** 2 for v, p in zip(values, probabilities))

die_values = [1, 2, 3, 4, 5, 6]
die_probs = [1/6] * 6
mu = expected_value(die_values, die_probs)
var = variance(die_values, die_probs)
print(f"Die: E[X] = {mu:.4f}, Var(X) = {var:.4f}, SD = {var**0.5:.4f}")
```

### Langkah 4: Pengambilan sampel dari distribusi

```python
def sample_bernoulli(p, n=1):
    return [1 if random.random() < p else 0 for _ in range(n)]

def sample_categorical(probs, n=1):
    cumulative = []
    total = 0
    for p in probs:
        total += p
        cumulative.append(total)
    samples = []
    for _ in range(n):
        r = random.random()
        for i, c in enumerate(cumulative):
            if r <= c:
                samples.append(i)
                break
    return samples

def sample_normal_box_muller(mu, sigma, n=1):
    samples = []
    for _ in range(n):
        u1 = random.random()
        u2 = random.random()
        z = math.sqrt(-2 * math.log(u1)) * math.cos(2 * math.pi * u2)
        samples.append(mu + sigma * z)
    return samples
```

### Langkah 5: Softmax dan log probabilitas

```python
def softmax(logits):
    max_logit = max(logits)
    shifted = [z - max_logit for z in logits]
    exps = [math.exp(z) for z in shifted]
    total = sum(exps)
    return [e / total for e in exps]

def log_softmax(logits):
    max_logit = max(logits)
    shifted = [z - max_logit for z in logits]
    log_sum_exp = max_logit + math.log(sum(math.exp(z) for z in shifted))
    return [z - log_sum_exp for z in logits]

def cross_entropy_loss(logits, target_index):
    log_probs = log_softmax(logits)
    return -log_probs[target_index]
```

### Langkah 6: Demonstrasi Teorema Limit Pusat

```python
def demonstrate_clt(dist_fn, n_samples, n_averages):
    averages = []
    for _ in range(n_averages):
        samples = [dist_fn() for _ in range(n_samples)]
        averages.append(sum(samples) / len(samples))
    return averages
```

### Langkah 7: Visualisasi

```python
import matplotlib.pyplot as plt

xs = [mu + sigma * (i - 500) / 100 for i in range(1001)]
ys = [normal_pdf(x, mu, sigma) for x, mu, sigma in ...]
plt.plot(xs, ys)
```

Implementasi lengkap dengan semua visualisasi ada di `code/probability.py`.

## Gunakanlah

Dengan NumPy dan SciPy, semua hal di atas menjadi satu baris:

```python
import numpy as np
from scipy import stats

normal = stats.norm(loc=0, scale=1)
samples = normal.rvs(size=10000)
print(f"Mean: {np.mean(samples):.4f}, Std: {np.std(samples):.4f}")
print(f"P(X < 1.96) = {normal.cdf(1.96):.4f}")

logits = np.array([2.0, 1.0, 0.1])
from scipy.special import softmax, log_softmax
probs = softmax(logits)
log_probs = log_softmax(logits)
print(f"Softmax: {probs}")
print(f"Log-softmax: {log_probs}")
```

Anda membangun semuanya dari awal. Sekarang Anda tahu apa yang dilakukan panggilan pustaka itu.

## Latihan

1. Implementasikan invers transformasi untuk distribusi eksponensial. Verifikasi dengan mengambil 10.000 sampel dan membandingkan histogramnya dengan PDF yang sebenarnya.

2. Bangun tabel distribusi gabungan untuk dua dadu yang tidak adil. Hitung distribusi marginalnya dan periksa apakah kedua dadu saling merdeka.

3. Hitung rugi cross-entropy untuk klasifikator 5 kelas yang menghasilkan logit `[2.0, 0.5, -1.0, 3.0, 0.1]` ketika kelas yang benar adalah indeks 3. Kemudian verifikasi jawaban Anda dengan `nn.CrossEntropyLoss` milik PyTorch.

4. Tulis fungsi yang menerima daftar log probabilitas dan mengembalikan urutan paling mungkin, total log probabilitas, dan probabilitas mentah yang setara. Uji dengan kalimat 50 kata di mana setiap kata memiliki probabilitas 0.01.

### Petunjuk

**Latihan 1.** Ambil `u` dari distribusi seragam dan kembalikan $-\ln(1 - u) / \lambda$. Periksa bahwa rata-rata sampel mendekati $1/\lambda$ dan bahwa bin histogram mengikuti $\lambda e^{-\lambda x}$.

**Latihan 2.** Isi tabel 6x6 di mana setiap sel adalah probabilitas dan jumlah seluruh sel adalah 1, lalu panggil `joint_to_marginals`. Gunakan `check_independence`: kedua dadu merdeka hanya jika setiap sel sama dengan hasil kali dua marginalnya.

**Latihan 3.** Terapkan `softmax` pada logit dan ambil $-\ln$ dari probabilitas pada indeks 3. Hasilnya harus mendekati 0.421, yaitu nilai yang diberikan `nn.CrossEntropyLoss` untuk logit dan target yang sama.

**Latihan 4.** Jumlahkan log probabilitasnya; jangan kalikan probabilitas mentah. Untuk 50 kata dengan probabilitas 0.01, totalnya harus $50 \ln(0.01) \approx -230.26$, dan probabilitas mentahnya adalah $10^{-100}$.

## Istilah Kunci

| Istilah | Yang sering dikatakan | Arti sebenarnya |
|---|---|---|
| Ruang sampel | "Semua kemungkinan" | Himpunan $S$ dari setiap hasil yang mungkin dari sebuah percobaan |
| PMF | "Fungsi probabilitas" | Fungsi yang memberikan probabilitas tepat dari setiap hasil diskrit, dengan jumlah 1 |
| PDF | "Kurva probabilitas" | Fungsi kepadatan untuk peubah kontinu. Integralkan pada suatu interval untuk memperoleh probabilitas |
| Probabilitas bersyarat | "Probabilitas dengan syarat tertentu" | $P(A \mid B) = P(A \text{ dan } B) / P(B)$. Dasar penalaran Bayes dan teorema Bayes |
| Kemerdekaan | "Keduanya tidak saling memengaruhi" | $P(A \text{ dan } B) = P(A) \cdot P(B)$. Mengetahui satu kejadian tidak memberi tahu apa pun tentang kejadian lainnya |
| Nilai harapan | "Rata-ratanya" | Jumlah semua hasil yang dibobot probabilitas. Fungsi rugi adalah sebuah nilai harapan |
| Varians | "Seberapa tersebar" | Rata-rata kuadrat selisih dari rata-rata. Varians tinggi = estimasi berisik dan tidak stabil |
| Distribusi normal | "Kurva lonceng" | $f(x) = (1/\sqrt{2\pi\sigma^2}) \cdot \exp(-(x-\mu)^2/(2\sigma^2))$. Muncul di mana-mana karena CLT |
| Teorema Limit Pusat | "Rata-rata menjadi normal" | Rata-rata yang dipusatkan dan diskalakan dari banyak sampel independen konvergen ke distribusi normal untuk sumber apa pun dengan varians berhingga |
| Distribusi gabungan | "Dua peubah sekaligus" | $P(X, Y)$ mendeskripsikan probabilitas setiap kombinasi hasil $X$ dan $Y$ |
| Distribusi marginal | "Jumlahkan peubah lainnya" | $P(X) = \sum_y P(X, Y)$. Memperoleh distribusi satu peubah dari distribusi gabungan |
| Log probabilitas | "Log dari probabilitas" | $\log P(x)$. Mengubah perkalian menjadi penjumlahan, mencegah underflow numerik pada urutan panjang |
| Softmax | "Ubah skor menjadi probabilitas" | $\text{softmax}(z_i) = \exp(z_i) / \sum_j \exp(z_j)$. Memetakan logit bernilai riil menjadi distribusi probabilitas yang valid |
| Cross-entropy | "Fungsi rugi" | $-\sum p_{\text{benar}} \cdot \log(p_{\text{prediksi}})$. Mengukur seberapa berbeda dua distribusi. Semakin rendah semakin baik |
| Logit | "Keluaran mentah model" | Skor belum ternormalisasi sebelum softmax. Dinamai dari fungsi logistik |
| Pengambilan sampel | "Menarik nilai acak" | Menghasilkan nilai sesuai distribusi probabilitas. Cara model menghasilkan keluaran |

## Bacaan Lanjutan

- [3Blue1Brown: But what is the Central Limit Theorem?](https://www.youtube.com/watch?v=zeJD6dqJ5lo) - bukti visual mengapa rata-rata menjadi normal
- [Stanford CS229 Probability Review](https://cs229.stanford.edu/section/cs229-prob.pdf) - referensi ringkas yang mencakup semua di sini dan lebih banyak lagi
- [The Log-Sum-Exp Trick](https://gregorygundersen.com/blog/2020/02/09/log-sum-exp/) - mengapa stabilitas numerik penting dan cara mencapainya
