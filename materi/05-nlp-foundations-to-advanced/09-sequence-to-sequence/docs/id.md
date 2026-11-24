# Model Sequence-to-Sequence

> Dua RNN yang berpura-pura menjadi penerjemah. Hambatan yang mereka temui adalah alasan atensi ada.

**Jenis:** Build
**Bahasa:** Python, TypeScript
**Prasyarat:** Fase 5 · 08 (CNN + RNN untuk Teks), Fase 3 · 11 (Pengantar PyTorch)
**Waktu:** ~75 menit

## Masalah

Klasifikasi memetakan sekuens dengan panjang variabel menjadi satu label. Terjemahan memetakan sekuens dengan panjang variabel menjadi sekuens lain dengan panjang variabel. Masukan dan keluaran hidup di kosakata yang berbeda, mungkin dalam bahasa yang berbeda, tanpa jaminan kesamaan panjang.

Arsitektur seq2seq (Sutskever, Vinyals, Le, 2014) memecahkan ini dengan resep yang sengaja sederhana. Dua RNN. Satu membaca kalimat sumber dan menghasilkan vektor konteks berukuran tetap. Satu lagi membaca vektor itu dan menghasilkan kalimat target token demi token. Kode yang sama yang Anda tulis untuk pelajaran 08, direkatkan dengan cara yang berbeda.

Ini layak dipelajari karena dua alasan. Pertama, hambatan vektor konteks adalah kegagalan yang paling berguna secara pedagogis dalam NLP. Kegagalan ini mendorong segala hal yang membuat atensi dan transformer unggul. Kedua, resep pelatihannya (teacher forcing, scheduled sampling, beam search saat inferensi) tetap berlaku pada setiap sistem generasi modern, termasuk LLM.

## Konsep

**Encoder.** RNN yang membaca kalimat sumber. Keadaan tersembunyi terakhirnya adalah **vektor konteks** — ringkasan berukuran tetap dari seluruh masukan. Konon, tidak ada yang hilang kecuali sumbernya.

**Decoder.** RNN lain yang diinisialisasi dari vektor konteks. Pada setiap langkah, ia mengambil token yang sebelumnya dihasilkan sebagai masukan dan menghasilkan distribusi atas kosakata target. Ambil sampel atau argmax untuk memilih token berikutnya. Masukkan kembali. Ulangi sampai token `<EOS>` dihasilkan atau panjang maksimum tercapai.

**Pelatihan:** Loss cross-entropy pada setiap langkah decoder, dijumlahkan sepanjang sekuens. Propagasi balik standar sepanjang waktu melalui kedua jaringan.

**Teacher forcing.** Saat pelatihan, masukan decoder pada langkah `t` adalah token *ground-truth* di posisi `t-1`, bukan prediksi decoder sebelumnya sendiri. Ini menstabilkan pelatihan; tanpanya, kesalahan awal berantai dan model tidak pernah belajar. Saat inferensi, Anda harus menggunakan prediksi model sendiri, sehingga selalu ada kesenjangan distribusi antara pelatihan dan inferensi. Kesenjangan ini disebut **bias paparan (exposure bias)**.

**Hambatan.** Semua yang dipelajari encoder tentang sumber harus dimampatkan ke dalam satu vektor konteks itu. Kalimat panjang kehilangan detail. Kata langka menjadi kabur. Penyusunan ulang (chat noir vs. black cat) harus dihafal, bukan dihitung.

Atensi (pelajaran 10) memperbaiki ini dengan membiarkan decoder melihat *setiap* keadaan tersembunyi encoder, bukan hanya yang terakhir. Itulah inti idenya.

```figure
lstm-gates
```

## Bangun Sendiri

### Langkah 1: encoder

```python
import torch
import torch.nn as nn


class Encoder(nn.Module):
    def __init__(self, src_vocab_size, embed_dim, hidden_dim):
        super().__init__()
        self.embed = nn.Embedding(src_vocab_size, embed_dim, padding_idx=0)
        self.gru = nn.GRU(embed_dim, hidden_dim, batch_first=True)

    def forward(self, src):
        e = self.embed(src)
        outputs, hidden = self.gru(e)
        return outputs, hidden
```

`outputs` berbentuk `[batch, seq_len, hidden_dim]` — satu keadaan tersembunyi per posisi masukan. `hidden` berbentuk `[1, batch, hidden_dim]` — langkah terakhir. Pelajaran 08 berkata "lakukan pooling atas outputs untuk klasifikasi." Di sini kita menyimpan keadaan tersembunyi terakhir sebagai vektor konteks, dan mengabaikan keluaran per langkah.

### Langkah 2: decoder

```python
class Decoder(nn.Module):
    def __init__(self, tgt_vocab_size, embed_dim, hidden_dim):
        super().__init__()
        self.embed = nn.Embedding(tgt_vocab_size, embed_dim, padding_idx=0)
        self.gru = nn.GRU(embed_dim, hidden_dim, batch_first=True)
        self.fc = nn.Linear(hidden_dim, tgt_vocab_size)

    def forward(self, token, hidden):
        e = self.embed(token)
        out, hidden = self.gru(e, hidden)
        logits = self.fc(out)
        return logits, hidden
```

Decoder dipanggil satu langkah per waktu. Masukan: batch token tunggal dan keadaan tersembunyi saat ini. Keluaran: logits kosakata untuk token berikutnya dan keadaan tersembunyi yang sudah diperbarui.

### Langkah 3: loop pelatihan dengan teacher forcing

```python
def train_batch(encoder, decoder, src, tgt, bos_id, optimizer, teacher_forcing_ratio=0.9):
    optimizer.zero_grad()
    _, hidden = encoder(src)
    batch_size, tgt_len = tgt.shape
    input_token = torch.full((batch_size, 1), bos_id, dtype=torch.long)
    loss = 0.0
    loss_fn = nn.CrossEntropyLoss(ignore_index=0)

    for t in range(tgt_len):
        logits, hidden = decoder(input_token, hidden)
        step_loss = loss_fn(logits.squeeze(1), tgt[:, t])
        loss += step_loss
        use_teacher = torch.rand(1).item() < teacher_forcing_ratio
        if use_teacher:
            input_token = tgt[:, t].unsqueeze(1)
        else:
            input_token = logits.argmax(dim=-1)

    loss.backward()
    optimizer.step()
    return loss.item() / tgt_len
```

Dua tombol yang perlu disebut. `ignore_index=0` melewati loss pada token padding. `teacher_forcing_ratio` adalah probabilitas menggunakan token asli dibanding prediksi model pada setiap langkah. Mulai dari 1.0 (teacher forcing penuh) dan turunkan secara bertahap menjadi ~0,5 selama pelatihan untuk menutup kesenjangan bias paparan.

### Langkah 4: loop inferensi (greedy)

```python
@torch.no_grad()
def greedy_decode(encoder, decoder, src, bos_id, eos_id, max_len=50):
    _, hidden = encoder(src)
    batch_size = src.shape[0]
    input_token = torch.full((batch_size, 1), bos_id, dtype=torch.long)
    output_ids = []
    for _ in range(max_len):
        logits, hidden = decoder(input_token, hidden)
        next_token = logits.argmax(dim=-1)
        output_ids.append(next_token)
        input_token = next_token
        if (next_token == eos_id).all():
            break
    return torch.cat(output_ids, dim=1)
```

Dekode greedy memilih token berprobabilitas tertinggi pada setiap langkah. Ia bisa menyimpang: begitu Anda memilih sebuah token, Anda tidak bisa membatalkannya. **Beam search** menjaga `k` urutan parsial teratas tetap hidup dan memilih urutan lengkap dengan skor tertinggi di akhir. Lebar beam 3-5 adalah standar.

### Langkah 5: hambatan, didemonstrasikan

Latih model pada tugas mainan menyalin: sumber `[a, b, c, d, e]`, target `[a, b, c, d, e]`. Naikkan panjang sekuens. Amati akurasi.

```
seq_len=5   akurasi salin: 98%
seq_len=10  akurasi salin: 91%
seq_len=20  akurasi salin: 62%
seq_len=40  akurasi salin: 23%
```

Satu keadaan tersembunyi GRU tidak dapat menghafal tanpa kehilangan masukan sepanjang 40 token. Informasinya ada di setiap langkah encoder, tetapi decoder hanya melihat keadaan terakhir. Atensi memperbaiki ini secara langsung.

## Gunakan

PyTorch memiliki template seq2seq berbasis `nn.Transformer` dan `nn.LSTM`. Pustaka `transformers` dari Hugging Face menyediakan model encoder-decoder lengkap (BART, T5, mBART, NLLB) yang dilatih pada miliaran token.

```python
from transformers import AutoTokenizer, AutoModelForSeq2SeqLM

tok = AutoTokenizer.from_pretrained("facebook/bart-base")
model = AutoModelForSeq2SeqLM.from_pretrained("facebook/bart-base")

src = tok("Translate this to French: Hello, how are you?", return_tensors="pt")
out = model.generate(**src, max_new_tokens=50, num_beams=4)
print(tok.decode(out[0], skip_special_tokens=True))
```

Encoder-decoder modern meninggalkan RNN demi transformer. Bentuk tingkat tingginya (encoder, decoder, menghasilkan token demi token) identik dengan makalah seq2seq 2014. Mekanisme di dalam setiap blok yang berbeda.

### Kapan masih menggunakan seq2seq berbasis RNN

Hampir tidak pernah, untuk proyek baru. Pengecualian spesifik:

- Terjemahan streaming di mana Anda mengonsumsi masukan satu token per waktu dengan memori terbatas.
- Generasi teks pada perangkat di mana biaya memori transformer terlalu besar.
- Pengajaran. Memahami hambatan encoder-decoder adalah jalan tercepat untuk memahami mengapa transformer menang.

### Bias paparan dan mitigasinya

- **Scheduled sampling.** Turunkan rasio teacher forcing secara bertahap selama pelatihan agar model belajar pulih dari kesalahannya sendiri.
- **Minimum risk training.** Latih pada skor BLEU tingkat kalimat, bukan cross-entropy tingkat token. Lebih dekat dengan apa yang benar-benar Anda inginkan.
- **Fine-tuning reinforcement learning.** Beri hadiah pada generator sekuens dengan sebuah metrik. Digunakan dalam RLHF LLM modern.

Ketiganya tetap berlaku pada generasi berbasis transformer.

## Kirim

Simpan sebagai `outputs/prompt-seq2seq-design.md`:

```markdown
---
name: seq2seq-design
description: Rancang pipeline sequence-to-sequence untuk sebuah tugas tertentu.
phase: 5
lesson: 09
---

Diberikan sebuah tugas (terjemahan, peringkasan, parafrasa, penulisan ulang kueri), hasilkan:

1. Arsitektur. Encoder-decoder transformer terlatih (BART, T5, mBART, NLLB) adalah standar. Seq2seq berbasis RNN hanya untuk kendala tertentu.
2. Checkpoint awal. Sebutkan namanya (`facebook/bart-base`, `google/flan-t5-base`, `facebook/nllb-200-distilled-600M`). Sesuaikan checkpoint dengan tugas dan cakupan bahasa.
3. Strategi dekode. Greedy untuk keluaran deterministik, beam search (lebar 4-5) untuk kualitas, sampling dengan temperatur untuk keberagaman. Satu kalimat justifikasi.
4. Satu mode kegagalan yang diverifikasi sebelum kirim. Bias paparan muncul sebagai pergeseran generasi pada keluaran yang lebih panjang; ambil sampel 20 keluaran pada panjang persentil ke-90 dan periksa secara visual.

Tolak merekomendasikan pelatihan seq2seq dari nol untuk di bawah satu juta contoh paralel. Tandai sebagai rapuh setiap pipeline yang menggunakan dekode greedy untuk konten yang dilihat pengguna (greedy cenderung mengulang dan membentuk loop).
```

## Latihan

1. **Mudah.** Implementasikan tugas salin mainan. Latih seq2seq GRU pada pasangan masukan-keluaran di mana target sama dengan sumber. Ukur akurasi pada panjang 5, 10, 20. Reproduksikan hambatannya.
2. **Sedang.** Tambahkan dekode beam search dengan lebar beam 3. Ukur BLEU pada korpus paralel kecil dan bandingkan dengan greedy. Dokumentasikan di mana beam search menang (biasanya pada token terakhir) dan di mana tidak ada bedanya.
3. **Sulit.** Fine-tune `facebook/bart-base` pada dataset parafrasa 10k pasangan. Bandingkan keluaran beam-4 model hasil fine-tuning dengan model dasar pada masukan yang disisihkan. Laporkan BLEU dan pilih 10 contoh kualitatif.

## Istilah Kunci

| Istilah | Yang orang katakan | Yang sebenarnya dimaksud |
|---------|--------------------|--------------------------|
| Encoder | RNN masukan | Membaca sumber. Menghasilkan keadaan tersembunyi per langkah dan vektor konteks akhir. |
| Decoder | RNN keluaran | Diinisialisasi dari vektor konteks. Menghasilkan token target satu per satu. |
| Vektor konteks | Ringkasan | Keadaan tersembunyi encoder terakhir. Berukuran tetap. Hambatan yang diselesaikan atensi. |
| Teacher forcing | Gunakan token asli | Memberikan token sebelumnya yang ground-truth saat pelatihan. Menstabilkan pembelajaran. |
| Bias paparan | Kesenjangan latih/uji | Model yang dilatih pada token asli tidak pernah berlatih pulih dari kesalahannya sendiri. |
| Beam search | Dekode yang lebih baik | Menjaga `k` urutan parsial teratas tetap hidup pada setiap langkah, bukan berkomitmen secara greedy. |

## Bacaan Lanjutan

- [Sutskever, Vinyals, Le (2014). Sequence to Sequence Learning with Neural Networks](https://arxiv.org/abs/1409.3215) — makalah seq2seq orisinal. Empat halaman.
- [Cho et al. (2014). Learning Phrase Representations using RNN Encoder-Decoder for Statistical Machine Translation](https://arxiv.org/abs/1406.1078) — memperkenalkan GRU dan kerangka encoder-decoder.
- [Bahdanau, Cho, Bengio (2014). Neural Machine Translation by Jointly Learning to Align and Translate](https://arxiv.org/abs/1409.0473) — makalah atensi. Bacalah segera setelah pelajaran ini.
- [Tutorial PyTorch NLP from Scratch](https://pytorch.org/tutorials/intermediate/seq2seq_translation_tutorial.html) — kode seq2seq + atensi yang dapat dibangun sendiri.
