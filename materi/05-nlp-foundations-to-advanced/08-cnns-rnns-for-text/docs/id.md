# CNN dan RNN untuk Teks

> Konvolusi mempelajari n-gram. Rekurensi mengingat. Keduanya telah tergantikan oleh atensi. Keduanya tetap relevan pada perangkat keras yang terbatas.

**Jenis:** Build
**Bahasa:** Python, TypeScript
**Prasyarat:** Fase 3 · 11 (Pengantar PyTorch), Fase 5 · 03 (Embedding Kata), Fase 4 · 02 (Konvolusi dari Nol)
**Waktu:** ~75 menit

## Masalah

TF-IDF dan Word2Vec menghasilkan vektor datar yang mengabaikan urutan kata. Sebuah classifier yang dibangun di atasnya tidak dapat membedakan `dog bites man` dari `man bites dog`. Terkadang urutan kata membawa sinyalnya.

Dua keluarga arsitektur mengisi celah itu sebelum transformer tiba.

**Jaringan konvolusional untuk teks (TextCNN).** Terapkan konvolusi 1D atas urutan embedding kata. Sebuah filter dengan lebar 3 adalah detektor trigram yang dapat dipelajari: ia merentangi tiga kata dan menghasilkan sebuah skor. Tumpukkan lebar yang berbeda (2, 3, 4, 5) untuk mendeteksi pola pada berbagai skala. Lakukan max-pooling menjadi representasi berukuran tetap. Datar, paralel, cepat.

**Jaringan rekuren (RNN, LSTM, GRU).** Memproses token satu per satu, dengan menjaga keadaan tersembunyi (hidden state) yang membawa informasi ke depan. Sekuensial, menyimpan memori, Panjang masukan fleksibel. Mendominasi pemodelan sekuens dari 2014 hingga 2017, lalu atensi pun muncul.

Pelajaran ini membangun keduanya, lalu menyebutkan kegagalan yang mendorong munculnya atensi.

## Konsep

**TextCNN** (Kim, 2014). Token di-embedding. Konvolusi 1D dengan lebar `k` menggeser sebuah filter atas `k`-gram embedding yang berurutan, menghasilkan peta fitur. Global max-pooling atas peta itu memilih aktivasi terkuat. Gabungkan keluaran hasil max-pooling dari beberapa lebar filter. Teruskan ke head classifier.

Mengapa ini berfungsi. Sebuah filter adalah n-gram yang dapat dipelajari. Max-pooling bersifat invarian terhadap posisi, sehingga "not good" memicu fitur yang sama di awal maupun di tengah ulasan. Tiga lebar filter dengan 100 filter setiap ukuran memberi Anda 300 detektor n-gram yang dipelajari. Pelatihan bersifat paralel; tidak ada ketergantungan sekuensial.

**RNN.** Pada setiap langkah waktu `t`, keadaan tersembunyi `h_t = f(W * x_t + U * h_{t-1} + b)`. Bagikan `W`, `U`, `b` lintas waktu. Keadaan tersembunyi pada waktu `T` adalah ringkasan seluruh prefiks. Untuk klasifikasi, lakukan pooling atas `h_1 ... h_T` (maks, rata-rata, atau yang terakhir).

RNN biasa mengalami gradien yang menghilang. **LSTM** menambahkan gerbang yang memutuskan apa yang dilupakan, apa yang disimpan, dan apa yang dikeluarkan, sehingga menstabilkan gradien sepanjang sekuens panjang. **GRU** menyederhanakan LSTM menjadi dua gerbang; kinerjanya serupa dengan lebih sedikit parameter.

**RNN bidirectional** menjalankan satu RNN ke depan dan satu lagi ke belakang, lalu menggabungkan keadaan tersembunyinya. Representasi setiap token melihat konteks kiri dan kanan. Esensial untuk tugas penandaan.

```figure
rnn-unroll
```

## Bangun Sendiri

### Langkah 1: TextCNN di PyTorch

```python
import torch
import torch.nn as nn
import torch.nn.functional as F


class TextCNN(nn.Module):
    def __init__(self, vocab_size, embed_dim, n_classes, filter_widths=(2, 3, 4), n_filters=64, dropout=0.3):
        super().__init__()
        self.embed = nn.Embedding(vocab_size, embed_dim, padding_idx=0)
        self.convs = nn.ModuleList([
            nn.Conv1d(embed_dim, n_filters, kernel_size=k)
            for k in filter_widths
        ])
        self.dropout = nn.Dropout(dropout)
        self.fc = nn.Linear(n_filters * len(filter_widths), n_classes)

    def forward(self, token_ids):
        x = self.embed(token_ids).transpose(1, 2)
        pooled = []
        for conv in self.convs:
            c = F.relu(conv(x))
            p = F.max_pool1d(c, c.size(2)).squeeze(2)
            pooled.append(p)
        h = torch.cat(pooled, dim=1)
        return self.fc(self.dropout(h))
```

`transpose(1, 2)` membentuk ulang `[batch, seq_len, embed_dim]` menjadi `[batch, embed_dim, seq_len]` karena `nn.Conv1d` memperlakukan sumbu tengah sebagai kanal. Keluaran hasil pooling berukuran tetap berapa pun panjang masukan.

### Langkah 2: classifier LSTM

```python
class LSTMClassifier(nn.Module):
    def __init__(self, vocab_size, embed_dim, hidden_dim, n_classes, bidirectional=True, dropout=0.3):
        super().__init__()
        self.embed = nn.Embedding(vocab_size, embed_dim, padding_idx=0)
        self.lstm = nn.LSTM(embed_dim, hidden_dim, batch_first=True, bidirectional=bidirectional)
        factor = 2 if bidirectional else 1
        self.dropout = nn.Dropout(dropout)
        self.fc = nn.Linear(hidden_dim * factor, n_classes)

    def forward(self, token_ids):
        x = self.embed(token_ids)
        out, _ = self.lstm(x)
        pooled = out.max(dim=1).values
        return self.fc(self.dropout(pooled))
```

Lakukan max-pooling atas sekuens, bukan pooling keadaan terakhir. Untuk klasifikasi, max-pooling biasanya mengalahkan pengambilan keadaan tersembunyi terakhir karena informasi di ujung sekuens panjang cenderung mendominasi keadaan terakhir.

### Langkah 3: demo gradien yang menghilang (intuisi)

RNN biasa tanpa gerbang tidak dapat mempelajari ketergantungan jarak jauh. Pertimbangkan tugas mainan: prediksi apakah token `A` muncul di suatu tempat dalam sekuens. Jika `A` ada di posisi 1 dan sekuens sepanjang 100 token, gradien dari loss harus mengalir balik melalui 100 perkalian bobot rekuren. Jika bobot kurang dari 1, gradien menghilang. Jika lebih dari 1, gradien meledak.

```python
def vanishing_gradient_sim(seq_len, recurrent_weight=0.9):
    import math
    return math.pow(recurrent_weight, seq_len)


# Pada weight=0.9 selama 100 langkah:
#   0.9 ^ 100 ≈ 2.7e-5
# Gradien dari langkah 100 ke langkah 1 secara efektif nol.
```

LSTM memperbaiki ini dengan **keadaan sel (cell state)** yang mengalir melalui jaringan hanya dengan interaksi aditif (gerbang lupa menskalakannya secara multiplikatif, tetapi gradien tetap mengalir sepanjang "jalan tol"). GRU melakukan hal serupa dengan lebih sedikit parameter. Keduanya memberi Anda pelatihan yang stabil sepanjang sekuens 100+ langkah.

### Langkah 4: mengapa ini masih belum cukup

Tiga masalah tetap ada bahkan dengan LSTM.

1. **Hambatan sekuensial.** Melatih RNN pada sekuens sepanjang 1000 memerlukan 1000 langkah maju/mundur serial. Tidak dapat diparalelkan lintas waktu.
2. **Vektor konteks berukuran tetap pada susunan encoder-decoder.** Decoder hanya melihat keadaan tersembunyi terakhir encoder, yang dikompresi atas seluruh masukan. Masukan panjang kehilangan detail. Pelajaran 09 membahas ini secara langsung.
3. **Batas akurasi pada ketergantungan jauh.** LSTM mengungguli RNN biasa tetapi masih kesulitan menyebarkan informasi spesifik lintas 200+ langkah.

Atensi menyelesaikan ketiganya. Transformer sepenuhnya meninggalkan rekurensi. Pelajaran 10 adalah titik baliknya.

## Gunakan

`nn.LSTM`, `nn.GRU`, dan `nn.Conv1d` dari PyTorch siap untuk produksi. Kode pelatihannya standar.

Hugging Face menyediakan embedding terlatih yang Anda pasang sebagai lapisan masukan:

```python
from transformers import AutoModel

encoder = AutoModel.from_pretrained("bert-base-uncased")
for param in encoder.parameters():
    param.requires_grad = False


class BertCNN(nn.Module):
    def __init__(self, n_classes, filter_widths=(2, 3, 4), n_filters=64):
        super().__init__()
        self.encoder = encoder
        self.convs = nn.ModuleList([nn.Conv1d(768, n_filters, kernel_size=k) for k in filter_widths])
        self.fc = nn.Linear(n_filters * len(filter_widths), n_classes)

    def forward(self, input_ids, attention_mask):
        with torch.no_grad():
            out = self.encoder(input_ids=input_ids, attention_mask=attention_mask).last_hidden_state
        x = out.transpose(1, 2)
        pooled = [F.max_pool1d(F.relu(conv(x)), kernel_size=conv(x).size(2)).squeeze(2) for conv in self.convs]
        return self.fc(torch.cat(pooled, dim=1))
```

Daftar periksa "pakai bila sesuai kendala".

- **Inferensi di tepi / pada perangkat.** TextCNN dengan embedding GloVe ukurannya 10-100x lebih kecil daripada transformer. Jika target deploy Anda adalah ponsel, inilah tumpukannya.
- **Klasifikasi streaming / daring.** RNN memproses satu token per waktu; transformer memerlukan sekuens lengkap. Untuk teks yang masuk secara real-time, LSTM tetap menang.
- **Model kecil untuk baseline.** Iterasi cepat pada tugas baru. Latih TextCNN dalam 5 menit pada CPU.
- **Penandaan sekuens dengan data terbatas.** BiLSTM-CRF (pelajaran 06) masih merupakan arsitektur NER setingkat produksi untuk 1k-10k kalimat berlabel.

Semua kasus lainnya menggunakan transformer.

## Kirim

Simpan sebagai `outputs/prompt-text-encoder-picker.md`:

```markdown
---
name: text-encoder-picker
description: Pilih arsitektur encoder teks untuk sekumpulan kendala tertentu.
phase: 5
lesson: 08
---

Diberikan kendala (tugas, volume data, anggaran latensi, target deploy, anggaran komputasi), hasilkan:

1. Arsitektur encoder: TextCNN, BiLSTM, BiLSTM-CRF, fine-tuning transformer, atau "gunakan transformer terlatih sebagai encoder beku + head kecil".
2. Masukan embedding: inisialisasi acak, GloVe / fastText beku, atau embedding transformer terkontekstualisasi.
3. Resep pelatihan dalam 5 baris: pengoptimal, laju pembelajaran, ukuran batch, epoch, regularisasi.
4. Satu sinyal pemantauan. Untuk model RNN/CNN: tidak adanya mekanisme atensi berarti mereka melewatkan ketergantungan jarak jauh; periksa akurasi per panjang. Untuk transformer: keruntuhan fine-tuning jika LR terlalu tinggi; periksa loss pelatihan.

Tolak merekomendasikan fine-tuning transformer ketika data di bawah ~500 contoh berlabel tanpa menunjukkan bahwa baseline TextCNN / BiLSTM sudah mencapai plateau. Tandai deploy di tepi sebagai kebutuhan yang mengutamakan arsitektur di atas segalanya.
```

## Latihan

1. **Mudah.** Latih TextCNN pada dataset mainan 3 kelas (Anda yang membuat datanya). Verifikasi bahwa lebar filter (2, 3, 4) mengungguli satu lebar (3) pada F1 rata-rata.
2. **Sedang.** Implementasikan max-pool, mean-pool, dan pooling keadaan terakhir pada classifier LSTM. Bandingkan pada dataset kecil; dokumentasikan pooling mana yang menang dan ajukan hipotesis mengapa.
3. **Sulit.** Bangun penanda NER BiLSTM-CRF (gabungkan pelajaran 06 dan pelajaran ini). Latih pada CoNLL-2003. Bandingkan dengan baseline CRF-saja dari pelajaran 06 dan dengan fine-tuning BERT. Laporkan waktu pelatihan, memori, dan F1.

## Istilah Kunci

| Istilah | Yang orang katakan | Yang sebenarnya dimaksud |
|---------|--------------------|--------------------------|
| TextCNN | CNN untuk teks | Tumpukan konvolusi 1D atas embedding kata dengan global max-pool. Kim (2014). |
| RNN | Jaringan rekuren | Keadaan tersembunyi diperbarui pada setiap langkah waktu: `h_t = f(W x_t + U h_{t-1})`. |
| LSTM | RNN bergerbang | Menambahkan gerbang masukan / lupa / keluaran + keadaan sel. Melatih secara stabil sepanjang sekuens panjang. |
| GRU | LSTM yang lebih sederhana | Dua gerbang, bukan tiga. Akurasi serupa, lebih sedikit parameter. |
| Bidirectional | Kedua arah | RNN maju + mundur digabungkan. Setiap token melihat kedua sisi konteksnya. |
| Gradien yang menghilang | Sinyal pelatihan mati | Perkalian berulang dengan bobot <1 pada RNN biasa membuat gradien langkah awal secara efektif nol. |

## Bacaan Lanjutan

- [Kim, Y. (2014). Convolutional Neural Networks for Sentence Classification](https://arxiv.org/abs/1408.5882) — makalah TextCNN. Delapan halaman. Mudah dibaca.
- [Hochreiter, S. and Schmidhuber, J. (1997). Long Short-Term Memory](https://www.bioinf.jku.at/publications/older/2604.pdf) — makalah LSTM. Cukup jelas secara tak terduga.
- [Olah, C. (2015). Understanding LSTM Networks](https://colah.github.io/posts/2015-08-Understanding-LSTMs/) — diagram yang membuat LSTM dapat dipahami oleh semua orang.
