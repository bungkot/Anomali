# Bag of Words, TF-IDF, dan Representasi Teks

> Hitung dulu, pikir kemudian. TF-IDF masih mengalahkan embedding pada tugas yang terdefinisi dengan baik di tahun 2026.

**Jenis:** Build
**Bahasa:** Python, TypeScript
**Prasyarat:** Fase 5 · 01 (Pemrosesan Teks), Fase 2 · 02 (Regresi Linear dari Awal)
**Waktu:** ~75 menit

## Masalahnya

Model butuh angka. Anda punya string.

Setiap pipeline NLP harus menjawab pertanyaan yang sama: bagaimana kita mengubah aliran token dengan panjang variabel menjadi vektor berukuran tetap yang dapat dikonsumsi oleh klasifier? Jawaban pertama yang ditemukan bidang ini adalah jawaban paling bodoh yang tetap berfungsi: hitung katanya, lalu buat vektornya.

Vektor itu telah menopang lebih banyak NLP produksi daripada model embedding mana pun. Filter spam, klasifier topik, deteksi anomali pada log, peringkat pencarian (sebelum BM25), gelombang pertama analisis sentimen, dekade pertama benchmark NLP akademik. Praktisi di tahun 2026 masih meraihnya lebih dulu untuk tugas klasifikasi yang sempit. Cepat, dapat ditafsirkan, dan sering kali tak terbedakan dari model embedding berparameter 400 juta pada tugas-tugas di mana kehadiran katanya yang menjadi penentu.

Pelajaran ini membangun bag of words, lalu TF-IDF, dari awal. Kemudian menunjukkan scikit-learn melakukan hal yang sama dalam tiga baris. Kemudian menyebutkan mode kegagalan yang membuat Anda beralih ke embedding.

## Konsepnya

**Bag of Words (BoW)** membuang urutan. Untuk setiap dokumen, hitung berapa kali setiap kata dalam kosakata muncul. Panjang vektor sama dengan ukuran kosakata. Posisi `i` adalah jumlah kemunculan kata `i`.

**TF-IDF** menimbang ulang BoW. Kata yang muncul di setiap dokumen tidak memberikan informasi, jadi perkecil bobotnya. Kata yang jarang di seluruh korpus tetapi sering muncul dalam satu dokumen adalah sinyal, jadi perbesar bobotnya.

```math
\begin{aligned}
\text{TF-IDF}(w, d) &= \text{TF}(w, d) \cdot \text{IDF}(w) \\
&= \frac{\mathop{\mathrm{count}}(w \text{ in } d)}{|d|} \cdot \log\frac{N}{\mathop{\mathrm{df}}(w)}
\end{aligned}
```

Di sini `TF` adalah frekuensi istilah dalam dokumen, `df` adalah frekuensi dokumen (berapa banyak dokumen mengandung kata tersebut), dan `N` adalah total dokumen. Fungsi `log` menjaga bobot tetap terikat untuk kata-kata yang ada di mana-mana.

Properti utamanya: keduanya menghasilkan vektor jarang dengan sumbu yang dapat ditafsirkan. Anda dapat melihat bobot klasifier yang telah dilatih lalu membaca kata-kata mana yang mendorong sebuah dokumen ke masing-masing kelas. Hal itu tidak dapat Anda lakukan dengan embedding BERT berdimensi 768.

```figure
bow-tfidf
```

## Bangun Sendiri

### Langkah 1: membangun kosakata

```python
def build_vocab(docs):
    vocab = {}
    for doc in docs:
        for token in doc:
            if token not in vocab:
                vocab[token] = len(vocab)
    return vocab
```

Input: daftar dokumen yang telah ditokenisasi (tokenizer level kata mana pun boleh; `code/main.py` dalam pelajaran ini menggunakan variasi huruf kecil yang disederhanakan). Output: dictionary `{word: index}`. Urutan penyisipan yang stabil berarti indeks kata 0 adalah kata pertama yang terlihat di dokumen pertama. Konvensi bisa berbeda; scikit-learn mengurutkan secara alfabetis.

### Langkah 2: bag of words

```python
def bag_of_words(docs, vocab):
    matrix = [[0] * len(vocab) for _ in docs]
    for i, doc in enumerate(docs):
        for token in doc:
            if token in vocab:
                matrix[i][vocab[token]] += 1
    return matrix
```

```python
>>> docs = [["cat", "sat", "on", "mat"], ["cat", "cat", "ran"]]
>>> vocab = build_vocab(docs)
>>> bag_of_words(docs, vocab)
[[1, 1, 1, 1, 0], [2, 0, 0, 0, 1]]
```

Baris adalah dokumen. Kolom adalah indeks kosakata. Entri `[i][j]` adalah "berapa kali kata `j` muncul di dokumen `i`". Dokumen 1 memuat `cat` dua kali karena memang demikian. Dokumen 0 memuat `ran` nol kali karena tidak memuatnya.

### Langkah 3: frekuensi istilah dan frekuensi dokumen

```python
import math


def term_frequency(doc_bow, doc_length):
    return [c / doc_length if doc_length else 0 for c in doc_bow]


def document_frequency(bow_matrix):
    df = [0] * len(bow_matrix[0])
    for row in bow_matrix:
        for j, count in enumerate(row):
            if count > 0:
                df[j] += 1
    return df


def inverse_document_frequency(df, n_docs):
    return [math.log((n_docs + 1) / (d + 1)) + 1 for d in df]
```

Dua trik pemulusan patut disebut. `(n+1)/(d+1)` menghindari `log(x/0)`. Tambahan `+1` di akhir memastikan kata yang muncul di semua dokumen tetap memiliki IDF 1 (bukan 0), sesuai dengan nilai bawaan scikit-learn. Implementasi lain memakai `log(N/df)` mentah. Keduanya berfungsi; versi yang dimuluskan lebih ramah.

### Langkah 4: TF-IDF

```python
def tfidf(bow_matrix):
    n_docs = len(bow_matrix)
    df = document_frequency(bow_matrix)
    idf = inverse_document_frequency(df, n_docs)
    out = []
    for row in bow_matrix:
        length = sum(row)
        tf = term_frequency(row, length)
        out.append([tf_j * idf_j for tf_j, idf_j in zip(tf, idf)])
    return out
```

```python
>>> docs = [
...     ["the", "cat", "sat"],
...     ["the", "dog", "sat"],
...     ["the", "cat", "ran"],
... ]
>>> vocab = build_vocab(docs)
>>> bow = bag_of_words(docs, vocab)
>>> tfidf(bow)
```

Tiga dokumen, lima kata kosakata (`the`, `cat`, `sat`, `dog`, `ran`). `the` muncul di ketiganya, sehingga IDF-nya rendah. `dog` muncul di satu dokumen, sehingga IDF-nya tinggi. Dengan kosakata sungguhan yang terdiri dari ribuan kata, vektor-vektor ini bersifat jarang (sebagian besar entri adalah nol). Bahkan dalam contoh mainan ini, kata-kata yang diskriminatif menonjol.

### Langkah 5: normalisasi L2 pada baris

```python
def l2_normalize(matrix):
    out = []
    for row in matrix:
        norm = math.sqrt(sum(x * x for x in row))
        out.append([x / norm if norm else 0 for x in row])
    return out
```

Tanpa normalisasi, dokumen yang lebih panjang mendapatkan vektor yang lebih besar dan mendominasi skor kemiripan. Normalisasi L2 menempatkan setiap dokumen pada hipersfera satuan. Kemiripan kosinus antarbaris kini sekadar produk titik.

## Pakai Sendiri

scikit-learn menyediakan versi produksinya.

```python
from sklearn.feature_extraction.text import CountVectorizer, TfidfVectorizer

docs = ["the cat sat on the mat", "the dog sat on the mat", "the cat ran"]

bow_vectorizer = CountVectorizer()
bow = bow_vectorizer.fit_transform(docs)
print(bow_vectorizer.get_feature_names_out())
print(bow.toarray())

tfidf_vectorizer = TfidfVectorizer()
tfidf = tfidf_vectorizer.fit_transform(docs)
print(tfidf.toarray().round(3))
```

`CountVectorizer` melakukan tokenisasi, pembangunan kosakata, dan BoW dalam satu panggilan. `TfidfVectorizer` menambahkan pembobotan IDF dan normalisasi L2. Keduanya mengembalikan matriks jarang. Untuk 100 ribu dokumen, versi padat tidak muat di memori; tetap gunakan format jarang sampai klasifier meminta format padat.

Opsi yang mengubah segalanya:

| Argumen | Efek |
|---------|------|
| `ngram_range=(1, 2)` | Sertakan bigram. Biasanya meningkatkan klasifikasi. |
| `min_df=2` | Buang kata yang muncul di kurang dari 2 dokumen. Memangkas kosakata pada data bising. |
| `max_df=0.95` | Buang kata yang muncul di lebih dari 95% dokumen. Mendekati penghapusan stopword tanpa daftar yang ditanam keras. |
| `stop_words="english"` | Daftar stopword bawaan scikit-learn. Bergantung pada tugas — analisis sentimen sebaiknya *tidak* membuang negasi. |
| `sublinear_tf=True` | Pakai `1 + log(tf)` alih-alih `tf` mentah. Membantu ketika sebuah istilah berulang kali muncul dalam satu dokumen. |

### Kapan TF-IDF masih menang (per 2026)

- Deteksi spam, pelabelan topik, penandaan anomali pada log. Yang penting adalah kehadiran kata, bukan nuansa semantik.
- Rezim data sedikit (ratusan contoh berlabel). TF-IDF plus regresi logistik tidak punya biaya pra-pelatihan.
- Di mana pun latensi penting. TF-IDF plus model linear menjawab dalam mikrodetik. Menyematkan dokumen melalui transformer membutuhkan 10-100 ms.
- Sistem yang wajib menjelaskan prediksinya. Periksa koefisien klasifier. Kata-kata dengan bobot positif terbesar adalah alasannya.

### Kapan TF-IDF gagal

Kegagalan buta-semantik. Perhatikan dua dokumen ini:

- "The movie was not good at all."
- "The movie was excellent."

Satu adalah ulasan negatif. Satu lagi positif. Tumpang tindih TF-IDF keduanya tepat adalah `{the, movie, was}`. Klasifier bag-of-words harus menghafal bahwa kata `not` di dekat `good` membalik label. Ia dapat mempelajari ini dengan cukup data, tetapi tidak pernah seanggun model yang memahami sintaksis.

Kegagalan lainnya: kata di luar kosakata saat inferensi. Model BoW yang dilatih pada ulasan IMDb tidak tahu apa yang harus dilakukan dengan `Zoomer-approved` jika token itu tidak pernah muncul saat pelatihan. Embedding subkata (pelajaran 04) menangani ini. TF-IDF tidak.

### Hibrida: embedding berbobot TF-IDF

Bawaan pragmatis tahun 2026 untuk klasifikasi dengan data menengah: pakai bobot TF-IDF sebagai atensi atas embedding kata.

```python
def tfidf_weighted_embedding(doc, tfidf_scores, embedding_table, dim):
    vec = [0.0] * dim
    total_weight = 0.0
    for token in doc:
        if token not in embedding_table or token not in tfidf_scores:
            continue
        weight = tfidf_scores[token]
        emb = embedding_table[token]
        for i in range(dim):
            vec[i] += weight * emb[i]
        total_weight += weight
    if total_weight == 0:
        return vec
    return [v / total_weight for v in vec]
```

Anda mendapatkan kapasitas semantik dari embedding dan penekanan kata langka dari TF-IDF. Klasifier dilatih pada vektor hasil penggabungan. Ini mengungguli masing-masing metode secara tunggal untuk klasifikasi sentimen, topik, dan niat di bawah sekitar 50 ribu contoh berlabel.

## Kirim Hasilnya

Simpan sebagai `outputs/prompt-vectorization-picker.md`:

```markdown
---
name: vectorization-picker
description: Diberi tugas klasifikasi teks, rekomendasikan BoW, TF-IDF, embedding, atau hibrida.
phase: 5
lesson: 02
---

Anda merekomendasikan strategi vektorisasi teks. Diberi deskripsi tugas, keluarkan:

1. Representasi (BoW, TF-IDF, embedding transformer, atau hibrida). Jelaskan alasannya dalam satu kalimat.
2. Konfigurasi vektorizer yang spesifik. Sebutkan pustakanya. Kutip argumennya (`ngram_range`, `min_df`, `max_df`, `sublinear_tf`, `stop_words`).
3. Satu mode kegagalan yang perlu diuji sebelum rilis.

Tolak merekomendasikan embedding ketika pengguna memiliki kurang dari 500 contoh berlabel, kecuali mereka menunjukkan bukti kegagalan semantik pada baseline TF-IDF. Tolak menghapus stopword untuk analisis sentimen (negasi membawa sinyal). Tandai ketidakseimbangan kelas sebagai masalah yang memerlukan lebih dari sekadar perubahan vektorizer.

Contoh input: "Mengklasifikasikan 30 ribu tiket dukungan pelanggan ke dalam 12 kategori. Sebagian besar tiket terdiri dari 2-3 kalimat. Bahasa Inggris saja. Dibutuhkan kemampuan penjelasan untuk log audit."

Contoh keluaran:

- Representasi: TF-IDF. 30 ribu contoh tidaklah sedikit; persyaratan kemampuan penjelasan menyingkirkan embedding padat.
- Konfigurasi: `TfidfVectorizer(ngram_range=(1, 2), min_df=3, max_df=0.95, sublinear_tf=True, stop_words=None)`. Pertahankan stopword karena kata kunci kategori kadang merupakan stopword ("not working" berbeda dengan "working").
- Kegagalan yang diuji: verifikasi bahwa `min_df=3` tidak membuang kata kunci kategori yang langka. Jalankan `get_feature_names_out` yang difilter menurut kelas lalu periksa secara visual.
```

## Latihan

1. **Mudah.** Terapkan `cosine_similarity(doc_vec_a, doc_vec_b)` pada keluaran TF-IDF yang telah dinormalisasi L2. Verifikasi bahwa dokumen identik mendapat skor 1,0 dan dokumen dengan kosakata yang tak bertumpang tindih mendapat skor 0,0.
2. **Sedang.** Tambahkan dukungan `n-gram` pada `bag_of_words`. Parameter `n` menghasilkan jumlah kemunculan atas `n`-gram. Uji bahwa `n=2` pada `["the", "cat", "sat"]` menghasilkan jumlah bigram untuk `["the cat", "cat sat"]`.
3. **Sulit.** Bangun hibrida embedding berbobot TF-IDF di atas menggunakan vektor GloVe 100d (unduh sekali, simpan di cache). Bandingkan akurasi klasifikasi dengan TF-IDF murni dan embedding rata-rata murni pada dataset 20 Newsgroups. Laporkan mana yang menang pada kondisi apa.

## Istilah Kunci

| Istilah | Yang orang katakan | Yang sebenarnya dimaksud |
|---------|--------------------|--------------------------|
| BoW | Vektor frekuensi kata | Jumlah kemunculan kata kosakata dalam satu dokumen. Urutan dibuang. |
| TF | Frekuensi istilah | Jumlah kemunculan sebuah kata dalam dokumen, opsional dinormalisasi terhadap panjang dokumen. |
| DF | Frekuensi dokumen | Jumlah dokumen yang mengandung kata itu sekurang-kurangnya sekali. |
| IDF | Frekuensi dokumen invers | `log(N / df)` yang dimuluskan. Menurunkan bobot kata yang muncul di mana-mana. |
| Vektor jarang | Sebagian besar berisi nol | Kosakata biasanya 10 ribu-100 ribu kata; sebagian besar tidak muncul dalam dokumen tertentu. |
| Kemiripan kosinus | Sudut antarvektor | Produk titik dari vektor yang telah dinormalisasi L2. 1 berarti identik, 0 berarti ortogonal. |

## Bacaan Lanjutan

- [scikit-learn — ekstraksi fitur dari teks](https://scikit-learn.org/stable/modules/feature_extraction.html#text-feature-extraction) — referensi API kanonik, plus catatan untuk setiap opsi.
- [Salton, G., & Buckley, C. (1988). Term-weighting approaches in automatic text retrieval](https://www.sciencedirect.com/science/article/pii/0306457388900210) — makalah yang menjadikan TF-IDF sebagai pilihan utama selama satu dekade.
- ["Why TF-IDF Still Beats Embeddings" — Ashfaque Thonikkadavan (Medium)](https://medium.com/@cmtwskb/why-tf-idf-still-beats-embeddings-ad85c123e1b2) — tinjauan tahun 2026 tentang kapan metode lama menang dan mengapa.
