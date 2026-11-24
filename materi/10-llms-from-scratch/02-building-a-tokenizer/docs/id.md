# Membangun Tokenizer dari Awal

> Pelajaran 01 memberi Anda mainan. Pelajaran ini memberi Anda senjata.

**Jenis:** Bangun
**Bahasa:** Python
**Prasyarat:** Fase 10, Pelajaran 01 (Tokenizer: BPE, WordPiece, SentencePiece)
**Waktu:** ~90 menit

## Tujuan Pembelajaran

- Membangun tokenizer BPE kelas produksi yang menangani Unicode, normalisasi spasi, dan token khusus
- Mengimplementasikan fallback tingkat byte agar tokenizer dapat mengodekan masukan apa pun (termasuk emoji, CJK, dan kode) tanpa token unknown
- Menambahkan pola regex pratokenisasi yang membagi teks pada batas kata sebelum menerapkan penggabungan BPE
- Melatih tokenizer kustom pada sebuah korpus dan mengevaluasi rasio kompresinya dibandingkan tiktoken pada teks multibahasa

## Masalahnya

Tokenizer BPE Anda dari Pelajaran 01 bekerja pada teks bahasa Inggris. Sekarang lemparkan bahasa Jepang ke padanya. Atau emoji. Atau kode Python dengan campuran tab dan spasi.

Ia rusak.

Bukan karena BPE salah -- melainkan karena implementasinya belum lengkap. Tokenizer kelas produksi menangani byte mentah dalam pengodean apa pun, menormalkan Unicode sebelum membagi teks, mengelola token khusus yang tidak pernah digabung, merangkai pratokenisasi dengan pemecahan subkata, dan melakukan semuanya cukup cepat sehingga tidak menjadi hambatan bagi pipeline pelatihan yang memproses 15 triliun token.

Tokenizer GPT-2 memiliki 50,257 token. Llama 3 memiliki 128,256. GPT-4 memiliki sekitar 100,000. Ini bukan angka mainan. Tabel penggabungan di balik kosakata itu dilatih pada ratusan gigabyte teks, dan mesin di sekitarnya -- normalisasi, pratokenisasi, penyisipan token khusus, pemformatan templat obrolan --lah yang membedakan tokenizer yang menangani "hello world" dari tokenizer yang menangani seluruh internet.

Anda akan membangun mesin itu.

## Konsep

### Pipeline Lengkap

Tokenizer kelas produksi bukan satu algoritme. Ia adalah pipeline lima tahap, masing-masing memecahkan masalah yang berbeda.

```mermaid
graph LR
    A[Teks Mentah] --> B[Normalisasi]
    B --> C[Pratokenisasi]
    C --> D[Penggabungan BPE]
    D --> E[Token Khusus]
    E --> F[ID Token]

    style A fill:#1a1a2e,stroke:#e94560,color:#fff
    style B fill:#1a1a2e,stroke:#e94560,color:#fff
    style C fill:#1a1a2e,stroke:#e94560,color:#fff
    style D fill:#1a1a2e,stroke:#e94560,color:#fff
    style E fill:#1a1a2e,stroke:#e94560,color:#fff
    style F fill:#1a1a2e,stroke:#e94560,color:#fff
```

Setiap tahap memiliki tugas tertentu:

| Tahap | Apa yang Dilakukannya | Mengapa Penting |
|-------|----------------------|-----------------|
| Normalisasi | Unicode NFKC, huruf kecil opsional, penghapusan aksen opsional | Ligatur "fi" (U+FB01) menjadi "fi" (dua karakter). Tanpa ini, kata yang sama mendapat token berbeda. |
| Pratokenisasi | Membagi teks menjadi potongan sebelum BPE | Mencegah BPE menggabungkan token lintas batas kata. "the cat" tidak boleh menghasilkan token "e c". |
| Penggabungan BPE | Menerapkan aturan penggabungan yang dipelajari pada urutan byte | Kompresi inti. Mengubah byte mentah menjadi token subkata. |
| Token Khusus | Menyisipkan [BOS], [EOS], [PAD], penanda templat obrolan | Token-token ini memiliki ID tetap. Ia tidak pernah ikut penggabungan BPE. Model memerlukannya untuk struktur. |
| Pemetaan ID | Mengubah string token menjadi ID bilangan bulat | Model melihat bilangan bulat, bukan string. |

### BPE Tingkat Byte

Tokenizer di Pelajaran 01 bekerja pada byte UTF-8. Itu keputusan yang tepat. Tetapi kita melewati sesuatu yang penting: apa yang terjadi ketika byte-byte itu bukan UTF-8 yang valid?

BPE tingkat byte mengatasi ini dengan memperlakukan setiap nilai byte yang mungkin (0-255) sebagai token yang valid. Kosakata dasar Anda tepat 256 entri. File apa pun -- teks, biner, rusak -- dapat ditokenisasi tanpa menghasilkan token unknown.

GPT-2 menambahkan trik: memetakan setiap byte ke karakter Unicode yang dapat dicetak agar kosakatanya tetap mudah dibaca manusia. Byte 0x20 (spasi) menjadi karakter "G" dalam pemetaan mereka. Ini murni kosmetik. Algoritme tidak peduli.

Kekuatan sesungguhnya: BPE tingkat byte menangani setiap bahasa di dunia. Karakter Mandarin masing-masing 3 byte UTF-8. Bahasa Jepang bisa 3-4 byte. Arab, Devanagari, emoji -- semuanya hanya urutan byte. Algoritme BPE menemukan pola dalam urutan byte ini dengan cara yang persis sama seperti ia menemukan pola pada byte ASCII bahasa Inggris.

### Pratokenisasi

Sebelum BPE menyentuh teks Anda, Anda perlu membaginya menjadi potongan-potongan. Ini mencegah algoritme penggabungan membuat token yang melintasi batas kata.

GPT-2 menggunakan pola regex untuk membagi teks:

```
'(?:[sdmt]|ll|ve|re)| ?\p{L}+| ?\p{N}+| ?[^\s\p{L}\p{N}]+|\s+(?!\S)|\s+
```

Pola ini membagi kontraksi ("don't" menjadi "don" + "'t"), kata dengan spasi awal opsional, angka, tanda baca, dan spasi kosong. Spasi awal tetap melekat pada kata yang mengikutinya -- sehingga "the cat" menjadi [" the", " cat"], bukan ["the", " ", "cat"].

Llama menggunakan SentencePiece, yang sepenuhnya melewati regex. Ia memperlakukan aliran byte mentah sebagai satu urutan panjang dan membiarkan algoritme BPE menentukan batasnya sendiri. Ini lebih sederhana, tetapi memberi BPE kebebasan lebih untuk membuat token lintas kata.

Pilihan ini penting. Regex GPT-2 mencegah tokenizer mempelajari bahwa "the" di akhir satu kata dan "the" di awal kata berikutnya harus digabung. SentencePiece mengizinkannya, yang kadang menghasilkan kompresi lebih efisien, tetapi token yang kurang dapat ditafsirkan.

### Token Khusus

Setiap tokenizer kelas produksi mencadangkan ID token untuk penanda struktural:

| Token | Tujuan | Digunakan Oleh |
|-------|--------|----------------|
| `[BOS]` / `<s>` | Awal urutan | Llama 3, GPT |
| `[EOS]` / `</s>` | Akhir urutan | Semua model |
| `[PAD]` | Padding untuk penyelarasan batch | BERT, T5 |
| `[UNK]` | Token unknown (BPE tingkat byte menghilangkannya) | BERT, WordPiece |
| `<\|im_start\|>` | Awal batas pesan obrolan | ChatGPT, Qwen |
| `<\|im_end\|>` | Akhir batas pesan obrolan | ChatGPT, Qwen |
| `<\|user\|>` | Penanda giliran pengguna | Llama 3 |
| `<\|assistant\|>` | Penanda giliran asisten | Llama 3 |

Token khusus tidak pernah dibagi oleh BPE. Token itu dicocokkan persis sebelum algoritme penggabungan berjalan, diganti dengan ID tetapnya, dan teks di sekitarnya ditokenisasi seperti biasa.

### Templat Obrolan

Di sinilah kebanyakan orang bingung dan kebanyakan implementasi rusak.

Saat Anda mengirim pesan ke model obrolan, API menerima daftar pesan:

```
[
  {"role": "system", "content": "You are helpful."},
  {"role": "user", "content": "Hello"},
  {"role": "assistant", "content": "Hi there!"}
]
```

Model tidak melihat JSON. Ia melihat urutan token datar. Templat obrolan mengubah pesan menjadi urutan datar itu menggunakan token khusus. Setiap model melakukannya dengan cara berbeda:

```
Llama 3:
<|begin_of_text|><|start_header_id|>system<|end_header_id|>

You are helpful.<|eot_id|><|start_header_id|>user<|end_header_id|>

Hello<|eot_id|><|start_header_id|>assistant<|end_header_id|>

Hi there!<|eot_id|>

ChatGPT:
<|im_start|>system
You are helpful.<|im_end|>
<|im_start|>user
Hello<|im_end|>
<|im_start|>assistant
Hi there!<|im_end|>
```

Salah templat dan model menghasilkan sampah. Ia dilatih pada satu format yang persis. Penyimpangan apa pun -- baris baru yang hilang, token yang tertukar, spasi ekstra -- menempatkan masukan di luar distribusi pelatihan.

### Kecepatan

Python terlalu lambat untuk tokenisasi kelas produksi.

tiktoken (OpenAI) ditulis dalam Rust dengan binding Python. HuggingFace tokenizers juga Rust. SentencePiece berbasis C++. Semuanya mencapai percepatan 10-100x dibanding Python murni.

Sebagai gambaran: menokenisasi 15 triliun token untuk pra-pelatihan Llama 3 pada 1 juta token per detik (Python cepat) membutuhkan 174 hari. Pada 100 juta token per detik (Rust), dibutuhkan 1,7 hari.

Anda membangun dengan Python untuk memahami algoritmenya. Di produksi, Anda akan menggunakan implementasi terkompilasi dan hanya menyentuh pembungkus Python-nya.

```figure
weight-tying
```

## Bangunlah

### Langkah 1: Pengodean Tingkat Byte

Fondasinya. Ubah string apa pun menjadi urutan byte, petakan setiap byte ke karakter yang dapat dicetak untuk ditampilkan, dan balikkan prosesnya.

```python
def bytes_to_tokens(text):
    return list(text.encode("utf-8"))

def tokens_to_text(token_bytes):
    return bytes(token_bytes).decode("utf-8", errors="replace")
```

Uji pada teks multibahasa untuk melihat jumlah byte:

```python
texts = [
    ("English", "hello"),
    ("Chinese", "你好"),
    ("Emoji", "🔥"),
    ("Mixed", "hello你好🔥"),
]

for label, text in texts:
    b = bytes_to_tokens(text)
    print(f"{label}: {len(text)} chars -> {len(b)} bytes -> {b}")
```

"hello" berisi 5 byte. "你好" berisi 6 byte (3 per karakter). Emoji api berisi 4 byte. Tokenizer tingkat byte tidak peduli bahasanya. Byte tetap byte.

### Langkah 2: Pratokenizer dengan Regex

Bagi teks menjadi potongan menggunakan pola regex GPT-2. Setiap potongan ditokenisasi secara independen oleh BPE.

```python
import re

try:
    import regex
    GPT2_PATTERN = regex.compile(
        r"""'(?:[sdmt]|ll|ve|re)| ?\p{L}+| ?\p{N}+| ?[^\s\p{L}\p{N}]+|\s+(?!\S)|\s+"""
    )
except ImportError:
    GPT2_PATTERN = re.compile(
        r"""'(?:[sdmt]|ll|ve|re)| ?[^\W\d_]+| ?\d+| ?(?:[^\s\w]|_)+|\s+(?!\S)|\s+"""
    )

def pre_tokenize(text):
    return [match.group() for match in GPT2_PATTERN.finditer(text)]
```

Modul `regex` mendukung escape properti Unicode (`\p{L}` untuk huruf, `\p{N}` untuk angka). Modul `re` dari pustaka standar tidak mendukungnya, sehingga fallback membangun kelas yang sama dari `\w`: `[^\W\d_]` mencocokkan huruf Unicode apa pun dan `\d` digit Unicode apa pun. Untuk tokenizer multibahasa kelas produksi, pasang `regex`.

Cobalah:

```python
print(pre_tokenize("Hello, world! Don't stop."))
# ['Hello', ',', ' world', '!', ' Don', "'t", ' stop', '.']
```

Spasi awal tetap melekat pada kata yang mengikutinya. Kata pertama tidak memiliki spasi sebelumnya, sehingga potongannya adalah `Hello`. Kontraksi dibagi pada apostrof. Tanda baca menjadi potongannya sendiri. BPE tidak akan pernah menggabungkan token lintas batas ini.

### Langkah 3: BPE pada Urutan Byte

Algoritme inti dari Pelajaran 01, tetapi sekarang beroperasi pada potongan yang sudah ditokenisasi awal secara independen.

```python
from collections import Counter

def get_byte_pairs(chunks):
    pairs = Counter()
    for chunk in chunks:
        byte_seq = list(chunk.encode("utf-8"))
        for i in range(len(byte_seq) - 1):
            pairs[(byte_seq[i], byte_seq[i + 1])] += 1
    return pairs

def apply_merge(byte_seq, pair, new_id):
    merged = []
    i = 0
    while i < len(byte_seq):
        if i < len(byte_seq) - 1 and byte_seq[i] == pair[0] and byte_seq[i + 1] == pair[1]:
            merged.append(new_id)
            i += 2
        else:
            merged.append(byte_seq[i])
            i += 1
    return merged
```

### Langkah 4: Penanganan Token Khusus

Token khusus memerlukan pencocokan persis dan ID tetap. Token ini sepenuhnya melewati BPE.

```python
class SpecialTokenHandler:
    def __init__(self):
        self.special_tokens = {}
        self.pattern = None

    def add_token(self, token_str, token_id):
        self.special_tokens[token_str] = token_id
        escaped = [re.escape(t) for t in sorted(self.special_tokens.keys(), key=len, reverse=True)]
        self.pattern = re.compile("|".join(escaped))

    def split_with_specials(self, text):
        if not self.pattern:
            return [(text, False)]
        parts = []
        last_end = 0
        for match in self.pattern.finditer(text):
            if match.start() > last_end:
                parts.append((text[last_end:match.start()], False))
            parts.append((match.group(), True))
            last_end = match.end()
        if last_end < len(text):
            parts.append((text[last_end:], False))
        return parts
```

### Langkah 5: Kelas Tokenizer Lengkap

Rangkai semuanya: normalisasi, pembagian pada token khusus, pratokenisasi, penggabungan BPE, pemetaan ke ID.

```python
import unicodedata

class ProductionTokenizer:
    def __init__(self):
        self.merges = {}
        self.vocab = {i: bytes([i]) for i in range(256)}
        self.special_handler = SpecialTokenHandler()
        self.next_id = 256

    def normalize(self, text):
        return unicodedata.normalize("NFKC", text)

    def train(self, text, num_merges):
        text = self.normalize(text)
        chunks = pre_tokenize(text)
        chunk_bytes = [list(chunk.encode("utf-8")) for chunk in chunks]

        for i in range(num_merges):
            pairs = Counter()
            for seq in chunk_bytes:
                for j in range(len(seq) - 1):
                    pairs[(seq[j], seq[j + 1])] += 1
            if not pairs:
                break
            best = max(pairs, key=pairs.get)
            new_id = self.next_id
            self.next_id += 1
            self.merges[best] = new_id
            self.vocab[new_id] = self.vocab[best[0]] + self.vocab[best[1]]
            chunk_bytes = [apply_merge(seq, best, new_id) for seq in chunk_bytes]

    def add_special_token(self, token_str):
        token_id = self.next_id
        self.next_id += 1
        self.special_handler.add_token(token_str, token_id)
        self.vocab[token_id] = token_str.encode("utf-8")
        return token_id

    def encode(self, text):
        text = self.normalize(text)
        parts = self.special_handler.split_with_specials(text)
        all_ids = []
        for part_text, is_special in parts:
            if is_special:
                all_ids.append(self.special_handler.special_tokens[part_text])
            else:
                for chunk in pre_tokenize(part_text):
                    byte_seq = list(chunk.encode("utf-8"))
                    for pair, new_id in self.merges.items():
                        byte_seq = apply_merge(byte_seq, pair, new_id)
                    all_ids.extend(byte_seq)
        return all_ids

    def decode(self, ids):
        byte_parts = []
        for token_id in ids:
            if token_id in self.vocab:
                byte_parts.append(self.vocab[token_id])
        return b"".join(byte_parts).decode("utf-8", errors="replace")

    def vocab_size(self):
        return len(self.vocab)
```

### Langkah 6: Uji Multibahasa

Ujian sesungguhnya. Lemparkan bahasa Inggris, Mandarin, emoji, dan kode ke padanya.

```python
corpus = (
    "The quick brown fox jumps over the lazy dog. "
    "The quick brown fox runs through the forest. "
    "Machine learning models process natural language. "
    "Deep learning transforms how we build software. "
    "def train(model, data): return model.fit(data) "
    "def predict(model, x): return model(x) "
)

tok = ProductionTokenizer()
tok.train(corpus, num_merges=50)

bos = tok.add_special_token("<|begin|>")
eos = tok.add_special_token("<|end|>")

test_texts = [
    "The quick brown fox.",
    "你好世界",
    "Hello 🌍 World",
    "def foo(x): return x + 1",
    f"<|begin|>Hello<|end|>",
]

for text in test_texts:
    ids = tok.encode(text)
    decoded = tok.decode(ids)
    print(f"Input:   {text}")
    print(f"Tokens:  {len(ids)} ids")
    print(f"Decoded: {decoded}")
    print()
```

Karakter Mandarin menghasilkan 3 byte setiap satu. Emoji menghasilkan 4 byte. Tidak ada yang membuat tokenizer mogok. Tidak ada yang menghasilkan token unknown. Itulah kekuatan BPE tingkat byte.

## Gunakanlah

### Membandingkan Tokenizer Nyata

Muat tokenizer asli dari Llama 3, GPT-4, dan Mistral. Lihat bagaimana masing-masing menangani paragraf multibahasa yang sama.

```python
import tiktoken

gpt4_enc = tiktoken.get_encoding("cl100k_base")

test_paragraph = "Machine learning is powerful. 机器学习很强大。 L'apprentissage automatique est puissant. 🤖💪"

tokens = gpt4_enc.encode(test_paragraph)
pieces = [gpt4_enc.decode([t]) for t in tokens]
print(f"GPT-4 ({len(tokens)} tokens): {pieces}")
```

```python
from transformers import AutoTokenizer

llama_tok = AutoTokenizer.from_pretrained("meta-llama/Meta-Llama-3-8B")
mistral_tok = AutoTokenizer.from_pretrained("mistralai/Mistral-7B-v0.1")

for name, tok in [("Llama 3", llama_tok), ("Mistral", mistral_tok)]:
    tokens = tok.encode(test_paragraph)
    pieces = tok.convert_ids_to_tokens(tokens)
    print(f"{name} ({len(tokens)} tokens): {pieces[:20]}...")
```

Anda akan melihat jumlah token yang berbeda untuk teks yang sama. Llama 3 dengan kosakata 128K lebih agresif menggabungkan pola umum. GPT-4 dengan 100K berada di tengah. Mistral dengan 32K menghasilkan lebih banyak token, tetapi memiliki lapisan embedding yang lebih kecil.

Trade-off-nya selalu sama: kosakata lebih besar berarti urutan lebih pendek, tetapi parameter lebih banyak.

## Kirimkan

Pelajaran ini menghasilkan sebuah prompt untuk membangun dan men-debug tokenizer kelas produksi. Lihat `outputs/prompt-tokenizer-builder.md`.

## Latihan

1. **Mudah:** Tambahkan metode `get_token_bytes(id)` yang menampilkan byte mentah untuk ID token apa pun. Gunakan untuk memeriksa apa yang sebenarnya diwakili oleh token hasil penggabungan yang paling sering muncul.
2. **Sedang:** Implementasikan pratokenizer bergaya Llama yang membagi pada spasi kosong dan angka, tetapi mempertahankan spasi awal. Bandingkan kosakatanya dengan pendekatan regex GPT-2 pada korpus yang sama.
3. **Sulit:** Tambahkan metode templat obrolan yang menerima daftar pesan `{"role": ..., "content": ...}` dan menghasilkan urutan token yang benar untuk format obrolan Llama 3. Uji terhadap implementasi HuggingFace.

## Istilah Kunci

| Istilah | Orang Bilang | Arti Sebenarnya |
|---------|--------------|-----------------|
| BPE tingkat byte | "Tokenizer yang bekerja pada byte" | BPE dengan kosakata dasar 256 nilai byte -- menangani masukan apa pun tanpa token unknown |
| Pratokenisasi | "Pembagian sebelum BPE" | Pembagian berbasis regex atau aturan yang mencegah BPE menggabungkan token lintas batas kata |
| Normalisasi NFKC | "Pembersihan Unicode" | Dekomposisi kanonik diikuti komposisi kompatibilitas -- ligatur "fi" menjadi "fi", "A" lebar penuh menjadi "A" |
| Templat obrolan | "Cara pesan menjadi token" | Format persis untuk mengubah daftar pesan peran/konten menjadi urutan token datar -- spesifik model dan harus cocok dengan format pelatihan |
| Token khusus | "Token kontrol" | ID token yang dicadangkan dan melewati BPE -- [BOS], [EOS], [PAD], penanda obrolan -- dicocokkan persis sebelum penggabungan |
| Fertilitas | "Token per kata" | Rasio token keluaran terhadap kata masukan -- 1.3 untuk bahasa Inggris pada GPT-4, 2-3 untuk bahasa Korea; lebih tinggi berarti konteks terbuang |
| tiktoken | "Tokenizer OpenAI" | Implementasi BPE dalam Rust dengan binding Python -- 10-100x lebih cepat daripada Python murni |
| Tabel penggabungan | "Kosakatanya" | Daftar urutan penggabungan pasangan byte yang dipelajari saat pelatihan -- inilah pengetahuan yang dipelajari tokenizer |

## Bacaan Lanjutan

- [Sumber tiktoken OpenAI](https://github.com/openai/tiktoken) -- implementasi BPE dalam Rust yang digunakan oleh GPT-3.5/4
- [HuggingFace tokenizers](https://github.com/huggingface/tokenizers) -- pustaka tokenizer berbasis Rust yang mendukung BPE, WordPiece, Unigram
- [Makalah Llama 3 (Meta, 2024)](https://arxiv.org/abs/2407.21783) -- rincian kosakata 128K dan pelatihan tokenizer
- [SentencePiece (Kudo & Richardson, 2018)](https://arxiv.org/abs/1808.06226) -- tokenisasi agnostik bahasa
- [Sumber tokenizer GPT-2](https://github.com/openai/gpt-2/blob/master/src/encoder.py) -- pemetaan byte-ke-Unicode yang asli
