# Pengenalan Entitas Bernama

> Tarik nama-namanya. Terdengar mudah sampai Anda berurusan dengan batas yang ambigu, entitas bersarang, dan jargon domain.

**Jenis:** Build
**Bahasa:** Python, TypeScript
**Prasyarat:** Fase 5 · 02 (BoW + TF-IDF), Fase 5 · 03 (Word Embedding)
**Waktu:** ~75 menit

## Masalahnya

"Apple sued Google over its iPhone search deal in the US." Lima entitas: Apple (ORG), Google (ORG), iPhone (PRODUCT), search deal (mungkin), US (GPE). Sistem NER yang baik mengekstrak semuanya dengan tipe yang benar. Yang buruk melewatkan iPhone, mengira Apple si buah sebagai Apple si perusahaan, dan memberi label "US" sebagai PERSON.

NER adalah kuda kerja di bawah setiap pipeline ekstraksi terstruktur. Penguraian CV, pemindaian log kepatuhan, anonimisasi rekam medis, pemahaman kueri pencarian, grounding untuk respons chatbot, ekstraksi kontrak hukum. Anda tidak pernah benar-benar melihatnya; Anda selalu bergantung padanya.

Pelajaran ini menelusuri jalur klasik (berbasis aturan, HMM, CRF) menuju jalur modern (BiLSTM-CRF, lalu transformer). Setiap langkah memecahkan keterbatasan spesifik dari langkah sebelumnya. Polanya adalah inti pelajarannya.

## Konsepnya

**Penandaan BIO** (atau BILOU) mengubah ekstraksi entitas menjadi masalah pelabelan sekuens. Beri label setiap token dengan `B-TYPE` (awal entitas), `I-TYPE` (di dalam entitas), atau `O` (di luar entitas mana pun).

```
Apple    B-ORG
sued     O
Google   B-ORG
over     O
its      O
iPhone   B-PRODUCT
search   O
deal     O
in       O
the      O
US       B-GPE
.        O
```

Entitas multi-token dirantai: `New B-GPE`, `York I-GPE`, `City I-GPE`. Model yang memahami BIO dapat mengekstrak span apa pun.

Urutan perkembangan arsitektur:

- **Berbasis aturan.** Regex + pencarian gazetteer. Presisi tinggi pada entitas yang dikenal, cakupan nol pada entitas baru.
- **HMM.** Hidden Markov Model. Probabilitas emisi token apabila tagnya diketahui, probabilitas transisi dari tag ke tag. Dekode Viterbi. Dilatih pada data berlabel.
- **CRF.** Conditional Random Field. Seperti HMM tetapi diskriminatif, sehingga Anda dapat mencampur fitur sembarang (bentuk kata, huruf kapital, kata tetangga). Masih menjadi kuda kerja produksi klasik di tahun 2026 untuk penyebaran dengan sumber daya rendah.
- **BiLSTM-CRF.** Fitur neural alih-alih buatan tangan. LSTM membaca kalimat dua arah, lapisan CRF di atasnya menegakkan urutan label yang konsisten.
- **Berbasis transformer.** Fine-tune BERT dengan head klasifikasi token. Akurasi terbaik. Komputasi terbesar.

```figure
ner-bio-tagging
```

## Bangun Sendiri

### Langkah 1: pembantu penandaan BIO

```python
def spans_to_bio(tokens, spans):
    labels = ["O"] * len(tokens)
    for start, end, label in spans:
        labels[start] = f"B-{label}"
        for i in range(start + 1, end):
            labels[i] = f"I-{label}"
    return labels


def bio_to_spans(tokens, labels):
    spans = []
    current = None
    for i, label in enumerate(labels):
        if label.startswith("B-"):
            if current:
                spans.append(current)
            current = (i, i + 1, label[2:])
        elif label.startswith("I-") and current and current[2] == label[2:]:
            current = (current[0], i + 1, current[2])
        else:
            if current:
                spans.append(current)
                current = None
    if current:
        spans.append(current)
    return spans
```

```python
>>> tokens = ["Apple", "sued", "Google", "over", "iPhone", "sales", "."]
>>> labels = ["B-ORG", "O", "B-ORG", "O", "B-PRODUCT", "O", "O"]
>>> bio_to_spans(tokens, labels)
[(0, 1, 'ORG'), (2, 3, 'ORG'), (4, 5, 'PRODUCT')]
```

### Langkah 2: fitur buatan tangan

Untuk NER klasik (non-neural), fitur adalah segalanya. Yang berguna:

```python
def token_features(token, prev_token, next_token):
    return {
        "lower": token.lower(),
        "is_upper": token.isupper(),
        "is_title": token.istitle(),
        "has_digit": any(c.isdigit() for c in token),
        "suffix_3": token[-3:].lower(),
        "shape": word_shape(token),
        "prev_lower": prev_token.lower() if prev_token else "<BOS>",
        "next_lower": next_token.lower() if next_token else "<EOS>",
    }


def word_shape(word):
    out = []
    for c in word:
        if c.isupper():
            out.append("X")
        elif c.islower():
            out.append("x")
        elif c.isdigit():
            out.append("d")
        else:
            out.append(c)
    return "".join(out)
```

`word_shape("iPhone")` mengembalikan `xXxxxx`. `word_shape("USA-2024")` mengembalikan `XXX-dddd`. Pola huruf kapital adalah sinyal kuat untuk nama diri.

### Langkah 3: baseline sederhana berbasis aturan + kamus

```python
ORG_GAZETTEER = {"Apple", "Google", "Microsoft", "OpenAI", "Meta", "Amazon", "Netflix"}
GPE_GAZETTEER = {"US", "USA", "UK", "India", "Germany", "France"}
PRODUCT_GAZETTEER = {"iPhone", "Android", "Windows", "ChatGPT", "Claude"}


def rule_based_ner(tokens):
    labels = []
    for token in tokens:
        if token in ORG_GAZETTEER:
            labels.append("B-ORG")
        elif token in GPE_GAZETTEER:
            labels.append("B-GPE")
        elif token in PRODUCT_GAZETTEER:
            labels.append("B-PRODUCT")
        else:
            labels.append("O")
    return labels
```

Gazetteer produksi memiliki jutaan entri yang diambil dari Wikipedia dan DBpedia. Cakupannya bagus. Disambiguasi (`Apple` si perusahaan versus si buah) buruk. Itulah sebabnya model statistik menang.

### Langkah 4: langkah CRF (sketsa, bukan implementasi penuh)

CRF penuh dari awal dalam 50 baris tidak mencerahkan tanpa fondasi teori probabilitas. Pakai `sklearn-crfsuite` saja:

```python
import sklearn_crfsuite

def to_features(tokens):
    out = []
    for i, tok in enumerate(tokens):
        prev = tokens[i - 1] if i > 0 else ""
        nxt = tokens[i + 1] if i + 1 < len(tokens) else ""
        out.append({
            "word.lower()": tok.lower(),
            "word.isupper()": tok.isupper(),
            "word.istitle()": tok.istitle(),
            "word.isdigit()": tok.isdigit(),
            "word.suffix3": tok[-3:].lower(),
            "word.shape": word_shape(tok),
            "prev.word.lower()": prev.lower(),
            "next.word.lower()": nxt.lower(),
            "BOS": i == 0,
            "EOS": i == len(tokens) - 1,
        })
    return out


crf = sklearn_crfsuite.CRF(algorithm="lbfgs", c1=0.1, c2=0.1, max_iterations=100, all_possible_transitions=True)
X_train = [to_features(s) for s in sentences_tokenized]
crf.fit(X_train, bio_labels_train)
```

`c1` dan `c2` adalah regularisasi L1 dan L2. `all_possible_transitions=True` memungkinkan model mempelajari bahwa urutan ilegal (misalnya `I-ORG` setelah `O`) tidak mungkin, itulah cara CRF menegakkan konsistensi BIO tanpa Anda menulis batasannya sendiri.

### Langkah 5: apa yang ditambahkan BiLSTM-CRF

Fitur menjadi dipelajari. Input: embedding token (GloVe atau fastText). LSTM membaca kiri-ke-kanan dan kanan-ke-kiri. Status tersembunyi yang digabungkan melewati lapisan keluaran CRF. CRF tetap menegakkan konsistensi urutan label; LSTM menggantikan fitur buatan tangan dengan fitur yang dipelajari.

```python
import torch
import torch.nn as nn


class BiLSTM_CRF_Head(nn.Module):
    def __init__(self, vocab_size, embed_dim, hidden_dim, n_labels):
        super().__init__()
        self.embed = nn.Embedding(vocab_size, embed_dim)
        self.lstm = nn.LSTM(embed_dim, hidden_dim, bidirectional=True, batch_first=True)
        self.fc = nn.Linear(hidden_dim * 2, n_labels)

    def forward(self, token_ids):
        e = self.embed(token_ids)
        h, _ = self.lstm(e)
        emissions = self.fc(h)
        return emissions
```

Untuk lapisan CRF, pakai `torchcrf.CRF` (pip install pytorch-crf). Keuntungan dibanding CRF buatan tangan memang terukur, tetapi lebih kecil dari yang Anda kira, kecuali Anda memiliki puluhan ribu kalimat berlabel.

## Pakai Sendiri

spaCy menyediakan NER kelas produksi apa adanya.

```python
import spacy

nlp = spacy.load("en_core_web_sm")
doc = nlp("Apple sued Google over its iPhone search deal in the US.")
for ent in doc.ents:
    print(f"{ent.text:20s} {ent.label_}")
```

```
Apple                ORG
Google               ORG
iPhone               ORG
US                   GPE
```

Perhatikan `iPhone` diberi label `ORG`, bukan `PRODUCT` — model kecil spaCy memiliki cakupan entitas produk yang lemah. Model besar (`en_core_web_lg`) lebih baik. Model transformer (`en_core_web_trf`) lebih baik lagi.

Hugging Face untuk NER berbasis BERT:

```python
from transformers import pipeline

ner = pipeline("ner", model="dslim/bert-base-NER", aggregation_strategy="simple")
print(ner("Apple sued Google over its iPhone in the US."))
```

```
[{'entity_group': 'ORG', 'word': 'Apple', ...},
 {'entity_group': 'ORG', 'word': 'Google', ...},
 {'entity_group': 'MISC', 'word': 'iPhone', ...},
 {'entity_group': 'LOC', 'word': 'US', ...}]
```

`aggregation_strategy="simple"` menggabungkan token `B-X`, `I-X` yang berurutan menjadi sebuah span. Tanpa itu, Anda mendapat label level token dan harus menggabungkannya sendiri.

### NER berbasis LLM (opsi tahun 2026)

NER LLM zero-shot dan few-shot kini bersaing dengan model yang di-fine-tune di banyak domain, dan jauh lebih baik ketika data berlabel langka.

- **Prompting zero-shot.** Beri LLM daftar tipe entitas dan skema contoh. Minta keluaran JSON. Berfungsi apa adanya; akurasinya sedang pada domain baru.
- **Prompting bergaya ZeroTuneBio.** Uraikan tugas menjadi ekstraksi kandidat → penjelasan makna → penilaian → pemeriksaan ulang. Prompt bertahap (bukan sekali jalan) meningkatkan akurasi secara substansial pada NER biomedik. Pola yang sama berfungsi untuk domain hukum, keuangan, dan ilmiah.
- **Prompting dinamis dengan RAG.** Ambil contoh berlabel paling mirip dari himpunan benih beranotasi kecil untuk setiap panggilan inferensi; susun prompt few-shot secara langsung. Pada benchmark tahun 2026, ini menaikkan F1 NER biomedik GPT-4 sebesar 11-12% dibanding prompting statis.
- **Dekomposisi per tipe entitas.** Untuk dokumen panjang, satu panggilan yang mengekstrak semua tipe entitas sekaligus kehilangan recall seiring bertambahnya panjang. Jalankan satu lintasan ekstraksi per tipe entitas. Biaya inferensi lebih tinggi, akurasi jauh lebih tinggi. Ini adalah pola standar untuk catatan klinis dan kontrak hukum.

Rekomendasi produksi per 2026: mulailah dengan baseline LLM zero-shot sebelum Anda mengumpulkan data latih. Sering kali F1-nya sudah cukup bagus sehingga Anda tidak perlu fine-tuning sama sekali.

### Di mana NER klasik masih menang

Bahkan dengan LLM yang tersedia, NER klasik menang ketika:

- Anggaran latensi di bawah 50 ms.
- Anda memiliki ribuan contoh berlabel dan membutuhkan F1 98% atau lebih.
- Domain memiliki ontologi stabil yang dapat ditransfer dengan baik oleh CRF atau BiLSTM pra-terlatih.
- Kendala regulasi mengharuskan model on-prem, non-generatif.

### Di mana ia pecah

- **Pergeseran domain.** NER yang dilatih pada CoNLL dan diterapkan pada kontrak hukum lebih buruk daripada gazetteer. Lakukan fine-tuning pada domain Anda.
- **Entitas bersarang.** "Bank of America Tower" secara bersamaan adalah ORG dan FACILITY. BIO standar tidak dapat merepresentasikan span yang tumpang tindih. Anda memerlukan NER bersarang (model multi-lintasan atau berbasis span).
- **Entitas panjang.** "United States Federal Deposit Insurance Corporation." Model level token kadang memecahnya. Pakai `aggregation_strategy` atau pasca-pemrosesan.
- **Tipe jarang.** Label NER medis seperti DRUG_BRAND, ADVERSE_EVENT, DOSE. Model umum tidak tahu apa-apa. Scispacy dan BioBERT adalah titik awal di sana.

## Kirim Hasilnya

Simpan sebagai `outputs/skill-ner-picker.md`:

```markdown
---
name: ner-picker
description: Pilih pendekatan NER yang tepat untuk tugas ekstraksi tertentu.
version: 1.0.0
phase: 5
lesson: 06
tags: [nlp, ner, extraction]
---

Diberi deskripsi tugas (domain, himpunan label, bahasa, latensi, volume data), keluarkan:

1. Pendekatan. Berbasis aturan + gazetteer, CRF, BiLSTM-CRF, atau fine-tuning transformer.
2. Model awal. Sebutkan namanya (ID model spaCy, ID checkpoint Hugging Face, atau "kustom, dilatih dari awal").
3. Strategi pelabelan. BIO, BILOU, atau berbasis span. Beri alasan satu kalimat.
4. Evaluasi. Pakai `seqeval`. Selalu laporkan F1 level entitas (bukan level token).

Tolak merekomendasikan fine-tuning transformer untuk kurang dari 500 contoh berlabel, kecuali pengguna sudah punya model domain pra-terlatih. Tandai entitas bersarang sebagai memerlukan model berbasis span atau multi-lintasan. Wajibkan audit gazetteer jika pengguna menyebut "skala produksi" sementara label masih sama dengan CoNLL-2003.
```

## Latihan

1. **Mudah.** Terapkan `bio_to_spans` (kebalikan dari `spans_to_bio`) dan verifikasi konsistensi pulang-pergi pada 10 kalimat.
2. **Sedang.** Latih CRF sklearn-crfsuite di atas pada dataset NER bahasa Inggris CoNLL-2003. Laporkan F1 per entitas dengan `seqeval`. Hasil khas: ~84 F1.
3. **Sulit.** Fine-tune `distilbert-base-cased` pada dataset NER spesifik-domain (medis, hukum, atau keuangan). Bandingkan dengan model kecil spaCy. Dokumentasikan pemeriksaan kebocoran data dan tulis apa yang mengejutkan Anda.

## Istilah Kunci

| Istilah | Yang orang katakan | Yang sebenarnya dimaksud |
|---------|--------------------|--------------------------|
| NER | Ekstrak nama | Beri label span token dengan tipe (PERSON, ORG, GPE, DATE, ...). |
| BIO | Skema penandaan | `B-X` memulai, `I-X` melanjutkan, `O` di luar. |
| BILOU | BIO yang lebih baik | Menambah `L-X` (terakhir), `U-X` (satuan) untuk batas yang lebih bersih. |
| CRF | Klasifier terstruktur | Memodelkan transisi antarlabel, bukan hanya emisi. Menegakkan urutan yang valid. |
| NER bersarang | Entitas tumpang tindih | Sebuah span adalah entitas yang berbeda dari sub-span di dalamnya. BIO tidak dapat menyatakannya. |
| F1 level entitas | Metrik NER yang tepat | Span yang diprediksi harus sama persis dengan span yang benar. F1 level token melebih-lebihkan akurasi. |

## Bacaan Lanjutan

- [Lample et al. (2016). Neural Architectures for Named Entity Recognition](https://arxiv.org/abs/1603.01360) — makalah BiLSTM-CRF. Kanonik.
- [Devlin et al. (2018). BERT: Pre-training of Deep Bidirectional Transformers](https://arxiv.org/abs/1810.04805) — memperkenalkan pola klasifikasi token yang menjadi standar.
- [Fitur linguistik spaCy — named entities](https://spacy.io/usage/linguistic-features#named-entities) — referensi praktis untuk setiap atribut pada `Doc.ents` dan `Span`.
- [seqeval](https://github.com/chakki-works/seqeval) — pustaka metrik yang benar. Selalu pakai.
