# Dekode Spekulatif — Draf, Verifikasi, Ulangi

> Dekode autoregresif bersifat serial. Setiap token menunggu token sebelumnya. Dekode spekulatif memutus rantai itu: model murah mendraf N token, model mahal memverifikasi semuanya dalam satu forward pass. Ketika drafnya benar, Anda membayar satu forward besar untuk N generasi.

**Jenis:** Bangun
**Bahasa:** Python, Rust
**Prasyarat:** Fase 7 · 07 (GPT Causal LM), Fase 7 · 12 (KV Cache & Flash Attention)
**Waktu:** ~60 menit

## Permasalahan

LLM 70B yang mengambil sampel satu token memakan ~30 ms di H100. Model draf 3B memakan ~3 ms. Jika kita membiarkan 3B mendraf 5 token lebih dulu, lalu menjalankan 70B *sekali* untuk memverifikasi kelimanya, totalnya $5 \times 3 + 30 = 45$ ms untuk hingga 5 token yang diterima — dibanding `5×30 = 150 ms` untuk generasi lurus. Itulah pitch lengkap dekode spekulatif: tukar sedikit memori GPU ekstra (model draf) dengan latensi dekode 2–4× lebih rendah.

Trik ini harus menjaga distribusi. Pengambilan sampel spekulatif, diperkenalkan oleh Leviathan dkk. (2023) dan secara bersamaan oleh Chen dkk., menjamin bahwa urutan keluaran **terdistribusi identik** dengan apa yang akan dihasilkan model besar sendiri. Tidak ada kompromi kualitas. Hanya lebih cepat.

Empat keluarga pasangan draf-verifikator mendominasi inferensi 2026:

1. **Spekulatif vanilla (Leviathan 2023).** Model draf terpisah (mis. Llama 3 1B) + verifikator (mis. Llama 3 70B).
2. **Medusa (Cai 2024).** Beberapa head dekode pada verifikator memprediksi posisi `t+1..t+k` secara paralel. Tanpa model draf terpisah.
3. **Keluarga EAGLE (Li 2024, 2025).** Draf ringan yang memakai ulang hidden state verifikator; tingkat penerimaan lebih dekat; 3–4× tipikal.
4. **Dekode lookahead (Fu 2024).** Iterasi Jacobi; sama sekali tidak memerlukan model draf. Swa-spekulasi. Niche, tetapi bebas dependensi.

Setiap stack inferensi produksi pada 2026 mengirimkan dekode spekulatif secara default. vLLM, TensorRT-LLM, SGLang, dan llama.cpp semuanya mendukung setidaknya vanilla + EAGLE-2.

## Konsep

### Algoritme inti

Diberikan verifikator $M_q$ dan draf $M_p$ yang lebih murah:

1. Misalkan $x_1 \ldots x_k$ adalah prefiks yang sudah didekode.
2. **Draf**: gunakan $M_p$ untuk mengusulkan secara autoregresif $d_{k+1}, d_{k+2}, \ldots, d_{k+N}$ dengan probabilitas draf $p_1 \ldots p_N$.
3. **Verifikasi secara paralel**: jalankan $M_q$ sekali pada $x_1 \ldots x_k, d_{k+1}, \ldots, d_{k+N}$, mendapatkan probabilitas verifikator $q_1 \ldots q_{N+1}$ untuk posisi $k+1 \ldots k+N+1$.
4. **Terima/tolak setiap token draf dari kiri ke kanan**: untuk setiap $i$, terima dengan probabilitas $\min(1, q_i(d_i) / p_i(d_i))$.
5. Pada penolakan pertama di posisi $j$: ambil sampel $t_j$ dari distribusi "residual" $(q_j - p_j)_+$ yang dinormalisasi. Semua draf setelah $j$ dibuang.
6. Pada penerimaan semua $N$: ambil satu token ekstra $t_{N+1}$ dari $q_{N+1}$ (token bonus gratis).

Trik distribusi residual adalah wawasan matematis yang menjaga keluaran terdistribusi tepat seolah $M_q$ mengambil sampel dari awal.

### Apa yang menentukan percepatan

Misalkan $\alpha$ = tingkat penerimaan ekspektasi per token draf. Misalkan $c$ = rasio biaya draf terhadap verifikator. Per langkah:

- Generasi naif melakukan 1 panggilan model besar per token.
- Spekulatif melakukan 1 panggilan model besar per $(1 - \alpha^{N+1}) / (1 - \alpha) \approx 1/(1-\alpha)$ token ketika $\alpha$ tinggi.

Aturan praktis umum pada $\alpha = 0,75$ dan $N = 5$: 3× lebih sedikit panggilan model besar. Biaya draf 5× lebih murah. Waktu tembok (wall-clock) total turun ~2,5×.

**$\alpha$ bergantung pada:**

- Seberapa baik draf mendekati verifikator. Keluarga yang sama / data pelatihan yang sama meningkatkan $\alpha$ secara signifikan.
- Strategi dekode. Draf greedy terhadap verifikator greedy: $\alpha$ tinggi. Pengambilan sampel bersuhu: lebih sulit dicocokkan; penerimaan turun.
- Jenis tugas. Kode dan keluaran terstruktur lebih banyak diterima (dapat diprediksi); tulisan kreatif bebas lebih sedikit.

### Medusa — draf tanpa model draf

Medusa menggantikan model draf dengan head keluaran ekstra pada verifikator. Pada posisi `t`:

```
batang bersama → hidden h_t
    ├── head_0: prediksi token di t+1  (LM head standar)
    ├── head_1: prediksi token di t+2
    ├── head_2: prediksi token di t+3
    ├── head_3: prediksi token di t+4
```

Setiap head menghasilkan logitnya sendiri. Saat inferensi Anda mengambil sampel dari setiap head untuk mendapatkan urutan kandidat, lalu memverifikasi dengan satu forward pass menggunakan skema tree-attention yang mempertimbangkan semua kelanjutan kandidat sekaligus.

Kelebihan: tanpa model kedua. Kekurangan: menambah parameter yang dapat dilatih; memerlukan tahap fine-tuning terawasi (~1B token); tingkat penerimaan sedikit lebih rendah daripada spekulatif vanilla dengan draf yang baik.

### EAGLE — draf lebih baik dengan memakai ulang hidden state

EAGLE-1/2/3 (Li dkk., 2024–2025) menjadikan model draf sebagai transformer sangat kecil (biasanya 1 lapisan) yang mengonsumsi hidden state lapisan terakhir verifikator. Karena draf melihat representasi fitur verifikator, prediksinya berkorelasi kuat dengan distribusi keluaran verifikator. Tingkat penerimaan naik dari ~0,6 (vanilla) menjadi 0,85+. Ini adalah angka per token pada benchmark chat, dengan draf yang dilatih pada data chat yang cocok. Pada lalu lintas produksi campuran, Fase 17 · 05 merencanakan 0,6 hingga 0,8 untuk EAGLE-3, dan lalu lintas domain bisa lebih rendah.

EAGLE-3 (2025) menambahkan pencarian pohon atas kelanjutan kandidat. vLLM dan SGLang mengirimkan EAGLE-2/3 sebagai jalur spekulatif default untuk Llama 3/4 dan Qwen 3.

### Tarian KV cache

Verifikasi memasukkan $N$ token draf ke verifikator dalam satu forward pass. Ini memperpanjang KV cache verifikator sebanyak $N$ entri. Jika sebagian draf ditolak, Anda harus menggulung kembali cache ke panjang prefiks yang diterima.

Implementasi produksi (`--speculative-config` vLLM, LookaheadDecoder TensorRT-LLM) menangani ini dengan buffer KV sementara. Tulis dulu, terapkan saat diterima. Secara konseptual tidak sulit, tetapi rumit.

```figure
draft-verify-tokens
```

## Bangunlah

Lihat `code/main.py`. Kami mengimplementasikan algoritme pengambilan sampel spekulatif inti (langkah penolakan + distribusi residual) dengan:

- Model "besar" yang berupa softmax deterministik atas distribusi yang ditulis tangan (agar kita dapat memverifikasi matematika penerimaan secara analitik).
- Model "draf" yang merupakan gangguan terhadap model besar.
- Loop terima/tolak yang menghasilkan distribusi marginal yang sama dengan pengambilan sampel langsung.

`code/main.rs` adalah port Rust dengan fungsi dan demo yang sama. Hanya memakai pustaka standar. Angka acaknya berbeda dari Python, tetapi kesimpulan yang dicetak sama.

```bash
python3 code/main.py
rustc --edition 2021 -O code/main.rs -o /tmp/lesson && /tmp/lesson
```

### Langkah 1: langkah penolakan

```python
def accept_or_reject(q_prob, p_prob, draft_token, u):
    ratio = q_prob / p_prob if p_prob > 0 else float("inf")
    return u < min(1.0, ratio)
```

`u` adalah bilangan acak seragam. `q_prob` adalah probabilitas verifikator untuk token yang didraf. `p_prob` adalah probabilitas model draf. Teorema Leviathan menyatakan bahwa keputusan Bernoulli ini, diikuti oleh pengambilan sampel dari residual saat penolakan, menjaga distribusi verifikator secara tepat.

### Langkah 2: distribusi residual

```python
def residual_dist(q, p):
    raw = [max(0.0, qi - pi) for qi, pi in zip(q, p)]
    s = sum(raw)
    return [r / s for r in raw]
```

Kurangi `p` dari `q` elemen per elemen, jepit nilai negatif menjadi nol, normalisasi ulang. Ambil sampel dari ini pada setiap penolakan.

### Langkah 3: satu langkah spekulatif

```python
def spec_step(prefix, q_model, p_model, N, rng):
    drafts = []
    p_probs = []
    ctx = list(prefix)
    for _ in range(N):
        p_dist = p_model(ctx)
        d = sample(p_dist, rng)
        drafts.append(d)
        p_probs.append(p_dist[d])
        ctx.append(d)

    q_dists = [q_model(prefix + drafts[:i]) for i in range(N + 1)]

    for i, d in enumerate(drafts):
        u = rng.random()
        q_prob = q_dists[i][d]
        p_prob = p_probs[i]
        if u < min(1.0, q_prob / p_prob if p_prob > 0 else float("inf")):
            prefix = prefix + [d]
        else:
            res = residual_dist(q_dists[i], p_model(prefix))
            prefix = prefix + [sample(res, rng)]
            return prefix
    prefix = prefix + [sample(q_dists[N], rng)]
    return prefix
```

Lima draf diterima → satu bonus → enam token dihasilkan dalam satu pass verifikator.

### Langkah 4: ukur tingkat penerimaan

Jalankan 10.000 langkah spekulatif pada berbagai tingkat kualitas draf. Plot tingkat penerimaan vs divergensi KL antara distribusi draf dan verifikator. Anda harus melihat hubungan monoton yang bersih.

### Langkah 5: verifikasi ekuivalensi distribusi

Secara empiris: histogram token yang dihasilkan oleh loop spekulatif harus cocok dengan histogram yang dihasilkan oleh pengambilan sampel langsung dari verifikator. Ini teorema Leviathan dalam praktik. Uji chi-square mengonfirmasi dalam batas galat sampel.

## Gunakanlah

Produksi:

```bash
# vLLM dengan EAGLE
vllm serve meta-llama/Llama-3.1-70B-Instruct \
    --speculative-config '{"method": "eagle", "model": "/models/llama-3.1-eagle-70b", "draft_tensor_parallel_size": 1, "num_speculative_tokens": 5}'

# vLLM dengan model draf vanilla
vllm serve meta-llama/Llama-3.1-70B-Instruct \
    --speculative-config '{"method": "draft_model", "model": "meta-llama/Llama-3.2-1B-Instruct", "num_speculative_tokens": 5}'
```

TensorRT-LLM memiliki jalur Medusa tercepat per pertengahan 2026. `faster-whisper` membungkus dekode spekulatif untuk Whisper-large dengan draf kecil.

**Memilih draf:**

| Strategi | Kapan memilih | Percepatan |
|----------|---------------|------------|
| Draf vanilla (keluarga Llama 1B/3B) | Prototipe cepat, tanpa pelatihan | 1,8–2,3× |
| Head Medusa | Anda dapat menyetel halus verifikator | 2–3× |
| EAGLE-2 / 3 | Produksi, kecepatan maksimal | 3–4× |
| Lookahead | Tanpa draf, tanpa pelatihan, tanpa parameter ekstra | 1,3–1,6× |

**Kapan TIDAK melakukan dekode spekulatif:**

- Generasi satu urutan 1–5 token. Overhead mendominasi.
- Pengambilan sampel sangat kreatif / bersuhu tinggi ($\alpha$ turun).
- Deploy yang dibatasi memori (model draf menambah VRAM).

## Terapkanlah

Lihat `outputs/skill-spec-decode-picker.md`. Skill ini memilih strategi dekode spekulatif (vanilla / Medusa / EAGLE / lookahead) dan parameter penyetelan (N, suhu draf) untuk beban kerja inferensi baru.

## Latihan

1. **Mudah.** Jalankan `code/main.py`. Konfirmasi bahwa distribusi token spekulatif cocok dengan distribusi sampel langsung verifikator pada 50.000 token dengan uji chi-square p > 0,05.
2. **Sedang.** Plot percepatan (token per forward model besar) sebagai fungsi $N$ untuk $\alpha = 0,5, 0,7, 0,85$. Identifikasi $N$ optimal untuk setiap $\alpha$. (Petunjuk: token ekspektasi per panggilan verifikasi = $(1 - \alpha^{N+1}) / (1 - \alpha)$.)
3. **Sulit.** Implementasikan Medusa mini: ambil GPT proyek akhir dari Pelajaran 14, tambahkan 3 LM head ekstra yang memprediksi posisi t+2, t+3, t+4. Latih pada tinyshakespeare dengan rugi multi-head bersama. Bandingkan tingkat penerimaan dengan draf vanilla yang dibuat dengan memotong model yang sama.
4. **Sulit.** Implementasikan rollback: mulai dengan KV cache prefiks 10 token, masukkan 5 token draf, simulasi penolakan di posisi 3. Verifikasi bahwa pembacaan cache Anda cocok dengan "prefiks + 2 draf pertama yang diterima" pada iterasi berikutnya.

## Istilah Kunci

| Istilah | Apa yang orang katakan | Arti sebenarnya |
|---------|------------------------|-----------------|
| Model draf | "Yang murah" | Model lebih kecil yang mengusulkan token kandidat; biasanya 10–50× lebih murah daripada verifikator. |
| Verifikator | "Yang besar" | Model target yang distribusinya kita jaga; berjalan sekali per langkah spekulatif. |
| Tingkat penerimaan ($\alpha$) | "Seberapa sering draf benar" | Probabilitas per token bahwa verifikator menerima draf. 0,7–0,9 tipikal. |
| Distribusi residual | "Jalan kembali saat penolakan" | `(q - p)_+` ternormalisasi; pengambilan sampel dari sini saat penolakan menjaga distribusi verifikator. |
| Token bonus | "Yang gratis" | Saat semua N draf diterima, ambil satu token lagi dari distribusi langkah berikutnya verifikator. |
| Medusa | "Spekulatif tanpa draf" | Beberapa LM head pada verifikator memprediksi posisi t+1..t+k secara paralel. |
| EAGLE | "Draf hidden state" | Draf transformer mini yang dikondisikan pada hidden state lapisan terakhir verifikator. |
| Dekode lookahead | "Iterasi Jacobi" | Swa-spekulasi menggunakan iterasi titik tetap; tanpa model draf. |
| Tree attention | "Verifikasi banyak kandidat sekaligus" | Verifikasi bercabang yang mempertimbangkan beberapa kelanjutan draf secara bersamaan. |
| KV rollback | "Urungkan draf yang ditolak" | Buffer KV sementara; terapkan saat diterima, buang saat ditolak. |

## Bacaan Lanjutan

- [Leviathan, Kalman, Matias (2023). Inferensi Cepat dari Transformer via Dekode Spekulatif](https://arxiv.org/abs/2211.17192) — algoritme inti dan teorema ekuivalensi.
- [Chen dkk. (2023). Mempercepat Dekode Model Bahasa Besar dengan Pengambilan Sampel Spekulatif](https://arxiv.org/abs/2302.01318) — pengenalan bersamaan; bukti penolakan Bernoulli yang bersih.
- [Cai dkk. (2024). Medusa: Kerangka Percepatan Inferensi LLM Sederhana dengan Beberapa Head Dekode](https://arxiv.org/abs/2401.10774) — makalah Medusa; verifikasi tree-attention.
- [Li dkk. (2024). EAGLE: Pengambilan Sampel Spekulatif Membutuhkan Pemikiran Ulang atas Ketidakpastian Fitur](https://arxiv.org/abs/2401.15077) — EAGLE-1; draf terkondisi hidden state.
- [Li dkk. (2024). EAGLE-2: Inferensi Model Bahasa Lebih Cepat dengan Pohon Draf Dinamis](https://arxiv.org/abs/2406.16858) — EAGLE-2; kedalaman pohon dinamis.
- [Li dkk. (2025). EAGLE-3: Meningkatkan Skala Percepatan Inferensi Model Bahasa Besar via Uji Saat Pelatihan](https://arxiv.org/abs/2503.01840) — EAGLE-3.
- [Fu dkk. (2024). Memutus Ketergantungan Sekuensial Inferensi LLM Menggunakan Dekode Lookahead](https://arxiv.org/abs/2402.02057) — lookahead, pendekatan tanpa draf.
- [Dokumen vLLM — Dekode Spekulatif](https://docs.vllm.ai/en/latest/features/spec_decode.html) — referensi produksi kanonik dengan keempat strategi terpasang.
- [Implementasi referensi SafeAILab / EAGLE](https://github.com/SafeAILab/EAGLE) — kode referensi untuk EAGLE-1/2/3.
