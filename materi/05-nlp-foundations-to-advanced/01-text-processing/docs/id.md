# Pemrosesan Teks — Tokenisasi, Stemming, Lemmatization

> Bahasa itu kontinu. Model itu diskrit. Praproses adalah jembatannya.

**Jenis:** Build
**Bahasa:** Python, TypeScript
**Prasyarat:** Fase 2 · 14 (Naive Bayes)
**Waktu:** ~45 menit

## Masalah

Model tidak bisa membaca "The cats were running." Model membaca bilangan bulat.

Setiap sistem NLP dibuka dengan tiga pertanyaan yang sama. Di mana sebuah kata dimulai. Apa akar katanya. Bagaimana kita memperlakukan "run", "running", "ran" sebagai hal yang sama saat itu membantu, dan sebagai hal yang berbeda saat tidak.

Salah tokenisasi dan model belajar dari sampah. Jika tokenizer Anda memperlakukan `don't` sebagai satu token tetapi `do n't` sebagai dua, distribusi pelatihan terbelah. Jika stemmer menyamakan `organization` dan `organ` menjadi stemming yang sama, pemodelan topik mati. Jika lemmatizer membutuhkan konteks kelas kata tetapi Anda tidak meneruskannya, kata kerja diperlakukan sebagai kata benda.

Pelajaran ini membangun tiga langkah praproses dari awal, lalu menunjukkan bagaimana NLTK dan spaCy melakukan pekerjaan yang sama agar Anda bisa melihat trade-off-nya.

## Konsep

Tiga operasi. Masing-masing punya tugas dan mode kegagalan.

**Tokenisasi** memecah string menjadi token. Kata "token" sengaja dibuat kabur karena tingkat granularitas yang tepat bergantung pada tugas. Tingkat kata untuk NLP klasik. Subkata untuk transformer. Karakter untuk bahasa tanpa spasi.

**Stemming** memotong akhiran dengan aturan. Cepat, agresif, bodoh. `running -> run`. `organization -> organ`. Yang kedua itulah mode kegagalannya.

**Lemmatization** mereduksi kata menjadi bentuk kamus menggunakan pengetahuan tata bahasa. Lebih lambat, akurat, membutuhkan tabel lookup atau analis morfologi. `ran -> run` (perlu tahu bahwa "ran" adalah bentuk lampau dari "run"). `better -> good` (perlu tahu bentuk perbandingan).

Aturan praktis. Lakukan stemming saat kecepatan penting dan Anda bisa menoleransi noise (indeks pencarian, klasifikasi kasar). Lakukan lemmatization saat makna penting (tanya jawab, pencarian semantik, apa pun yang akan dibaca pengguna).

```figure
edit-distance
```

## Bangun

### Langkah 1: tokenizer kata berbasis regex

Tokenizer berguna paling sederhana memecah pada karakter bukan-alfanumerik sambil mempertahankan tanda baca sebagai token tersendiri. Tidak sempurna, tidak final, tetapi berjalan dalam satu baris.

```python
import re

def tokenize(text):
    return re.findall(r"[A-Za-z]+(?:'[A-Za-z]+)?|[0-9]+|[^\sA-Za-z0-9]", text)
```

Tiga pola berurutan berdasarkan prioritas. Kata dengan apostrof dalam opsional (`don't`, `it's`). Bilangan murni. Semua karakter tunggal bukan-spasi-bukan-alfanumerik sebagai token mandiri (tanda baca).

```python
>>> tokenize("The cats weren't running at 3pm.")
['The', 'cats', "weren't", 'running', 'at', '3', 'pm', '.']
```

Mode kegagalan yang perlu dicatat. `3pm` terpecah menjadi `['3', 'pm']` karena kita bergantian antara deretan huruf dan deretan angka. Cukup baik untuk sebagian besar tugas. URL, email, dan hashtag semuanya rusak. Untuk produksi, tambahkan pola khusus sebelum pola umum.

### Langkah 2: stemmer Porter (langkah 1a saja)

Algoritma Porter lengkap memiliki lima fase aturan. Langkah 1a saja sudah mencakup akhiran bahasa Inggris paling sering dan mengajarkan polanya.

```python
def stem_step_1a(word):
    if word.endswith("sses"):
        return word[:-2]
    if word.endswith("ies"):
        return word[:-2]
    if word.endswith("ss"):
        return word
    if word.endswith("s") and len(word) > 1:
        return word[:-1]
    return word
```

```python
>>> [stem_step_1a(w) for w in ["caresses", "ponies", "caress", "cats"]]
['caress', 'poni', 'caress', 'cat']
```

Baca aturan dari atas ke bawah. Aturan `ies -> i` lah yang menyebabkan `ponies -> poni`, bukan `pony`. Porter asli memiliki langkah 1b yang memperbaikinya. Aturan saling bersaing. Aturan lebih awal menang. Urutan lebih penting daripada aturan tunggal mana pun.

### Langkah 3: lemmatizer berbasis tabel lookup

Lemmatization yang sesungguhnya membutuhkan morfologi. Versi pengajaran yang bisa dikelola memakai tabel lemma kecil dan fallback.

```python
LEMMMA_TABLE = {
    ("running", "VERB"): "run",
    ("ran", "VERB"): "run",
    ("runs", "VERB"): "run",
    ("better", "ADJ"): "good",
    ("best", "ADJ"): "good",
    ("cats", "NOUN"): "cat",
    ("cat", "NOUN"): "cat",
    ("were", "VERB"): "be",
    ("was", "VERB"): "be",
    ("is", "VERB"): "be",
}

def lemmatize(word, pos):
    key = (word.lower(), pos)
    if key in LEMMA_TABLE:
        return LEMMA_TABLE[key]
    if pos == "VERB" and word.endswith("ing"):
        return word[:-3]
    if pos == "NOUN" and word.endswith("s"):
        return word[:-1]
    return word.lower()
```

```python
>>> lemmatize("running", "VERB")
'run'
>>> lemmatize("cats", "NOUN")
'cat'
>>> lemmatize("better", "ADJ")
'good'
>>> lemmatize("watched", "VERB")
'watched'
```

Kasus terakhir adalah momen pengajaran yang kunci. `watched` tidak ada dalam tabel kita dan fallback kita hanya menangani `ing`. Lemmatization sesungguhnya mencakup `ed`, kata kerja tidak beraturan, adjektiva perbandingan, kata jamak dengan perubahan bunyi (`children -> child`). Inilah sebabnya sistem produksi menggunakan WordNet, morphologizer spaCy, atau analis morfologi lengkap.

### Langkah 4: merangkainya

```python
def preprocess(text, pos_tagger=None):
    tokens = tokenize(text)
    stems = [stem_step_1a(t.lower()) for t in tokens]
    tags = pos_tagger(tokens) if pos_tagger else [(t, "NOUN") for t in tokens]
    lemmas = [lemmatize(word, pos) for word, pos in tags]
    return {"tokens": tokens, "stems": stems, "lemmas": lemmas}
```

Bagian yang masih hilang adalah penanda POS. Fase 5 · 07 (POS Tagging) membangunnya. Untuk saat ini, tetapkan semuanya ke `NOUN` dan akui keterbatasannya.

## Gunakan

NLTK dan spaCy menyediakan versi produksinya. Masing-masing beberapa baris.

### NLTK

```python
import nltk
nltk.download("punkt_tab")
nltk.download("wordnet")
nltk.download("averaged_perceptron_tagger_eng")

from nltk.tokenize import word_tokenize
from nltk.stem import PorterStemmer, WordNetLemmatizer
from nltk import pos_tag

text = "The cats were running."
tokens = word_tokenize(text)
stems = [PorterStemmer().stem(t) for t in tokens]
lemmatizer = WordNetLemmatizer()
tagged = pos_tag(tokens)


def nltk_pos_to_wordnet(tag):
    if tag.startswith("V"):
        return "v"
    if tag.startswith("J"):
        return "a"
    if tag.startswith("R"):
        return "r"
    return "n"


lemmas = [lemmatizer.lemmatize(t, nltk_pos_to_wordnet(tag)) for t, tag in tagged]
```

`word_tokenize` menangani kontraksi, Unicode, dan kasus tepi yang terlewat oleh regex Anda. `PorterStemmer` menjalankan kelima fase. `WordNetLemmatizer` membutuhkan tag POS yang diterjemahkan dari skema Penn Treebank NLTK ke himpunan singkatan WordNet. Rangkaian terjemahan di atas adalah bagian yang kebanyakan tutorial lewatkan.

### spaCy

```python
import spacy

nlp = spacy.load("en_core_web_sm")
doc = nlp("The cats were running.")

for token in doc:
    print(token.text, token.lemma_, token.pos_)
```

```
The      the     DET
cats     cat     NOUN
were     be      AUX
running  run     VERB
.        .       PUNCT
```

spaCy menyembunyikan seluruh pipeline di balik `nlp(text)`. Tokenisasi, penandaan POS, dan lemmatization semuanya berjalan. Lebih cepat daripada NLTK dalam skala besar. Lebih akurat secara bawaan. Trade-off-nya adalah Anda tidak bisa dengan mudah mengganti komponen individual.

### Kapan memilih mana

| Situasi | Pilihan |
|---------|------|
| Pengajaran, riset, mengganti komponen | NLTK |
| Produksi, banyak bahasa, kecepatan penting | spaCy |
| Pipeline transformer (Anda tetap akan melakukan tokenisasi dengan tokenizer model) | Gunakan `tokenizers` / `transformers` dan lewati praproses klasik |

### Dua mode kegagalan yang tidak diperingatkan siapa pun

Kebanyakan tutorial mengajarkan algoritma lalu berhenti. Dua hal akan menggigit pipeline praproses sungguhan, dan keduanya hampir tidak pernah dibahas.

**Pergeseran keterulangan (reproducibility drift).** NLTK dan spaCy mengubah perilaku tokenisasi dan lemmatizer antarversi. Apa yang menghasilkan `['do', "n't"]` pada spaCy 2.x mungkin menghasilkan `["don't"]` pada 3.x. Model Anda dilatih pada satu distribusi. Inferensi kini berjalan pada distribusi yang berbeda. Akurasi menurun diam-diam dan tidak ada yang tahu mengapa. Kunci versi pustaka di `requirements.txt`. Tulis uji regresi praproses yang membekukan tokenisasi yang diharapkan dari 20 kalimat sampel. Jalankan pada setiap pemutakhiran.

**Ketidaksesuaian latih/inferensi.** Latih dengan praproses agresif (huruf kecil, penghapusan stopword, stemming), terapkan pada masukan pengguna mentah, lalu saksikan kinerja anjlok. Ini adalah kegagalan NLP produksi paling umum. Jika Anda melakukan praproses saat pelatihan, Anda menjalankan fungsi yang identik saat inferensi. Kirim praproses sebagai fungsi di dalam paket model, bukan sebagai sel notebook yang ditulis ulang oleh tim layanan.

## Kirim

Prompt yang dapat digunakan ulang untuk membantu insinyur memilih strategi praproses tanpa membaca tiga buku teks.

Simpan sebagai `outputs/prompt-preprocessing-advisor.md`:

```markdown
---
name: preprocessing-advisor
description: Merekomendasikan konfigurasi tokenisasi, stemming, dan lemmatization untuk suatu tugas NLP.
phase: 5
lesson: 01
---

Anda memberi saran tentang praproses NLP klasik. Diberikan deskripsi tugas, Anda menghasilkan:

1. Pilihan tokenisasi (regex, NLTK word_tokenize, spaCy, atau tokenizer transformer). Jelaskan alasannya.
2. Apakah akan melakukan stemming, lemmatization, keduanya, atau tidak sama sekali. Jelaskan alasannya.
3. Panggilan pustaka spesifik. Sebutkan fungsinya. Kutip terjemahan tag POS jika NLTK terlibat.
4. Satu mode kegagalan yang harus diuji pengguna.

Tolak merekomendasikan stemming untuk teks yang terlihat pengguna. Tolak merekomendasikan lemmatization tanpa tag POS. Tandai masukan bukan bahasa Inggris sebagai memerlukan pipeline berbeda.
```

## Latihan

1. **Mudah.** Perluas `tokenize` agar mempertahankan URL sebagai satu token. Uji: `tokenize("Visit https://example.com today.")` harus menghasilkan satu token URL.
2. **Sedang.** Implementasikan langkah 1b Porter. Jika sebuah kata mengandung vokal dan berakhir dengan `ed` atau `ing`, hapus akhiran itu. Tangani aturan konsonan ganda (`hopping -> hop`, bukan `hopp`).
3. **Sulit.** Bangun lemmatizer yang menggunakan WordNet sebagai tabel lookup tetapi kembali ke stemmer Porter Anda ketika WordNet tidak memiliki entri. Ukur akurasi pada korpus beranotasi dibandingkan dengan WordNet murni dan Porter murni.

## Istilah Kunci

| Istilah | Apa yang orang katakan | Apa artinya sebenarnya |
|--------|-----------------|-----------------------|
| Token | Sebuah kata | Unit apa pun yang dikonsumsi model. Bisa berupa kata, subkata, karakter, atau byte. |
| Stem | Akar kata | Hasil penghapusan akhiran berbasis aturan. Tidak selalu berupa kata nyata. |
| Lemma | Bentuk kamus | Bentuk yang Anda cari di kamus. Memerlukan konteks tata bahasa untuk dihitung dengan benar. |
| Tag POS | Kelas kata | Kategori seperti NOUN, VERB, ADJ. Dibutuhkan untuk melakukan lemmatization secara akurat. |
| Morfologi | Aturan bentuk kata | Bagaimana sebuah kata berubah bentuk berdasarkan kala, jumlah, dan kasus. Lemmatization bergantung padanya. |

## Bacaan Lanjutan

- [Porter, M. F. (1980). An algorithm for suffix stripping](https://tartarus.org/martin/PorterStemmer/def.txt) — makalah aslinya, lima halaman, hingga kini masih penjelasan paling jelas.
- [spaCy 101 — linguistic features](https://spacy.io/usage/linguistic-features) — bagaimana pipeline sungguhan dirangkai.
- [NLTK book, chapter 3](https://www.nltk.org/book/ch03.html) — kasus tepi tokenisasi yang belum Anda pikirkan.
