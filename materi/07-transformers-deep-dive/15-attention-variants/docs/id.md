# Varian Atensi — Jendela Geser, Jarang, Diferensial

> Atensi penuh adalah sebuah lingkaran. Setiap token melihat token lain, dan memori yang menanggung biayanya. Tiga varian membengkokkan bentuk lingkaran itu dan menghemat separuh biayanya.

**Jenis:** Bangun
**Bahasa:** Python, Rust
**Prasyarat:** Fase 7 · 02 (Self-Attention), Fase 7 · 03 (Multi-Head), Fase 7 · 12 (KV Cache / Flash Attention)
**Waktu:** ~60 menit

## Permasalahan

Atensi penuh memakan memori $O(N^2)$ dan komputasi $O(N^2)$ terhadap panjang urutan. Untuk Llama 3 70B berkonteks 128K, itu 16 miliar entri atensi per lapisan, dikali 80 lapisan. Flash Attention (Pelajaran 12) menyembunyikan memori aktivasi $O(N^2)$ tetapi tidak mengubah biaya aritmetika — setiap token tetap menghadiri setiap token lainnya.

Tiga kelas varian mengubah topologi matriks atensi itu sendiri:

1. **Sliding window attention (SWA).** Setiap token menghadiri jendela tetangga yang tetap, bukan seluruh prefiks. Memori dan komputasi turun ke $O(N \cdot W)$ dengan $W$ sebagai jendela. Gemma 2/3, lapisan awal Mistral 7B, Phi-3-Long.
2. **Atensi jarang / blok.** Hanya pasangan `(i, j)` terpilih yang diberi skor; sisanya dipaksa berbobot nol. Longformer, BigBird, sparse transformer OpenAI.
3. **Atensi diferensial.** Hitung dua peta atensi dengan proyeksi Q/K terpisah, kurangkan satu dari yang lain. Menghilangkan "attention sink" yang menyalurkan bobot ke beberapa token pertama. DIFF Transformer milik Microsoft (2024).

Ketiganya hidup berdampingan. Model frontier 2026 sering mencampurnya: sebagian besar lapisan SWA-1024, setiap kelima adalah atensi penuh global, dan segelintir adalah head diferensial yang membersihkan hasil pengambilan (retrieval). Rasio 5:1 SWA-to-global dari Gemma 3 adalah default buku teks saat ini.

## Konsep

### Sliding Window Attention (SWA)

Setiap query pada posisi $i$ hanya menghadiri posisi dalam $[i - W, i]$ (SWA kausal) atau $[i - W/2, i + W/2]$ (bidireksional). Token di luar jendela mendapat $-\infty$ pada matriks skor.

```
atensi penuh:            jendela geser (W=4):
posisi 0-7               posisi 0-7, W=4
    0 1 2 3 4 5 6 7         0 1 2 3 4 5 6 7
0 | x                0 |  x
1 | x x              1 |  x x
2 | x x x            2 |  x x x
3 | x x x x          3 |  x x x x
4 | x x x x x        4 |    x x x x
5 | x x x x x x      5 |      x x x x
6 | x x x x x x x    6 |        x x x x
7 | x x x x x x x x  7 |          x x x x
```

Untuk $N = 8192$ dan $W = 1024$, matriks skor memiliki 1024 × 8192 baris bukan nol secara ekspektasi — pengurangan 8×.

**KV cache menyusut bersama SWA.** Hanya $W$ token terakhir dari K dan V yang perlu disimpan per lapisan. Untuk konfigurasi ala Gemma-3 (jendela 1024, konteks 128K), KV cache turun 128×.

**Biaya kualitas.** Transformer SWA-saja kesulitan pada pengambilan jarak jauh. Solusinya: selang-selingkan lapisan SWA dengan lapisan atensi penuh. Gemma 3 memakai 5:1 SWA:global. Mistral 7B memakai tumpukan SWA kausal di mana informasi "mengalir maju" melalui jendela yang tumpang tindih — setiap lapisan memperluas bidang penerimaan efektif sebesar $W$, dan setelah `L` lapisan model dapat menghadiri `L × W` token ke belakang.

### Atensi Jarang / Blok

Pilih pola jarangan `N × N` terlebih dahulu. Tiga bentuk kanonik:

- **Lokal + berlangkah (sparse transformer OpenAI).** Hadiri $W$ token terakhir ditambah setiap token ke-`stride` sebelum itu. Menangkap informasi lokal dan jarak jauh dengan komputasi $O(N \cdot \sqrt{N})$.
- **Longformer / BigBird.** Jendela lokal + sekumpulan kecil token global (mis. `[CLS]`) yang menghadiri semua dan dihadiri semua + tautan acak-jarang. Secara empiris konteks 2× pada kualitas yang setara.
- **Native Sparse Attention (DeepSeek, 2025).** Pelajari blok `(Q, K)` mana yang penting; lewati blok nol di tingkat kernel. Kompatibel dengan FlashAttention.

Atensi jarang adalah kisah rekayasa kernel. Matematikanya sederhana (topengi matriks skor); keuntungannya datang dari tidak pernah memuat entri nol ke SRAM. FlashAttention-3 dan API FlexAttention 2026 menjadikan pola jarang kustom sebagai warga kelas satu di PyTorch.

### Atensi Diferensial (DIFF Transformer, 2024)

Atensi reguler memiliki masalah "attention sink": softmax memaksa setiap baris berjumlah 1, sehingga token yang tidak ingin menghadiri sesuatu secara khusus membuang bobot ke token pertama (atau beberapa token pertama). Ini mencuri kapasitas yang seharusnya untuk konten nyata.

Atensi diferensial memperbaikinya dengan menghitung **dua** peta atensi dan mengurangkannya:

```math
\begin{aligned}
A_1 &= \mathop{\mathrm{softmax}}(Q_1 K_1^T / \sqrt{d}) \\
A_2 &= \mathop{\mathrm{softmax}}(Q_2 K_2^T / \sqrt{d}) \\
\text{DiffAttn} &= (A_1 - \lambda \cdot A_2) V
\end{aligned}
```

dengan $\lambda$ sebagai skalar yang dipelajari (biasanya 0,5–0,8). A1 menangkap bobot konten nyata; A2 menangkap sink. Pengurangan menghilangkan sink dan mengalokasikan ulang bobot ke token yang relevan.

Hasil yang dilaporkan (Microsoft 2024): perplexity 5–10% lebih rendah, konteks efektif 1,5–2× lebih panjang pada panjang pelatihan yang sama, pengambilan needle-in-haystack yang lebih tajam.

### Perbandingan Varian

| Varian | Komputasi | KV cache | Kualitas vs penuh | Penggunaan produksi |
|--------|-----------|----------|-------------------|---------------------|
| Atensi penuh | $O(N^2)$ | $O(N)$ per lapisan | baseline | lapisan default setiap model |
| SWA (jendela 1024) | $O(N \cdot W)$ | $O(W)$ per lapisan | -0.1 ppl, baik dengan lapisan global | Gemma 2/3, Phi-3-Long |
| Jarang lokal + berlangkah | $O(N \cdot \sqrt{N})$ | campuran | mirip SWA | sparse transformer OpenAI, Longformer |
| BigBird (lokal + global + acak) | $O(N)$ approx | campuran | setara penuh pada konteks 2× | BERT konteks panjang awal |
| Jarang native (DeepSeek-V3.2) | $O(N \cdot \text{active fraction})$ | $O(N)$ | dalam 0.05 ppl | DeepSeek-V3.2, 2025 |
| Diferensial | $O(2 \cdot N^2)$ | $O(2N)$ | -5 hingga -10% ppl | DIFF Transformer, model awal 2026 |

```figure
gqa-kv-sharing
```

## Bangunlah

Lihat `code/main.py`, atau port Rust-nya `code/main.rs`. Untuk menjalankan versi Rust: `rustc --edition 2021 -O code/main.rs -o /tmp/lesson && /tmp/lesson`. Kami mengimplementasikan pembanding mask kausal yang menampilkan atensi penuh, SWA, lokal+berlangkah, dan diferensial berdampingan pada urutan mainan.

### Langkah 1: mask kausal penuh (baseline)

```python
def causal_mask(n):
    return [[0.0 if j <= i else float("-inf") for j in range(n)] for i in range(n)]
```

Baseline dari Pelajaran 07. Segitiga bawah; bobot nol di atas diagonal.

### Langkah 2: mask kausal jendela geser

```python
def swa_mask(n, window):
    M = [[float("-inf")] * n for _ in range(n)]
    for i in range(n):
        lo = max(0, i - window + 1)
        for j in range(lo, i + 1):
            M[i][j] = 0.0
    return M
```

Satu parameter — `window`. Untuk `window >= n`, Anda mendapatkan kembali atensi kausal penuh. Untuk `window = 1`, setiap token hanya menghadiri dirinya sendiri.

### Langkah 3: mask jarang lokal + berlangkah

```python
def strided_mask(n, window, stride):
    M = [[float("-inf")] * n for _ in range(n)]
    for i in range(n):
        lo = max(0, i - window + 1)
        for j in range(lo, i + 1):
            M[i][j] = 0.0
        for j in range(0, i + 1, stride):
            M[i][j] = 0.0
    return M
```

Jendela lokal padat ditambah setiap token ke-`stride` kembali ke awal urutan. Bidang penerimaan bertumbuh dalam langkah logaritmik dengan lapisan tambahan.

### Langkah 4: atensi diferensial

```python
def diff_attention(Q1, K1, Q2, K2, V, lam):
    A1 = softmax_causal(Q1 @ K1.T / sqrt_d)
    A2 = softmax_causal(Q2 @ K2.T / sqrt_d)
    return (A1 - lam * A2) @ V
```

Dua lintasan atensi, dikurangi dengan koefisien pencampuran yang dipelajari. Dalam kode kami membandingkan peta panas attention-sink antara tunggal vs diferensial dan melihat sink itu runtuh.

### Langkah 5: ukuran KV cache

Cetak ukuran cache per lapisan pada $N = 131072$ untuk setiap varian. SWA dan varian jarang turun 10–100×. Diferensial dua kali lipat. Bayar tagihan memori Anda dengan sadar.

## Gunakanlah

Pola produksi 2026:

```python
from transformers import AutoModelForCausalLM
# Gemma 3 mencampur SWA (window=1024) dan lapisan global dengan rasio 5:1.
model = AutoModelForCausalLM.from_pretrained("google/gemma-3-27b-it")
# print(model.config.sliding_window, model.config.layer_types)
```

FlexAttention di PyTorch 2.5+ menerima fungsi mask:

```python
from torch.nn.attention.flex_attention import flex_attention, create_block_mask

def swa_pattern(b, h, q_idx, kv_idx):
    return (q_idx - kv_idx < 1024) & (q_idx >= kv_idx)

mask = create_block_mask(swa_pattern, B=batch, H=heads, Q_LEN=n, KV_LEN=n)
out = flex_attention(q, k, v, block_mask=mask)
```

Ini dikompilasi menjadi kernel Triton kustom. Dalam 10% dari kecepatan FlashAttention-3 untuk pola umum, dan fungsi mask-nya adalah callable Python.

**Kapan memilih masing-masing:**

- **Atensi penuh murni** — semua lapisan hingga ~16K konteks, atau ketika kualitas pengambilan menjadi yang terpenting.
- **Campuran SWA + global** — konteks panjang (>32K), memori pelatihan dan inferensi menjadi hambatan. Default 2026 di atas 32K.
- **Atensi blok jarang** — kernel kustom, pola kustom. Disiapkan untuk beban kerja khusus (pengambilan, audio).
- **Atensi diferensial** — beban kerja apa pun di mana kontaminasi attention sink merugikan (RAG konteks panjang, needle-in-haystack).

## Terapkanlah

Lihat `outputs/skill-attention-variant-picker.md`. Skill ini memilih topologi atensi untuk model baru diberikan panjang konteks target, tuntutan pengambilan, dan profil komputasi pelatihan/inferensi.

## Latihan

1. **Mudah.** Jalankan `code/main.py`. Verifikasi bahwa SWA pada `window=4` mengnolkan segala sesuatu di luar 4 token terakhir per baris. Verifikasi bahwa `window=n` mereproduksi atensi kausal penuh secara bit-identik.
2. **Sedang.** Implementasikan SWA kausal dengan `window=1024` di atas proyek akhir Pelajaran 07. Latih selama 1.000 langkah pada tinyshakespeare. Seberapa besar rugi val mundur dibanding atensi penuh? Seberapa besar memori puncak turun?
3. **Sulit.** Implementasikan campuran lapisan 5:1 ala Gemma-3 (5 SWA, 1 global) pada model proyek akhir. Bandingkan rugi, memori, dan kualitas generasi terhadap baseline SWA-murni dan global-murni pada parameter yang setara.
4. **Sulit.** Implementasikan atensi diferensial dengan $\lambda$ yang dipelajari per head. Latih pada tugas pengambilan sintetis (satu needle, 2.000 pengganggu). Ukur akurasi pengambilan vs baseline atensi tunggal pada parameter yang setara.

## Istilah Kunci

| Istilah | Apa yang orang katakan | Arti sebenarnya |
|---------|------------------------|-----------------|
| Sliding window attention (SWA) | "Atensi lokal" | Setiap query menghadiri $W$ token terakhirnya; KV cache menyusut ke $O(W)$. |
| Bidang penerimaan efektif | "Seberapa jauh ke belakang model melihat" | Dalam tumpukan SWA $L$ lapisan dengan jendela $W$, hingga $L \times W$ token. |
| Longformer / BigBird | "Lokal + global + acak" | Pola jarang dengan beberapa token global yang selalu menghadiri; pendekatan awal konteks panjang. |
| Native Sparse Attention | "Trik kernel DeepSeek" | Memelajari jarangan tingkat blok; melewati blok nol di tingkat kernel sambil menjaga kualitas. |
| Atensi diferensial | "Dua peta, yang satu mengurangi" | DIFF Transformer: kurangi peta atensi kedua yang dikali $\lambda$ dari yang pertama untuk menghilangkan attention sink. |
| Attention sink | "Bobot bocor ke token 0" | Normalisasi softmax memaksa baris berjumlah 1; query tanpa informasi membuang bobot ke posisi 0. |
| FlexAttention | "Mask sebagai Python" | API PyTorch 2.5+ yang mengompilasi fungsi mask sembarang menjadi kernel berbentuk FlashAttention. |
| Campuran tipe lapisan | "5:1 SWA-to-global" | Selang-selingkan lapisan atensi jarang dan penuh dalam tumpukan untuk menjaga kualitas dengan memori lebih rendah. |

## Bacaan Lanjutan

- [Beltagy, Peters, Cohan (2020). Longformer: Transformer untuk Dokumen Panjang](https://arxiv.org/abs/2004.05150) — makalah kanonik jendela geser + token global.
- [Zaheer dkk. (2020). Big Bird: Transformer untuk Urutan Lebih Panjang](https://arxiv.org/abs/2007.14062) — lokal + global + acak.
- [Child dkk. (2019). Menghasilkan Urutan Panjang dengan Sparse Transformer](https://arxiv.org/abs/1904.10509) — pola lokal+berlangkah milik OpenAI.
- [Tim Gemma (2024). Gemma 2: Meningkatkan Model Bahasa Terbuka pada Ukuran Praktis](https://arxiv.org/abs/2408.00118) — campuran 1:1 SWA:global.
- [Tim Gemma (2025). Laporan teknis Gemma 3](https://arxiv.org/abs/2503.19786) — campuran 5:1 dengan window=1024 yang kini menjadi default buku teks.
- [Ye dkk. (2024). Differential Transformer](https://arxiv.org/abs/2410.05258) — makalah DIFF Transformer.
- [Yuan dkk. (2025). Native Sparse Attention](https://arxiv.org/abs/2502.11089) — atensi jarang terpelajari milik DeepSeek-V3.2.
- [PyTorch — blog dan dokumen FlexAttention](https://pytorch.org/blog/flexattention/) — referensi API untuk pola mask-sebagai-callable di Gunakanlah.
