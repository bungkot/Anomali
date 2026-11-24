# Analisis Sentimen

> Tugas NLP kanonik. Sebagian besar yang perlu Anda tahu tentang klasifikasi teks klasik muncul di sini.

**Jenis:** Build
**Bahasa:** Python, TypeScript
**Prasyarat:** Fase 5 · 02 (BoW + TF-IDF), Fase 2 · 14 (Naive Bayes)
**Waktu:** ~75 menit

## Masalahnya

"The food was not great." Positif atau negatif?

Sentimen terdengar sederhana. Seorang pengulas mengatakan bahwa mereka menyukai atau tidak menyukai sesuatu. Beri label pada kalimatnya. Alasan ia menjadi tugas NLP kanonik adalah karena setiap kasus yang tampak mudah menyembunyikan kasus yang sulit. Negasi membalik makna. Sarkasme membalikkannya. "Not bad at all" adalah positif meski mengandung dua kata berkonotasi negatif. Emoji membawa sinyal lebih kuat daripada teks di sekitarnya. Kosakata domain penting (`tight` dalam ulasan musik berbeda dengan `tight` dalam ulasan fashion).

Sentimen adalah laboratorium kerja untuk NLP klasik. Jika Anda memahami mengapa setiap baseline naif memiliki mode kegagalan tertentu, Anda memahami mengapa setiap model yang lebih kaya diciptakan. Pelajaran ini membangun baseline Naive Bayes dari awal, menambahkan regresi logistik, dan menyebut jebakan yang membuat sentimen produksi menjadi masalah setingkat kepatuhan.

## Konsepnya

Sentimen klasik adalah resep dua langkah.

1. **Representasikan.** Ubah teks menjadi vektor fitur. BoW, TF-IDF, atau n-gram.
2. **Klasifikasikan.** Pasang model linear (Naive Bayes, regresi logistik, SVM) pada contoh berlabel.

Naive Bayes adalah model paling bodoh yang berfungsi. Anggap setiap fitur independen apabila labelnya diketahui. Estimasi `P(word | positive)` dan `P(word | negative)` dari jumlah kemunculan. Saat inferensi, kalikan probabilitasnya. Asumsi independensi yang "naif" itu lucu untuk dipercaya, namun hasilnya mengejutkan kuat. Alasannya: dengan fitur teks yang jarang dan data dalam jumlah sedang, klasifier lebih peduli ke arah mana sebuah kata condong daripada seberapa besar kecenderungannya.

Regresi logistik memperbaiki asumsi independensi. Ia mempelajari satu bobot per fitur, termasuk bobot negatif. `not good` sebagai fitur bigram mendapat bobot negatif. Naive Bayes tidak dapat melakukan itu untuk bigram yang belum pernah diberi label.

```figure
sentiment-logits
```

## Bangun Sendiri

### Langkah 1: mini-dataset sungguhan

```python
POSITIVE = [
    "absolutely loved this movie",
    "beautiful cinematography and a great story",
    "one of the best films of the year",
    "brilliant acting from the lead",
    "heartwarming and funny",
]

NEGATIVE = [
    "boring and far too long",
    "not worth your time",
    "the plot made no sense",
    "terrible acting, awful script",
    "i want my two hours back",
]
```

Sengaja kecil. Pekerjaan sungguhan memakai puluhan ribu contoh (IMDb, SST-2, polaritas Yelp). Matematikanya identik.

### Langkah 2: multinomial Naive Bayes dari awal

```python
import math
from collections import Counter


def train_nb(docs_by_class, vocab, alpha=1.0):
    class_priors = {}
    class_word_probs = {}
    total_docs = sum(len(d) for d in docs_by_class.values())

    for cls, docs in docs_by_class.items():
        class_priors[cls] = len(docs) / total_docs
        counts = Counter()
        for doc in docs:
            for token in doc:
                counts[token] += 1
        total = sum(counts.values()) + alpha * len(vocab)
        class_word_probs[cls] = {
            w: (counts[w] + alpha) / total for w in vocab
        }
    return class_priors, class_word_probs


def predict_nb(doc, class_priors, class_word_probs):
    scores = {}
    for cls in class_priors:
        s = math.log(class_priors[cls])
        for token in doc:
            if token in class_word_probs[cls]:
                s += math.log(class_word_probs[cls][token])
        scores[cls] = s
    return max(scores, key=scores.get)
```

Pemulusan aditif (alpha=1.0) adalah pemulusan Laplace. Tanpa itu, kata yang tidak terlihat dalam suatu kelas memiliki probabilitas nol dan log meledak. `alpha=0.01` umum dipakai dalam praktik. `alpha=1.0` adalah nilai bawaan untuk pengajaran.

### Langkah 3: regresi logistik dari awal

```python
import numpy as np


def sigmoid(x):
    return 1.0 / (1.0 + np.exp(-np.clip(x, -20, 20)))


def train_lr(X, y, epochs=500, lr=0.05, l2=0.01):
    n_features = X.shape[1]
    w = np.zeros(n_features)
    b = 0.0
    for _ in range(epochs):
        logits = X @ w + b
        preds = sigmoid(logits)
        err = preds - y
        grad_w = X.T @ err / len(y) + l2 * w
        grad_b = err.mean()
        w -= lr * grad_w
        b -= lr * grad_b
    return w, b


def predict_lr(X, w, b):
    return (sigmoid(X @ w + b) >= 0.5).astype(int)
```

Regularisasi L2 penting di sini. Fitur teks bersifat jarang; tanpa L2 model menghafal contoh pelatihan. Mulai dari `0.01` lalu setel.

### Langkah 4: menangani negasi (mode kegagalan)

Perhatikan "not good" dan "not bad". Klasifier BoW melihat `{not, good}` dan `{not, bad}` serta belajar dari mana yang lebih sering muncul di data latih. Klasifier bigram melihat `not_good` dan `not_bad` serta mempelajarinya sebagai fitur yang berbeda. Itu biasanya cukup.

Perbaikan yang lebih kasar tetapi tetap berfungsi ketika Anda tidak punya bigram: **cakupan negasi**. Beri awalan `NOT_` pada token setelah kata negasi sampai tanda baca berikutnya.

```python
NEGATION_WORDS = {"not", "no", "never", "nor", "none", "nothing", "neither"}
NEGATION_TERMINATORS = {".", "!", "?", ",", ";"}


def apply_negation(tokens):
    out = []
    negate = False
    for token in tokens:
        if token in NEGATION_TERMINATORS:
            negate = False
            out.append(token)
            continue
        if token in NEGATION_WORDS:
            negate = True
            out.append(token)
            continue
        out.append(f"NOT_{token}" if negate else token)
    return out
```

```python
>>> apply_negation(["not", "good", "at", "all", ".", "but", "funny"])
['not', 'NOT_good', 'NOT_at', 'NOT_all', '.', 'but', 'funny']
```

Sekarang `good` dan `NOT_good` adalah fitur yang berbeda. Klasifier dapat memberi bobot yang berlawanan. Tiga baris praproses, lompatan akurasi yang terukur pada benchmark sentimen.

### Langkah 5: metrik evaluasi yang penting

Akurasi saja menyesatkan jika kelas tidak seimbang. Korpus sentimen sungguhan biasanya 70-80% positif atau 70-80% negatif; klasifier yang selalu menebak kelas mayoritas mendapat akurasi 80% dan tidak berguna. Laporkan semua yang berikut:

- **Presisi dan recall per kelas.** Satu pasangan per kelas. Rata-ratakan secara makro untuk mendapatkan satu angka yang menghormati keseimbangan kelas.
- **Macro-F1 (metrik utama untuk data tidak seimbang).** Rata-rata skor F1 per kelas dengan bobot sama. Pakai ini alih-alih akurasi ketika kelas tidak seimbang.
- **Weighted-F1 (alternatif).** Sama seperti makro tetapi dibobot menurut frekuensi kelas. Laporkan bersama macro-F1 apabila ketidakseimbangannya sendiri memiliki arti bisnis.
- **Matriks kebingungan.** Jumlah mentah. Selalu periksa sebelum mempercayai metrik skalar mana pun; matriks ini mengungkap pasangan kelas apa yang tertukar oleh model.
- **Contoh kesalahan per kelas.** Ambil 5 prediksi salah per kelas. Bacalah. Tidak ada yang dapat menggantikan membaca kesalahan yang sebenarnya.

Untuk data sangat tidak seimbang (rasio > 95-5), laporkan **AUROC** dan **AUPRC** alih-alih akurasi. AUPRC lebih sensitif terhadap kelas minoritas, yang biasanya menjadi perhatian Anda (spam, penipuan, sentimen langka).

**Bug umum yang harus dihindari.** Melaporkan micro-F1 alih-alih macro-F1 pada data tidak seimbang menghasilkan angka yang tampak tinggi karena didominasi kelas mayoritas. Macro-F1 memaksa Anda melihat kinerja kelas minoritas.

```python
def evaluate(y_true, y_pred):
    tp = sum(1 for t, p in zip(y_true, y_pred) if t == 1 and p == 1)
    fp = sum(1 for t, p in zip(y_true, y_pred) if t == 0 and p == 1)
    fn = sum(1 for t, p in zip(y_true, y_pred) if t == 1 and p == 0)
    tn = sum(1 for t, p in zip(y_true, y_pred) if t == 0 and p == 0)
    precision = tp / (tp + fp) if tp + fp else 0
    recall = tp / (tp + fn) if tp + fn else 0
    f1 = 2 * precision * recall / (precision + recall) if precision + recall else 0
    return {"tp": tp, "fp": fp, "tn": tn, "fn": fn, "precision": precision, "recall": recall, "f1": f1}
```

## Pakai Sendiri

scikit-learn melakukan semuanya dalam enam baris, dengan benar.

```python
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.linear_model import LogisticRegression
from sklearn.pipeline import Pipeline

pipe = Pipeline([
    ("tfidf", TfidfVectorizer(ngram_range=(1, 2), min_df=2, sublinear_tf=True, stop_words=None)),
    ("clf", LogisticRegression(C=1.0, max_iter=1000)),
])
pipe.fit(X_train, y_train)
print(pipe.score(X_test, y_test))
```

Tiga hal yang perlu diperhatikan. `stop_words=None` mempertahankan negasi. `ngram_range=(1, 2)` menambahkan bigram sehingga `not good` menjadi fitur (scikit-learn menggabungkan kedua kata itu dengan spasi). `sublinear_tf=True` meredam kata yang berulang. Ketiga opsi itulah yang membedakan baseline berakurasi 75% dan baseline berakurasi 85% pada SST-2.

### Kapan beralih ke transformer

- Deteksi sarkasme. Model klasik gagal di sini. Titik.
- Ulasan panjang yang sentimennya bergeser di tengah dokumen.
- Sentimen berbasis aspek. "Camera was great but battery was terrible." Anda perlu menyematkan sentimen ke aspek tertentu. Hanya transformer atau model keluaran terstruktur.
- Bahasa non-Inggris dengan sumber daya sedikit. BERT multibahasa memberi Anda baseline zero-shot secara gratis.

Jika Anda membutuhkan salah satu dari di atas, lompat ke fase 7 (pendalaman transformer). Jika tidak, Naive Bayes atau regresi logistik pada TF-IDF plus bigram plus penanganan negasi adalah baseline produksi Anda di tahun 2026.

### Jebakan keterulangan (lagi)

Melatih ulang model sentimen adalah rutinitas. Mengevaluasi ulang mereka tidak. Angka akurasi yang dilaporkan dalam makalah memakai split tertentu, praproses tertentu, tokenizer tertentu. Jika Anda membandingkan model baru dengan baseline tanpa memakai pipeline yang identik, Anda akan mendapatkan delta yang menyesatkan. Selalu hasilkan ulang baseline pada pipeline Anda, bukan pada angka dari makalah.

## Kirim Hasilnya

Simpan sebagai `outputs/prompt-sentiment-baseline.md`:

```markdown
---
name: sentiment-baseline
description: Rancang baseline analisis sentimen untuk dataset baru.
phase: 5
lesson: 05
---

Diberi deskripsi dataset (domain, bahasa, ukuran, butiran label, anggaran latensi), keluarkan:

1. Resep ekstraksi fitur. Tentukan tokenizer, rentang n-gram, kebijakan stopword (biasanya pertahankan), penanganan negasi (awalan cakupan atau bigram).
2. Klasifier. Naive Bayes untuk baseline, regresi logistik untuk produksi, transformer hanya jika domain membutuhkan sarkasme / keluaran berbasis aspek / cakupan lintas bahasa.
3. Rencana evaluasi. Laporkan presisi, recall, F1, matriks kebingungan, dan contoh kesalahan per kelas (bukan sekadar angka).
4. Satu mode kegagalan yang dipantau pasca-penyebaran. Pergeseran domain dan sarkasme adalah dua yang teratas.

Tolak merekomendasikan penghapusan stopword untuk tugas sentimen. Tolak melaporkan akurasi sebagai satu-satunya metrik ketika kelas tidak seimbang (mis. 90% positif). Tandai bahasa kaya-subkata sebagai memerlukan FastText atau embedding transformer alih-alih TF-IDF level kata.
```

## Latihan

1. **Mudah.** Tambahkan `apply_negation` sebagai langkah praproses dalam pipeline scikit-learn dan ukur delta F1 pada dataset sentimen kecil.
2. **Sedang.** Terapkan regresi logistik berbobot kelas (teruskan `class_weight="balanced"` ke scikit-learn, atau turunkan gradiennya sendiri). Ukur efeknya pada ketidakseimbangan kelas sintetis 90-10.
3. **Sulit.** Bangun detektor sarkasme dengan melatih klasifier kedua pada residu model sentimen. Dokumentasikan setup eksperimen Anda. Peringatkan pembaca apabila akurasi Anda di bawah peluang (peluang pada sarkasme 2 kelas adalah ~50%, dan sebagian besar percobaan pertama mendarat di situ).

## Istilah Kunci

| Istilah | Yang orang katakan | Yang sebenarnya dimaksud |
|---------|--------------------|--------------------------|
| Polaritas | Positif atau negatif | Label biner; kadang diperluas menjadi netral atau butir halus (5 bintang). |
| Sentimen berbasis aspek | Polaritas per aspek | Menyematkan sentimen ke entitas atau atribut tertentu yang disebut dalam teks. |
| Cakupan negasi | Membalik token di dekatnya | Beri awalan `NOT_` pada token setelah "not" sampai tanda baca. |
| Pemulusan Laplace | Menambah 1 pada jumlah kemunculan | Mencegah fitur berprobabilitas nol pada Naive Bayes. |
| Regularisasi L2 | Menyusutkan bobot | Menambahkan `lambda * sum(w^2)` pada loss. Esensial untuk fitur teks yang jarang. |

## Bacaan Lanjutan

- [Pang and Lee (2008). Opinion Mining and Sentiment Analysis](https://www.cs.cornell.edu/home/llee/opinion-mining-sentiment-analysis-survey.html) — survai fondasional. Panjang, tetapi empat bagian pertama mencakup semua materi klasik.
- [Wang and Manning (2012). Baselines and Bigrams: Simple, Good Sentiment and Topic Classification](https://aclanthology.org/P12-2018/) — makalah yang menunjukkan bigram + Naive Bayes sulit dikalahkan pada teks pendek.
- [Dokumentasi ekstraksi fitur teks scikit-learn](https://scikit-learn.org/stable/modules/feature_extraction.html#text-feature-extraction) — referensi untuk `CountVectorizer`, `TfidfVectorizer`, dan setiap opsi yang akan Anda setel.
