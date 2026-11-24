# GloVe, FastText, dan Embedding Subkata

> Word2Vec melatih satu embedding per kata. GloVe memfaktorkan matriks kejadian-bersama. FastText menyematkan bagian-bagiannya. BPE menjembatani ke transformer.

**Jenis:** Build
**Bahasa:** Python, TypeScript
**Prasyarat:** Fase 5 · 03 (Word2Vec dari Awal)
**Waktu:** ~45 menit

## Masalahnya

Word2Vec meninggalkan dua pertanyaan terbuka.

Pertama, ada jalur penelitian paralel yang memfaktorkan langsung matriks kejadian-bersama (LSA, HAL), alih-alih melakukan pembaruan skip-gram daring. Apakah pendekatan iteratif Word2Vec secara fundamental lebih baik, atau perbedaan itu artefak dari cara kedua metode menangani jumlah kemunculan? **GloVe** menjawabnya: faktorisasi matriks dengan loss yang dipilih dengan cermat menyamai atau mengalahkan Word2Vec, dengan biaya pelatihan lebih rendah.

Kedua, tidak ada metode yang punya cerita untuk kata-kata yang belum pernah dilihat. `Zoomer-approved`, `dogecoin`, nama diri apa pun yang diciptakan minggu lalu, setiap bentuk infleksi dari akar kata langka. **FastText** memperbaiki ini dengan menyematkan n-gram karakter: sebuah kata adalah jumlah dari bagian-bagiannya, termasuk morfem, sehingga bahkan kata di luar kosakata mendapat vektor yang masuk akal.

Ketiga, begitu transformer muncul, pertanyaannya bergeser lagi. Kosakata level kata mentok sekitar satu juta entri; bahasa sungguhan lebih terbuka daripada itu. **Byte-pair encoding (BPE)** dan kerabatnya memecahkan ini dengan mempelajari kosakata unit subkata yang sering muncul dan mencakup semuanya. Setiap tokenizer modern untuk setiap LLM modern adalah tokenizer subkata.

Pelajaran ini menelusuri ketiganya, lalu menjelaskan mana yang harus dipakai kapan.

## Konsepnya

**GloVe (Global Vectors).** Bangun matriks kejadian-bersama kata-kata `X` di mana `X[i][j]` adalah seberapa sering kata `j` muncul dalam konteks kata `i`. Latih vektor sedemikian rupa sehingga `v_i · v_j + b_i + b_j ≈ log(X[i][j])`. Boboti loss agar pasangan yang sering tidak mendominasi. Selesai.

**FastText.** Sebuah kata adalah jumlah n-gram karakternya plus kata itu sendiri. `where` menjadi `<wh, whe, her, ere, re>, <where>`. Vektor kata adalah jumlah dari vektor-vektor komponen itu. Latih seperti Word2Vec. Keuntungannya: kata yang belum terlihat (`whereupon`) disusun dari n-gram yang dikenal.

**BPE (Byte-Pair Encoding).** Mulai dengan kosakata berupa byte (atau karakter) tunggal. Hitung setiap pasangan berurutan dalam korpus. Gabungkan pasangan paling sering menjadi token baru. Ulangi sebanyak `k` iterasi. Hasilnya: kosakata berupa simbol dasar plus `k` penggabungan (`k + 256` token jika mulai dari byte) di mana urutan yang sering (`ing`, `tion`, `the`) menjadi token tunggal dan kata langka dipecah menjadi bagian-bagian yang dikenal. Setiap kalimat tersusun menjadi sesuatu.

```figure
n5-subword-merge
```

## Bangun Sendiri

### GloVe: memfaktorkan matriks kejadian-bersama

```python
import numpy as np
from collections import Counter


def build_cooccurrence(docs, window=5):
    pair_counts = Counter()
    vocab = {}
    for doc in docs:
        for token in doc:
            if token not in vocab:
                vocab[token] = len(vocab)
    for doc in docs:
        indexed = [vocab[t] for t in doc]
        for i, center in enumerate(indexed):
            for j in range(max(0, i - window), min(len(indexed), i + window + 1)):
                if i != j:
                    distance = abs(i - j)
                    pair_counts[(center, indexed[j])] += 1.0 / distance
    return vocab, pair_counts


def glove_train(vocab, pair_counts, dim=16, epochs=100, lr=0.05, x_max=100, alpha=0.75, seed=0):
    n = len(vocab)
    rng = np.random.default_rng(seed)
    W = rng.normal(0, 0.1, size=(n, dim))
    W_tilde = rng.normal(0, 0.1, size=(n, dim))
    b = np.zeros(n)
    b_tilde = np.zeros(n)

    for epoch in range(epochs):
        for (i, j), x_ij in pair_counts.items():
            weight = (x_ij / x_max) ** alpha if x_ij < x_max else 1.0
            diff = W[i] @ W_tilde[j] + b[i] + b_tilde[j] - np.log(x_ij)
            coef = weight * diff

            grad_W_i = coef * W_tilde[j]
            grad_W_tilde_j = coef * W[i]
            W[i] -= lr * grad_W_i
            W_tilde[j] -= lr * grad_W_tilde_j
            b[i] -= lr * coef
            b_tilde[j] -= lr * coef

    return W + W_tilde
```

Dua bagian yang bergerak patut disebut. Fungsi pembobotan `f(x) = (x/x_max)^alpha` menurunkan bobot pasangan yang sangat sering (seperti `(the, and)`) agar tidak mendominasi loss. Embedding akhir adalah jumlah dari tabel `W` (pusat) dan `W_tilde` (konteks). Menjumlahkan keduanya adalah trik yang dipublikasikan dan cenderung mengungguli penggunaan salah satu saja.

### FastText: embedding sadar-subkata

```python
def char_ngrams(word, n_min=3, n_max=6):
    wrapped = f"<{word}>"
    grams = {wrapped}
    for n in range(n_min, n_max + 1):
        for i in range(len(wrapped) - n + 1):
            grams.add(wrapped[i:i + n])
    return grams
```

```python
>>> char_ngrams("where")
{'<where>', '<wh', 'whe', 'her', 'ere', 're>', '<whe', 'wher', 'here', 'ere>', '<wher', 'where', 'here>'}
```

Setiap kata direpresentasikan oleh himpunan n-gram-nya (biasanya 3 hingga 6 karakter). Embedding kata adalah jumlah dari embedding n-gram-nya. Untuk pelatihan skip-gram, colok ini di tempat Word2Vec memakai satu vektor.

```python
def fasttext_vector(word, ngram_table):
    grams = char_ngrams(word)
    vecs = [ngram_table[g] for g in grams if g in ngram_table]
    if not vecs:
        return None
    return np.sum(vecs, axis=0)
```

Untuk kata yang belum terlihat, Anda tetap mendapat vektor selama beberapa n-gram-nya dikenal. `whereupon` berbagi `<wh`, `her`, `ere`, dan `<where` dengan `where`, sehingga keduanya mendarat berdekatan.

### BPE: kosakata subkata yang dipelajari

```python
def learn_bpe(corpus, k_merges):
    vocab = Counter()
    for word, freq in corpus.items():
        tokens = tuple(word) + ("</w>",)
        vocab[tokens] = freq

    merges = []
    for _ in range(k_merges):
        pair_freq = Counter()
        for tokens, freq in vocab.items():
            for a, b in zip(tokens, tokens[1:]):
                pair_freq[(a, b)] += freq
        if not pair_freq:
            break
        best = pair_freq.most_common(1)[0][0]
        merges.append(best)

        new_vocab = Counter()
        for tokens, freq in vocab.items():
            new_tokens = []
            i = 0
            while i < len(tokens):
                if i + 1 < len(tokens) and (tokens[i], tokens[i + 1]) == best:
                    new_tokens.append(tokens[i] + tokens[i + 1])
                    i += 2
                else:
                    new_tokens.append(tokens[i])
                    i += 1
            new_vocab[tuple(new_tokens)] = freq
        vocab = new_vocab
    return merges


def apply_bpe(word, merges):
    tokens = list(word) + ["</w>"]
    for a, b in merges:
        new_tokens = []
        i = 0
        while i < len(tokens):
            if i + 1 < len(tokens) and tokens[i] == a and tokens[i + 1] == b:
                new_tokens.append(a + b)
                i += 2
            else:
                new_tokens.append(tokens[i])
                i += 1
        tokens = new_tokens
    return tokens
```

```python
>>> corpus = Counter({"low": 5, "lower": 2, "newest": 6, "widest": 3})
>>> merges = learn_bpe(corpus, k_merges=10)
>>> apply_bpe("lowest", merges)
['low', 'est</w>']
```

Iterasi pertama menggabungkan pasangan berurutan paling umum. Setelah iterasi yang cukup, substring yang sering (`low`, `est`, `tion`) menjadi token tunggal dan kata langka terpecah dengan rapi.

Tokenizer GPT / BERT / T5 sungguhan mempelajari 30 ribu-100 ribu penggabungan. Hasilnya: teks apa pun tersusun menjadi urutan ID yang dikenal dengan panjang terbatas, tidak ada OOV sama sekali.

## Pakai Sendiri

Dalam praktiknya, Anda jarang melatih salah satu dari ini sendiri. Anda memuat checkpoint yang telah dilatih sebelumnya.

```python
import fasttext.util
fasttext.util.download_model("en", if_exists="ignore")
ft = fasttext.load_model("cc.en.300.bin")
print(ft.get_word_vector("whereupon").shape)
print(ft.get_word_vector("zoomerapproved").shape)
```

Untuk tokenisasi subkata bergaya BPE di era transformer:

```python
from transformers import AutoTokenizer

tok = AutoTokenizer.from_pretrained("gpt2")
print(tok.tokenize("unbelievably tokenized"))
```

```
['un', 'bel', 'iev', 'ably', 'Ġtoken', 'ized']
```

Awalan `Ġ` menandai batas kata (konvensi GPT-2). Setiap tokenizer modern adalah varian BPE, WordPiece (BERT), atau SentencePiece (T5, LLaMA).

### Kapan memilih mana

| Situasi | Pilih |
|---------|-------|
| Vektor kata umum pra-terlatih, tidak perlu menangani OOV | GloVe 300d |
| Vektor kata umum pra-terlatih, harus menangani salah eja / neologisme / bahasa dengan morfologi kaya | FastText |
| Apa pun yang masuk ke transformer (pelatihan maupun inferensi) | Tokenizer yang disertakan model. Jangan pernah menggantinya. |
| Melatih model bahasa Anda sendiri dari awal | Latih tokenizer BPE atau SentencePiece pada korpus Anda terlebih dahulu |
| Klasifikasi teks produksi dengan model linear | Tetap TF-IDF. Pelajaran 02. |

## Kirim Hasilnya

Simpan sebagai `outputs/skill-embeddings-picker.md`:

```markdown
---
name: tokenizer-picker
description: Pilih pendekatan tokenisasi untuk model bahasa atau pipeline teks baru.
version: 1.0.0
phase: 5
lesson: 04
tags: [nlp, tokenization, embeddings]
---

Diberi deskripsi tugas dan dataset, keluarkan:

1. Strategi tokenisasi (level kata, BPE, WordPiece, SentencePiece, BPE level byte). Alasan dalam satu kalimat.
2. Target ukuran kosakata (mis. 32k untuk LM bahasa Inggris saja, 64k-100k untuk multibahasa).
3. Panggilan pustaka dengan perintah pelatihan yang persis. Sebutkan pustakanya. Kutip argumennya.
4. Satu jebakan keterulangan. Ketidakcocokan tokenizer-model adalah bug produksi senyap yang paling umum; sebutkan pasangan mana yang wajib dipakai bersama.

Tolak merekomendasikan pelatihan tokenizer kustom ketika pengguna melakukan fine-tuning LLM pra-terlatih. Tolak merekomendasikan tokenisasi level kata untuk model apa pun yang menargetkan inferensi produksi. Tandai korpus non-Inggris atau multi-aksara sebagai memerlukan SentencePiece dengan fallback byte.
```

## Latihan

1. **Mudah.** Jalankan `char_ngrams("playing")` dan `char_ngrams("played")`. Hitung tumpang tindih Jaccard dari kedua himpunan n-gram. Anda akan melihat bagian bersama yang substansial (`pla`, `lay`, `play`), itulah sebabnya FastText transfer dengan baik antarvariasi morfologis.
2. **Sedang.** Kembangkan `learn_bpe` untuk melacak pertumbuhan kosakata. Petakan token per karakter korpus sebagai fungsi jumlah penggabungan. Anda akan melihat kompresi cepat pada awalnya, lalu mendekati ~2-3 karakter per token.
3. **Sulit.** Latih BPE 1000 penggabungan pada seluruh karya Shakespeare. Bandingkan tokenisasi kata umum versus nama diri langka. Ukur rata-rata token per kata sebelum dan sesudah. Tulis apa yang mengejutkan Anda.

## Istilah Kunci

| Istilah | Yang orang katakan | Yang sebenarnya dimaksud |
|---------|--------------------|--------------------------|
| Matriks kejadian-bersama | Tabel frekuensi kata-kata | `X[i][j]` = seberapa sering kata `j` muncul dalam jendela di sekitar kata `i`. |
| Subkata | Bagian dari sebuah kata | N-gram karakter (FastText) atau token yang dipelajari (BPE/WordPiece/SentencePiece). |
| BPE | Byte-pair encoding | Penggabungan iteratif pasangan berurutan paling sering sampai kosakata mencapai ukuran target. |
| OOV | Di luar kosakata | Kata yang belum pernah dilihat model. Word2Vec/GloVe gagal. FastText dan BPE menangani. |
| BPE level byte | BPE pada byte mentah | Skema GPT-2. Kosakata diawali 256 byte, sehingga tidak ada yang pernah OOV. |

## Bacaan Lanjutan

- [Pennington, Socher, Manning (2014). GloVe: Global Vectors for Word Representation](https://nlp.stanford.edu/pubs/glove.pdf) — makalah GloVe, tujuh halaman, masih turunan loss terbaik.
- [Bojanowski et al. (2017). Enriching Word Vectors with Subword Information](https://arxiv.org/abs/1607.04606) — FastText.
- [Sennrich, Haddow, Birch (2016). Neural Machine Translation of Rare Words with Subword Units](https://arxiv.org/abs/1508.07909) — makalah yang memperkenalkan BPE ke NLP modern.
- [Ringkasan tokenizer Hugging Face](https://huggingface.co/docs/transformers/tokenizer_summary) — bagaimana BPE, WordPiece, dan SentencePiece benar-benar berbeda dalam praktik.
