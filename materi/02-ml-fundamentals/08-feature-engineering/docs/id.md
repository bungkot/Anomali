# Rekayasa Fitur & Seleksi

> Fitur yang baik sepadan dengan seribu titik data.

**Jenis:** Build
**Bahasa:** Python, TypeScript
**Prasyarat:** Fase 1 (Statistik untuk ML, Aljabar Linear), Fase 2 Pelajaran 1-7
**Waktu:** ~90 menit

## Tujuan Pembelajaran

- Mengimplementasikan transformasi numerik (standardisasi, penskalaan min-max, transformasi log, pembinning) dan menjelaskan kapan masing-masing tepat digunakan
- Membangun enkode one-hot, label, dan target untuk fitur kategorikal serta mengidentifikasi risiko kebocoran data pada enkode target
- Membangun vectorizer TF-IDF dari awal dan menjelaskan mengapa algoritma ini mengungguli hitungan kata mentah untuk klasifikasi teks
- Menerapkan seleksi fitur berbasis filter (ambang varians, korelasi, informasi bersama) untuk mereduksi dimensi

## Masalahnya

Anda memiliki dataset. Anda memilih algoritma. Anda melatihnya. Hasilnya biasa saja. Anda mencoba algoritma yang lebih canggih. Tetap biasa saja. Anda menghabiskan satu minggu menyetel hyperparameter. Perbaikan yang marginal.

Lalu seseorang mengubah data mentah menjadi fitur yang lebih baik, dan regresi logistik sederhana mengalahkan ensemble gradient boosting yang sudah Anda setel dengan teliti.

Ini terjadi terus-menerus. Dalam ML klasik, representasi data lebih penting daripada pilihan algoritma. Model harga rumah dengan fitur "luas lantai" dan "jumlah kamar tidur" akan mengalahkan model dengan "alamat sebagai string mentah", betapapun canggihnya pembelajar itu. Algoritma hanya dapat bekerja dengan apa yang Anda berikan kepadanya.

Rekayasa fitur adalah proses mengubah data mentah menjadi representasi yang membuat pola lebih mudah ditemukan model. Seleksi fitur adalah proses membuang fitur yang menambah noise tanpa menambah sinyal. Bersama-sama, keduanya adalah aktivitas dengan daya ungkit tertinggi dalam ML klasik.

## Konsepnya

### Pipeline Fitur

```mermaid
flowchart LR
    A[Data Mentah] --> B[Tangani Nilai Hilang]
    B --> C[Transformasi Numerik]
    B --> D[Enkode Kategorikal]
    B --> E[Fitur Teks]
    C --> F[Interaksi Fitur]
    D --> F
    E --> F
    F --> G[Seleksi Fitur]
    G --> H[Data Siap Model]
```

### Fitur Numerik

Angka mentah jarang siap dipakai model. Transformasi umum:

**Penskalaan:** Menempatkan fitur pada rentang yang sama sehingga algoritma berbasis jarak (K-Means, KNN, SVM) memperlakukan semua fitur secara setara. Penskalaan min-max memetakan ke [0, 1]. Standardisasi (skor-z) memetakan ke rata-rata=0, standar deviasi=1.

**Transformasi log:** Mengompresi distribusi yang miring ke kanan (pendapatan, populasi, jumlah kata). Mengubah hubungan multiplikatif menjadi hubungan aditif.

**Pembinning:** Mengubah nilai kontinu menjadi kategori. Berguna ketika hubungan antara fitur dan target bersifat nonlinear tetapi bertahap (mis. kelompok usia).

**Fitur polinomial:** Membuat suku $x^2$, $x^3$, $x_1 x_2$. Memungkinkan model linear menangkap hubungan nonlinear dengan biaya lebih banyak fitur.

### Fitur Kategorikal

Model memerlukan angka. Kategori perlu dienodekan.

**Enkode one-hot:** Membuat satu kolom biner untuk setiap kategori. "warna = merah/biru/hijau" menjadi tiga kolom: is_red, is_blue, is_green. Bekerja baik untuk fitur berkardinalitas rendah, tetapi meledak ketika banyak kategori.

**Enkode label:** Memetakan setiap kategori ke sebuah bilangan bulat: merah=0, biru=1, hijau=2. Memperkenalkan urutan palsu (model mungkin mengira hijau > biru > merah). Hanya tepat untuk model berbasis pohon yang memisahkan pada nilai individu.

**Enkode target:** Mengganti setiap kategori dengan rata-rata variabel target untuk kategori itu. Kuat tetapi berbahaya: risiko kebocoran data tinggi. Harus dihitung hanya pada data latihan dan diterapkan pada data uji.

### Fitur Teks

**Count vectorizer:** Menghitung berapa kali setiap kata muncul dalam sebuah dokumen. "the cat sat on the mat" menjadi {the: 2, cat: 1, sat: 1, on: 1, mat: 1}.

**TF-IDF:** Term Frequency-Inverse Document Frequency. Menimbang kata berdasarkan keunikan mereka di seluruh dokumen. Kata umum seperti "the" mendapat bobot rendah. Kata langka dan khas mendapat bobot tinggi.

```math
\begin{aligned}
\text{TF}(\text{word}, \text{doc}) &= \frac{\text{count(word in doc)}}{\text{total words in doc}} \\
\text{IDF}(\text{word}) &= \log \frac{\text{total docs}}{\text{docs containing word}} \\
\text{TF-IDF} &= \text{TF} \cdot \text{IDF}
\end{aligned}
```

### Nilai Hilang

Data dunia nyata punya lubang. Strateginya:

- **Buang baris:** Hanya ketika data yang hilang jarang dan acak
- **Imputasi rata-rata/median:** Sederhana, mempertahankan pusat fitur (median lebih tangguh terhadap outlier), tetapi menyusutkan variansnya dan menumpuk semua nilai hasil isian pada satu paku, sehingga bentuk distribusinya berubah
- **Imputasi modus:** Untuk fitur kategorikal
- **Kolom indikator:** Tambahkan kolom biner "was_this_missing" sebelum imputasi. Fakta bahwa data hilang itu sendiri bisa bermakna
- **Forward/backward fill:** Untuk data deret waktu

### Interaksi Fitur

Kadang hubungannya ada pada kombinasinya. "Tinggi badan" dan "berat badan" saja kurang prediktif dibanding $\text{BMI} = \text{weight} / \text{height}^2$. Interaksi fitur memperbesar ruang fitur, jadi gunakan pengetahuan domain untuk memilih yang tepat.

### Seleksi Fitur

Lebih banyak fitur tidak selalu lebih baik. Fitur tidak relevan menambah noise, meningkatkan waktu latihan, dan dapat menyebabkan overfitting.

**Metode filter (sebelum model):**
- Korelasi: buang fitur yang berkorelasi tinggi satu sama lain (redundan)
- Informasi bersama: mengukur seberapa besar mengetahui sebuah fitur mengurangi ketidakpastian tentang target
- Ambang varians: buang fitur yang hampir tidak berubah

**Metode wrapper (berbasis model):**
- Regularisasi L1 (Lasso): mendorong bobot fitur tidak relevan menjadi tepat nol
- Eliminasi fitur rekursif: latih, buang fitur paling tidak penting, ulangi

**Mengapa seleksi penting:** Model dengan 10 fitur baik biasanya mengalahkan model dengan 10 fitur baik ditambah 90 fitur bising. Fitur bising memberi model kesempatan untuk overfit pada pola data latihan yang tidak dapat digeneralisasi.

```figure
feature-scaling
```

## Bangun Sendiri

### Langkah 1: Transformasi numerik dari awal

```python
import math


def min_max_scale(values):
    min_val = min(values)
    max_val = max(values)
    if max_val == min_val:
        return [0.0] * len(values)
    return [(v - min_val) / (max_val - min_val) for v in values]


def standardize(values):
    n = len(values)
    mean = sum(values) / n
    variance = sum((v - mean) ** 2 for v in values) / n
    std = math.sqrt(variance) if variance > 0 else 1.0
    return [(v - mean) / std for v in values]


def log_transform(values):
    return [math.log(v + 1) for v in values]


def bin_values(values, n_bins=5):
    min_val = min(values)
    max_val = max(values)
    bin_width = (max_val - min_val) / n_bins
    if bin_width == 0:
        return [0] * len(values)
    result = []
    for v in values:
        bin_idx = int((v - min_val) / bin_width)
        bin_idx = min(bin_idx, n_bins - 1)
        result.append(bin_idx)
    return result


def polynomial_features(row, degree=2):
    n = len(row)
    result = list(row)
    if degree >= 2:
        for i in range(n):
            result.append(row[i] ** 2)
        for i in range(n):
            for j in range(i + 1, n):
                result.append(row[i] * row[j])
    return result
```

### Langkah 2: Enkode kategorikal dari awal

```python
def one_hot_encode(values):
    categories = sorted(set(values))
    cat_to_idx = {cat: i for i, cat in enumerate(categories)}
    n_cats = len(categories)

    encoded = []
    for v in values:
        row = [0] * n_cats
        row[cat_to_idx[v]] = 1
        encoded.append(row)

    return encoded, categories


def label_encode(values):
    categories = sorted(set(values))
    cat_to_int = {cat: i for i, cat in enumerate(categories)}
    return [cat_to_int[v] for v in values], cat_to_int


def target_encode(feature_values, target_values, smoothing=10):
    global_mean = sum(target_values) / len(target_values)

    category_stats = {}
    for feat, target in zip(feature_values, target_values):
        if feat not in category_stats:
            category_stats[feat] = {"sum": 0.0, "count": 0}
        category_stats[feat]["sum"] += target
        category_stats[feat]["count"] += 1

    encoding = {}
    for cat, stats in category_stats.items():
        cat_mean = stats["sum"] / stats["count"]
        weight = stats["count"] / (stats["count"] + smoothing)
        encoding[cat] = weight * cat_mean + (1 - weight) * global_mean

    return [encoding[v] for v in feature_values], encoding
```

### Langkah 3: Fitur teks dari awal

```python
def count_vectorize(documents):
    vocab = {}
    idx = 0
    for doc in documents:
        for word in doc.lower().split():
            if word not in vocab:
                vocab[word] = idx
                idx += 1

    vectors = []
    for doc in documents:
        vec = [0] * len(vocab)
        for word in doc.lower().split():
            vec[vocab[word]] += 1
        vectors.append(vec)

    return vectors, vocab


def tfidf(documents):
    n_docs = len(documents)

    vocab = {}
    idx = 0
    for doc in documents:
        for word in doc.lower().split():
            if word not in vocab:
                vocab[word] = idx
                idx += 1

    doc_freq = {}
    for doc in documents:
        seen = set()
        for word in doc.lower().split():
            if word not in seen:
                doc_freq[word] = doc_freq.get(word, 0) + 1
                seen.add(word)

    vectors = []
    for doc in documents:
        words = doc.lower().split()
        word_count = len(words)
        tf_map = {}
        for word in words:
            tf_map[word] = tf_map.get(word, 0) + 1

        vec = [0.0] * len(vocab)
        for word, count in tf_map.items():
            tf = count / word_count
            idf = math.log(n_docs / doc_freq[word])
            vec[vocab[word]] = tf * idf
        vectors.append(vec)

    return vectors, vocab
```

### Langkah 4: Imputasi nilai hilang dari awal

```python
def impute_mean(values):
    present = [v for v in values if v is not None]
    if not present:
        return [0.0] * len(values), 0.0
    mean = sum(present) / len(present)
    return [v if v is not None else mean for v in values], mean


def impute_median(values):
    present = sorted(v for v in values if v is not None)
    if not present:
        return [0.0] * len(values), 0.0
    n = len(present)
    if n % 2 == 0:
        median = (present[n // 2 - 1] + present[n // 2]) / 2
    else:
        median = present[n // 2]
    return [v if v is not None else median for v in values], median


def impute_mode(values):
    present = [v for v in values if v is not None]
    if not present:
        return values, None
    counts = {}
    for v in present:
        counts[v] = counts.get(v, 0) + 1
    mode = max(counts, key=counts.get)
    return [v if v is not None else mode for v in values], mode


def add_missing_indicator(values):
    return [0 if v is not None else 1 for v in values]
```

### Langkah 5: Seleksi fitur dari awal

```python
def correlation(x, y):
    n = len(x)
    mean_x = sum(x) / n
    mean_y = sum(y) / n
    cov = sum((xi - mean_x) * (yi - mean_y) for xi, yi in zip(x, y)) / n
    std_x = math.sqrt(sum((xi - mean_x) ** 2 for xi in x) / n)
    std_y = math.sqrt(sum((yi - mean_y) ** 2 for yi in y) / n)
    if std_x == 0 or std_y == 0:
        return 0.0
    return cov / (std_x * std_y)


def mutual_information(feature, target, n_bins=10):
    feat_min = min(feature)
    feat_max = max(feature)
    bin_width = (feat_max - feat_min) / n_bins if feat_max != feat_min else 1.0
    feat_binned = [
        min(int((f - feat_min) / bin_width), n_bins - 1) for f in feature
    ]

    n = len(feature)
    target_classes = sorted(set(target))

    feat_bins = sorted(set(feat_binned))
    p_feat = {}
    for b in feat_bins:
        p_feat[b] = feat_binned.count(b) / n

    p_target = {}
    for t in target_classes:
        p_target[t] = target.count(t) / n

    mi = 0.0
    for b in feat_bins:
        for t in target_classes:
            joint_count = sum(
                1 for fb, tv in zip(feat_binned, target) if fb == b and tv == t
            )
            p_joint = joint_count / n
            if p_joint > 0:
                mi += p_joint * math.log(p_joint / (p_feat[b] * p_target[t]))

    return mi


def variance_threshold(features, threshold=0.01):
    n_features = len(features[0])
    n_samples = len(features)
    selected = []

    for j in range(n_features):
        col = [features[i][j] for i in range(n_samples)]
        mean = sum(col) / n_samples
        var = sum((v - mean) ** 2 for v in col) / n_samples
        if var >= threshold:
            selected.append(j)

    return selected


def remove_correlated(features, threshold=0.9):
    n_features = len(features[0])
    n_samples = len(features)

    to_remove = set()
    for i in range(n_features):
        if i in to_remove:
            continue
        col_i = [features[r][i] for r in range(n_samples)]
        for j in range(i + 1, n_features):
            if j in to_remove:
                continue
            col_j = [features[r][j] for r in range(n_samples)]
            corr = abs(correlation(col_i, col_j))
            if corr >= threshold:
                to_remove.add(j)

    return [i for i in range(n_features) if i not in to_remove]
```

### Langkah 6: Pipeline lengkap dan demo

```python
import random


def make_housing_data(n=200, seed=42):
    random.seed(seed)
    data = []
    for _ in range(n):
        sqft = random.uniform(500, 5000)
        bedrooms = random.choice([1, 2, 3, 4, 5])
        age = random.uniform(0, 50)
        neighborhood = random.choice(["downtown", "suburbs", "rural"])
        has_pool = random.choice([True, False])

        sqft_with_missing = sqft if random.random() > 0.05 else None
        age_with_missing = age if random.random() > 0.08 else None

        price = (
            50 * sqft
            + 20000 * bedrooms
            - 1000 * age
            + (50000 if neighborhood == "downtown" else 10000 if neighborhood == "suburbs" else 0)
            + (15000 if has_pool else 0)
            + random.gauss(0, 20000)
        )

        data.append({
            "sqft": sqft_with_missing,
            "bedrooms": bedrooms,
            "age": age_with_missing,
            "neighborhood": neighborhood,
            "has_pool": has_pool,
            "price": price,
        })
    return data


if __name__ == "__main__":
    data = make_housing_data(200)

    print("=== Raw Data Sample ===")
    for row in data[:3]:
        print(f"  {row}")

    sqft_raw = [d["sqft"] for d in data]
    age_raw = [d["age"] for d in data]
    prices = [d["price"] for d in data]

    print("\n=== Missing Value Handling ===")
    sqft_missing = sum(1 for v in sqft_raw if v is None)
    age_missing = sum(1 for v in age_raw if v is None)
    print(f"  sqft missing: {sqft_missing}/{len(sqft_raw)}")
    print(f"  age missing: {age_missing}/{len(age_raw)}")

    sqft_indicator = add_missing_indicator(sqft_raw)
    age_indicator = add_missing_indicator(age_raw)
    sqft_imputed, sqft_fill = impute_median(sqft_raw)
    age_imputed, age_fill = impute_mean(age_raw)
    print(f"  sqft filled with median: {sqft_fill:.0f}")
    print(f"  age filled with mean: {age_fill:.1f}")

    print("\n=== Numerical Transforms ===")
    sqft_scaled = standardize(sqft_imputed)
    age_scaled = min_max_scale(age_imputed)
    sqft_log = log_transform(sqft_imputed)
    age_binned = bin_values(age_imputed, n_bins=5)
    print(f"  sqft standardized: mean={sum(sqft_scaled)/len(sqft_scaled):.4f}, std={math.sqrt(sum(v**2 for v in sqft_scaled)/len(sqft_scaled)):.4f}")
    print(f"  age min-max: [{min(age_scaled):.2f}, {max(age_scaled):.2f}]")
    print(f"  age bins: {sorted(set(age_binned))}")

    print("\n=== Categorical Encoding ===")
    neighborhoods = [d["neighborhood"] for d in data]

    ohe, ohe_cats = one_hot_encode(neighborhoods)
    print(f"  One-hot categories: {ohe_cats}")
    print(f"  Sample encoding: {neighborhoods[0]} -> {ohe[0]}")

    le, le_map = label_encode(neighborhoods)
    print(f"  Label encoding map: {le_map}")

    te, te_map = target_encode(neighborhoods, prices, smoothing=10)
    print(f"  Target encoding: {({k: round(v) for k, v in te_map.items()})}")

    print("\n=== Text Features ===")
    descriptions = [
        "large modern house with pool",
        "small cozy cottage near downtown",
        "spacious family home with large yard",
        "modern apartment downtown with view",
        "rustic cabin in rural area",
    ]
    cv, cv_vocab = count_vectorize(descriptions)
    print(f"  Vocabulary size: {len(cv_vocab)}")
    print(f"  Doc 0 non-zero features: {sum(1 for v in cv[0] if v > 0)}")

    tf, tf_vocab = tfidf(descriptions)
    print(f"  TF-IDF vocabulary size: {len(tf_vocab)}")
    top_words = sorted(tf_vocab.keys(), key=lambda w: tf[0][tf_vocab[w]], reverse=True)[:3]
    print(f"  Doc 0 top TF-IDF words: {top_words}")

    print("\n=== Polynomial Features ===")
    sample_row = [sqft_scaled[0], age_scaled[0]]
    poly = polynomial_features(sample_row, degree=2)
    print(f"  Input: {[round(v, 4) for v in sample_row]}")
    print(f"  Polynomial: {[round(v, 4) for v in poly]}")
    print(f"  Features: [x1, x2, x1^2, x2^2, x1*x2]")

    print("\n=== Feature Selection ===")
    feature_matrix = [
        [sqft_scaled[i], age_scaled[i], float(sqft_indicator[i]), float(age_indicator[i])]
        + ohe[i]
        for i in range(len(data))
    ]

    print(f"  Total features: {len(feature_matrix[0])}")

    surviving_var = variance_threshold(feature_matrix, threshold=0.01)
    print(f"  After variance threshold (0.01): {len(surviving_var)} features kept")

    surviving_corr = remove_correlated(feature_matrix, threshold=0.9)
    print(f"  After correlation filter (0.9): {len(surviving_corr)} features kept")

    binary_prices = [1 if p > sum(prices) / len(prices) else 0 for p in prices]
    print("\n  Mutual information with target:")
    feature_names = ["sqft", "age", "sqft_missing", "age_missing"] + [f"neigh_{c}" for c in ohe_cats]
    for j in range(len(feature_matrix[0])):
        col = [feature_matrix[i][j] for i in range(len(feature_matrix))]
        mi = mutual_information(col, binary_prices, n_bins=10)
        print(f"    {feature_names[j]}: MI={mi:.4f}")

    print("\n  Correlation with price:")
    for j in range(len(feature_matrix[0])):
        col = [feature_matrix[i][j] for i in range(len(feature_matrix))]
        corr = correlation(col, prices)
        print(f"    {feature_names[j]}: r={corr:.4f}")
```

## Gunakan

Dengan scikit-learn, transformasi-transformasi ini adalah pipeline yang dapat disusun:

```python
from sklearn.preprocessing import StandardScaler, OneHotEncoder, PolynomialFeatures
from sklearn.impute import SimpleImputer
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.feature_selection import mutual_info_classif, VarianceThreshold
from sklearn.compose import ColumnTransformer
from sklearn.pipeline import Pipeline

numeric_pipe = Pipeline([
    ("imputer", SimpleImputer(strategy="median")),
    ("scaler", StandardScaler()),
])

categorical_pipe = Pipeline([
    ("encoder", OneHotEncoder(sparse_output=False)),
])

preprocessor = ColumnTransformer([
    ("num", numeric_pipe, ["sqft", "age"]),
    ("cat", categorical_pipe, ["neighborhood"]),
])
```

Versi dari awal menunjukkan persis apa yang terjadi di dalam setiap transformasi. Versi pustaka menambahkan penanganan kasus tepi, dukungan matriks jarang, dan komposisi pipeline, tetapi matematikanya sama.

## Kirim

Pelajaran ini menghasilkan:
- `outputs/prompt-feature-engineer.md` - sebuah prompt untuk merekayasa fitur secara sistematis dari data mentah

## Latihan

1. Tambahkan penskalaan robust (menggunakan median dan jangkauan antarkuartil alih-alih rata-rata dan standar deviasi) ke dalam transformasi numerik. Bandingkan dengan penskalaan standar pada data yang memiliki outlier ekstrem.
2. Implementasikan enkode target leave-one-out: untuk setiap baris, hitung rata-rata target dengan mengecualikan nilai target baris itu sendiri. Tunjukkan bagaimana cara ini mengurangi overfitting dibandingkan enkode target naif.
3. Bangun pipeline seleksi fitur otomatis yang menggabungkan ambang varians, filter korelasi, dan peringkat informasi bersama. Terapkan pada dataset perumahan dan bandingkan kinerja model (gunakan regresi linear sederhana) antara semua fitur dan fitur hasil seleksi.

## Istilah Kunci

| Istilah | Apa yang orang katakan | Arti sebenarnya |
|------|----------------|----------------------|
| Rekayasa fitur | "Membuat kolom baru" | Mengubah data mentah menjadi representasi yang menyingkap pola bagi model |
| Standardisasi | "Membuatnya normal" | Mengurangi rata-rata dan membagi dengan standar deviasi agar fitur memiliki rata-rata=0 dan std=1 |
| Enkode one-hot | "Membuat variabel dummy" | Membuat satu kolom biner per kategori, dengan tepat satu kolom bernilai 1 pada setiap baris |
| Enkode target | "Menggunakan jawaban untuk mengodekan" | Mengganti setiap kategori dengan nilai target rata-rata untuk kategori itu, dengan smoothing untuk mencegah overfitting |
| TF-IDF | "Hitungan kata yang canggih" | Term Frequency dikali Inverse Document Frequency: kata diberi bobot berdasarkan kekhasannya di seluruh korpus |
| Imputasi | "Mengisi bagian kosong" | Mengganti nilai hilang dengan nilai estimasi (rata-rata, median, modus, atau prediksi model) |
| Seleksi fitur | "Membuang kolom buruk" | Menghapus fitur yang menambah noise atau redundansi, menyisakan hanya yang memiliki sinyal tentang target |
| Informasi bersama | "Seberapa banyak satu hal memberi tahu Anda tentang hal lain" | Ukuran pengurangan ketidakpastian tentang variabel Y yang diperoleh dengan mengamati variabel X |
| Kebocoran data | "Curang tanpa sengaja" | Menggunakan informasi saat latihan yang tidak akan tersedia saat prediksi, menghasilkan hasil yang palsu optimistis |

## Bacaan Lanjutan

- [Feature Engineering and Selection (Max Kuhn & Kjell Johnson)](http://www.feat.engineering/) - buku daring gratis yang mencakup seluruh lanskap rekayasa fitur
- [scikit-learn Preprocessing Guide](https://scikit-learn.org/stable/modules/preprocessing.html) - referensi praktis untuk semua transformasi standar
- [Target Encoding Done Right (Micci-Barreca, 2001)](https://dl.acm.org/doi/10.1145/507533.507538) - makalah awal tentang enkode target dengan smoothing
