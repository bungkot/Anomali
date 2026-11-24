# Norma dan Jarak

> Fungsi jarak Anda mendefinisikan arti "serupa". Pilih yang salah dan semua yang berikutnya rusak.

**Jenis:** Build
**Bahasa:** Python, TypeScript
**Prasyarat:** Fase 1, Pelajaran 01 (Intuisi Aljabar Linear), 02 (Vektor, Matriks & Operasi)
**Waktu:** ~90 menit

## Tujuan Pembelajaran

- Mengimplementasikan fungsi jarak L1, L2, kosinus, Mahalanobis, Jaccard, dan jarak edit dari awal
- Memilih metrik jarak yang tepat untuk suatu tugas ML dan menjelaskan mengapa alternatifnya gagal
- Menghubungkan norma L1 dan L2 dengan regularisasi LASSO dan Ridge serta wilayah kendala geometrisnya
- Mendemonstrasikan bagaimana dataset yang sama menghasilkan tetangga terdekat yang berbeda di bawah metrik yang berbeda

## Masalahnya

Anda memiliki dua vektor. Mungkin keduanya adalah embedding kata. Mungkin profil pengguna. Mungkin larik piksel. Anda perlu tahu: seberapa dekat keduanya?

Jawabannya sepenuhnya bergantung pada fungsi jarak yang Anda pilih. Dua titik data dapat menjadi tetangga terdekat pada satu metrik dan saling jauh pada metrik lain. Klasifier KNN Anda, mesin rekomendasi Anda, basis data vektor Anda, algoritma pengelompokan Anda, fungsi loss Anda -- semuanya bergantung pada pilihan ini. Salah pilih dan model Anda mengoptimalkan hal yang keliru.

Tidak ada jarak terbaik secara universal. L2 bekerja untuk data spasial. Kesamaan kosinus mendominasi NLP. Jaccard menangani himpunan. Jarak edit menangani string. Mahalanobis memperhitungkan korelasi. Wasserstein memindahkan massa probabilitas. Masing-masing mengodekan asumsi yang berbeda tentang arti "serupa".

Pelajaran ini membangun setiap fungsi jarak utama dari awal, menunjukkan kapan masing-masing merupakan alat yang tepat, dan mendemonstrasikan bagaimana data yang sama menghasilkan tetangga terdekat yang benar-benar berbeda tergantung pada metrik yang Anda pakai.

## Konsepnya

### Norma: mengukur besar vektor

Sebuah norma mengukur "ukuran" dari sebuah vektor. Setiap fungsi jarak antara dua vektor dapat ditulis sebagai norma dari selisih keduanya: $d(a, b) = \lVert a - b \rVert$. Jadi memahami norma berarti memahami jarak.

### Norma L1 (jarak Manhattan)

Norma L1 menjumlahkan nilai absolut dari semua komponen.

```math
\lVert x \rVert_1 = |x_1| + |x_2| + \cdots + |x_n|
```

Norma ini disebut jarak Manhattan karena mengukur seberapa jauh Anda berjalan di sebuah jaringan kota di mana Anda hanya bisa bergerak searah sumbu. Tidak ada jalur diagonal.

```
Titik A = (1, 1)
Titik B = (4, 5)

Jarak L1 = |4-1| + |5-1| = 3 + 4 = 7

Di atas jaringan, Anda berjalan 3 blok ke timur dan 4 blok ke utara.
```

Kapan menggunakan L1:
- Data jarang berdimensi tinggi (fitur teks, enkode one-hot)
- Ketika Anda menginginkan ketahanan terhadap pencilan (satu perbedaan besar tidak mendominasi)
- Masalah pemilihan fitur (regularisasi L1 mendorong kejarangan)

Kaitan dengan regularisasi L1 (Lasso): menambahkan $\lVert w \rVert_1$ pada fungsi loss menghukum jumlah nilai absolut bobot. Ini mendorong bobot kecil menjadi tepat nol, melakukan pemilihan fitur otomatis. Penalti L1 menciptakan wilayah kendala berbentuk belah ketupat dalam ruang bobot, dan sudut-sudut belah ketupat terletak pada sumbu di mana beberapa bobot bernilai nol.

Kaitan dengan fungsi loss: Mean Absolute Error (MAE) adalah rata-rata jarak L1 antara prediksi dan target. Hal ini menghukum semua galat secara linear, sehingga lebih tahan terhadap pencilan dibandingkan MSE.

### Norma L2 (jarak Euclidean)

Norma L2 adalah jarak garis lurus. Akar kuadrat dari jumlah kuadrat komponen.

```math
\lVert x \rVert_2 = \sqrt{x_1^2 + x_2^2 + \cdots + x_n^2}
```

Inilah jarak yang Anda pelajari di pelajaran geometri. Pythagoras dalam $n$ dimensi.

```
Titik A = (1, 1)
Titik B = (4, 5)

Jarak L2 = sqrt((4-1)^2 + (5-1)^2) = sqrt(9 + 16) = sqrt(25) = 5.0

Garis lurus, memotong jaringan secara diagonal.
```

Kapan menggunakan L2:
- Data kontinu berdimensi rendah hingga sedang
- Ketika skala fitur sebanding
- Jarak fisik (data spasial, bacaan sensor)
- Kesamaan gambar pada tingkat piksel

Kaitan dengan regularisasi L2 (Ridge): menambahkan $\lVert w \rVert_2^2$ pada fungsi loss menghukum bobot besar. Berbeda dengan L1, regularisasi ini tidak mendorong bobot menjadi nol. Ia menyusutkan semua bobot ke arah nol secara proporsional. Penalti L2 menciptakan wilayah kendala berbentuk lingkaran, sehingga tidak ada sudut pada sumbu. Bobot menjadi kecil tetapi jarang tepat nol.

Kaitan dengan fungsi loss: Mean Squared Error (MSE) adalah rata-rata jarak L2 kuadrat. Pengkuadratan menghukum galat besar lebih berat daripada galat kecil.

```math
\begin{aligned}
\text{MAE (L1 loss):} \quad & |y - \hat{y}| && \text{Hukuman linear. Tahan terhadap pencilan.} \\
\text{MSE (L2 loss):} \quad & (y - \hat{y})^2 && \text{Hukuman kuadratik. Sensitif terhadap pencilan.}
\end{aligned}
```

### Norma Lp: keluarga umum

L1 dan L2 adalah kasus khusus dari norma Lp:

```math
\lVert x \rVert_p = (|x_1|^p + |x_2|^p + \cdots + |x_n|^p)^{1/p}
```

Nilai $p$ yang berbeda menghasilkan "bola satuan" dengan bentuk berbeda (himpunan semua titik yang jaraknya paling besar 1 dari titik asal; bentuk-bentuk di bawah adalah batasnya):

```
p=1:    Belah ketupat      (sudut pada sumbu)
p=2:    Lingkaran/bola     (bola bulat seperti biasa)
p=3:    Superelips         (persegi dengan sudut membulat)
p=inf:  Persegi/hiperkubus (sisi datar sejajar sumbu)
```

### Norma L-infinity (jarak Chebyshev)

Ketika $p$ mendekati tak hingga, norma Lp konvergen ke komponen absolut maksimum.

```math
\lVert x \rVert_\infty = \max(|x_1|, |x_2|, \ldots, |x_n|)
```

Jarak antara dua titik ditentukan oleh satu dimensi tempat keduanya paling berbeda. Semua dimensi lain diabaikan.

```math
\begin{aligned}
\text{Titik } A &= (1, 1) \\
\text{Titik } B &= (4, 5) \\[6pt]
\text{Jarak } L_\infty &= \max(|4-1|, |5-1|) = \max(3, 4) = 4
\end{aligned}
```

Kapan menggunakan L-infinity:
- Ketika penyimpangan terburuk pada satu dimensi pun penting
- Papan permainan (raja catur bergerak dengan L-infinity: satu langkah ke arah mana pun biayanya 1)
- Toleransi manufaktur (setiap dimensi harus dalam spesifikasi)

### Kesamaan Kosinus dan Jarak Kosinus

Kesamaan kosinus mengukur sudut antara dua vektor, mengabaikan besarnya.

```math
\text{cos\_sim}(a, b) = \frac{a \cdot b}{\lVert a \rVert_2 \, \lVert b \rVert_2}
```

Nilainya berkisar dari -1 (arah berlawanan) sampai +1 (arah sama). Vektor yang saling tegak lurus memiliki kesamaan kosinus 0.

Jarak kosinus mengubahnya menjadi sebuah jarak: cosine_distance = 1 - cosine_similarity. Nilainya berkisar dari 0 (arah identik) sampai 2 (arah berlawanan).

```math
\begin{aligned}
& a = (1, 0) \qquad b = (1, 1) \\[6pt]
& \text{cos\_sim} = \frac{1 \cdot 1 + 0 \cdot 1}{1 \cdot \sqrt{2}} = \frac{1}{\sqrt{2}} = 0.707 \\
& \text{cos\_dist} = 1 - 0.707 = 0.293
\end{aligned}
```

Mengapa kosinus mendominasi NLP dan embedding: dalam teks, panjang dokumen tidak boleh memengaruhi kesamaan. Sebuah dokumen tentang kucing yang dua kali lebih panjang daripada dokumen lain tentang kucing tetap harus "serupa". Kesamaan kosinus mengabaikan besaran (panjang) dan hanya peduli pada arah. Dua dokumen dengan distribusi kata yang sama tetapi panjang berbeda mengarah ke arah yang sama dan mendapat kesamaan kosinus 1.0.

Kapan menggunakan kesamaan kosinus:
- Kesamaan teks (vektor TF-IDF, embedding kata, embedding kalimat)
- Domain apa pun di mana besaran adalah derau dan arah adalah sinyal
- Sistem rekomendasi (vektor preferensi pengguna)
- Pencarian embedding (basis data vektor hampir selalu memakai kosinus atau hasil kali titik)

### Kesamaan Hasil Kali Titik vs Kesamaan Kosinus

Hasil kali titik dari dua vektor adalah:

```math
\begin{aligned}
a \cdot b &= a_1 b_1 + a_2 b_2 + \cdots + a_n b_n \\
&= \lVert a \rVert \, \lVert b \rVert \cos(\text{angle})
\end{aligned}
```

Kesamaan kosinus adalah hasil kali titik yang dinormalisasi oleh kedua besaran. Ketika kedua vektor sudah dinormalisasi ke satuan (besaran = 1), hasil kali titik dan kesamaan kosinus identik.

```math
\text{Jika } \lVert a \rVert = 1 \text{ dan } \lVert b \rVert = 1: \quad a \cdot b = \cos(\text{sudut antara } a \text{ dan } b)
```

Kapan keduanya berbeda: hasil kali titik memasukkan informasi besaran. Vektor dengan besaran lebih besar mendapat skor hasil kali titik lebih tinggi. Hal ini penting pada beberapa sistem pengambilan (retrieval) di mana Anda ingin item "populer" menduduki peringkat lebih tinggi. Besaran bertindak sebagai sinyal kualitas atau kepentingan implisit.

```
a = (3, 0)    b = (1, 0)    c = (0, 1)

dot(a, b) = 3     dot(a, c) = 0
cos(a, b) = 1.0   cos(a, c) = 0.0

Keduanya sepakat soal arah, tetapi hasil kali titik juga mencerminkan besaran.
```

Dalam praktiknya:
- Gunakan kesamaan kosinus ketika Anda menginginkan kesamaan arah murni
- Gunakan hasil kali titik ketika besaran membawa informasi yang bermakna
- Banyak basis data vektor (Pinecone, Weaviate, Qdrant) memungkinkan Anda memilih di antara keduanya
- Jika embedding Anda sudah dinormalisasi-L2, pilihannya tidak berpengaruh

### Jarak Mahalanobis

Jarak Euclidean memperlakukan semua dimensi secara sama. Tetapi jika fitur Anda berkorelasi atau memiliki skala berbeda, L2 memberikan hasil yang menyesatkan.

Jarak Mahalanobis memperhitungkan struktur kovarians data.

```math
d_M(x, y) = \sqrt{(x - y)^T S^{-1} (x - y)}
```

di mana $S$ adalah matriks kovarians dari data.

Secara intuitif: jarak Mahalanobis pertama-tama mendekorelasi dan menormalisasi data (whitening), lalu menghitung jarak L2 dalam ruang yang telah ditransformasikan. Jika $S$ adalah matriks identitas (fitur tak berkorelasi, varians satuan), jarak Mahalanobis tereduksi menjadi jarak Euclidean.

```
Contoh: tinggi dan berat badan berkorelasi.
Seseorang dengan tinggi 6'2" dan berat 180 lbs tidaklah aneh.
Seseorang dengan tinggi 5'0" dan berat 180 lbs adalah aneh.

Jarak Euclidean mungkin mengatakan keduanya sama jauhnya dari rata-rata.
Jarak Mahalanobis dengan tepat mengidentifikasi yang kedua sebagai pencilan
karena ia memperhitungkan korelasi tinggi-berat.
```

Kapan menggunakan jarak Mahalanobis:
- Deteksi pencilan (titik dengan jarak Mahalanobis besar dari rata-rata adalah pencilan)
- Klasifikasi ketika fitur memiliki skala dan korelasi berbeda
- Ketika Anda memiliki cukup data untuk mengestimasi matriks kovarians yang andal
- Pengendalian mutu dalam manufaktur (pemantauan proses multivariat)

### Kesamaan Jaccard (untuk himpunan)

Kesamaan Jaccard mengukur tumpang tindih antara dua himpunan.

```math
J(A, B) = \frac{|A \cap B|}{|A \cup B|}
```

Nilainya berkisar dari 0 (tidak ada tumpang tindih) sampai 1 (himpunan identik). Jarak Jaccard = 1 - kesamaan Jaccard.

```
A = {cat, dog, fish}
B = {cat, bird, fish, snake}

Irisan = {cat, fish}                  ukuran = 2
Gabungan = {cat, dog, fish, bird, snake}  ukuran = 5

Kesamaan Jaccard = 2/5 = 0.4
Jarak Jaccard = 0.6
```

Kapan menggunakan Jaccard:
- Membandingkan himpunan tag, kategori, atau fitur
- Kesamaan dokumen berdasarkan kehadiran kata (bukan frekuensi)
- Deteksi duplikat dekat (pendekatan Jaccard dengan MinHash)
- Membandingkan vektor fitur biner (data hadir/tidak hadir)
- Evaluasi model segmentasi (Intersection over Union = Jaccard)

### Jarak Edit (Jarak Levenshtein)

Jarak edit menghitung jumlah minimum operasi satu karakter yang dibutuhkan untuk mengubah satu string menjadi string lain. Operasinya adalah: sisipkan, hapus, atau substitusi.

```
"kitten" -> "sitting"

kitten -> sitten  (substitusi k -> s)
sitten -> sittin  (substitusi e -> i)
sittin -> sitting (sisipkan g)

Jarak edit = 3
```

Dihitung menggunakan pemrograman dinamis. Isi sebuah matriks yang mana entri $(i, j)$ adalah jarak edit antara $i$ karakter pertama string A dan $j$ karakter pertama string B.

```
         ""  s  i  t  t  i  n  g
     ""   0  1  2  3  4  5  6  7
     k    1  1  2  3  4  5  6  7
     i    2  2  1  2  3  4  5  6
     t    3  3  2  1  2  3  4  5
     t    4  4  3  2  1  2  3  4
     e    5  5  4  3  2  2  3  4
     n    6  6  5  4  3  3  2  3
```

Kapan menggunakan jarak edit:
- Pemeriksaan dan koreksi ejaan
- Penyejajaran urutan DNA (dengan operasi berbobot)
- Pencocokan string kabur
- Deduplikasi data teks yang berantakan

### Divergensi KL (bukan jarak, tetapi dipakai seolah-olah jarak)

Divergensi KL mengukur bagaimana satu distribusi probabilitas berbeda dari distribusi lain. Dibahas di Pelajaran 09, tetapi ia termasuk dalam pembahasan ini karena orang memakainya sebagai "jarak" meskipun ia bukan jarak.

```math
D_{KL}(P \,\|\, Q) = \sum_x p(x) \log\frac{p(x)}{q(x)}
```

Sifat krusial: divergensi KL TIDAK simetris.

```math
D_{KL}(P \,\|\, Q) \ne D_{KL}(Q \,\|\, P)
```

Artinya, ia gagal memenuhi persyaratan dasar sebuah metrik jarak. Ia juga tidak memenuhi ketidaksamaan segitiga. Ia adalah divergensi, bukan jarak.

KL maju $(D_{KL}(P \Vert Q))$ bersifat "mencari rata-rata" (mean-seeking): $Q$ berusaha mencakup semua mode dari $P$.
KL mundur $(D_{KL}(Q \Vert P))$ bersifat "mencari mode" (mode-seeking): $Q$ fokus pada satu mode dari $P$.

Ketika Anda melihat divergensi KL:
- VAE (suku KL dalam ELBO mendorong distribusi laten menuju prior)
- Distilasi pengetahuan (murid berusaha mencocokkan distribusi guru)
- RLHF (penalti KL menjaga model hasil fine-tuning tetap dekat dengan model dasar)
- Metode gradien kebijakan (membatasi pembaruan kebijakan)

### Jarak Wasserstein (Jarak Earth Mover)

Jarak Wasserstein mengukur "usaha" minimum yang dibutuhkan untuk mengubah satu distribusi probabilitas menjadi distribusi lain. Pikirkan begini: jika satu distribusi adalah timbunan tanah dan yang lain adalah lubang, berapa banyak tanah yang harus Anda pindahkan dan seberapa jauh?

```math
W(P, Q) = \inf_{\gamma} \mathbb{E}[d(x, y)] \quad \text{(untuk semua rencana transpor } \gamma\text{)}
```

Untuk distribusi 1D, rumusnya menyederhana menjadi integral dari selisih absolut fungsi distribusi kumulatif:

```math
W_1(P, Q) = \int |\text{CDF}_P(x) - \text{CDF}_Q(x)| \, dx
```

Mengapa Wasserstein penting:
- Ia adalah metrik sejati (simetris, memenuhi ketidaksamaan segitiga)
- Ia memberikan gradien bahkan ketika distribusi tidak tumpang tindih (divergensi KL menjadi tak hingga)
- Sifat ini menjadikannya inti dari Wasserstein GAN (WGAN), yang menyelesaikan ketidakstabilan pelatihan GAN original

```
Distribusi yang tidak tumpang tindih:

P: [1, 0, 0, 0, 0]    Q: [0, 0, 0, 0, 1]

Divergensi KL: tak hingga (log dari nol)
Wasserstein: 4 (pindahkan seluruh massa 4 bin)

Wasserstein memberi gradien yang bermakna. KL tidak.
```

Kapan menggunakan Wasserstein:
- Pelatihan GAN (WGAN, WGAN-GP)
- Membandingkan distribusi yang mungkin tidak tumpang tindih
- Masalah transpor optimal
- Pengambilan gambar (membandingkan histogram warna)

### Mengapa Tugas Berbeda Membutuhkan Jarak Berbeda

| Tugas | Jarak terbaik | Mengapa |
|---|---|---|
| Kesamaan teks | Kosinus | Besaran adalah derau, arah adalah makna |
| Perbandingan piksel gambar | L2 | Hubungan spasial penting, fitur berskala sebanding |
| Fitur jarang berdimensi tinggi | L1 | Tangguh, tidak memperbesar perbedaan besar yang langka |
| Tumpang tindih himpunan (tag, kategori) | Jaccard | Data secara alami berhimpun, bukan vektorial |
| Pencocokan string | Jarak edit | Operasi memetakan ke intuisi penyuntingan manusia |
| Deteksi pencilan | Mahalanobis | Memperhitungkan korelasi dan skala fitur |
| Membandingkan distribusi | Divergensi KL | Mengukur informasi yang hilang dengan memakai $Q$, bukan $P$ |
| Pelatihan GAN | Wasserstein | Memberikan gradien meski distribusi tidak tumpang tindih |
| Embedding (basis data vektor) | Kosinus atau hasil kali titik | Embedding dilatih untuk mengodekan makna pada arah |
| Rekomendasi | Hasil kali titik | Besaran dapat mengodekan popularitas atau keyakinan |
| Urutan DNA | Jarak edit berbobot | Biaya substitusi berbeda untuk setiap pasangan nukleotida |
| QC manufaktur | L-infinity | Penyimpangan terburuk pada satu dimensi yang penting |

### Kaitan dengan Fungsi Loss

Fungsi loss adalah fungsi jarak yang diterapkan pada prediksi versus target.

```
Fungsi loss       Jarak yang dipakai      Perilaku
MSE               L2 kuadrat              Menghukum galat besar dengan berat
MAE               L1                      Menghukum semua galat secara sama
Huber loss        L1 untuk galat besar,   Terbaik dari keduanya: tangguh terhadap pencilan,
                  L2 untuk galat kecil    gradien halus di dekat nol
Cross-entropy     Divergensi KL           Mengukur ketidakcocokan distribusi
Hinge loss        max(0, margin - d)      Hanya menghukum di bawah margin
Triplet loss      L2 (umumnya)            Menarik positif mendekat, mendorong
                                          negatif menjauh
Contrastive loss  L2                      Pasangan serupa dekat, pasangan tak
                                          serupa di luar margin
```

### Kaitan dengan Regularisasi

Regularisasi menambahkan penalti norma pada bobot ke dalam fungsi loss.

```
Regularisasi L1 (Lasso):   loss + lambda * ||w||_1
  -> Bobot jarang. Beberapa bobot menjadi tepat nol.
  -> Pemilihan fitur otomatis.
  -> Solusi memiliki sudut (tidak terdiferensialkan di nol).

Regularisasi L2 (Ridge):   loss + lambda * ||w||_2^2
  -> Bobot kecil. Semua bobot menyusut ke arah nol.
  -> Tidak ada pemilihan fitur (tidak ada yang mencapai tepat nol).
  -> Solusi halus di semua titik.

Elastic Net:                  loss + lambda_1 * ||w||_1 + lambda_2 * ||w||_2^2
  -> Menggabungkan kejarangan L1 dengan kestabilan L2.
  -> Kelompok fitur yang berkorelasi dipertahankan atau dibuang bersama.
```

Mengapa L1 menghasilkan kejarangan tetapi L2 tidak: bayangkan wilayah kendala dalam ruang bobot 2D. L1 adalah belah ketupat, L2 adalah lingkaran. Kontur fungsi loss (elips) paling mungkin menyentuh belah ketupat pada sebuah sudut, di mana satu bobot bernilai nol. Kontur itu menyentuh lingkaran pada titik yang halus, di mana kedua bobot tidak nol.

### Pencarian Tetangga Terdekat

Setiap fungsi jarak mengimplikasikan masalah pencarian tetangga terdekat: diberi sebuah titik kueri, temukan titik terdekat dalam sebuah dataset.

Pencarian tetangga terdekat eksak memerlukan $O(n \cdot d)$ per kueri pada dataset dengan $n$ titik dan $d$ dimensi. Untuk dataset besar, ini terlalu lambat.

Algoritma Approximate Nearest Neighbor (ANN) menukar sedikit akurasi dengan kecepatan yang jauh lebih besar:

```
Algoritma         Pendekatan                    Digunakan oleh
KD-trees          Partisi ruang searah sumbu    scikit-learn (dimensi rendah)
Ball trees        Hipersfera bertingkat         scikit-learn (dimensi sedang)
LSH               Proyeksi hash acak            Deteksi duplikat dekat
HNSW              Graf small-world hierarkis    FAISS, Qdrant, Weaviate
                  yang dapat dinavigasi
IVF               Indeks file terbalik dengan   FAISS (skala miliaran)
                  pencarian berbasis klaster
Kuant. produk.    Kompresi vektor, cari dalam   FAISS (memori terbatas)
                  ruang terkompresi
```

HNSW (Hierarchical Navigable Small World) adalah algoritma dominan pada basis data vektor modern. Ia membangun graf multilapis di mana setiap simpul terhubung ke tetangga terdekat perkiraannya. Pencarian dimulai dari lapisan atas (jarang, lompatan panjang) dan turun ke lapisan bawah (rapat, lompatan pendek).

```figure
norm-unit-balls
```

## Bangun Sendiri

### Langkah 1: Semua fungsi norma dan jarak

Lihat `code/distances.py` untuk implementasi lengkapnya. Setiap fungsi dibangun dari awal hanya dengan matematika dasar Python. Berkas `code/distances.ts` adalah port TypeScript dengan fungsi dan demo yang sama.

### Langkah 2: Data sama, jarak beda, tetangga beda

Demo dalam `distances.py` membuat sebuah dataset, memilih satu titik kueri, dan menunjukkan bagaimana tetangga terdekat berubah tergantung metrik jarak. Titik yang "terdekat" pada L1 mungkin tidak terdekat pada L2 maupun kosinus.

### Langkah 3: Pencarian kesamaan embedding

Kode ini menyertakan pencarian kesamaan embedding tiruan yang menemukan "dokumen" paling mirip dengan sebuah kueri memakai kesamaan kosinus versus jarak L2, menunjukkan bahwa peringkatnya dapat berbeda.

## Gunakan

Penggunaan praktis yang paling umum: menemukan item serupa dalam basis data vektor.

```python
import numpy as np

def cosine_similarity_matrix(X):
    norms = np.linalg.norm(X, axis=1, keepdims=True)
    norms = np.where(norms == 0, 1, norms)
    X_normalized = X / norms
    return X_normalized @ X_normalized.T

embeddings = np.random.randn(1000, 768)

sim_matrix = cosine_similarity_matrix(embeddings)

query_idx = 0
similarities = sim_matrix[query_idx]
top_k = np.argsort(similarities)[::-1][1:6]
print(f"Top 5 most similar to item 0: {top_k}")
print(f"Similarities: {similarities[top_k]}")
```

Ketika Anda memanggil `model.encode(text)` lalu mencari basis data vektor, inilah yang terjadi di balik layar. Model embedding memetakan teks menjadi vektor. Basis data vektor menghitung kesamaan kosinus (atau hasil kali titik) antara vektor kueri Anda dan setiap vektor yang tersimpan, dengan memakai algoritma ANN agar tidak perlu memeriksa semuanya.

## Latihan

1. Hitung jarak L1, L2, dan L-infinity antara (1, 2, 3) dan (4, 0, 6). Verifikasi bahwa $L_\infty \le L_2 \le L_1$ selalu berlaku untuk setiap pasangan titik. Buktikan mengapa urutan ini terjamin.

2. Buat dua vektor yang kesamaan kosinusnya tinggi (> 0,9) tetapi jarak L2-nya besar (> 10). Jelaskan secara geometris apa yang terjadi. Lalu buat dua vektor yang kesamaan kosinusnya rendah (< 0,3) tetapi jarak L2-nya kecil (< 0,5).

3. Implementasikan sebuah fungsi yang menerima dataset dan titik kueri, lalu mengembalikan tetangga terdekat di bawah jarak L1, L2, kosinus, dan Mahalanobis. Temukan dataset di mana keempatnya tidak sepakat soal titik mana yang terdekat.

4. Hitung jarak Wasserstein antara [0.5, 0.5, 0, 0] dan [0, 0, 0.5, 0.5] secara manual dengan metode CDF. Lalu hitunglah antara [0.25, 0.25, 0.25, 0.25] dan [0, 0, 0.5, 0.5]. Mana yang lebih besar dan mengapa?

5. Implementasikan MinHash untuk kesamaan Jaccard perkiraan. Bangkitkan 100 himpunan acak, hitung Jaccard eksak untuk semua pasangan, lalu bandingkan dengan pendekatan MinHash memakai 50, 100, dan 200 fungsi hash. Plot galat pendekatannya.

### Petunjuk

**Latihan 1.** Vektor selisihnya adalah (-3, 2, -3), jadi hitung ketiga norma vektor itu dengan `l1_norm`, `l2_norm`, dan `linf_norm`. Untuk pembuktiannya, bandingkan $\lvert d_i \rvert$ terbesar dengan jumlah kuadrat, lalu bandingkan jumlah kuadrat dengan kuadrat dari jumlah nilai absolut.

**Latihan 2.** Kesamaan kosinus mengabaikan panjang, jadi mulailah dengan dua vektor yang mengarah hampir sama lalu buat salah satunya jauh lebih panjang. Untuk kasus kedua, gunakan dua vektor pendek di dekat titik asal yang mengarah ke arah berbeda, lalu periksa kedua syarat dengan `cosine_similarity` dan `l2_distance`.

**Latihan 3.** Panggil `find_nearest_neighbor(query, dataset, distance_fn)` sekali untuk masing-masing `l1_distance`, `l2_distance`, `cosine_distance`, dan `mahalanobis_distance`. `demo_different_neighbors` menunjukkan dataset di mana metrik-metrik itu tidak sepakat, jadi bacalah dulu lalu ubah titik-titiknya sampai keempat jawabannya berbeda.

**Latihan 4.** Tulis jumlahan kumulatif setiap distribusi, lalu jumlahkan selisih absolut dari kedua jumlahan kumulatif tersebut. Periksa kedua hasil hitungan tangan dengan `wasserstein_1d(p, q)`, lalu jelaskan nilai yang lebih besar melalui seberapa jauh massa probabilitas harus bergerak.

**Latihan 5.** Untuk setiap fungsi hash, tanda tangan MinHash dari sebuah himpunan adalah nilai hash minimum pada elemennya, dan fraksi fungsi hash di mana dua himpunan memperoleh minimum yang sama mengestimasi kesamaan Jaccard mereka. Bandingkan setiap estimasi dengan `jaccard_similarity`; galatnya harus turun kurang lebih sebanding dengan $1/\sqrt{k}$ untuk $k$ fungsi hash.

## Istilah Kunci

| Istilah | Apa yang orang katakan | Arti sebenarnya |
|---|---|---|
| Norma | "Ukuran sebuah vektor" | Fungsi yang memetakan vektor menjadi skalar nonnegatif, memenuhi ketidaksamaan segitiga, homogenitas absolut, dan bernol hanya untuk vektor nol |
| Norma L1 | "Jarak Manhattan" | Jumlah nilai absolut komponen. Menghasilkan kejarangan dalam optimasi. Tangguh terhadap pencilan |
| Norma L2 | "Jarak Euclidean" | Akar kuadrat dari jumlah kuadrat komponen. Jarak garis lurus dalam ruang Euclidean |
| Norma Lp | "Norma umum" | Akar ke-$p$ dari jumlah pangkat ke-$p$ nilai absolut komponen. L1 dan L2 adalah kasus khusus |
| Norma L-infinity | "Norma maks" atau "jarak Chebyshev" | Nilai komponen absolut terbesar. Limit dari Lp saat $p$ mendekati tak hingga |
| Kesamaan kosinus | "Sudut antarvektor" | Hasil kali titik yang dinormalisasi oleh kedua besaran. Berkisar -1 sampai +1. Mengabaikan panjang vektor |
| Jarak kosinus | "1 dikurangi kesamaan kosinus" | Mengubah kesamaan kosinus menjadi jarak. Berkisar 0 sampai 2 |
| Hasil kali titik | "Kosinus takternormalisasi" | Jumlah hasil kali komponen per komponen. Sama dengan kesamaan kosinus dikali kedua besaran |
| Jarak Mahalanobis | "Jarak yang sadar korelasi" | Jarak L2 dalam ruang yang telah di-whitening (didekorelasi dan dinormalisasi) memakai matriks kovarians data |
| Kesamaan Jaccard | "Tumpang tindih himpunan" | Ukuran irisan dibagi ukuran gabungan. Untuk himpunan, bukan vektor |
| Jarak edit | "Jarak Levenshtein" | Jumlah minimum sisipan, penghapusan, dan substitusi untuk mengubah satu string menjadi string lain |
| Divergensi KL | "Jarak antardistribusi" | Bukan jarak sejati (tidak simetris). Mengukur bit ekstra dari pemakaian $Q$ untuk mengodekan $P$ |
| Jarak Wasserstein | "Jarak earth mover" | Usaha minimum untuk memindahkan massa dari satu distribusi ke distribusi lain. Metrik sejati |
| Tetangga terdekat perkiraan | "Pencarian ANN" | Algoritma (HNSW, LSH, IVF) yang menemukan titik hampir terdekat jauh lebih cepat daripada pencarian eksak |
| HNSW | "Algoritma basis data vektor" | Graf Hierarchical Navigable Small World. Graf multilapis untuk pencarian tetangga terdekat perkiraan yang cepat |
| Regularisasi L1 | "Lasso" | Menambahkan norma L1 bobot ke loss. Mendorong bobot ke nol (kejarangan) |
| Regularisasi L2 | "Ridge" atau "weight decay" | Menambahkan norma L2 kuadrat bobot ke loss. Menyusutkan bobot ke arah nol tanpa kejarangan |
| Elastic Net | "L1 + L2" | Menggabungkan regularisasi L1 dan L2. Menangani kelompok fitur berkorelasi lebih baik daripada salah satunya saja |

## Bacaan Lanjutan

- [FAISS: A Library for Efficient Similarity Search](https://github.com/facebookresearch/faiss) - pustaka Meta untuk pencarian ANN skala miliaran
- [Wasserstein GAN (Arjovsky et al., 2017)](https://arxiv.org/abs/1701.07875) - makalah yang memperkenalkan jarak Earth Mover ke dalam GAN
- [Locality-Sensitive Hashing (Indyk & Motwani, 1998)](https://dl.acm.org/doi/10.1145/276698.276876) - algoritma ANN fundamental
- [Efficient Estimation of Word Representations (Mikolov et al., 2013)](https://arxiv.org/abs/1301.3781) - Word2Vec, tempat kesamaan kosinus menjadi pilihan utama untuk embedding
- [Dokumentasi sklearn.neighbors](https://scikit-learn.org/stable/modules/neighbors.html) - panduan praktis metrik jarak dan algoritma tetangga di scikit-learn
