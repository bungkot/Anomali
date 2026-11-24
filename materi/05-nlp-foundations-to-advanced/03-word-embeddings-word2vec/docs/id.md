# Word Embedding — Word2Vec dari Awal

> Sebuah kata dikenal dari perusahaan yang disimpannya. Latih jaringan dangkal atas gagasan itu, dan geometri pun muncul dengan sendirinya.

**Jenis:** Build
**Bahasa:** Python
**Prasyarat:** Fase 5 · 02 (BoW + TF-IDF), Fase 3 · 03 (Propagasi Balik dari Awal)
**Waktu:** ~75 menit

## Masalahnya

TF-IDF tahu bahwa `dog` dan `puppy` adalah dua kata yang berbeda. Ia tidak tahu bahwa keduanya hampir bermakna sama. Klasifier yang dilatih pada `dog` tidak dapat digeneralisasi ke ulasan tentang `puppy`. Anda dapat menambalnya dengan mendaftar sinonim, tetapi itu gagal pada istilah langka, jargon domain, dan setiap bahasa yang tidak Anda antisipasi.

Anda menginginkan representasi di mana `dog` dan `puppy` berada berdekatan di ruang. Di mana `king - man + woman` mendarat dekat `queen`. Di mana model yang dilatih pada `dog` mentransfer sebagian sinyal ke `puppy` secara gratis.

Word2Vec memberi kita ruang itu. Jaringan saraf dua lapis, pelatihan triliun token, diterbitkan tahun 2013. Arsitekturnya nyaris memalukan karena begitu sederhananya. Hasilnya membentuk ulang NLP selama satu dekade.

## Konsepnya

**Hipotesis distribusional** (Firth, 1957): "Anda akan mengenal sebuah kata dari perusahaan yang disimpannya." Jika dua kata muncul dalam konteks yang serupa, keduanya kemungkinan bermakna serupa.

Word2Vec hadir dalam dua rasa, keduanya memanfaatkan gagasan itu.

- **Skip-gram.** Diberi kata pusat, prediksi kata-kata di sekitarnya. `cat -> (the, sat, on)` dengan ukuran jendela 2.
- **CBOW (continuous bag of words).** Diberi kata-kata di sekitar, prediksi kata pusatnya. `(the, sat, on) -> cat`.

Skip-gram lebih lambat dilatih tetapi lebih baik menangani kata langka. Ia menjadi pilihan utama.

Jaringan ini memiliki satu lapisan tersembunyi tanpa nonlinieritas. Inputnya adalah vektor one-hot atas kosakata. Keluarannya adalah softmax atas kosakata. Setelah pelatihan, Anda buang lapisan keluaran. Bobot lapisan tersembunyilah yang menjadi embedding.

```
one-hot(center) ── W ──▶ hidden (d-dim) ── W' ──▶ softmax(vocab)
                          ^
                          inilah embedding-nya
```

Triknya: softmax atas 100 ribu kata sangat mahal. Word2Vec memakai **negative sampling** untuk mengubahnya menjadi tugas klasifikasi biner. Prediksi "apakah kata konteks ini muncul di dekat kata pusat ini, ya atau tidak". Ambil segelintir kata negatif (yang tidak ikut muncul) per pasangan pelatihan, alih-alih menghitung softmax atas seluruh kosakata.

```figure
word-vector-arithmetic
```

## Bangun Sendiri

### Langkah 1: pasangan pelatihan dari korpus

```python
def skipgram_pairs(docs, window=2):
    pairs = []
    for doc in docs:
        for i, center in enumerate(doc):
            for j in range(max(0, i - window), min(len(doc), i + window + 1)):
                if i == j:
                    continue
                pairs.append((center, doc[j]))
    return pairs
```

```python
>>> skipgram_pairs([["the", "cat", "sat", "on", "mat"]], window=2)
[('the', 'cat'), ('the', 'sat'),
 ('cat', 'the'), ('cat', 'sat'), ('cat', 'on'),
 ('sat', 'the'), ('sat', 'cat'), ('sat', 'on'), ('sat', 'mat'),
 ...]
```

Setiap pasangan (pusat, konteks) dalam jendela adalah contoh pelatihan positif.

### Langkah 2: tabel embedding

Dua matriks. `W` adalah tabel embedding kata pusat (yang Anda simpan). `W'` adalah tabel kata konteks (sering dibuang, kadang digabungkan dengan `W`).

```python
import numpy as np


def init_embeddings(vocab_size, dim, seed=0):
    rng = np.random.default_rng(seed)
    W = rng.normal(0, 0.1, size=(vocab_size, dim))
    W_prime = rng.normal(0, 0.1, size=(vocab_size, dim))
    return W, W_prime
```

Inisialisasi acak yang kecil. Ukuran kosakata 10 ribu dan dimensi 100 adalah ukuran realistis; untuk pengajaran, 50 kata kali 16 dimensi sudah cukup untuk melihat geometrinya.

### Langkah 3: objektif negative sampling

Untuk setiap pasangan positif `(center, context)`, ambil `k` kata acak dari kosakata sebagai negatif. Latih model agar produk titik `W[center] · W'[context]` bernilai tinggi untuk pasangan positif dan rendah untuk pasangan negatif.

```python
def sigmoid(x):
    return 1.0 / (1.0 + np.exp(-np.clip(x, -20, 20)))


def train_pair(W, W_prime, center_idx, context_idx, negative_indices, lr):
    v_c = W[center_idx]
    u_pos = W_prime[context_idx]
    u_negs = W_prime[negative_indices]

    pos_score = sigmoid(v_c @ u_pos)
    neg_scores = sigmoid(u_negs @ v_c)

    grad_center = (pos_score - 1) * u_pos
    for i, u in enumerate(u_negs):
        grad_center += neg_scores[i] * u

    W[context_idx] = W[context_idx]
    W_prime[context_idx] -= lr * (pos_score - 1) * v_c
    for i, neg_idx in enumerate(negative_indices):
        W_prime[neg_idx] -= lr * neg_scores[i] * v_c
    W[center_idx] -= lr * grad_center
```

Rumus ajaibnya: loss logistik pada pasangan positif (ingin sigmoid mendekati 1) plus loss logistik pada pasangan negatif (ingin sigmoid mendekati 0). Gradien mengalir ke kedua tabel. Turunan lengkapnya ada di makalah asli; jalankan sekali dengan pensil dan kertas jika Anda ingin benar-benar memahaminya.

### Langkah 4: latih pada korpus mainan

```python
def train(docs, dim=16, window=2, k_neg=5, epochs=100, lr=0.05, seed=0):
    vocab = build_vocab(docs)
    vocab_size = len(vocab)
    rng = np.random.default_rng(seed)
    W, W_prime = init_embeddings(vocab_size, dim, seed=seed)
    pairs = skipgram_pairs(docs, window=window)

    for epoch in range(epochs):
        rng.shuffle(pairs)
        for center, context in pairs:
            c_idx = vocab[center]
            ctx_idx = vocab[context]
            negs = rng.integers(0, vocab_size, size=k_neg)
            negs = [n for n in negs if n != ctx_idx and n != c_idx]
            train_pair(W, W_prime, c_idx, ctx_idx, negs, lr)
    return vocab, W
```

Setelah epoch yang cukup banyak pada korpus besar, kata-kata yang berbagi konteks memiliki embedding pusat yang serupa. Pada korpus mainan, efeknya terlihat samar. Pada miliaran token, efeknya terlihat jelas.

### Langkah 5: trik analogi

```python
def nearest(vocab, W, target_vec, topk=5, exclude=None):
    exclude = exclude or set()
    inv_vocab = {i: w for w, i in vocab.items()}
    norms = np.linalg.norm(W, axis=1, keepdims=True) + 1e-9
    W_norm = W / norms
    target = target_vec / (np.linalg.norm(target_vec) + 1e-9)
    sims = W_norm @ target
    order = np.argsort(-sims)
    out = []
    for i in order:
        if i in exclude:
            continue
        out.append((inv_vocab[i], float(sims[i])))
        if len(out) == topk:
            break
    return out


def analogy(vocab, W, a, b, c, topk=5):
    v = W[vocab[b]] - W[vocab[a]] + W[vocab[c]]
    return nearest(vocab, W, v, topk=topk, exclude={vocab[a], vocab[b], vocab[c]})
```

Pada vektor Google News 300d yang telah dilatih sebelumnya:

```python
>>> analogy(vocab, W, "man", "king", "woman")
[('queen', 0.71), ('monarch', 0.62), ('princess', 0.59), ...]
```

`king - man + woman = queen`. Bukan karena model tahu apa itu kerajaan. Melainkan karena vektor `(king - man)` menangkap sesuatu seperti "kerajaan", dan menambahkannya pada `woman` mendarat di dekat wilayah perempuan-kerajaan.

## Pakai Sendiri

Menulis Word2Vec dari awal adalah untuk belajar. NLP produksi memakai `gensim`.

```python
from gensim.models import Word2Vec

sentences = [
    ["the", "cat", "sat", "on", "the", "mat"],
    ["the", "dog", "ran", "across", "the", "room"],
]

model = Word2Vec(
    sentences,
    vector_size=100,
    window=5,
    min_count=1,
    sg=1,
    negative=5,
    workers=4,
    epochs=30,
)

print(model.wv["cat"])
print(model.wv.most_similar("cat", topn=3))
```

Untuk pekerjaan sungguhan, Anda hampir tidak pernah melatih Word2Vec sendiri. Anda mengunduh vektor yang telah dilatih sebelumnya.

- **GloVe** — pendekatan faktorisasi matriks kejadian-bersama milik Stanford. Checkpoint 50d, 100d, 200d, 300d. Cakupan umum yang baik. Pelajaran 04 membahas GloVe secara khusus.
- **fastText** — perluasan Word2Vec dari Facebook yang menyematkan n-gram karakter. Menangani kata di luar kosakata dengan menyusun subkata. Pelajaran 04.
- **Word2Vec pra-terlatih pada Google News** — 300d, kosakata 3 juta kata, diterbitkan 2013. Masih diunduh setiap hari.

### Kapan Word2Vec masih menang di tahun 2026

- Pengambilan informasi spesifik-domain yang ringan. Latih pada abstrak medis dalam satu jam di laptop, dapatkan vektor khusus yang tidak ditangkap model umum mana pun.
- Rekayasa fitur bergaya analogi. `gender_vector = mean(man - woman pairs)`. Kurangkan dari kata lain untuk mendapatkan sumbu netral-gender. Masih dipakai dalam penelitian keadilan.
- Interpretabilitas. 100d cukup kecil untuk dipetakan lewat PCA atau t-SNE dan benar-benar melihat klaster terbentuk.
- Di mana pun inferensi harus berjalan di perangkat tanpa GPU. Pengambilan Word2Vec cukup mengambil satu baris.

### Di mana Word2Vec gagal

Dinding polisemi. `bank` hanya punya satu vektor. `river bank` dan `financial bank` berbagi vektor itu. `table` (lembar kerja versus furnitur) juga berbagi. Klasifier di hilir tidak dapat membedakan makna-makna itu dari vektornya.

Embedding kontekstual (ELMo, BERT, semua transformer setelahnya) memecahkan ini dengan menghasilkan vektor berbeda untuk setiap kemunculan kata berdasarkan konteks di sekitarnya. Itulah lompatan dari Word2Vec ke BERT: dari statis ke kontekstual. Fase 7 membahas bagian transformer.

Masalah kata di luar kosakata adalah kegagalan lainnya. Word2Vec tidak pernah melihat `Zoomer-approved` jika tidak ada dalam data latih. Tidak ada fallback. fastText memperbaikinya dengan komposisi subkata (pelajaran 04).

## Kirim Hasilnya

Simpan sebagai `outputs/skill-embedding-probe.md`:

```markdown
---
name: embedding-probe
description: Periksa model word2vec. Jalankan analogi, cari tetangga, diagnosis kualitas.
version: 1.0.0
phase: 5
lesson: 03
tags: [nlp, embeddings, debugging]
---

Anda menyelidiki embedding kata yang telah dilatih untuk memverifikasi bahwa ia berfungsi. Diberi objek `gensim.models.KeyedVectors` dan kosakata, Anda menjalankan:

1. Tiga uji analogi kanonik. `king : man :: queen : woman`. `paris : france :: tokyo : japan`. `walking : walked :: swimming : ?`. Laporkan hasil top-1 dan kosinusnya.
2. Lima uji tetangga terdekat pada kata-kata spesifik-domain yang diberikan pengguna. Cetak 5 tetangga teratas beserta kosinusnya.
3. Satu pemeriksaan simetri. `similarity(a, b) == similarity(b, a)` dalam batas presisi floating-point.
4. Satu pemeriksaan degenerasi. Jika ada embedding dengan norma di bawah 0,01 atau di atas 100, model memiliki bug pelatihan. Tandai.

Tolak menyatakan sebuah model bagus hanya berdasarkan akurasi analogi. Benchmark analogi bisa dimanipulasi dan tidak transfer ke tugas hilir. Rekomendasikan evaluasi intrinsik + hilir secara bersamaan.
```

## Latihan

1. **Mudah.** Jalankan loop pelatihan pada korpus sangat kecil (20 kalimat tentang kucing dan anjing). Setelah 200 epoch, verifikasi bahwa `nearest(vocab, W, W[vocab["cat"]])` mengembalikan `dog` dalam 3 teratas. Jika tidak, tambah epoch atau kosakata.
2. **Sedang.** Tambahkan subsampling kata-kata yang sering muncul. Kata dengan frekuensi di atas `10^-5` dibuang dari pasangan pelatihan dengan probabilitas sebanding dengan frekuensinya. Ukur efeknya pada kemiripan kata langka.
3. **Sulit.** Latih sebuah model pada korpus 20 Newsgroups. Hitung dua sumbu bias: `he - she` dan `doctor - nurse`. Proyeksikan kata-kata pekerjaan ke kedua sumbu itu. Laporkan pekerjaan mana yang memiliki celah bias terbesar. Ini adalah jenis penyelidikan yang dipakai peneliti keadilan.

## Istilah Kunci

| Istilah | Yang orang katakan | Yang sebenarnya dimaksud |
|---------|--------------------|--------------------------|
| Embedding kata | Kata sebagai vektor | Representasi padat berdimensi rendah (biasanya 100-300) yang dipelajari dari konteks. |
| Skip-gram | Trik Word2Vec | Prediksi kata konteks dari kata pusat. Lebih lambat dari CBOW, lebih baik untuk kata langka. |
| Negative sampling | Pintasan pelatihan | Mengganti softmax atas seluruh kosakata dengan klasifikasi biner terhadap `k` kata acak. |
| Embedding statis | Satu vektor per kata | Vektor yang sama terlepas dari konteks. Gagal pada polisemi. |
| Embedding kontekstual | Vektor sensitif-konteks | Vektor berbeda untuk setiap kemunculan berdasarkan kata di sekitarnya. Yang dihasilkan transformer. |
| OOV | Di luar kosakata | Kata yang tidak terlihat saat pelatihan. Word2Vec tidak dapat menghasilkan vektor untuk kata ini. |

## Bacaan Lanjutan

- [Mikolov et al. (2013). Distributed Representations of Words and Phrases and their Compositionality](https://arxiv.org/abs/1310.4546) — makalah negative sampling. Pendek dan mudah dibaca.
- [Rong, X. (2014). word2vec Parameter Learning Explained](https://arxiv.org/abs/1411.2738) — turunan gradien yang paling jelas, jika matematika makalah asli terasa padat.
- [Tutorial gensim Word2Vec](https://radimrehurek.com/gensim/models/word2vec.html) — pengaturan pelatihan produksi yang benar-benar berfungsi.
