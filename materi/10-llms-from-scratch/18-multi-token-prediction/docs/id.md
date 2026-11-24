# Multi-Token Prediction (MTP)

> Setiap LLM autoregresif dari GPT-2 sampai Llama 3 dilatih dengan satu loss per posisi: prediksi token berikutnya. DeepSeek-V3 menambah loss kedua per posisi: prediksi token setelahnya. Ekstra 14B parameter (pada model 671B) disuling kembali ke model utama melalui aliran gradien, dan head MTP yang telah dilatih digunakan ulang saat inferensi sebagai draf dekode spekulatif dengan penerimaan di atas 80%. Throughput generasi 1,8× didapat secara gratis. Pelajaran ini membangun modul MTP sekuensial dari laporan teknis DeepSeek, menghitung loss dan tata letak parameter head bersama, serta menjelaskan mengapa MTP mempertahankan rantai kausal sementara MTP paralel Gloeckle dkk. yang orisinal memutuskannya.

**Jenis:** Build
**Bahasa:** Python (stdlib), Rust
**Prasyarat:** Fase 10 · 04 (pra-pelatihan mini GPT), Fase 10 · 15 (dekode spekulatif)
**Waktu:** ~60 menit

## Tujuan Pembelajaran

- Menyatakan objektif pelatihan MTP dan menurunkan loss gabungan lintas kedalaman prediksi.
- Menjelaskan perbedaan antara head MTP paralel Gloeckle dkk. (2024) dan modul MTP sekuensial DeepSeek-V3 serta mengapa desain sekuensial mempertahankan rantai kausal.
- Menghitung overhead parameter dan memori dari penambahan modul MTP pada suatu proses pra-pelatihan.
- Mengimplementasikan satu modul MTP dari nol: embedding bersama, blok transformer per kedalaman, proyeksi, dan head keluaran bersama.

## Masalahnya

Prediksi token berikutnya adalah objektif pelatihan LLM yang standar. Setiap keadaan tersembunyi diawasi untuk memprediksi tepat satu hal: token yang langsung mengikutinya. Itu adalah sinyal yang mengejutkan lemah. Sebagian besar informasi dalam suatu urutan melampaui satu token — struktur, koherensi, faktualitas, alur aritmetika. Model harus mempelajari semuanya dengan mengakumulasi banyak sinyal satu-token melintasi triliunan token.

MTP bertanya: bagaimana kalau setiap keadaan tersembunyi diawasi untuk memprediksi beberapa token masa depan sekaligus? Gloeckle dkk. (Meta, 2024) menunjukkan bahwa ini membantu. Implementasi mereka meletakkan beberapa head keluaran independen di atas backbone, masing-masing memprediksi offset yang berbeda. Paralel, sederhana, tetapi head-head itu melihat keadaan tersembunyi yang sama tanpa penyempurnaan hierarkis — dan prediksinya tidak membentuk rantai kausal, sehingga tidak dapat dipakai untuk dekode spekulatif.

DeepSeek-V3 (Desember 2024) mendesain ulang MTP sebagai modul sekuensial yang mempertahankan rantai kausal pada setiap kedalaman prediksi. Model memprediksi `t+1` dari `h_i^(0)`, lalu memprediksi `t+2` dari keadaan tersembunyi baru `h_i^(1)` yang menggabungkan `h_i^(0)` dengan embedding `E(t+1)`, dan seterusnya. Setiap kedalaman adalah blok transformer kecilnya sendiri. Embedding bersama dan head keluaran bersama menjaga overhead parameter tetap moderat. Pada skala DeepSeek-V3, 14B parameter ekstra pada modul MTP di atas 671B bobot model utama. Overhead 2% itu membeli sinyal pelatihan yang lebih padat DAN draf dekode spekulatif yang siap pakai saat inferensi.

Pelajaran ini membangun satu modul MTP dan loss berkedalaman D dari nol. Matematikanya rapi. Implementasinya 150 baris.

## Konsepnya

### Resep MTP sekuensial

DeepSeek-V3 menambah `D` modul MTP di atas model utama. Model utama memprediksi `t_{i+1}`. Setiap modul `k` (untuk `k = 1..D`) memprediksi token `k` langkah lebih jauh, `t_{i+k+1}`, diberi prefiks sampai posisi `i`.

Modul `k` terdiri dari:

- Sebuah blok transformer `T_k` dengan atensi dan MLP miliknya sendiri.
- Matriks proyeksi `M_k` yang menggabungkan keadaan tersembunyi kedalaman sebelumnya dengan embedding token ground-truth kedalaman berikutnya.
- Embedding bersama `E` (sama seperti model utama).
- Head keluaran bersama `Out` (sama seperti model utama).

Saat pelatihan, untuk prefiks sampai posisi `i`, keadaan tersembunyi per kedalaman adalah:

```math
\begin{aligned}
h_i^{(0)} &= \text{main model backbone at position } i \\
h_i^{(k)} &= T_k\Big( M_k \cdot \mathop{\mathrm{concat}}\big(\mathop{\mathrm{RMSNorm}}(h_i^{(k-1)}), \mathop{\mathrm{RMSNorm}}(E(t_{i+k}))\big) \Big) \quad \text{for } k \ge 1
\end{aligned}
```

Prediksi per kedalaman adalah:

```math
\text{logits}_{i+k+1} = \mathop{\mathrm{Out}}(h_i^{(k)}) \quad \text{for } k = 1, \ldots, D
```

Loss per kedalaman adalah cross-entropy terhadap ground-truth `t_{i+k+1}`:

```math
L_k = \mathop{\mathrm{CE}}(\text{logits}_{i+k+1}, t_{i+k+1})
```

Loss gabungan lintas kedalaman:

```math
L_{\text{MTP}} = \frac{\lambda}{D} \sum_{k=1}^{D} L_k
```

`lambda` adalah faktor pembobot kecil — DeepSeek-V3 memakai 0,3 untuk 10% pertama pelatihan dan 0,1 setelahnya. Total loss pelatihan adalah `L_main + L_MTP`.

### Mengapa sekuensial, bukan paralel

MTP paralel orisinal Gloeckle memiliki D head keluaran, masing-masing diterapkan langsung ke `h_i^(0)`. Setiap head memprediksi `t_{i+k}` dari keadaan tersembunyi backbone yang sama. Itu melatih dengan baik, tetapi prediksinya tidak terkondisi satu sama lain. Anda tidak bisa memakai keluaran `head_1` untuk membantu `head_2` — head-head itu bekerja secara paralel.

Desain sekuensial DeepSeek-V3 membangun `h_i^(k)` dari `h_i^(k-1)` ditambah embedding token berikutnya yang sebenarnya, `E(t_{i+k})`. Ini mempertahankan rantai kausal: untuk memprediksi `t_{i+k+1}`, modul pada kedalaman `k` melihat apa yang ada di `t_{i+k}`. Secara struktural ini identik dengan cara dekoder autoregresif mengonsumsi keluarannya sendiri, yang membuat modul MTP langsung dapat dipakai sebagai draf dekode spekulatif.

Saat inferensi: masukkan `h_i^(k-1)` dan `t_{i+k}` yang didraf ke modul `k`, dapatkan prediksi untuk `t_{i+k+1}`. Ulangi. Itu persis draf gaya EAGLE, dengan modul MTP terlatih sebagai jaringan draf. DeepSeek-V3 melaporkan penerimaan di atas 80% pada modul MTP pertama dan percepatan ~1,8×.

### Pembukuan parameter

Untuk model dengan hidden `h` dan kosakata `V`:

- Model utama: miliaran parameter, ditambah satu head keluaran berukuran `V * h`.
- Head keluaran bersama: pakai ulang head model utama. Tanpa parameter tambahan.
- Embedding bersama: pakai ulang embedding model utama. Tanpa parameter tambahan.
- Per modul MTP:
  - Proyeksi `M_k`: `(2h) * h = 2h^2`.
  - Blok transformer `T_k`: atensi (`4h^2` untuk MHA) ditambah MLP (biasanya `8h^2` untuk SwiGLU dengan rasio 8/3). Sekitar `12h^2` per blok.

Total tambahan per modul: `~14h^2`. Untuk `h = 7168` milik DeepSeek-V3, modul D = 1: `~14 * 7168^2 = ~720M` parameter di atas kertas. DeepSeek-V3 melaporkan 14B — selisihnya sebagian besar karena lapisan pakar juga berupa MoE di dalam modul MTP.

### Imbalan dekode spekulatif

Selama pra-pelatihan, modul MTP memperlambat pelatihan sekitar 10% (lebih banyak komputasi forward, loss tambahan). Imbalannya dua kali lipat:

1. Sinyal pelatihan lebih padat. Setiap keadaan tersembunyi melihat D+1 target supervisi. Efek terukur pada MMLU, GSM8K, MATH, HumanEval: perbaikan konsisten beberapa poin persentase dalam ablasi DeepSeek-V3.

2. Draf dekode spekulatif gratis saat inferensi. Modul MTP sudah dilatih untuk memprediksi beberapa token berikutnya. Digunakan ulang sebagai jaringan draf, ia memberi tingkat penerimaan di atas 80%. Pada level itu, dekode spekulatif dengan N=3 atau N=5 memberi throughput 1,8×. Biaya waktu pelatihan 10% terbayar pada pertama kali Anda menjalankan inferensi.

### Hubungan dengan EAGLE

EAGLE melatih model draf kecil secara TERPISAH setelah pra-pelatihan. MTP memanggang draf ke dalam pra-pelatihan. Kedua pendekatan bertemu pada tingkat penerimaan serupa, tetapi melalui pipeline yang berbeda:

| Dimensi | EAGLE-3 | MTP (DeepSeek-V3) |
|---------|---------|------------------|
| Kapan dilatih | Pasca-pra-pelatihan | Selama pra-pelatihan |
| Kompatibel mundur dengan bobot yang ada | Ya | Tidak (perlu dilatih ulang) |
| Parameter draf | 1-2 lapisan transformer | 1 blok transformer + proyeksi |
| Tingkat penerimaan | 0,88-0,92 | 0,80+ pada kedalaman 1 |
| Manfaat di luar percepatan | Hanya dekode spekulatif | Sinyal pelatihan lebih padat + percepatan |

```figure
multi-token-predict
```

## Bangunlah

`code/main.py` membangun satu modul MTP dari awal sampai akhir: embedding bersama, proyeksi, blok transformer, head keluaran bersama. Selanjutnya ia menghitung loss cross-entropy per kedalaman pada urutan sintetis pendek dan mencetak jumlah parameter per komponen. Kosakata mainan berisi 32 token agar angka-angkanya tetap mudah dibaca.

### Langkah 1: tabel embedding bersama

Satu tabel `vocab_size x hidden` dipakai oleh model utama DAN oleh setiap modul MTP pada setiap kedalaman. Bukan salinan kedua — benar-benar tensor yang sama.

### Langkah 2: kombinasi per kedalaman

```python
def combine(prev_hidden, next_token_embed, M_k):
    # gabungkan sepanjang dimensi fitur, lalu proyeksikan ke hidden
    concat = rms_norm(prev_hidden) + rms_norm(next_token_embed)  # pengganti penjumlahan vektor
    projected = matvec(M_k, concat)
    return projected
```

DeepSeek-V3 yang asli menggabungkan kedua vektor hasil RMSNorm menjadi `[2h]` dan memproyeksikannya dengan matriks `h x 2h`. Versi mainan memakai penjumlahan vektor agar tetap ringkas dengan stdlib.

### Langkah 3: blok transformer pada kedalaman k

Self-attention ditambah MLP. Pada versi mainan, blok atensi linear satu lapis dan MLP SwiGLU menjaga strukturnya tetap terlihat tanpa numpy.

### Langkah 4: head keluaran bersama

Pakai ulang proyeksi keluaran model utama. Logits di seluruh kosakata.

### Langkah 5: loss per kedalaman

Cross-entropy dari softmax(logits) terhadap token ground-truth pada offset `k+1`. Agregat lintas kedalaman dengan faktor penskalaan `lambda / D`.

### Langkah 6: pembukuan parameter

Cetak total jumlah parameter, jumlah parameter bersama (embedding, head), dan jumlah tambahan per modul. Tampilkan rasio tambahan MTP terhadap ukuran model utama.

## Pakailah

MTP terintegrasi ke dalam DeepSeek-V3 (Desember 2024) dan seri DeepSeek-R1. Saat inferensi:

- Stack serving milik DeepSeek sendiri mengonsumsi modul MTP sebagai dekode spekulatif secara langsung.
- vLLM dan SGLang memiliki jalur integrasi untuk MTP DeepSeek-V3 per April 2026.
- Tutorial ROCm SGLang dari AMD menunjukkan konfigurasi dekode spekulatif MTP spesifik dengan percepatan terukur 1,8× pada checkpoint V3.

Kapan memakai MTP dalam proses pra-pelatihan baru:

- Anda mengendalikan pipeline pra-pelatihan penuh dan ingin mendapat sinyal pelatihan yang lebih padat.
- Anda tahu akan melayani model dalam skala besar dan ingin dekode spekulatif secara gratis.
- Ukuran hidden Anda minimal 4096. Pada skala 1B, overhead lebih merugikan daripada keuntungannya.

Kapan tidak:

- Menyetel halus model dense yang sudah pra-dilatih. Modul MTP tidak dilatih.
- Model riset yang ingin baseline bersih untuk dibandingkan. MTP mengubah arsitektur.

## Kirimkan

Pelajaran ini menghasilkan `outputs/skill-mtp-planner.md`. Diberikan spesifikasi proses pra-pelatihan (ukuran model, data, komputasi), berkas itu mengembalikan rencana integrasi MTP: jumlah kedalaman D, jadwal `lambda`, overhead memori, dan pengkabelan dekode spekulatif saat inferensi.

## Latihan

1. Jalankan `code/main.py`. Tunjukkan bahwa loss per kedalaman turun secara monoton ketika sinyal sintetis diperkuat. Ubah data sintetis agar memakai pola tetap dan verifikasi bahwa loss kedalaman 1 dan kedalaman 2 sama-sama konvergen.

2. Hitung overhead parameter untuk model dense 70B (hidden 8192, 80 layer) dengan modul MTP D=1. Bandingkan dengan overhead 14B yang dilaporkan DeepSeek-V3. Jelaskan mengapa angka DeepSeek lebih tinggi: blok transformer MTP mewarisi struktur MoE yang sama, sehingga membengkakkan jumlah parameter per modul.

3. Implementasikan D=2 pada versi mainan: tambah modul MTP kedua yang menerima $h^{(1)}$ dan memprediksi `t_{i+3}`. Verifikasi bahwa loss gabungan dan pembukuan parameter cocok dengan persamaan 19-21 dalam makalah DeepSeek.

4. Ubah versi mainan menjadi MTP paralel (gaya Gloeckle): tambah D head keluaran di atas keadaan tersembunyi model utama, masing-masing memprediksi offset berbeda. Ukur bagaimana loss per kedalaman dibandingkan dengan versi sekuensial pada sinyal sintetis yang sama. Versi sekuensial seharusnya menghasilkan loss kedalaman-k lebih rendah untuk k > 1 karena ia terkondisi pada prediksi antara.

5. Pakai modul MTP terlatih sebagai draf gaya EAGLE: panggil modul k untuk mengusulkan `t_{i+k+1}` saat inferensi. Ukur tingkat penerimaan token draf ini terhadap prediksi model utama pada urutan yang disisihkan. Jika Anda mencapai 50%+ pada versi mainan, Anda telah mereproduksi sifat empiris MTP-sebagai-draf.

## Istilah Kunci

| Istilah | Yang orang katakan | Arti sebenarnya |
|---------|--------------------|-----------------|
| Modul MTP | "Blok loss tambahan" | Blok transformer kecil ditambah proyeksi yang memprediksi token `k` posisi di depan model utama |
| Kedalaman prediksi | "Offset ke berapa" | Bilangan bulat `k` sedemikian sehingga modul `k` memprediksi `t_{i+k+1}` dari prefiks sampai posisi `i` (model utama berada di kedalaman 0 dan memprediksi `t_{i+1}`) |
| MTP paralel | "Gaya Gloeckle" | D head independen pada keadaan tersembunyi backbone yang sama, tanpa rantai kondisional |
| MTP sekuensial | "Gaya DeepSeek-V3" | Setiap modul terkondisi pada keadaan tersembunyi kedalaman sebelumnya ditambah embedding token berikutnya; mempertahankan rantai kausal |
| Head keluaran bersama | "Pakai ulang head utama" | Modul MTP memanggil LM head model utama, bukan proyeksi keluaran terpisah |
| Embedding bersama | "Pakai ulang tabel utama" | Tabel embedding kosakata yang sama dipakai di mana saja; tanpa parameter duplikat |
| Matriks proyeksi $M_k$ | "Gabungkan hidden + token berikutnya" | Lapisan linear `h x 2h` yang melipat keadaan tersembunyi sebelumnya dan embedding token target menjadi masukan kedalaman berikutnya |
| Loss gabungan $L_{\text{MTP}}$ | "Rata-rata loss tambahan" | Rata-rata aritmetika dari loss cross-entropy per kedalaman, diskalakan oleh `lambda` |
| Tingkat penerimaan pada kedalaman 1 | "Seberapa sering draf MTP benar" | Tingkat prediksi top-1 modul MTP D=1 yang sama dengan prediksi top-1 model utama; 80%+ pada DeepSeek-V3 |
| Pembobotan lambda | "Pentingnya loss tambahan" | Faktor penskalaan per kedalaman; 0,3 di awal pelatihan, 0,1 kemudian pada DeepSeek-V3 |

## Bacaan Lanjutan

- [DeepSeek-AI — DeepSeek-V3 Technical Report (arXiv:2412.19437)](https://arxiv.org/abs/2412.19437) — deskripsi lengkap MTP sekuensial (Bagian 2.2), termasuk persamaan loss gabungan dan percepatan 1,8× saat inferensi
- [Gloeckle dkk. — Better & Faster Large Language Models via Multi-token Prediction (arXiv:2404.19737)](https://arxiv.org/abs/2404.19737) — baseline MTP paralel yang menjadi titik perbaikan desain DeepSeek
- [Model card DeepSeek-V3 di Hugging Face](https://huggingface.co/deepseek-ai/DeepSeek-V3) — total 685B (671B utama + 14B MTP), catatan deployment
- [Leviathan dkk. — Fast Inference from Transformers via Speculative Decoding (arXiv:2211.17192)](https://arxiv.org/abs/2211.17192) — kerangka dekode spekulatif yang dimasuki MTP
- [Li dkk. — EAGLE-3 (arXiv:2503.01840)](https://arxiv.org/abs/2503.01840) — arsitektur draf EAGLE tahun 2025, tandingan yang bersaing dengan MTP
