# Penandaan POS dan Parsing Sintaksis

> Tata bahasa sempat tidak lagi populer. Lalu setiap pipeline LLM perlu memvalidasi ekstraksi terstruktur, dan tata bahasa pun kembali relevan.

**Jenis:** Build
**Bahasa:** Python, TypeScript
**Prasyarat:** Fase 5 · 01 (Pemrosesan Teks), Fase 2 · 14 (Naive Bayes)
**Waktu:** ~45 menit

## Masalah

Pelajaran 01 menjanjikan bahwa lemmatisasi memerlukan tag bagian-bagian bicara (part-of-speech). Tanpa mengetahui bahwa `running` adalah kata kerja, sebuah lemmatizer tidak dapat mereduksinya menjadi `run`. Tanpa mengetahui bahwa `better` adalah kata sifat, lemmatizer tidak dapat mereduksinya menjadi `good`.

Janji itu menyembunyikan satu subbidang penuh. Penandaan bagian-bagian bicara (POS tagging) menetapkan kategori gramatikal. Parsing sintaksis memulihkan struktur pohon kalimat: kata mana yang mengubah kata mana, kata kerja mana yang mengatur argumen mana. NLP klasik menghabiskan dua dekade untuk menyempurnakan keduanya. Lalu deep learning menyatukan keduanya menjadi sebuah tugas klasifikasi token di atas transformer terlatih (pretrained), dan komunitas riset pun beralih.

Bukan komunitas terapan. Setiap pipeline ekstraksi terstruktur masih menggunakan POS dan pohon dependensi di balik layar. JSON yang dihasilkan LLM divalidasi terhadap batasan gramatikal. Sistem tanya-jawab mendekomposisi kueri menggunakan parse dependensi. evaluator kualitas terjemahan mesin memeriksa keselarasan pohon parse.

Perlu diketahui. Pelajaran ini memperkenalkan himpunan tag, baseline, dan titik di mana Anda berhenti mengimplementasikan dari nol dan mulai memanggil spaCy.

## Konsep

**Penandaan POS** memberi label pada setiap token dengan sebuah kategori gramatikal. Himpunan tag **Penn Treebank (PTB)** adalah standar untuk bahasa Inggris. 36 tag dengan perbedaan yang dianggap terlalu teliti oleh pembaca awam: `NN` kata benda tunggal, `NNS` kata benda jamak, `NNP` kata benda proper tunggal, `VBD` kata kerja lampau, `VBZ` kata kerja orang ketiga tunggal kini, dan seterusnya. Himpunan tag **Universal Dependencies (UD)** lebih kasar (17 tag) dan agnostik terhadap bahasa; himpunan ini menjadi standar untuk kerja lintas bahasa.

```
The/DET cats/NOUN were/AUX running/VERB at/ADP 3pm/NOUN ./PUNCT
```

**Parsing sintaksis** menghasilkan sebuah pohon. Dua gaya utama:

- **Parsing konstituen.** Frasa kata benda, frasa kata kerja, dan frasa preposisi bersarang satu di dalam lainnya. Keluarannya adalah pohon kategori bukan-terminal (NP, VP, PP) dengan kata-kata sebagai daun.
- **Parsing dependensi.** Setiap kata memiliki satu kata kepala (head) yang menjadi tempatnya bergantung, diberi label berdasarkan relasi gramatikal. Keluarannya adalah pohon di mana setiap sisi adalah triple (head, dependen, relasi).

Parsing dependensi menang pada dekade 2010-an karena ia digeneralisasi dengan bersih lintas bahasa, terutama bahasa dengan urutan kata bebas.

```
running adalah ROOT
cats adalah nsubj dari running
were adalah aux dari running
at adalah prep dari running
3pm adalah pobj dari at
```

```figure
pos-tagger
```

```figure
dependency-arcs
```

## Bangun Sendiri

### Langkah 1: baseline tag paling sering

Penanda POS paling sederhana yang tetap berfungsi. Untuk setiap kata, prediksi tag yang paling sering muncul saat pelatihan.

```python
from collections import Counter, defaultdict


def train_mft(train_examples):
    word_tag_counts = defaultdict(Counter)
    all_tags = Counter()
    for tokens, tags in train_examples:
        for token, tag in zip(tokens, tags):
            word_tag_counts[token.lower()][tag] += 1
            all_tags[tag] += 1
    word_best = {w: c.most_common(1)[0][0] for w, c in word_tag_counts.items()}
    default_tag = all_tags.most_common(1)[0][0]
    return word_best, default_tag


def predict_mft(tokens, word_best, default_tag):
    return [word_best.get(t.lower(), default_tag) for t in tokens]
```

Pada korpus Brown, baseline ini mencapai akurasi ~85%. Tidak bagus, tetapi merupakan batas bawah yang tidak boleh dilampaui oleh model yang serius.

### Langkah 2: penanda HMM bigram

Modelkan probabilitas gabungan dari urutan tersebut:

```math
P(\text{tags}, \text{words}) = \prod_i P(\text{tag}_i \mid \text{tag}_{i-1}) \cdot P(\text{word}_i \mid \text{tag}_i)
```

Dua tabel: probabilitas transisi (tag diberikan tag sebelumnya) dan probabilitas emisi (kata diberikan tag). Estimasi keduanya dari hitungan dengan perhalusan Laplace. Lakukan dekode dengan Viterbi (pemrograman dinamis atas kisi tag).

```python
import math


def train_hmm(train_examples, alpha=0.01):
    transitions = defaultdict(Counter)
    emissions = defaultdict(Counter)
    tags = set()
    vocab = set()

    for tokens, ts in train_examples:
        prev = "<BOS>"
        for token, tag in zip(tokens, ts):
            transitions[prev][tag] += 1
            emissions[tag][token.lower()] += 1
            tags.add(tag)
            vocab.add(token.lower())
            prev = tag
        transitions[prev]["<EOS>"] += 1

    return transitions, emissions, tags, vocab


def log_prob(table, given, key, smooth_denom, alpha):
    return math.log((table[given].get(key, 0) + alpha) / smooth_denom)


def viterbi(tokens, transitions, emissions, tags, vocab, alpha=0.01):
    tags_list = list(tags)
    n = len(tokens)
    V = [[0.0] * len(tags_list) for _ in range(n)]
    back = [[0] * len(tags_list) for _ in range(n)]

    for j, tag in enumerate(tags_list):
        em_denom = sum(emissions[tag].values()) + alpha * (len(vocab) + 1)
        tr_denom = sum(transitions["<BOS>"].values()) + alpha * (len(tags_list) + 1)
        tr = log_prob(transitions, "<BOS>", tag, tr_denom, alpha)
        em = log_prob(emissions, tag, tokens[0].lower(), em_denom, alpha)
        V[0][j] = tr + em
        back[0][j] = 0

    for i in range(1, n):
        for j, tag in enumerate(tags_list):
            em_denom = sum(emissions[tag].values()) + alpha * (len(vocab) + 1)
            em = log_prob(emissions, tag, tokens[i].lower(), em_denom, alpha)
            best_prev = 0
            best_score = -1e30
            for k, prev_tag in enumerate(tags_list):
                tr_denom = sum(transitions[prev_tag].values()) + alpha * (len(tags_list) + 1)
                tr = log_prob(transitions, prev_tag, tag, tr_denom, alpha)
                score = V[i - 1][k] + tr + em
                if score > best_score:
                    best_score = score
                    best_prev = k
            V[i][j] = best_score
            back[i][j] = best_prev

    last_best = max(range(len(tags_list)), key=lambda j: V[n - 1][j])
    path = [last_best]
    for i in range(n - 1, 0, -1):
        path.append(back[i][path[-1]])
    return [tags_list[j] for j in reversed(path)]
```

HMM bigram pada Brown mencapai akurasi ~93%. Lompatan dari 85% ke 93% terutama berasal dari probabilitas transisi — model mempelajari bahwa `DET NOUN` itu umum dan `NOUN DET` itu langka.

### Langkah 3: mengapa penanda modern mengalahkan ini

Probabilitas transisi + emisi bersifat lokal. Keduanya tidak dapat menangkap bahwa `saw` adalah kata benda dalam "I bought a saw" tetapi kata kerja dalam "I saw the movie." Sebuah CRF dengan fitur sewenang-wenang (akhiran, bentuk kata, kata sebelum dan sesudah, kata itu sendiri) mencapai ~97%. BiLSTM-CRF atau transformer mencapai ~98%+.

Batas atas pada tugas ini ditetapkan oleh ketidaksepakatan anotator. Manusia anotator setuju sekitar 97% dari waktu pada Penn Treebank. Model di atas 98% kemungkinan besar mengalami overfitting pada himpunan uji.

### Langkah 4: sketsa parsing dependensi

Parsing dependensi penuh dari nol berada di luar cakupan; pembahasan buku teks kanonik ada pada Jurafsky dan Martin. Dua keluarga klasik yang perlu diketahui:

- **Parser berbasis transisi** (arc-eager, arc-standard) bertindak seperti parser shift-reduce: mereka membaca token, mendorongnya ke sebuah tumpukan (stack), dan menerapkan aksi reduce yang menciptakan busur. Dekode greedy cepat. Implementasi klasiknya adalah MaltParser. Versi neural modern: parser berbasis transisi dari Chen and Manning.
- **Parser berbasis graf** (algoritma Eisner, biaffine Dozat-Manning) memberi skor pada setiap sisi kepala-dependen yang mungkin dan memilih pohon rentang maksimum. Lebih lambat tetapi lebih akurat.

Untuk sebagian besar kerja terapan, panggil spaCy:

```python
import spacy

nlp = spacy.load("en_core_web_sm")
doc = nlp("The cats were running at 3pm.")
for token in doc:
    print(f"{token.text:10s} tag={token.tag_:5s} pos={token.pos_:6s} dep={token.dep_:10s} head={token.head.text}")
```

```
The        tag=DT    pos=DET    dep=det        head=cats
cats       tag=NNS   pos=NOUN   dep=nsubj      head=running
were       tag=VBD   pos=AUX    dep=aux        head=running
running    tag=VBG   pos=VERB   dep=ROOT       head=running
at         tag=IN    pos=ADP    dep=prep       head=running
3pm        tag=NN    pos=NOUN   dep=pobj       head=at
.          tag=.     pos=PUNCT  dep=punct      head=running
```

Baca kolom `dep` dari bawah ke atas dan struktur gramatikal kalimat pun tampak.

## Gunakan

Setiap pustaka NLP produksi menyertakan penanda POS dan parser dependensi sebagai bagian dari pipeline standar.

- **spaCy** (`en_core_web_sm` / `md` / `lg` / `trf`). Cepat, akurat, terintegrasi dengan tokenisasi + NER + lemmatisasi. `token.tag_` (Penn), `token.pos_` (UD), `token.dep_` (relasi dependensi).
- **Stanford NLP (stanza)**. Penerus CoreNLP dari Stanford. Mutu terkini pada 60+ bahasa.
- **trankit**. Berbasis transformer, akurasi UD yang baik.
- **NLTK**. `pos_tag`. Dapat digunakan, lambat, lebih tua. Cukup untuk pengajaran.

### Di mana ini masih relevan pada 2026

- **Lemmatisasi.** Pelajaran 01 memerlukan POS untuk melematisasi dengan benar. Selalu.
- **Ekstraksi terstruktur dari keluaran LLM.** Validasi bahwa kalimat yang dihasilkan mematuhi batasan gramatikal (mis. keselarasan subjek-kata kerja, pengubah yang wajib ada).
- **Sentimen berbasis aspek.** Parse dependensi memberi tahu Anda kata sifat mana yang mengubah kata benda mana.
- **Pemahaman kueri.** "movies directed by Wes Anderson starring Bill Murray" terdekomposisi menjadi batasan terstruktur melalui parse.
- **Transfer lintas bahasa.** Tag UD dan relasi dependensi bersifat agnostik terhadap bahasa, memungkinkan analisis terstruktur zero-shot pada bahasa baru.
- **Pipeline komputasi rendah.** Jika Anda tidak dapat mengirimkan transformer, POS + parse dependensi + gazetteer membawa Anda cukup jauh.

## Kirim

Simpan sebagai `outputs/skill-grammar-pipeline.md`:

```markdown
---
name: grammar-pipeline
description: Rancang pipeline POS + dependensi klasik untuk sebuah tugas NLP hilir.
version: 1.0.0
phase: 5
lesson: 07
tags: [nlp, pos, parsing]
---

Diberikan sebuah tugas hilir (ekstraksi informasi, validasi penulisan ulang, dekomposisi kueri, lemmatisasi), Anda menghasilkan:

1. Himpunan tag yang digunakan. Penn Treebank untuk pipeline lawas khusus Inggris, Universal Dependencies untuk multilingual atau lintas bahasa.
2. Pustaka. spaCy untuk sebagian besar produksi, stanza untuk multilingual setingkat akademik, trankit untuk akurasi UD tertinggi. Sebutkan ID model spesifiknya.
3. Pola integrasi. Tampilkan 3-5 baris yang memanggil pustaka dan mengonsumsi atribut yang dibutuhkan (`.pos_`, `.dep_`, `.head`).
4. Mode kegagalan yang diuji. Ambiguitas kata benda-kata kerja (`saw`, `book`, `can`) dan ambiguitas pelekatan PP adalah jebakan klasik. Ambil sampel 20 keluaran dan periksa secara visual.

Tolak merekomendasikan parser buatan sendiri. Membangun parser dari nol adalah proyek riset, bukan tugas aplikasi. Tandai sebagai rapuh setiap pipeline yang mengonsumsi tag POS tanpa menangani varian huruf kecil/huruf besar.
```

## Latihan

1. **Mudah.** Dengan menggunakan baseline tag paling sering pada korpus bertag kecil (mis. subhimpunan Brown dari NLTK), ukur akurasi pada kalimat yang disisihkan. Verifikasi hasil ~85%.
2. **Sedang.** Latih HMM bigram di atas dan laporkan presisi/recall per tag. Tag mana yang paling sering tertukar oleh HMM?
3. **Sulit.** Gunakan parse dependensi spaCy untuk mengekstrak triple subjek-kata kerja-objek dari sampel 1000 kalimat. Evaluasi pada 50 triple yang diberi label manual. Dokumentasikan di mana ekstraksi gagal (sering kali pada kalimat pasif, koordinasi, dan subjek yang dielipsis).

## Istilah Kunci

| Istilah | Yang orang katakan | Yang sebenarnya dimaksud |
|---------|--------------------|--------------------------|
| Tag POS | Jenis kata | Kategori gramatikal. PTB punya 36; UD punya 17. |
| Penn Treebank | Himpunan tag standar | Khusus bahasa Inggris. Kata kerja dengan waktu yang butir halus dan bilangan kata benda. |
| Universal Dependencies | Himpunan tag multilingual | Lebih kasar dari PTB; netral terhadap bahasa; standar untuk kerja lintas bahasa. |
| Parse dependensi | Pohon kalimat | Setiap kata memiliki satu head, setiap sisi memiliki relasi gramatikal. |
| Viterbi | Pemrograman dinamis | Menemukan urutan tag berprobabilitas tertinggi given emisi dan transisi. |

## Bacaan Lanjutan

- [Jurafsky and Martin — Speech and Language Processing, bab 8 dan 18](https://web.stanford.edu/~jurafsky/slp3/) — pembahasan buku teks kanonik tentang POS dan parsing.
- [Proyek Universal Dependencies](https://universaldependencies.org/) — himpunan tag lintas bahasa dan kumpulan treebank yang digunakan oleh setiap parser multilingual.
- [Panduan fitur linguistik spaCy](https://spacy.io/usage/linguistic-features) — referensi praktis untuk setiap atribut yang diekspos pada `Token`.
- [Chen and Manning (2014). A Fast and Accurate Dependency Parser using Neural Networks](https://nlp.stanford.edu/pubs/emnlp2014-depparser.pdf) — makalah yang membawa parser neural ke arus utama.
