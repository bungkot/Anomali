# Metode Pengambilan Sampel

> Pengambilan sampel adalah cara AI menjelajahi ruang kemungkinan.

**Jenis:** Bangun
**Bahasa:** Python, TypeScript
**Prasyarat:** Fase 1, Pelajaran 06-07 (Probabilitas, Teorema Bayes)
**Waktu:** ~120 menit

## Tujuan Pembelajaran

- Mengimplementasikan CDF invers, rejection sampling, dan importance sampling dari awal hanya dengan bilangan acak seragam
- Membangun sampling suhu, top-k, dan top-p (nucleus) untuk pembangkitan token model bahasa
- Menjelaskan trik reparameterisasi dan mengapa trik itu memungkinkan propagasi balik melalui pengambilan sampel pada VAE
- Menjalankan MCMC Metropolis-Hastings untuk mengambil sampel dari distribusi target yang belum ternormalisasi

## Masalahnya

Sebuah model bahasa selesai memproses prompt Anda dan menghasilkan vektor 50.000 logit. Satu untuk setiap token dalam kosakatanya. Sekarang model itu harus memilih satu. Bagaimana caranya?

Jika model itu selalu memilih token dengan probabilitas tertinggi, setiap jawaban identik. Deterministik. Membosankan. Jika ia memilih secara acak seragam, keluarannya adalah omong kosong. Jawabannya terletak di antara dua ekstrem itu, dan titik itu dikendalikan oleh pengambilan sampel.

Pengambilan sampel tidak hanya terbatas pada pembangkitan teks. Pembelajaran penguatan mengestimasi gradien kebijakan dengan mengambil sampel lintasan. VAE mempelajari representasi latent dengan mengambil sampel dari distribusi yang dipelajari dan melakukan propagasi balik melalui keacakan itu. Model difusi menghasilkan gambar dengan mengambil sampel derau dan berulang kali menghilangkannya. Metode Monte Carlo mengestimasi integral yang tidak punya solusi bentuk tertutup. Algoritma MCMC menjelajahi distribusi posterior berdimensi tinggi yang mustahil untuk dihitung satu per satu.

Setiap sistem AI generatif adalah sistem pengambilan sampel. Strategi pengambilan sampel menentukan kualitas, keragaman, dan kendali atas keluaran. Pelajaran ini membangun semua metode pengambilan sampel utama dari awal, mulai dari bilangan acak seragam hingga teknik yang tenaga LLM dan model generatif modern.

## Konsepnya

### Mengapa Pengambilan Sampel Penting

Pengambilan sampel muncul dalam empat peran fundamental di seluruh AI dan pembelajaran mesin:

**Pembangkitan.** Model bahasa, model difusi, dan GAN semuanya menghasilkan keluaran dengan mengambil sampel. Algoritma pengambilan sampel secara langsung mengendalikan kreativitas, koherensi, dan keragaman. Suhu, top-k, dan nucleus sampling adalah tombol yang setiap hari diputar para insinyur.

**Pelatihan.** Stochastic gradient descent mengambil sampel mini-batch. Dropout mengambil sampel neuron yang akan dinonaktifkan. Augmentasi data mengambil sampel transformasi acak. Importance sampling memberi bobot ulang sampel untuk mengurangi varians gradien dalam pembelajaran penguatan (PPO, TRPO).

**Estimasi.** Banyak besaran dalam ML tidak memiliki solusi bentuk tertutup. Rugi harapan atas distribusi data, fungsi partisi model berbasis energi, evidence dalam inferensi Bayes. Estimasi Monte Carlo mendekati semuanya dengan merata-ratakan sampel.

**Eksplorasi.** Algoritma MCMC menjelajahi distribusi posterior dalam inferensi Bayes. Strategi evolusioner mengambil sampel gangguan parameter. Thompson sampling menyeimbangkan eksplorasi dan eksploitasi pada bandit.

Tantangan intinya: Anda hanya dapat mengambil sampel langsung dari distribusi sederhana (seragam, normal). Untuk semua hal lain, Anda memerlukan metode untuk mengubah sampel sederhana menjadi sampel dari distribusi target Anda.

### Pengambilan Sampel Acak Seragam

Setiap metode pengambilan sampel dimulai dari sini. Pembangkit bilangan acak seragam menghasilkan nilai dalam $[0, 1)$ di mana setiap subinterval dengan panjang sama memiliki probabilitas sama.

```
U ~ Uniform(0, 1)

P(a <= U <= b) = b - a    untuk 0 <= a <= b <= 1

Sifat-sifat:
  E[U] = 0.5
  Var(U) = 1/12
```

Untuk mengambil sampel secara seragam dari himpunan diskrit berisi $n$ butir, bangkitkan $U$ dan kembalikan $\lfloor n U \rfloor$. Untuk mengambil sampel dari rentang kontinu $[a, b]$, hitung $a + (b - a) U$.

Wawasan kuncinya: satu bilangan acak seragam memuat tepat jumlah keacakan yang dibutuhkan untuk menghasilkan satu sampel dari distribusi apa pun. Triknya adalah menemukan transformasi yang tepat.

### Metode CDF Invers (Invers Transformasi)

Fungsi distribusi kumulatif (CDF) memetakan nilai ke probabilitas:

```
F(x) = P(X <= x)

Sifat-sifat:
  F tidak menurun
  F(-inf) = 0
  F(+inf) = 1
  F memetakan garis bilangan riil ke [0, 1]
```

CDF invers memetakan probabilitas kembali ke nilai. Jika $U \sim \text{Uniform}(0, 1)$, maka $X = F^{-1}(U)$ mengikuti distribusi target.

```
Algoritma:
  1. Bangkitkan u ~ Uniform(0, 1)
  2. Kembalikan F_inverse(u)

Mengapa ini bekerja:
  P(X <= x) = P(F_inverse(U) <= x) = P(U <= F(x)) = F(x)
```

**Contoh distribusi eksponensial:**

```
PDF: f(x) = lambda * exp(-lambda * x),   x >= 0
CDF: F(x) = 1 - exp(-lambda * x)

Selesaikan F(x) = u untuk x:
  u = 1 - exp(-lambda * x)
  exp(-lambda * x) = 1 - u
  x = -ln(1 - u) / lambda

Karena (1 - U) dan U memiliki distribusi yang sama:
  x = -ln(u) / lambda
```

Ini bekerja sempurna ketika Anda dapat menuliskan $F^{-1}$ dalam bentuk tertutup. Untuk distribusi normal, tidak ada CDF invers bentuk tertutup, sehingga kami menggunakan metode lain (Box-Muller atau aproksimasi numerik).

**Versi diskrit:** Untuk distribusi diskrit, bangun CDF sebagai jumlah kumulatif, bangkitkan $U$, dan temukan indeks pertama di mana jumlah kumulatif melebihi $U$. Inilah cara kerja `sample_categorical` di Pelajaran 06.

### Rejection Sampling

Ketika Anda tidak dapat membalik CDF tetapi dapat mengevaluasi PDF target hingga konstanta, rejection sampling bekerja.

```
Distribusi target: p(x)  (dapat dievaluasi, mungkin belum ternormalisasi)
Distribusi proposal: q(x)  (dapat diambil sampelnya)
Batas: M sedemikian sehingga p(x) <= M * q(x) untuk semua x

Algoritma:
  1. Ambil sampel x ~ q(x)
  2. Ambil sampel u ~ Uniform(0, 1)
  3. Jika u < p(x) / (M * q(x)), terima x
  4. Jika tidak, tolak dan kembali ke langkah 1

Laju penerimaan = Z/M, dengan Z adalah integral p(x)
                 (Z = 1, sehingga lajunya 1/M, ketika p ternormalisasi)
```

Semakin ketat batas $M$, semakin tinggi laju penerimaan. Pada dimensi rendah (1-3), rejection sampling bekerja dengan baik. Pada dimensi tinggi, laju penerimaan turun secara eksponensial karena sebagian besar volume proposal ditolak. Inilah kutukan dimensi untuk rejection sampling.

**Contoh: pengambilan sampel dari normal terpotong.** Gunakan proposal seragam pada rentang yang terpotong. Amplop $M$ adalah nilai maksimum PDF normal pada rentang itu.

**Contoh: pengambilan sampel dari setengah lingkaran.** Usulkan secara seragam dalam persegi panjang pembatas. Terima jika titik itu jatuh di dalam setengah lingkaran. Inilah cara Monte Carlo menghitung $\pi$: laju penerimaan sama dengan rasio luas $\pi/4$.

### Importance Sampling

Kadang Anda tidak memerlukan sampel dari distribusi target $p(x)$. Anda perlu mengestimasi nilai harapan di bawah $p(x)$, dan Anda memiliki sampel dari distribusi lain $q(x)$.

```
Tujuan: estimasi E_p[f(x)] = integral f(x) * p(x) dx

Tulis ulang:
  E_p[f(x)] = integral f(x) * (p(x)/q(x)) * q(x) dx
            = E_q[f(x) * w(x)]

dengan w(x) = p(x) / q(x)  adalah bobot kepentingan.

Estimator:
  E_p[f(x)] ~ (1/N) * sum(f(x_i) * w(x_i))    dengan x_i ~ q(x)
```

Ini sangat penting dalam pembelajaran penguatan. Pada PPO (Proximal Policy Optimization), Anda mengumpulkan lintasan di bawah kebijakan lama $\pi_{\text{old}}$ tetapi ingin mengoptimalkan kebijakan baru $\pi_{\text{new}}$. Bobot kepentingannya adalah $\pi_{\text{new}}(a \mid s) / \pi_{\text{old}}(a \mid s)$. PPO memotong bobot ini agar kebijakan baru tidak menyimpang terlalu jauh dari kebijakan lama.

Varians estimator importance sampling bergantung pada seberapa mirip $q$ dengan $p$. Jika $q$ sangat berbeda dari $p$, beberapa sampel mendapat bobot sangat besar dan mendominasi estimasi. Self-normalized importance sampling membagi dengan jumlah bobot untuk mengurangi masalah ini:

```math
\mathbb{E}_p[f(x)] \approx \frac{\sum_i w_i \, f(x_i)}{\sum_i w_i}
```

### Estimasi Monte Carlo

Estimasi Monte Carlo mendekati integral dengan merata-ratakan sampel acak. Hukum bilangan besar menjamin konvergensi.

```
Tujuan: estimasi I = integral g(x) dx atas domain D

Metode:
  1. Ambil sampel x_1, ..., x_N secara seragam dari D
  2. I ~ (Volume D / N) * sum(g(x_i))

Galat: O(1 / sqrt(N))   terlepas dari dimensi
```

Laju galatnya tidak bergantung pada dimensi. Inilah sebabnya metode Monte Carlo mendominasi pada dimensi tinggi di mana integrasi berbasis grid mustahil.

**Mengestimasi pi:**

```
Ambil sampel (x, y) secara seragam dari [-1, 1] x [-1, 1]
Hitung berapa banyak yang jatuh di dalam lingkaran satuan: x^2 + y^2 <= 1
pi ~ 4 * (banyak di dalam) / (banyak total)
```

**Mengestimasi nilai harapan:**

```
E[f(X)] ~ (1/N) * sum(f(x_i))    dengan x_i ~ p(x)

Rata-rata sampel konvergen ke nilai harapan sejati.
Varians estimator = Var(f(X)) / N
```

### MCMC: Metropolis-Hastings

MCMC membangun rantai Markov yang distribusi stasionernya adalah distribusi target $p(x)$. Setelah cukup banyak langkah, sampel dari rantai tersebut adalah (kira-kira) sampel dari $p(x)$.

```
Target: p(x)  (diketahui hingga konstanta penormalisasi)
Proposal: q(x'|x)  (cara mengusulkan keadaan berikutnya dari keadaan saat ini)

Algoritma Metropolis-Hastings:
  1. Mulai dari suatu x_0
  2. Untuk t = 1, 2, ..., T:
     a. Usulkan x' ~ q(x'|x_t)
     b. Hitung rasio penerimaan:
        alpha = [p(x') * q(x_t|x')] / [p(x_t) * q(x'|x_t)]
     c. Terima dengan probabilitas min(1, alpha):
        - Jika u < alpha (u ~ Uniform(0,1)): x_{t+1} = x'
        - Jika tidak: x_{t+1} = x_t
  3. Buang B sampel pertama (burn-in)
  4. Kembalikan sampel sisanya
```

Untuk proposal simetris $(q(x' \mid x) = q(x \mid x'))$, rasio menyederhana menjadi $p(x')/p(x)$. Inilah algoritma Metropolis yang semula.

**Mengapa ini bekerja.** Aturan penerimaan menjamin keseimbangan rinci: probabilitas berada di $x$ lalu bergerak ke $x'$ sama dengan probabilitas berada di $x'$ lalu bergerak ke $x$. Keseimbangan rinci menyiratkan bahwa $p(x)$ adalah distribusi stasioner rantai itu.

**Pertimbangan praktis:**
- Burn-in: buang sampel awal sebelum rantai mencapai keseimbangan
- Thinning: simpan setiap sampel ke-$k$ untuk mengurangi autokorelasi
- Skala proposal: terlalu kecil membuat rantai bergerak lambat (penerimaan tinggi, eksplorasi lambat); terlalu besar membuat sebagian besar proposal ditolak (penerimaan rendah, rantai mandek)
- Laju penerimaan optimal untuk proposal Gauss pada dimensi tinggi adalah sekitar 0,234

### Gibbs Sampling

Gibbs sampling adalah kasus khusus MCMC untuk distribusi multivariat. Alih-alih mengusulkan pergerakan di semua dimensi sekaligus, metode ini memperbarui satu peubah pada satu waktu dari distribusi bersyaratnya.

```
Target: p(x_1, x_2, ..., x_d)

Algoritma:
  Untuk setiap iterasi t:
    Ambil sampel x_1^{t+1} ~ p(x_1 | x_2^t, x_3^t, ..., x_d^t)
    Ambil sampel x_2^{t+1} ~ p(x_2 | x_1^{t+1}, x_3^t, ..., x_d^t)
    ...
    Ambil sampel x_d^{t+1} ~ p(x_d | x_1^{t+1}, x_2^{t+1}, ..., x_{d-1}^{t+1})
```

Gibbs sampling mensyaratkan bahwa Anda dapat mengambil sampel dari setiap distribusi bersyarat $p(x_i \mid x_{-i})$. Ini mudah dilakukan pada banyak model:
- Jaringan Bayes: kondisional mengikuti struktur graf
- Campuran Gauss: kondisional berbentuk Gauss
- Model Ising: kondisional setiap spin hanya bergantung pada tetangganya

Laju penerimaannya selalu 1 (setiap proposal diterima) karena pengambilan sampel dari kondisional yang tepat secara otomatis memenuhi keseimbangan rinci.

**Keterbatasan.** Ketika peubah-peubah sangat berkorelasi, Gibbs sampling bercampur lambat karena memperbarui satu peubah pada satu waktu tidak dapat melakukan pergerakan diagonal besar melintasi distribusi.

### Sampling Suhu (Digunakan dalam LLM)

Model bahasa menghasilkan logit $z_1, \ldots, z_V$ untuk setiap token dalam kosakata. Softmax mengubahnya menjadi probabilitas. Suhu menskalakan ulang logit sebelum softmax:

```
p_i = exp(z_i / T) / sum(exp(z_j / T))

T = 1.0: softmax standar (distribusi asli)
T -> 0:  argmax (deterministik, selalu memilih logit tertinggi)
T -> inf: seragam (semua token sama mungkin)
T < 1.0: mempertajam distribusi (lebih percaya diri, kurang beragam)
T > 1.0: meratakan distribusi (kurang percaya diri, lebih beragam)
```

**Mengapa ini bekerja.** Membagi logit dengan $T < 1$ memperbesar perbedaan antarlogit. Jika $z_1 = 2$ dan $z_2 = 1$, pembagian dengan $T = 0.5$ menghasilkan $z_1/T = 4$ dan $z_2/T = 2$, sehingga jaraknya semakin besar. Setelah softmax, token dengan logit tertinggi mendapat porsi jauh lebih besar.

**Dalam praktik:**
- $T = 0.0$: dekode greedy, terbaik untuk tanya-jawab faktual
- $T = 0.3\text{-}0.7$: agak kreatif, baik untuk pembangkitan kode
- $T = 0.7\text{-}1.0$: seimbang, baik untuk percakapan umum
- $T = 1.0\text{-}1.5$: penulisan kreatif, brainstorming
- $T > 1.5$: semakin acak, jarang berguna

Suhu tidak mengubah token mana yang mungkin. Suhu mengubah massa probabilitas yang dialokasikan pada setiap token.

### Sampling Top-k

Sampling top-k membatasi himpunan kandidat pada $k$ token dengan probabilitas tertinggi, lalu menormalisasi ulang dan mengambil sampel dari himpunan yang sudah dibatasi itu.

```
Algoritma:
  1. Hitung probabilitas softmax untuk semua V token
  2. Urutkan token berdasarkan probabilitas (menurun)
  3. Simpan hanya k token teratas
  4. Normalisasi ulang: p_i' = p_i / sum(p_j untuk j di top-k)
  5. Ambil sampel dari distribusi yang dinormalisasi ulang

k = 1:  dekode greedy
k = V:  tanpa filter (sampling standar)
k = 40: pengaturan umum, membuang ekor panjang token yang tidak mungkin
```

Top-k mencegah model memilih token yang sangat tidak mungkin (salah ketik, omong kosong) yang ada di ekor panjang distribusi kosakata. Masalahnya: $k$ tetap tanpa memedulikan konteks. Ketika model yakin (satu token memiliki probabilitas 95%), $k = 40$ tetap mengizinkan 39 alternatif. Ketika model tidak yakin (probabilitas tersebar di 1000 token), $k = 40$ memotong opsi yang masuk akal.

### Sampling Top-p (Nucleus)

Sampling top-p menyesuaikan secara dinamis ukuran himpunan kandidat. Alih-alih menyimpan sejumlah token yang tetap, metode ini menyimpan himpunan token terkecil yang probabilitas kumulatifnya melebihi $p$.

```
Algoritma:
  1. Hitung probabilitas softmax untuk semua V token
  2. Urutkan token berdasarkan probabilitas (menurun)
  3. Temukan k terkecil sedemikian sehingga jumlah probabilitas top-k >= p
  4. Simpan hanya k token itu
  5. Normalisasi ulang dan ambil sampel

p = 0.9:  menyimpan token yang mencakup 90% massa probabilitas
p = 1.0:  tanpa filter
p = 0.1:  sangat ketat, hampir greedy
```

Ketika model yakin, nucleus sampling menyimpan sedikit token (mungkin 2-3). Ketika model tidak yakin, ia menyimpan banyak token (mungkin 200). Perilaku adaptif inilah yang membuat nucleus sampling umumnya menghasilkan teks lebih baik daripada top-k.

**Kombinasi umum:**
- Suhu 0,7 + top-p 0,9: pengaturan serbaguna yang baik
- Suhu 0,0 (greedy): terbaik untuk tugas deterministik
- Suhu 1,0 + top-k 50: pengaturan makalah asli Fan et al. (2018)

Top-k dan top-p dapat digabungkan. Terapkan top-k dahulu, lalu top-p pada himpunan yang tersisa.

### Trik Reparameterisasi (Digunakan dalam VAE)

Variational autoencoder (VAE) belajar dengan mengodekan masukan ke dalam distribusi di ruang latent, mengambil sampel dari distribusi itu, lalu mendekode sampel kembali. Masalahnya: Anda tidak dapat melakukan propagasi balik melalui operasi pengambilan sampel.

```
Sampling standar (tidak terdiferensialkan):
  z ~ N(mu, sigma^2)

  Keacakan memblokir aliran gradien.
  d/d_mu [sampel dari N(mu, sigma^2)] = ???
```

Trik reparameterisasi memisahkan keacakan dari parameter:

```
Sampling dengan reparameterisasi:
  epsilon ~ N(0, 1)          (derau acak tetap, tanpa parameter)
  z = mu + sigma * epsilon   (fungsi deterministik dari parameter)

  Sekarang z adalah fungsi deterministik yang terdiferensialkan dari mu dan sigma.
  d(z)/d(mu) = 1
  d(z)/d(sigma) = epsilon

  Gradien mengalir melalui mu dan sigma.
```

Ini bekerja karena $\mathcal{N}(\mu, \sigma^2)$ memiliki distribusi yang sama dengan $\mu + \sigma \cdot \mathcal{N}(0, 1)$. Wawasan kuncinya: pindahkan keacakan ke sumber tanpa parameter ($\epsilon$), lalu nyatakan sampel sebagai transformasi parameter yang terdiferensialkan.

**Dalam loop pelatihan VAE:**
1. Encoder menghasilkan $\mu$ dan $\log(\sigma^2)$ untuk setiap masukan
2. Ambil sampel $\epsilon \sim \mathcal{N}(0, 1)$
3. Hitung $z = \mu + \sigma \epsilon$
4. Dekode $z$ untuk merekonstruksi masukan
5. Lakukan propagasi balik melalui langkah 4, 3, 2, 1 (memungkinkan karena langkah 3 terdiferensialkan)

Tanpa trik reparameterisasi, VAE tidak dapat dilatih dengan propagasi balik standar. Satu wawasan ini membuat VAE menjadi praktis.

### Gumbel-Softmax (Sampling Kategorikal yang Terdiferensialkan)

Trik reparameterisasi bekerja untuk distribusi kontinu (Gauss). Untuk distribusi kategorikal diskrit, kita memerlukan pendekatan berbeda. Gumbel-Softmax menyediakan aproksimasi yang terdiferensialkan untuk sampling kategorikal.

**Trik Gumbel-Max (tidak terdiferensialkan):**

```
Untuk mengambil sampel dari distribusi kategorikal dengan log-probabilitas log(p_1), ..., log(p_k):
  1. Ambil sampel g_i ~ Gumbel(0, 1) untuk setiap kategori
     (g = -log(-log(u)), dengan u ~ Uniform(0, 1))
  2. Kembalikan argmax(log(p_i) + g_i)

Ini menghasilkan sampel kategorikal yang tepat.
```

**Gumbel-Softmax (aproksimasi terdiferensialkan):**

```
Ganti argmax keras dengan softmax lunak:
  y_i = exp((log(p_i) + g_i) / tau) / sum(exp((log(p_j) + g_j) / tau))

tau (suhu) mengendalikan aproksimasi:
  tau -> 0:  mendekati vektor one-hot (kategorikal keras)
  tau -> inf: mendekati seragam (1/k, 1/k, ..., 1/k)
  tau = 1.0: aproksimasi lunak
```

Gumbel-Softmax menghasilkan relaksasi kontinu dari sampel diskrit. Keluarannya adalah vektor probabilitas (one-hot lunak), bukan one-hot keras. Gradien mengalir melalui softmax. Saat forward pass dalam pelatihan, Anda dapat menggunakan estimator "straight-through": memakai argmax keras untuk forward pass tetapi gradien Gumbel-Softmax lunak untuk backward pass.

**Aplikasi:**
- Peubah latent diskrit dalam VAE
- Pencarian arsitektur saraf (memilih operasi diskrit)
- Mekanisme atensi keras
- Pembelajaran penguatan dengan aksi diskrit

### Sampling Berlapis (Stratified Sampling)

Sampling Monte Carlo standar dapat meninggalkan celah dalam ruang sampel karena kebetulan. Sampling berlapis memaksa cakupan merata dengan membagi ruang menjadi lapisan dan mengambil sampel dari masing-masing.

```
Monte Carlo standar:
  Ambil N titik secara seragam dari [0, 1]
  Beberapa wilayah mungkin memiliki gumpalan, yang lain celah

Sampling berlapis:
  Bagi [0, 1] menjadi N lapisan sama: [0, 1/N), [1/N, 2/N), ..., [(N-1)/N, 1)
  Ambil satu titik secara seragam di dalam setiap lapisan
  x_i = (i + u_i) / N   dengan u_i ~ Uniform(0, 1),  i = 0, ..., N-1
```

Sampling berlapis selalu memiliki varians lebih rendah atau sama dibandingkan Monte Carlo standar:

```
Var(berlapis) <= Var(Monte Carlo standar)

Perbaikan terbesar terjadi ketika f(x) berubah secara halus.
Untuk fungsi konstan per bagian, sampling berlapis bersifat eksak.
```

**Aplikasi:**
- Integrasi numerik (quasi-Monte Carlo)
- Pembagian data pelatihan (memastikan keseimbangan kelas di setiap lipatan)
- Importance sampling dengan pelapisan (menggabungkan kedua teknik)
- NeRF (Neural Radiance Fields) menggunakan sampling berlapis sepanjang sinar kamera

### Hubungan dengan Model Difusi

Model difusi menghasilkan gambar melalui proses pengambilan sampel. Proses maju menambahkan derau Gauss pada gambar selama $T$ langkah sampai gambar itu menjadi derau murni. Proses mundur mempelajari cara menghilangkan derau, memulihkan gambar asli langkah demi langkah.

```
Proses maju (diketahui):
  x_t = sqrt(alpha_t) * x_{t-1} + sqrt(1 - alpha_t) * epsilon
  dengan epsilon ~ N(0, I)

  Setelah T langkah: x_T ~ N(0, I)  (derau murni)

Proses mundur (dipelajari):
  x_{t-1} = (1/sqrt(alpha_t)) * (x_t - (1 - alpha_t)/sqrt(1 - alpha_bar_t) * epsilon_theta(x_t, t)) + sigma_t * z
  dengan z ~ N(0, I)

  Setiap langkah penghilangan derau adalah satu langkah pengambilan sampel.
```

Hubungan dengan metode dalam pelajaran ini:
- Setiap langkah penghilangan derau menggunakan trik reparameterisasi (ambil sampel derau, terapkan transformasi deterministik)
- Jadwal derau $\lbrace \alpha_t \rbrace$ mengendalikan sejenis annealing suhu
- Pelatihan menggunakan estimasi Monte Carlo untuk mendekati ELBO (evidence lower bound)
- Sampling leluhur pada model difusi adalah rantai Markov (setiap langkah hanya bergantung pada keadaan saat ini)

Seluruh proses pembangkitan gambar adalah pengambilan sampel iteratif: mulai dari derau, dan pada setiap langkah ambil sampel versi yang sedikit lebih bersih dengan syarat model penghilang derau yang dipelajari.

```figure
monte-carlo-pi
```

## Bangunlah

### Langkah 1: Sampling seragam dan CDF invers

```python
import math
import random

def sample_uniform(a, b):
    return a + (b - a) * random.random()

def sample_exponential_inverse_cdf(lam):
    u = random.random()
    return -math.log(u) / lam
```

Bangkitkan 10.000 sampel eksponensial dan verifikasi bahwa rata-ratanya adalah $1/\lambda$.

### Langkah 2: Rejection sampling

```python
def rejection_sample(target_pdf, proposal_sample, proposal_pdf, M):
    while True:
        x = proposal_sample()
        u = random.random()
        if u < target_pdf(x) / (M * proposal_pdf(x)):
            return x
```

Gunakan rejection sampling untuk mengambil sampel dari distribusi normal terpotong. Verifikasi bentuknya dengan membuat histogram sampel tersebut.

### Langkah 3: Importance sampling

```python
def importance_sampling_estimate(f, target_pdf, proposal_pdf, proposal_sample, n):
    total = 0
    for _ in range(n):
        x = proposal_sample()
        w = target_pdf(x) / proposal_pdf(x)
        total += f(x) * w
    return total / n
```

Estimasi $\mathbb{E}[X^2]$ di bawah distribusi normal menggunakan proposal seragam. Bandingkan dengan jawaban yang sudah diketahui ($\mu^2 + \sigma^2$).

### Langkah 4: Estimasi Monte Carlo untuk pi

```python
def monte_carlo_pi(n):
    inside = 0
    for _ in range(n):
        x = random.uniform(-1, 1)
        y = random.uniform(-1, 1)
        if x*x + y*y <= 1:
            inside += 1
    return 4 * inside / n
```

### Langkah 5: MCMC Metropolis-Hastings

```python
def metropolis_hastings(target_log_pdf, proposal_sample, proposal_log_pdf, x0, n_samples, burn_in):
    samples = []
    x = x0
    for i in range(n_samples + burn_in):
        x_new = proposal_sample(x)
        log_alpha = (target_log_pdf(x_new) + proposal_log_pdf(x, x_new)
                     - target_log_pdf(x) - proposal_log_pdf(x_new, x))
        if math.log(random.random()) < log_alpha:
            x = x_new
        if i >= burn_in:
            samples.append(x)
    return samples
```

Ambil sampel dari distribusi bimodal (campuran dua Gauss). Visualisasikan lintasan rantainya.

### Langkah 6: Gibbs sampling

```python
def gibbs_sampling_2d(conditional_x_given_y, conditional_y_given_x, x0, y0, n_samples, burn_in):
    x, y = x0, y0
    samples = []
    for i in range(n_samples + burn_in):
        x = conditional_x_given_y(y)
        y = conditional_y_given_x(x)
        if i >= burn_in:
            samples.append((x, y))
    return samples
```

### Langkah 7: Sampling suhu

```python
def softmax(logits):
    max_l = max(logits)
    exps = [math.exp(z - max_l) for z in logits]
    total = sum(exps)
    return [e / total for e in exps]

def temperature_sample(logits, temperature):
    scaled = [z / temperature for z in logits]
    probs = softmax(scaled)
    return sample_from_probs(probs)
```

Tunjukkan bagaimana suhu mengubah distribusi keluaran untuk sekumpulan logit token.

### Langkah 8: Sampling top-k dan top-p

```python
def top_k_sample(logits, k):
    indexed = sorted(enumerate(logits), key=lambda x: -x[1])
    top = indexed[:k]
    top_logits = [l for _, l in top]
    probs = softmax(top_logits)
    idx = sample_from_probs(probs)
    return top[idx][0]

def top_p_sample(logits, p):
    probs = softmax(logits)
    indexed = sorted(enumerate(probs), key=lambda x: -x[1])
    cumsum = 0
    selected = []
    for token_idx, prob in indexed:
        cumsum += prob
        selected.append((token_idx, prob))
        if cumsum >= p:
            break
    sel_probs = [pr for _, pr in selected]
    total = sum(sel_probs)
    sel_probs = [pr / total for pr in sel_probs]
    idx = sample_from_probs(sel_probs)
    return selected[idx][0]
```

### Langkah 9: Trik reparameterisasi

```python
def reparam_sample(mu, sigma):
    epsilon = random.gauss(0, 1)
    return mu + sigma * epsilon

def reparam_gradient(mu, sigma, epsilon):
    dz_dmu = 1.0
    dz_dsigma = epsilon
    return dz_dmu, dz_dsigma
```

Tunjukkan bahwa gradien mengalir melalui sampel yang direparameterisasi, tetapi tidak melalui sampling langsung.

### Langkah 10: Gumbel-Softmax

```python
def gumbel_sample():
    u = random.random()
    return -math.log(-math.log(u))

def gumbel_softmax(logits, temperature):
    gumbels = [math.log(p) + gumbel_sample() for p in logits]
    return softmax([g / temperature for g in gumbels])
```

Tunjukkan bagaimana penurunan suhu membuat keluaran mendekati vektor one-hot.

Implementasi lengkap beserta semua visualisasi ada di `code/sampling.py`.

Port TypeScript ada di `code/sampling.ts`. Port itu menggunakan pembangkit bilangan acak ber-seed sendiri, sehingga bilangan acaknya berbeda dari keluaran Python.

## Gunakanlah

Dengan NumPy dan SciPy, berikut versi produksinya:

```python
import numpy as np

rng = np.random.default_rng(42)

exponential_samples = rng.exponential(scale=2.0, size=10000)
print(f"Exponential mean: {exponential_samples.mean():.4f} (expected 2.0)")

from scipy import stats
normal = stats.norm(loc=0, scale=1)
print(f"CDF at 1.96: {normal.cdf(1.96):.4f}")
print(f"Inverse CDF at 0.975: {normal.ppf(0.975):.4f}")

logits = np.array([2.0, 1.0, 0.5, 0.1, -1.0])
temperature = 0.7
scaled = logits / temperature
probs = np.exp(scaled - scaled.max()) / np.exp(scaled - scaled.max()).sum()
token = rng.choice(len(logits), p=probs)
print(f"Sampled token index: {token}")
```

Untuk MCMC dalam skala besar, gunakan pustaka khusus:
- PyMC: pemodelan Bayes lengkap dengan NUTS (HMC adaptif)
- emcee: sampler MCMC ensemble
- NumPyro/JAX: MCMC yang dipercepat GPU

Anda membangun semuanya dari awal. Sekarang Anda tahu apa yang dilakukan panggilan pustaka itu.

## Latihan

1. Implementasikan sampling CDF invers untuk distribusi Cauchy. CDF-nya adalah $F(x) = 0,5 + \arctan(x)/\pi$. Bangkitkan 10.000 sampel dan plot histogramnya terhadap PDF yang sebenarnya. Perhatikan ekor beratnya (nilai ekstrem yang jauh dari pusat).

2. Gunakan rejection sampling untuk membangkitkan sampel dari distribusi Beta(2, 5) dengan proposal Uniform(0, 1). Plot sampel yang diterima terhadap PDF Beta yang sebenarnya. Berapa laju penerimaan teoritisnya?

3. Estimasi integral $\sin(x)$ dari 0 sampai $\pi$ menggunakan Monte Carlo dengan 1.000, 10.000, dan 100.000 sampel. Bandingkan galat pada setiap tingkat. Verifikasi bahwa galat berskala $O(1/\sqrt{N})$.

4. Implementasikan Metropolis-Hastings untuk mengambil sampel dari distribusi 2D $p(x, y)$ yang proposional terhadap $\exp(-(x^2 y^2 + x^2 + y^2 - 8x - 8y) / 2)$. Plot sampel dan lintasan rantainya. Eksperimen dengan berbagai simpangan baku proposal.

5. Bangun demo pembangkitan teks lengkap: dengan kosakata 10 kata beserta logitnya, bangkitkan urutan 20 token menggunakan (a) greedy, (b) suhu=0,7, (c) top-k=3, (d) top-p=0,9. Bandingkan keragaman keluaran dalam 5 kali percobaan.

### Petunjuk

**Latihan 1.** Selesaikan $u = F(x)$ untuk $x$, yang menghasilkan $x = \tan(\pi(u - 0,5))$. Periksa bahwa median sampel mendekati 0 dan bahwa sekitar 6% sampel memiliki $\lvert x \rvert > 10$, yaitu $1 - (2/\pi)\arctan(10)$.

**Latihan 2.** Kepadatan Beta(2, 5) adalah $30x(1-x)^4$. Temukan nilai terbesarnya pada modus $x = 1/5$ dan gunakan nilai itu sebagai $M$ dalam `rejection_sample`. Laju penerimaan teoritis adalah $1/M$, jadi bandingkan dengan fraksi proposal yang Anda terima.

**Latihan 3.** Nilai eksaknya adalah 2. Panggil `monte_carlo_integral(math.sin, 0, math.pi, n)` untuk setiap $n$ dan ulangi setiap percobaan dengan beberapa seed. Ketika $n$ bertambah 10 kali, galat rata-rata harus turun sekitar $\sqrt{10} \approx 3,2$.

**Latihan 4.** Tulis log dari target, yaitu $-(x^2 y^2 + x^2 + y^2 - 8x - 8y)/2$, dan teruskan ke `metropolis_hastings_2d`. Catat laju penerimaan untuk setiap simpangan baku proposal: nilai yang sangat kecil menerima hampir setiap langkah tetapi bergerak lambat, dan nilai yang sangat besar menolak hampir setiap langkah.

**Latihan 5.** Dekode greedy mengambil indeks logit terbesar di setiap langkah. Gunakan `temperature_sample`, `top_k_sample`, dan `top_p_sample` untuk tiga metode lainnya. Greedy harus menghasilkan urutan yang sama dalam semua 5 percobaan, jadi hitung jumlah urutan berbeda untuk setiap metode sebagai ukuran keragaman.

## Istilah Kunci

| Istilah | Yang sering dikatakan | Arti sebenarnya |
|---|---|---|
| Pengambilan sampel | "Menarik nilai acak" | Menghasilkan nilai sesuai distribusi probabilitas. Mekanisme di balik semua AI generatif |
| Distribusi seragam | "Semua sama mungkin" | Setiap nilai dalam $[a, b]$ memiliki kepadatan probabilitas $1/(b-a)$ yang sama. Titik awal semua metode pengambilan sampel |
| CDF invers | "Transformasi probabilitas" | $F^{-1}(U)$ mengubah sampel seragam menjadi sampel dari distribusi apa pun dengan CDF yang diketahui. Tepat dan efisien |
| Rejection sampling | "Usulkan lalu terima/tolak" | Bangkitkan dari proposal sederhana, terima dengan probabilitas proposional terhadap rasio target/proposal. Tepat, tetapi membuang sampel |
| Importance sampling | "Berbobot ulang sampel" | Estimasi nilai harapan di bawah $p(x)$ dengan sampel dari $q(x)$ menggunakan bobot $p(x)/q(x)$. Inti dari PPO dalam RL |
| Monte Carlo | "Rata-ratakan sampel acak" | Mendekati integral sebagai rata-rata sampel. Galat $O(1/\sqrt{N})$ terlepas dari dimensi |
| MCMC | "Jalan acak yang konvergen" | Membangun rantai Markov yang distribusi stasionernya adalah target. Metropolis-Hastings adalah algoritma dasarnya |
| Metropolis-Hastings | "Terima ke atas, kadang ke bawah" | Usulkan pergerakan, terima berdasarkan rasio kepadatan. Keseimbangan rinci menjamin konvergensi ke distribusi target |
| Gibbs sampling | "Satu peubah pada satu waktu" | Perbarui setiap peubah dari distribusi bersyaratnya sambil memegang yang lain tetap. Laju penerimaan 100% |
| Suhu | "Tombol kepercayaan" | Membagi logit dengan $T$ sebelum softmax. $T<1$ mempertajam (lebih percaya diri), $T>1$ meratakan (lebih beragam) |
| Sampling top-k | "Simpan yang k terbaik" | Nolkan semua kecuali $k$ token dengan probabilitas tertinggi, normalisasi ulang, ambil sampel. Ukuran himpunan kandidat tetap |
| Nucleus sampling (top-p) | "Simpan yang mungkin" | Simpan himpunan token terkecil yang probabilitas kumulatifnya melebihi $p$. Ukuran himpunan kandidat adaptif |
| Trik reparameterisasi | "Pindahkan keacakan ke luar" | Tulis $z = \mu + \sigma \epsilon$ dengan $\epsilon \sim \mathcal{N}(0,1)$. Membuat sampling terdiferensialkan. Penting untuk pelatihan VAE |
| Gumbel-Softmax | "Sampling kategorikal lunak" | Aproksimasi terdiferensialkan untuk sampling kategorikal dengan derau Gumbel + softmax bersuhu |
| Sampling berlapis | "Cakupan paksa" | Bagi ruang sampel menjadi lapisan, ambil sampel dari masing-masing. Selalu varians lebih rendah daripada Monte Carlo naif |
| Burn-in | "Masa pemanasan" | Sampel MCMC awal yang dibuang sebelum rantai mencapai distribusi stasionernya |
| Keseimbangan rinci | "Syarat kebalikan" | $p(x) T(x \to y) = p(y) T(y \to x)$. Syarat cukup agar $p$ menjadi distribusi stasioner suatu rantai Markov |
| Sampling difusi | "Penghilangan derau iteratif" | Bangkitkan data dengan mulai dari derau dan menerapkan langkah penghilangan derau yang dipelajari. Setiap langkah adalah operasi sampling bersyarat |

## Bacaan Lanjutan

- [Holbrook (2023): The Metropolis-Hastings Algorithm](https://arxiv.org/abs/2304.07010) - tutorial mendetail tentang dasar-dasar MCMC
- [Jang, Gu, Poole (2017): Categorical Reparameterization with Gumbel-Softmax](https://arxiv.org/abs/1611.01144) - makalah asli Gumbel-Softmax
- [Holtzman et al. (2020): The Curious Case of Neural Text Degeneration](https://arxiv.org/abs/1904.09751) - makalah nucleus (top-p) sampling
- [Kingma & Welling (2014): Auto-Encoding Variational Bayes](https://arxiv.org/abs/1312.6114) - makalah VAE yang memperkenalkan trik reparameterisasi
- [Ho, Jain, Abbeel (2020): Denoising Diffusion Probabilistic Models](https://arxiv.org/abs/2006.11239) - DDPM menghubungkan sampling dengan pembangkitan gambar
