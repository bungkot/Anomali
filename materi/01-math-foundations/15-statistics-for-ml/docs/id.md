# Statistik untuk Pembelajaran Mesin

> Statistik adalah cara Anda mengetahui apakah model Anda benar-benar bekerja atau hanya beruntung.

**Jenis:** Bangun
**Bahasa:** Python, TypeScript
**Prasyarat:** Fase 1, Pelajaran 06 (Probabilitas dan Distribusi), 07 (Teorema Bayes)
**Waktu:** ~120 menit

## Tujuan Pembelajaran

- Menghitung statistik deskriptif, korelasi Pearson/Spearman, dan matriks kovarians dari awal
- Melakukan pengujian hipotesis (uji-t, chi-kuadrat) serta menafsirkan nilai-p dan selang kepercayaan dengan benar
- Menggunakan resampling bootstrap untuk membangun selang kepercayaan bagi metrik apa pun tanpa asumsi distribusi
- Membedakan signifikansi statistik dari signifikansi praktis menggunakan ukuran efek

## Masalahnya

Anda melatih dua model. Model A memperoleh 0,87 pada himpunan uji Anda. Model B memperoleh 0,89. Anda merilis Model B. Tiga minggu kemudian, metrik produksi lebih buruk daripada sebelumnya. Apa yang terjadi?

Model B sebenarnya tidak mengungguli Model A. Selisih 0,02 itu adalah derakan. Himpunan uji Anda terlalu kecil, atau variansnya terlalu tinggi, atau keduanya. Anda merilis keacakan yang menyamar sebagai peningkatan.

Ini terjadi terus-menerus. Peringkat Kaggle yang berguncang. Makalah yang gagal direproduksi. Pengujian A/B yang menyatakan pemenang hanya dari beberapa ratus sampel. Penyebabnya selalu sama: seseorang melewati statistik.

Statistik memberi Anda alat untuk membedakan sinyal dari derakan. Statistik memberi tahu Anda kapan suatu perbedaan nyata, seberapa yakin Anda seharusnya, dan berapa banyak data yang Anda butuhkan sebelum dapat mempercayai hasil. Setiap pipeline ML, setiap perbandingan model, setiap percobaan memerlukan statistik. Tanpa statistik, Anda hanya menebak.

## Konsepnya

### Statistik Deskriptif: Merangkum Data Anda

Sebelum memodelkan apa pun, Anda perlu tahu seperti apa data Anda. Statistik deskriptif memampatkan dataset menjadi beberapa angka yang menangkap bentuknya.

**Ukuran kecenderungan pusat** menjawab "di mana titik tengahnya?"

```
Rata-rata:   jumlah semua nilai / banyaknya nilai
             mu = (1/n) * sum(x_i)

Median:      nilai tengah setelah diurutkan
             Tahan terhadap pencilan. Jika Anda punya [1, 2, 3, 4, 1000],
             rata-ratanya 202 tetapi mediannya 3.

Modus:       nilai yang paling sering muncul
             Berguna untuk data kategorikal. Untuk data kontinu, jarang informatif.
```

Rata-rata adalah titik keseimbangan. Median adalah titik tengahnya. Ketika keduanya berbeda, distribusi Anda menceng. Distribusi pendapatan memiliki $\text{rata-rata} \gg \text{median}$ (keceng kanan karena miliarder). Distribusi rugi selama pelatihan sering memiliki $\text{rata-rata} \ll \text{median}$ (keceng kiri karena sampel yang mudah).

**Ukuran penyebaran** menjawab "seberapa tersebar datanya?"

```
Varians:    rata-rata kuadrat selisih dari rata-rata
            sigma^2 = (1/n) * sum((x_i - mu)^2)

Simpangan baku: akar kuadrat varians
                sigma = sqrt(sigma^2)
                Satuan sama dengan data, jadi lebih mudah ditafsirkan.

Rentang:    maks - min
            Sensitif terhadap pencilan. Hampir tidak pernah berguna sendiri.

IQR:        Q3 - Q1 (rentang antarkuartil)
            Rentang 50% data bagian tengah.
            Tahan terhadap pencilan. Digunakan untuk box plot dan deteksi pencilan.
```

**Persentil** membagi data terurut menjadi 100 bagian sama. Persentil ke-25 (Q1) berarti 25% nilai berada di bawah titik itu. Persentil ke-50 adalah median. Persentil ke-75 adalah Q3.

```
Untuk pemantauan latensi:
  P50 = latensi median        (pengalaman pengguna pada umumnya)
  P95 = persentil ke-95       (buruk, tetapi bukan kasus terburuk)
  P99 = persentil ke-99       (latensi ekor, sering 10x median)
```

Dalam ML, Anda peduli pada persentil untuk latensi inferensi, distribusi kepercayaan prediksi, dan pemahaman distribusi galat. Model dengan galat rata-rata rendah tetapi galat P99 buruk mungkin tidak berguna untuk aplikasi kritis keselamatan.

**Statistik sampel versus populasi.** Ketika menghitung varians dari sebuah sampel, bagi dengan $(n-1)$ alih-alih $n$. Ini adalah koreksi Bessel. Koreksi itu mengompensasi fakta bahwa rata-rata sampel Anda bukan rata-rata populasi yang sebenarnya. Dengan $n$ sebagai penyebut, Anda secara sistematis meremehkan varians sejati. Dengan $(n-1)$, estimasinya tak berbias.

```math
\begin{aligned}
\text{Varians populasi:} \quad \sigma^2 &= \frac{1}{N} \sum (x_i - \mu)^2 \\
\text{Varians sampel:} \quad s^2 &= \frac{1}{n-1} \sum (x_i - \bar{x})^2
\end{aligned}
```

Dalam praktik: jika $n$ besar (ribuan sampel), selisihnya tidak berarti. Jika $n$ kecil (beberapa lusin sampel), selisihnya penting.

### Korelasi: Bagaimana Peubah Bergerak Bersama

Korelasi mengukur kekuatan dan arah hubungan linear antara dua peubah.

**Koefisien korelasi Pearson** mengukur asosiasi linear:

```
r = sum((x_i - x_bar)(y_i - y_bar)) / (n * s_x * s_y)

r = +1:  hubungan linear positif sempurna
r = -1:  hubungan linear negatif sempurna
r =  0:  tidak ada hubungan linear (tetapi mungkin ada hubungan nonlinear!)

Rentang: [-1, 1]
```

Pearson mengasumsikan hubungannya linear dan kedua peubah kira-kira berdistribusi normal. Pearson sensitif terhadap pencilan. Satu titik ekstrem dapat menarik $r$ dari 0,1 menjadi 0,9.

**Korelasi peringkat Spearman** mengukur asosiasi monotonik:

```
1. Ganti setiap nilai dengan peringkatnya (1, 2, 3, ...)
2. Hitung korelasi Pearson pada peringkat tersebut

Spearman menangkap hubungan monotonik apa pun, bukan hanya linear.
Jika y = x^3, Pearson memberi r < 1 tetapi Spearman memberi rho = 1.
```

**Kapan menggunakan masing-masing:**

```
Pearson:    Kedua peubah kontinu dan kira-kira normal.
            Anda secara khusus peduli pada hubungan linear.
            Tidak ada pencilan ekstrem.

Spearman:   Data ordinal (peringkat, penilaian).
            Data tidak berdistribusi normal.
            Anda menduga ada hubungan monotonik tetapi tidak linear.
            Ada pencilan.
```

**Aturan emas:** korelasi tidak menyiratkan kausalitas. Penjualan es krim dan kematian akibat tenggelam berkorelasi karena keduanya meningkat di musim panas. Akurasi model Anda dan jumlah parameter berkorelasi, tetapi menambah parameter tidak otomatis meningkatkan akurasi (lihat: overfitting).

### Matriks Kovarians

Kovarians antara dua peubah mengukur bagaimana keduanya berubah bersama:

```
Cov(X, Y) = (1/n) * sum((x_i - x_bar)(y_i - y_bar))

Cov(X, Y) > 0:  X dan Y cenderung meningkat bersama
Cov(X, Y) < 0:  ketika X meningkat, Y cenderung menurun
Cov(X, Y) = 0:  tidak ada pergerakan bersama linear
```

Untuk $d$ fitur, matriks kovarians $C$ adalah matriks $d \times d$ dengan $C[i][j] = \mathop{\mathrm{Cov}}(\text{fitur}_i, \text{fitur}_j)$. Entri diagonal $C[i][i]$ adalah varians dari masing-masing fitur.

```
C = | Var(x1)      Cov(x1,x2)  Cov(x1,x3) |
    | Cov(x2,x1)  Var(x2)      Cov(x2,x3) |
    | Cov(x3,x1)  Cov(x3,x2)  Var(x3)     |

Sifat-sifat:
  - Simetris: C[i][j] = C[j][i]
  - Semi-definit positif: semua nilai eigen >= 0
  - Diagonal = varians
  - Di luar diagonal = kovarians
```

**Hubungan dengan PCA.** PCA melakukan dekomposisi eigendecomposition pada matriks kovarians. Vektor eigennya adalah komponen utama (arah varians maksimum). Nilai eigennya memberi tahu Anda berapa banyak varians yang ditangkap setiap komponen. Ini persis yang dibahas Pelajaran 10, tetapi sekarang Anda melihat mengapa matriks kovarians adalah hal yang tepat untuk didekomposisi: matriks itu menyandikan semua hubungan linear berpasangan dalam data Anda.

**Hubungan dengan korelasi.** Matriks korelasi adalah matriks kovarians dari peubah yang distandardisasi (masing-masing dibagi dengan simpangan bakunya). Korelasi menormalkan kovarians sehingga semua nilai berada di $[-1, 1]$.

### Pengujian Hipotesis

Pengujian hipotesis adalah kerangka untuk mengambil keputusan dalam ketidakpastian. Anda mulai dengan sebuah klaim, mengumpulkan data, dan menentukan apakah data konsisten dengan klaim itu.

**Kerangkanya:**

```
Hipotesis nol (H0):        asumsi bawaan, biasanya "tidak ada efek"
Hipotesis alternatif (H1): apa yang ingin Anda tunjukkan

Contoh:
  H0: Model A dan Model B memiliki akurasi yang sama
  H1: Model B memiliki akurasi lebih tinggi daripada Model A
```

**Nilai-p** adalah probabilitas melihat data sektrem itu, dengan mengasumsikan H0 benar. Nilai-p **bukan** probabilitas H0 benar. Ini adalah kesalahpahaman tunggal paling umum dalam statistik.

```
nilai-p = P(data sektrem ini | H0 benar)

Jika nilai-p < alpha (biasanya 0,05):
    Tolak H0. Hasilnya "signifikan secara statistik."
Jika nilai-p >= alpha:
    Gagal menolak H0. Anda tidak punya cukup bukti.
    Ini TIDAK berarti H0 benar.
```

**Selang kepercayaan** memberi rentang nilai yang mungkin bagi sebuah parameter:

```
Selang kepercayaan 95% untuk rata-rata:
    x_bar +/- z * (s / sqrt(n))

dengan z = 1,96 untuk kepercayaan 95%

Penafsiran: jika Anda mengulang percobaan ini berkali-kali, 95% dari
selang yang dihitung akan memuat rata-rata sejati. Ini TIDAK berarti
ada peluang 95% bahwa rata-rata sejati berada di selang khusus ini.
```

Lebar selang kepercayaan memberi tahu Anda tentang presisi. Selang lebar berarti ketidakpastian tinggi. Selang sempit berarti estimasi Anda presisi (tetapi belum tentu akurat, jika data Anda bias).

### Uji-t

Uji-t membandingkan rata-rata. Ada beberapa variannya.

**Uji-t satu sampel:** apakah rata-rata populasi berbeda dari nilai yang diduga?

```math
\begin{aligned}
t &= \frac{\bar{x} - \mu_0}{s / \sqrt{n}} \\[4pt]
\text{derajat kebebasan} &= n - 1
\end{aligned}
```

**Uji-t dua sampel (independen):** apakah rata-rata dua kelompok berbeda?

```
t = (x_bar_1 - x_bar_2) / sqrt(s1^2/n1 + s2^2/n2)

Ini adalah uji-t Welch, yang tidak mengasumsikan varians sama.
Selalu gunakan Welch kecuali Anda punya alasan khusus untuk varians sama.
```

**Uji-t berpasangan:** ketika pengukuran datang berpasangan (model yang sama dievaluasi pada split data yang sama):

```
Hitung d_i = x_i - y_i untuk setiap pasangan
Lalu jalankan uji-t satu sampel pada nilai d_i terhadap mu_0 = 0
```

Dalam ML, uji-t berpasangan umum dilakukan: Anda menjalankan kedua model pada 10 lipatan validasi silang yang sama dan membandingkan skornya secara berpasangan.

### Uji Chi-Kuadrat

Uji chi-kuadrat memeriksa apakah frekuensi teramati cocok dengan frekuensi yang diharapkan. Berguna untuk data kategorikal.

```
chi^2 = sum((observed - expected)^2 / expected)

Contoh: apakah distribusi keluaran model bahasa cocok dengan
distribusi pelatihan di berbagai kategori?

Kategori    Teramati   Diharapkan
Positif       120        100
Negatif        80        100
chi^2 = (120-100)^2/100 + (80-100)^2/100 = 4 + 4 = 8

Dengan 1 derajat kebebasan, chi^2 = 8 memberi p < 0,005.
Perbedaan itu signifikan.
```

### Pengujian A/B untuk Model ML

Pengujian A/B dalam ML tidak sama dengan pengujian A/B di web. Perbandingan model memiliki tantangan khusus:

```
1. Himpunan uji yang sama:    Kedua model harus dievaluasi pada data yang identik.
                              Himpunan uji berbeda membuat perbandingan tidak bermakna.

2. Banyak metrik: Akurasi saja tidak cukup. Anda memerlukan presisi,
                  recall, F1, latensi, dan metrik keadilan.

3. Varians:           Gunakan validasi silang atau bootstrap untuk
                      mengestimasi varians setiap metrik, bukan sekadar estimasi titik.

4. Kebocoran data:    Jika himpunan uji dipakai saat pemilihan model,
                      perbandingan Anda bias. Sisihkan himpunan uji akhir.
```

**Prosedurnya:**

```
1. Tentukan metrik dan tingkat signifikansi Anda (alpha = 0,05)
2. Jalankan kedua model pada split validasi silang k-lipat yang sama
3. Kumpulkan skor berpasangan: [(a1, b1), (a2, b2), ..., (ak, bk)]
4. Hitung selisih: d_i = b_i - a_i
5. Jalankan uji-t berpasangan pada selisih tersebut
6. Periksa: apakah rata-rata selisih signifikan berbeda dari 0?
7. Hitung selang kepercayaan untuk rata-rata selisih
8. Hitung ukuran efek (Cohen's d) untuk menilai signifikansi praktis
```

### Signifikansi Statistik versus Signifikansi Praktis

Suatu hasil dapat signifikan secara statistik tetapi tidak bermakna secara praktis. Dengan cukup banyak data, bahkan perbedaan kecil menjadi signifikan secara statistik.

```
Contoh:
  Akurasi Model A: 0,9234
  Akurasi Model B: 0,9237
  n = 1.000.000 sampel uji
  nilai-p = 0,001

Signifikan secara statistik? Ya.
Signifikan secara praktis? Peningkatan 0,03% tidak sepadan dengan
biaya rekayasa untuk merilis model baru.
```

**Ukuran efek** mengkuantifikasi seberapa besar perbedaan itu, terlepas dari ukuran sampel:

```math
\begin{aligned}
\text{Cohen's } d &= \frac{\text{rata-rata}_1 - \text{rata-rata}_2}{\text{standar\_gabungan}} \\[4pt]
d &= 0.2: \quad \text{efek kecil} \\
d &= 0.5: \quad \text{efek sedang} \\
d &= 0.8: \quad \text{efek besar}
\end{aligned}
```

Selalu laporkan nilai-p dan ukuran efek. Nilai-p memberi tahu Anda apakah perbedaan itu nyata. Ukuran efek memberi tahu Anda apakah perbedaan itu penting.

### Masalah Perbandingan Berganda

Ketika Anda menguji banyak hipotesis, sebagian akan "signifikan" karena kebetulan. Jika Anda menguji 20 hal pada $\alpha = 0,05$, Anda mengharapkan 1 positif palsu bahkan ketika tidak ada yang benar-benar berbeda.

```
P(setidaknya satu positif palsu) = 1 - (1 - alpha)^m

m = 20 uji, alpha = 0,05:
P(positif palsu) = 1 - 0,95^20 = 0,64

Anda memiliki peluang 64% untuk mendapat setidaknya satu positif palsu.
```

**Koreksi Bonferroni:** bagi $\alpha$ dengan jumlah uji.

```
Alpha yang disesuaikan = alpha / m = 0,05 / 20 = 0,0025

Hanya tolak H0 jika nilai-p < 0,0025.
Konservatif tetapi sederhana. Berfungsi ketika uji-uji saling merdeka.
```

Dalam ML, ini penting ketika Anda membandingkan sebuah model pada banyak metrik, menguji banyak konfigurasi hyperparameter, atau mengevaluasi pada banyak dataset.

### Metode Bootstrap

Bootstrap mengestimasi distribusi sampling suatu statistik dengan mengambil sampel ulang data Anda dengan penggantian. Tidak diperlukan asumsi tentang distribusi yang mendasarinya.

**Algoritmanya:**

```
1. Anda memiliki n titik data
2. Ambil n sampel DENGAN penggantian (beberapa titik muncul berkali-kali,
   ada yang sama sekali tidak muncul)
3. Hitung statistik Anda pada sampel bootstrap ini
4. Ulangi B kali (biasanya B = 1000 sampai 10000)
5. Distribusi statistik bootstrap mendekati
   distribusi sampel
```

**Selang kepercayaan bootstrap (metode persentil):**

```
Urutkan B statistik bootstrap
KI 95% = [persentil ke-2,5, persentil ke-97,5]
```

**Mengapa bootstrap penting bagi ML:**

```
- Akurasi himpunan uji adalah estimasi titik. Bootstrap memberi Anda
  selang kepercayaan.
- Anda tidak dapat mengasumsikan distribusi metrik berbentuk normal
  (terutama untuk AUC, F1, presisi at k).
- Bootstrap berfungsi untuk STATISTIK APA PUN: median, rasio dua rata-rata,
  selisih AUC antara dua model.
- Tidak perlu rumus bentuk tertutup.
```

**Bootstrap untuk perbandingan model:**

```
1. Anda memiliki prediksi dari Model A dan Model B pada himpunan uji yang sama
2. Untuk setiap iterasi bootstrap:
   a. Ambil sampel ulang indeks uji dengan penggantian
   b. Hitung metrik_A dan metrik_B pada himpunan yang diambil ulang
   c. Simpan diff = metrik_B - metrik_A
3. KI 95% untuk selisih:
   [persentil ke-2,5 dari diff, persentil ke-97,5 dari diff]
4. Jika KI tidak memuat 0, selisihnya signifikan
```

Ini lebih tangguh daripada uji-t berpasangan karena tidak membuat asumsi distribusi apa pun.

### Uji Parametrik versus Non-parametrik

**Uji parametrik** mengasumsikan distribusi tertentu (biasanya normal):

```
Uji-t:         mengasumsikan data berdistribusi normal (atau n besar by CLT)
ANOVA:         mengasumsikan normalitas dan varians sama
Pearson r:     mengasumsikan normalitas bivariat
```

**Uji non-parametrik** tidak membuat asumsi distribusi:

```
Mann-Whitney U:     membandingkan dua kelompok (menggantikan uji-t independen)
Wilcoxon signed-rank: membandingkan data berpasangan (menggantikan uji-t berpasangan)
Spearman rho:       korelasi pada peringkat (menggantikan Pearson)
Kruskal-Wallis:     membandingkan banyak kelompok (menggantikan ANOVA)
```

**Kapan menggunakan non-parametrik:**

```
- Ukuran sampel kecil (n < 30) dan data jelas tidak normal
- Data ordinal (penilaian, peringkat)
- Pencilan berat yang tidak bisa Anda buang
- Distribusi menceng
```

**Kapan menggunakan parametrik:**

```
- Ukuran sampel besar (CLT membuat statistik uji kira-kira normal)
- Data kira-kira simetris tanpa pencilan ekstrem
- Daya statistik lebih besar (lebih baik mendeteksi perbedaan nyata)
```

Dalam percobaan ML, Anda biasanya memiliki $n$ kecil (5 atau 10 lipatan validasi silang), sehingga uji non-parametrik seperti Wilcoxon signed-rank sering lebih tepat daripada uji-t.

### Teorema Limit Pusat: Implikasi Praktis

CLT menyatakan bahwa distribusi rata-rata sampel mendekati distribusi normal ketika $n$ bertambah, apa pun distribusi populasi yang mendasarinya.

```
Jika X_1, X_2, ..., X_n saling independen dan berdistribusi sama dengan rata-rata mu dan varians sigma^2:

    X_bar ~ Normal(mu, sigma^2 / n)    saat n -> tak hingga

Berfungsi untuk n >= 30 dalam kebanyakan kasus.
Untuk distribusi sangat menceng, Anda mungkin memerlukan n >= 100.
```

**Mengapa ini penting bagi ML:**

```
1. Membenarkan selang kepercayaan dan uji-t pada metrik agregat
2. Menjelaskan mengapa perataan atas lipatan validasi silang memberikan
   estimasi stabil meskipun tiap lipatan sangat berfluktuasi
3. Penurunan gradien mini-batch bekerja karena rata-rata gradien
   atas satu batch mendekati gradien sejati (CLT beraksi)
4. Metode ensemble: merata-ratakan prediksi dari banyak model memberi
   keluaran lebih stabil daripada model tunggal mana pun
```

**Apa yang TIDAK dilakukan CLT:**

```
- TIDAK membuat data Anda normal. CLT membuat RATA-RATA sampel yang normal.
- TIDAK berfungsi untuk distribusi berekor berat dengan varians tak hingga
  (distribusi Cauchy).
- TIDAK berlaku untuk data yang saling bergantung (deret waktu tanpa koreksi).
```

### Kesalahan Statistik Umum dalam Makalah ML

1. **Menguji pada himpunan pelatihan.** Menjamin overfitting. Selalu sisihkan data yang tidak pernah dilihat model saat pelatihan.

2. **Tanpa selang kepercayaan.** Melaporkan satu angka akurasi tanpa ketidakpastian membuat hasil tidak dapat direproduksi atau diverifikasi.

3. **Mengabaikan perbandingan berganda.** Menguji 50 konfigurasi dan melaporkan yang terbaik tanpa koreksi meningkatkan laju positif palsu.

4. **Mencampur signifikansi statistik dan praktis.** Nilai-p sebesar 0,001 pada peningkatan akurasi 0,01% tidak bermakna.

5. **Menggunakan akurasi pada data tidak seimbang.** Akurasi 99% pada dataset dengan 99% kelas negatif berarti model tidak mempelajari apa pun. Gunakan presisi, recall, F1, atau AUC.

6. **Memilih metrik secara sepihak.** Hanya melaporkan metrik tempat model Anda menang. Evaluasi yang jujur melaporkan semua metrik yang relevan.

7. **Membocorkan informasi antar-split latih/uji.** Menormalisasi sebelum membagi, atau menggunakan data masa depan untuk memprediksi masa lalu.

8. **Himpunan uji kecil tanpa estimasi varians.** Evaluasi pada 100 sampel dan mengklaim peningkatan 2% adalah derakan, bukan sinyal.

9. **Mengasumsikan kemerdekaan padahal datanya tidak merdeka.** Citra medis dari pasien yang sama, beberapa kalimat dari dokumen yang sama. Pengamatan dalam satu kelompok saling berkorelasi.

10. **P-hacking.** Mencoba uji, subset, atau kriteria pengecualian berbeda sampai Anda mendapat $p < 0,05$. Hasilnya adalah artefak dari proses pencarian itu.

## Membangunnya

Anda akan mengimplementasikan:

1. **Statistik deskriptif dari awal** (rata-rata, median, modus, simpangan baku, persentil, IQR)
2. **Fungsi korelasi** (Pearson dan Spearman, beserta matriks kovarians)
3. **Pengujian hipotesis** (uji-t satu sampel, uji-t dua sampel, uji chi-kuadrat)
4. **Selang kepercayaan bootstrap** (untuk statistik apa pun, tanpa asumsi)
5. **Simulator pengujian A/B** (membangkitkan data, menguji, dan memeriksa galat tipe I dan tipe II)
6. **Demonstrasi signifikansi statistik versus praktis** (menunjukkan bahwa $n$ besar membuat semuanya "signifikan")

Semua dari awal, hanya menggunakan `math` dan `random`. Tanpa numpy, tanpa scipy.

```figure
f3-bootstrap-resample
```

## Istilah Kunci

| Istilah | Definisi |
|---|---|
| Rata-rata | Jumlah nilai dibagi banyaknya nilai. Sensitif terhadap pencilan. |
| Median | Nilai tengah data yang diurutkan. Tahan terhadap pencilan. |
| Simpangan baku | Akar kuadrat varians. Mengukur penyebaran dalam satuan asli. |
| Persentil | Nilai di bawah mana persentase data tertentu berada. |
| IQR | Rentang antarkuartil. Q3 dikurangi Q1. Penyebaran 50% bagian tengah. |
| Korelasi Pearson | Mengukur asosiasi linear antara dua peubah. Rentang $[-1, 1]$. |
| Korelasi Spearman | Mengukur asosiasi monotonik menggunakan peringkat. |
| Matriks kovarians | Matriks kovarians berpasangan antarsemua fitur. |
| Hipotesis nol | Asumsi bawaan berupa tidak ada efek atau tidak ada perbedaan. |
| Nilai-p | Probabilitas data sektrem ini jika hipotesis nol benar. |
| Selang kepercayaan | Rentang nilai yang mungkin bagi parameter pada tingkat kepercayaan tertentu. |
| Uji-t | Menguji apakah rata-rata berbeda secara signifikan. Menggunakan distribusi-t. |
| Uji chi-kuadrat | Menguji apakah frekuensi teramati berbeda dari frekuensi yang diharapkan. |
| Ukuran efek | Besarnya perbedaan, terlepas dari ukuran sampel. Cohen's d umum digunakan. |
| Koreksi Bonferroni | Membagi ambang signifikansi dengan jumlah uji untuk mengendalikan positif palsu. |
| Bootstrap | Pengambilan sampel ulang dengan penggantian untuk mengestimasi distribusi sampling. |
| Galat tipe I | Positif palsu. Menolak H0 ketika H0 benar. |
| Galat tipe II | Negatif palsu. Gagal menolak H0 ketika H0 salah. |
| Daya statistik | Probabilitas menolak H0 yang salah dengan tepat. Daya = 1 dikurangi laju galat tipe II. |
| Teorema limit pusat | Rata-rata sampel konvergen ke distribusi normal seiring pertumbuhan ukuran sampel. |
| Uji parametrik | Mengasumsikan distribusi tertentu untuk data (biasanya normal). |
| Uji non-parametrik | Tidak membuat asumsi distribusi. Bekerja pada peringkat atau tanda. |
