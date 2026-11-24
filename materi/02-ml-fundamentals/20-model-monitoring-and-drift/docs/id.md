# Memantau Model di Produksi — Pergeseran Data dan Pergeseran Konsep

> Model yang dipasang gagal tanpa pesan galat. Model itu terus mengembalikan skor sementara dunia menjauh dari data latihannya, dan hanya pemantauan yang menunjukkan jurang itu.

**Jenis:** Build
**Bahasa:** Python, TypeScript
**Prasyarat:** Fase 2, Pelajaran 09 (Evaluasi Model), 16 (Deteksi Anomali), dan 19 (Desain Sistem ML)
**Waktu:** ~110 menit

## Tujuan Pembelajaran

- Pilih sinyal pemantauan pada empat lapisan: kesehatan sistem, masukan, prediksi, dan hasil
- Bedakan pergeseran kovariat, pergeseran prior, dan pergeseran konsep berdasarkan bagian dari distribusi bersama yang berubah
- Terapkan Population Stability Index, statistik Kolmogorov-Smirnov dua sampel, dan uji chi-kuadrat dari awal
- Bandingkan jendela referensi tetap dengan jendela terkini yang bergeser, dan jelaskan bagaimana ukuran jendela mengubah derau serta keterlambatan
- Rancang kebijakan alarm yang membatasi alarm palsu dengan ukuran efek, koreksi pengujian ganda, dan aturan kegigihan
- Putuskan jenis pergeseran mana yang memerlukan pelatihan ulang, mana yang memerlukan label baru, dan mana yang tidak memerlukan tindakan

## Masalahnya

Sebuah model risiko pembayaran diluncurkan dengan akurasi 95 persen. Dua bulan kemudian, sebuah kampanye pemasaran membawa pengguna aplikasi dengan keranjang lebih besar. Empat bulan kemudian, pelaku penipuan mengubah taktik mereka, dan pembayaran yang dulu aman mulai berubah menjadi chargeback.

Sepanjang itu, layanan model mengembalikan HTTP 200 untuk setiap permintaan. Latensinya normal, dan log tidak menunjukkan galat. Tanda pertama masalah adalah laporan keuangan tentang chargeback, berminggu-minggu setelah kerugian terjadi.

Pemantauan perangkat lunak memeriksa apakah sebuah layanan menjawab. Pemantauan model memeriksa apakah jawabannya masih benar, dan apakah datanya masih tampak seperti data yang melatih model. Pelajaran ini membangun statistik, jendela, dan kebijakan alarm untuk pekerjaan kedua itu. Anda menjalankannya pada aliran sintetis dengan dua perubahan yang disuntikkan dan melihat monitor mana yang bereaksi terhadap perubahan mana.

## Konsepnya

### Apa yang dipantau

Pantau empat lapisan. Tiga pertama memberikan peringatan dini. Hanya yang keempat yang mengukur kebenaran, dan ia tiba terlambat.

| Lapisan | Sinyal | Tersedia | Pertanyaan yang dijawab |
|---|---|---|---|
| Kesehatan sistem | Latensi p50 dan p99, tingkat galat, throughput, memori | Seketika | Apakah layanan menjawab? |
| Masukan | Distribusi setiap fitur, tingkat nilai hilang, skema, kategori baru | Seketika | Apakah data langsung tampak seperti data latihan? |
| Prediksi | Distribusi skor, tingkat positif, bagian jawaban default | Seketika | Apakah model berperilaku seperti sebelumnya? |
| Hasil | Akurasi, presisi, kalibrasi, metrik bisnis | Setelah keterlambatan label | Apakah model masih benar? |

Pemantauan masukan adalah pokok bahasan pelajaran ini, karena ia bekerja sebelum label tiba. Ia memiliki batas yang ditunjukkan demo: beberapa perubahan tidak menyentuh masukan sama sekali.

### Tiga jenis pergeseran

Tulis distribusi bersama dari masukan `x` dan label `y` dengan dua cara:

```math
P(x, y) = P(x) \cdot P(y \mid x) = P(y) \cdot P(x \mid y)
```

Setiap jenis pergeseran mengubah faktor yang berbeda:

| Pergeseran | Apa yang berubah | Apa yang tetap | Contoh pembayaran | Monitor mana yang melihatnya |
|---|---|---|---|---|
| Pergeseran kovariat | `P(x)` | `P(y \| x)` | Lebih banyak pengguna aplikasi dengan keranjang lebih besar | Monitor masukan |
| Pergeseran prior | `P(y)` | `P(x \| y)` | Tingkat penipuan berlipat saat serangan, dan penipuan tampak sama | Monitor prediksi dan hasil |
| Pergeseran konsep | `P(y \| x)` | Dapat tetap | Keranjang yang sama kini memiliki risiko penipuan berbeda | Hanya monitor hasil |

Gama dkk. menyebut perubahan pada `P(y | x)` sebagai pergeseran konsep nyata. Mereka menyebut perubahan pada `P(x)` yang membiarkan `P(y | x)` saja sebagai pergeseran virtual. Tinjauan mereka juga mengelompokkan pergeseran berdasarkan cara terjadinya dari waktu ke waktu: mendadak, inkremental, bertahap, atau sebagai konsep yang muncul kembali.

Pergeseran kovariat tidak selalu merugikan. Jika model mempelajari `P(y | x)` dengan baik, model tetap benar untuk setiap masukan. Model tetap dapat kehilangan akurasi ketika lalu lintas bergerak ke wilayah dengan sedikit data latihan atau dekat batas keputusan.

### Label yang terlambat

Hasil tiba setelah prediksi. Keterlambatannya bergantung pada masalah:

| Masalah | Kapan label tiba |
|---|---|
| Klik iklan | Beberapa detik |
| Churn dalam 30 hari ke depan | 30 hari |
| Penipuan kartu | Saat chargeback tiba, sering berminggu-minggu kemudian |
| Gagal bayar pinjaman | Berbulan-bulan atau bertahun-tahun |

Selama masa tunggu itu, Anda memiliki masukan dan prediksi tetapi tidak memiliki hasil. Pantau masukan dan prediksi, gabungkan label saat tiba, dan hitung metrik hasil hanya pada bagian yang berlabel. Demo memakai keterlambatan label sebesar 500 peristiwa, sehingga monitor hasil selalu melihat 500 peristiwa ke masa lalu.

### Jendela referensi dan jendela terkini

Setiap uji pergeseran membandingkan dua sampel:

- **Jendela referensi.** Data latihan, atau periode produksi yang telah Anda periksa manual. Jendela ini tetap tidak berubah sampai Anda melatih ulang.
- **Jendela terkini.** Peristiwa paling baru, entah `N` peristiwa terakhir atau `T` jam terakhir.

Jendela geser bergerak maju dengan langkah yang lebih kecil dari ukurannya, sehingga jendela berturut-turut tumpang tindih. Jendela loncat bergerak sepenuh ukurannya. Ukuran jendela adalah trade-off. Jendela kecil bereaksi cepat tetapi menilai derau sebagai pergeseran. Jendela besar stabil tetapi melaporkan perubahan terlambat.

Musiman merusak jendela naif. Lalu lintas Senin berbeda dengan lalu lintas Sabtu pada banyak produk. Buatlah referensi mencakup satu siklus penuh, atau bandingkan setiap jendela dengan periode yang sama pada siklus sebelumnya.

### Population Stability Index

PSI berasal dari penilaian kredit. Potong nilai referensi ke dalam bin, biasanya sepuluh bin pada desil referensi, sehingga setiap bin memuat 10 persen dari referensi. Hitung bagian nilai terkini di setiap bin. Lalu jumlahkan satu suku per bin:

```math
\text{PSI} = \sum_{\text{bins}} (\text{current\_share} - \text{reference\_share}) \cdot \ln\left(\frac{\text{current\_share}}{\text{reference\_share}}\right)
```

Bin dengan bagian nol membuat logaritma menjadi tak hingga, jadi klip setiap bagian ke nilai kecil seperti 0,0001. PSI bersifat simetris. Ia sama dengan divergensi KL dari terkini ke referensi ditambah divergensi KL dari referensi ke terkini.

Praktisi membaca PSI dengan pita tetap: di bawah 0,1 stabil, 0,1 sampai 0,25 adalah pergeseran sedang, dan di atas 0,25 adalah pergeseran signifikan. Perlakukan pita-pita ini sebagai konvensi, bukan uji statistik. PSI bertambah dengan jumlah bin dan menyusut dengan ukuran sampel. Dalam demo, jendela 500 peristiwa tanpa pergeseran mencetak antara 0,01 dan 0,04.

### Statistik Kolmogorov-Smirnov

Statistik KS dua sampel `D` adalah jurang vertikal terbesar antara dua fungsi distribusi kumulatif empiris:

```math
D = \max_x \left\lvert F_{\text{reference}}(x) - F_{\text{current}}(x) \right\rvert
```

`D` bernilai 0 untuk sampel yang identik dan 1 untuk sampel yang tidak tumpang tindih. Ia tidak memerlukan bin. Nilai-p berasal dari distribusi Kolmogorov dengan ukuran sampel efektif `n * m / (n + m)`.

Pada sampel besar, nilai-p menuju nol untuk jurang yang tidak peduli siapa pun. Pergeseran 0,02 simpangan baku adalah "signifikan" dengan sejuta peristiwa per jendela. Pakai `D` sendiri sebagai ukuran efek, dan beri alarm hanya ketika `D` dan nilai-p sama-sama melewati ambang.

### Uji chi-kuadrat untuk fitur kategorikal

Untuk fitur kategorikal, seperti kanal pembayaran, bangun tabel dengan dua baris (referensi, terkini) dan satu kolom per kategori. Jumlah yang diharapkan pada setiap sel berasal dari total baris dan kolom:

```math
\begin{aligned}
\text{expected}[r][c] &= \text{row\_total}[r] \cdot \text{column\_total}[c] / \text{grand\_total} \\
\text{chi2} &= \sum_{\text{cells}} (\text{observed} - \text{expected})^2 / \text{expected}
\end{aligned}
```

Uji ini memiliki `k - 1` derajat kebebasan untuk `k` kategori. Sebuah kategori yang hanya muncul di jendela terkini menambah suku besar, sehingga uji bereaksi terhadap nilai baru. V Cramér adalah ukuran efeknya. Untuk tabel dengan dua baris, nilainya sama dengan akar kuadrat dari `chi2 / grand_total`.

### Ambang alarm dan alarm palsu

Setiap uji memiliki tingkat alarm palsu `alpha` ketika tidak ada yang berubah. Pemantauan menjalankan banyak uji. Dengan 50 fitur yang diperiksa setiap jam pada `alpha = 0,05`, Anda mengharapkan sekitar 60 alarm palsu per hari ketika tidak ada yang berubah. Orang berhenti membaca alarm jauh sebelum itu.

Empat aturan menjaga laju alarm tetap berguna:

1. **Koreksi pengujian ganda.** Koreksi Bonferroni membagi `alpha` dengan jumlah uji. Demo memakai `alpha` keluarga sebesar 0,01 untuk dua uji, sehingga setiap uji memakai 0,005.
2. **Minta ukuran efek.** Beri alarm hanya ketika PSI, `D`, atau V Cramér juga melewati ambang.
3. **Minta kegigihan.** Beri alarm hanya setelah `k` jendela pergeseran berturut-turut. Demo memakai `k = 2`.
4. **Pisahkan peringatan dari alarm.** Peringatan masuk ke dasbor. Alarm masuk ke orang.

Demo menunjukkan aturan 2 beraksi. Satu jendela bersih memiliki nilai-p chi-kuadrat sebesar 0,0009, di bawah `alpha` per uji. V Cramér-nya di bawah 0,1, sehingga kebijakan mencatat peringatan, bukan alarm.

### Pemicu pelatihan ulang

Ada tiga pemicu umum untuk pelatihan ulang:

- **Jadwal.** Latih ulang setiap minggu atau bulan. Ini sederhana, tetapi melakukan pelatihan ulang saat tidak ada yang berubah dan menunggu saat ada perubahan.
- **Pergeseran.** Latih ulang ketika pergeseran masukan pada fitur penting tetap berada di atas ambang.
- **Kinerja.** Latih ulang ketika metrik hasil turun di bawah nilai referensi dikurangi toleransi.

Jenis pergeseran menentukan apa yang dapat dilakukan pelatihan ulang:

| Pergeseran | Tindakan yang berguna |
|---|---|
| Pergeseran kovariat, akurasi stabil | Sering tidak ada. Terus awasi metrik hasil. |
| Pergeseran kovariat ke wilayah dengan sedikit data | Latih ulang pada data terbaru yang mencakup wilayah itu |
| Pergeseran prior | Kalibrasi ulang skor atau geser ambang keputusan |
| Pergeseran konsep | Kumpulkan label baru terlebih dahulu. Pelatihan ulang pada label lama mengajarkan konsep lama lagi. |

Setelah pelatihan ulang, uji model baru pada data berlabel paling terbaru. Pasang model itu dengan tahap shadow dan canary dari pelajaran desain sistem. Lalu atur ulang jendela referensi ke data latihan yang baru.

```figure
mlprod-drift-psi
```

## Bangunlah

Kode dalam `code/main.py` hanya menggunakan pustaka standar.

### Langkah 1: Bin kuantil dan PSI

Tepi bin adalah desil referensi. `bisect_right` menemukan bin dari setiap nilai.

```python
def quantile_edges(reference: list[float], bins: int = 10) -> list[float]:
    ordered = sorted(reference)
    n = len(ordered)
    edges = [ordered[min(n - 1, (n * i) // bins)] for i in range(1, bins)]
    return sorted(set(edges))


def psi_from_fractions(expected: list[float], actual: list[float], eps: float = 1e-4) -> float:
    total = 0.0
    for e, a in zip(expected, actual):
        e, a = max(e, eps), max(a, eps)
        total += (a - e) * math.log(a / e)
    return total
```

### Langkah 2: Statistik KS dan nilai-p-nya

Jalani kedua sampel yang telah diurutkan bersama. Pada setiap nilai yang berbeda, lewati semua salinan nilainya di kedua sampel, lalu bandingkan dua bagian kumulatif. Ini menangani ikatan dengan benar.

```python
def ks_statistic(sample_a: list[float], sample_b: list[float]) -> float:
    a, b = sorted(sample_a), sorted(sample_b)
    n, m = len(a), len(b)
    i = j = 0
    d = 0.0
    while i < n and j < m:
        x = min(a[i], b[j])
        while i < n and a[i] <= x:
            i += 1
        while j < m and b[j] <= x:
            j += 1
        d = max(d, abs(i / n - j / m))
    return d
```

Nilai-p memakai deret untuk distribusi Kolmogorov, `2 * sum((-1)^(k-1) * exp(-2 k^2 lambda^2))`, dengan `lambda = sqrt(n * m / (n + m)) * D`.

### Langkah 3: Uji chi-kuadrat

`chi2_sf` menghitung fungsi survivors chi-kuadrat melalui fungsi gamma taklengkap atas yang teratur. Ia memakai deret untuk argumen kecil dan pecahan berlanjut untuk argumen besar. `chi_square_test` membangun tabel dua baris, menghitung statistik, dan mengembalikan nilai-p serta V Cramér.

```python
def chi_square_test(reference_counts: dict[str, int], current_counts: dict[str, int]) -> ChiSquareResult:
    categories = [c for c in sorted(set(reference_counts) | set(current_counts))
                  if reference_counts.get(c, 0) + current_counts.get(c, 0) > 0]
    rows = [[reference_counts.get(c, 0) for c in categories], [current_counts.get(c, 0) for c in categories]]
    row_totals = [sum(r) for r in rows]
    grand = sum(row_totals)
    if len(categories) < 2 or 0 in row_totals:
        return ChiSquareResult(0.0, 0, 1.0, 0.0)
    statistic = 0.0
    for col, _ in enumerate(categories):
        col_total = rows[0][col] + rows[1][col]
        for r in range(2):
            expected = row_totals[r] * col_total / grand
            statistic += (rows[r][col] - expected) ** 2 / expected
    dof = len(categories) - 1
    return ChiSquareResult(statistic, dof, chi2_sf(statistic, dof), math.sqrt(statistic / grand))
```

### Langkah 4: Jendela dan monitor pergeseran

`SlidingWindow` menyimpan `size` peristiwa terakhir dalam `deque`. `DriftMonitor` menghitung bin dan hitungan referensi sekali. Metode `check`-nya mengembalikan PSI, `D`, dan nilai-p KS untuk setiap fitur numerik, serta hasil chi-kuadrat untuk setiap fitur kategorikal.

### Langkah 5: Kebijakan alarm

Kebijakan mengubah setiap rangkaian metrik menjadi level dan menghitung jendela pergeseran berturut-turut per sinyal:

```python
def level(self, metrics: dict[str, float]) -> str:
    if "psi" in metrics:
        if metrics["psi"] >= self.psi_drift or (metrics["ks"] >= self.ks_drift and metrics["ks_p"] < self.alpha):
            return "drift"
        return "warn" if metrics["psi"] >= self.psi_warn else "ok"
    if metrics["p"] < self.alpha and metrics["v"] >= self.min_cramers_v:
        return "drift"
    return "warn" if metrics["p"] < self.alpha else "ok"
```

`update` mengembalikan `ALERT` pada jendela yang melengkapi rentetan, dan `resolved` pada jendela bersih pertama setelah alarm.

### Langkah 6: Aliran dengan pergeseran yang disuntikkan

Aliran ini memiliki 9000 peristiwa pembayaran dengan `amount`, `channel`, dan `label` penipuan. Empat segmen menentukannya:

| Peristiwa | Amount | Campuran kanal (app, store, web) | Batas label | Perubahan |
|---|---|---|---|---|
| 0 sampai 2999 | rata-rata 100, sd 20 | 35, 15, 50 persen | 120 | Periode referensi |
| 3000 sampai 4999 | rata-rata 118, sd 24 | 60, 10, 30 persen | 120 | Pergeseran kovariat |
| 5000 sampai 6499 | rata-rata 100, sd 20 | 35, 15, 50 persen | 120 | Kembali normal |
| 6500 sampai 8999 | rata-rata 100, sd 20 | 35, 15, 50 persen | 105 | Pergeseran konsep |

Model adalah ambang pada `amount`, yang dicocokkan pada 2000 peristiwa pertama. Jendela memuat 500 peristiwa dan bergerak 250. Label tiba 500 peristiwa terlambat.

```bash
python3 code/main.py
npx tsx code/main.ts
```

```console
  end    PSI   KS D   chi2 p    acc  levels (amount/channel/accuracy)  events
  3000  0.010  0.041  3.0e-01  0.932  ok/ok/ok
  3250  0.152  0.163  7.6e-06  0.924  drift/warn/ok
  3500  0.463  0.298  2.8e-26  0.936  drift/drift/ok                     ALERT amount
  3750  0.609  0.343  3.5e-35  0.942  drift/drift/ok                     ALERT channel
  ...
  5500  0.026  0.044  2.6e-01  0.898  ok/ok/ok                           resolved amount, resolved channel
  ...
  7250  0.040  0.048  7.1e-01  0.848  ok/ok/drift
  7500  0.022  0.039  6.1e-01  0.778  ok/ok/drift                        ALERT accuracy
```

Baca hasilnya dalam tiga bagian:

- **Pergeseran kovariat.** Pergeseran dimulai pada peristiwa 3000, dan alarm `amount` mengikuti pada 3500. Akurasi tetap antara 0,89 dan 0,95, karena aturan label tidak berubah.
- **Pemulihan.** Masukan kembali normal pada 5000, dan kedua alarm selesai pada 5500.
- **Pergeseran konsep.** Pergeseran dimulai pada 6500, dan monitor masukan tetap diam karena masukan tidak berubah. Akurasi turun dari 0,946 menjadi antara 0,76 dan 0,79. Alarm akurasi muncul pada 7500, 1000 peristiwa setelah perubahan, dan keterlambatan label menyumbang 500 di antaranya.

## Gunakanlah

**scipy.** `scipy.stats` menerapkan kedua uji itu. Statistiknya cocok dengan kode dari awal. scipy tidak diperlukan untuk menjalankan pelajaran ini.

```python
from scipy import stats

result = stats.ks_2samp(reference_amounts, current_amounts)
print(result.statistic, result.pvalue)

table = [[1000, 700, 300], [140, 290, 70]]
chi2 = stats.chi2_contingency(table)
print(chi2.statistic, chi2.pvalue, chi2.dof)
```

Kita memeriksa `ks_statistic` dan `chi_square_test` terhadap scipy 1.18.1 pada sampel acak. Statistik KS, statistik chi-kuadrat, dan nilai-p chi-kuadrat cocok sampai presisi titik mengambang. Pada 200 pasangan sampel acak, nilai-p KS berbeda dari scipy hingga 18 persen dari nilai scipy. Jurang terbesar terjadi pada nilai-p yang sangat kecil.

Jurang itu berasal dari metodenya. Kode kita memakai distribusi Kolmogorov limit. Secara default, `ks_2samp` memakai metode eksak ketika kedua sampel berisi paling banyak 10.000 nilai. Mode asimtotiknya memakai distribusi sampel terbatas dengan ukuran sampel efektif yang dibulatkan. Dengan dua kategori, `chi2_contingency` menerapkan koreksi kontinuitas Yates secara default, sehingga statistiknya berbeda di situ.

**Pustaka pemantauan.** Beberapa pustaka sumber terbuka membungkus uji-uji ini dengan laporan dan dasbor:

- **Evidently** memilih uji pergeseran per kolom. Untuk referensi dengan 1000 baris atau lebih sedikit, ia memakai uji KS untuk kolom numerik dan uji chi-kuadrat untuk kolom kategorikal. Untuk data lebih besar, ia memakai jarak Wasserstein dan divergensi Jensen-Shannon dengan ambang 0,1. PSI adalah salah satu metode lain yang dapat Anda pilih.
- **NannyML** mengestimasi kinerja model sebelum label tiba. Confidence-based Performance Estimation (CBPE)-nya memakai probabilitas prediksi model, yang harus terkalibrasi dengan baik. Dokumentasi NannyML menyatakan bahwa CBPE menangani pergeseran kovariat tetapi tidak mengatasi pergeseran konsep. Aliran demo menunjukkan alasannya: hanya label yang mengungkap pergeseran konsep itu.
- **TensorFlow Data Validation** membandingkan span data berturut-turut. Ia memakai jarak L-infinity untuk fitur kategorikal dan divergensi Jensen-Shannon pendekatan untuk fitur numerik. Ia juga membandingkan data latihan dengan data penyajian, yang dibahas dalam pelajaran feature store.

Kode dari awal menunjukkan apa yang dihitung setiap alat ini. Alat-alat itu menambahkan penyimpanan, penjadwalan, dasbor, dan jauh lebih banyak uji.

Untuk aplikasi LLM, [pelajaran observabilitas LLM](../../../17-infrastructure-and-production/13-llm-observability/docs/en.md) dalam fase infrastruktur membahas alat penelusuran dan evaluasi. Gagasan dalam pelajaran ini tetap berlaku pada masukan dan keluaran sistem-sistem itu.

## Kirimlah

Pelajaran ini menghasilkan `outputs/skill-drift-monitoring.md`. Skill ini merencanakan monitor untuk model baru dan menangani alarm pergeseran. Ia menanyakan lapisan mana yang berbunyi, jenis pergeseran apa yang sesuai dengan bukti, dan apakah tindakan yang tepat adalah menunggu, kalibrasi ulang, mengumpulkan label, atau melatih ulang.

## Latihan

1. Jalankan aliran dengan jendela 200 dan 2000 peristiwa. Catat peringatan palsu sebelum peristiwa 3000 dan keterlambatan alarm.
2. Tetapkan `n_tests=1`, `patience=1`, dan `min_cramers_v=0`. Hitung alarm pada periode bersih dari 5000 sampai 6500.
3. Tambahkan fitur yang rata-ratanya bertambah 0,01 per 100 peristiwa. Cari sinyal mana yang bereaksi lebih dulu, PSI atau KS.
4. Terapkan jarak Jensen-Shannon pada bin desil yang sama. Bandingkan dengan PSI pada setiap jendela.
5. Tambahkan monitor untuk tingkat prediksi positif model. Jelaskan mengapa monitor itu tetap diam selama pergeseran konsep.

## Istilah Kunci

| Istilah | Apa yang orang katakan | Apa artinya |
|---|---|---|
| Pergeseran data | "Datanya berubah" | Perubahan distribusi masukan, prediksi, atau label dibandingkan dengan jendela referensi |
| Pergeseran kovariat | "Jenis pengguna baru" | `P(x)` berubah sementara `P(y \| x)` tetap |
| Pergeseran prior | "Lebih banyak positif dari sebelumnya" | `P(y)` berubah sementara `P(x \| y)` tetap, juga disebut pergeseran label |
| Pergeseran konsep | "Aturan mainnya berubah" | `P(y \| x)` berubah, sehingga masukan yang sama kini memiliki jawaban benar yang berbeda |
| Jendela referensi | "Data baseline" | Sampel tetap, sering berupa data latihan, yang dijadikan pembanding setiap jendela terkini |
| PSI | "Skor stabilitas" | Jumlah simetris per bin dari (bagian terkini dikurangi bagian referensi) kali logaritma rasionya |
| Statistik KS | "Uji KS" | Jurang terbesar antara dua fungsi distribusi kumulatif empiris |
| V Cramér | "Ukuran efek untuk kategori" | Statistik chi-kuadrat yang diskalakan ke rentang 0 sampai 1 |
| Keterlambatan label | "Kebenaran datang belakangan" | Waktu antara prediksi dan tibanya hasil yang sebenarnya |
| Kelelahan alarm | "Terlalu banyak panggilan" | Orang mengabaikan alarm karena sebagian besar di antaranya alarm palsu |

## Bacaan Lanjutan

- [Gama et al., A Survey on Concept Drift Adaptation (ACM Computing Surveys, 2014)](https://research.tue.nl/en/publications/a-survey-on-concept-drift-adaptation): pergeseran nyata dan virtual, pola pergeseran, serta metode adaptasi
- [Rabanser, Günnemann, and Lipton, Failing Loudly: An Empirical Study of Methods for Detecting Dataset Shift (NeurIPS 2019)](https://arxiv.org/abs/1810.11953): uji dua sampel untuk deteksi pergeseran
- [Lu et al., Learning under Concept Drift: A Review (2020)](https://arxiv.org/abs/2004.05785): deteksi, pemahaman, dan metode adaptasi pergeseran
- [Yurdakul, Statistical Properties of Population Stability Index (Western Michigan University, 2018)](https://scholarworks.wmich.edu/dissertations/3208/): perilaku PSI terhadap ukuran sampel dan jumlah bin
- [scipy.stats.ks_2samp documentation](https://docs.scipy.org/doc/scipy/reference/generated/scipy.stats.ks_2samp.html): metode nilai-p eksak dan asimtotik untuk uji KS dua sampel
- [Evidently, How data drift detection works](https://docs.evidentlyai.com/metrics/explainer_drift): pemilihan uji default menurut jenis kolom dan ukuran data
- [NannyML, Performance estimation](https://nannyml.readthedocs.io/en/stable/how_it_works/performance_estimation.html): CBPE dan asumsi-asumsinya
