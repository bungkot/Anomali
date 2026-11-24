# Dekode Spekulatif dan EAGLE

> Sebuah LLM frontier yang menghasilkan satu token memerlukan satu forward pass penuh atas miliaran parameter. Forward pass itu sangat berlebih kapasitasnya: sebagian besar waktu, model yang jauh lebih kecil dapat menebak 3-5 token berikutnya dengan benar, dan model besar hanya perlu *memverifikasi* tebakan tersebut. Saat tebakan itu benar, Anda mendapat 5 token dengan harga satu. Dekode spekulatif (Leviathan dkk. 2023) membuat ini eksak, dan EAGLE-3 (2025) mendorong tingkat penerimaan hingga ~4.5 token per verifikasi — akselerasi 4-5x dengan distribusi keluaran yang sama.

**Jenis:** Bangun
**Bahasa:** Python (dengan numpy)
**Prasyarat:** Fase 10 Pelajaran 12 (Optimasi Inferensi), Fase 10 Pelajaran 04 (Pra-pelatihan Mini-GPT)
**Waktu:** ~75 menit

## Masalahnya

Throughput dekode untuk model kelas 70B pada H100 biasanya 40-80 token/detik. Setiap token memerlukan satu forward pass penuh yang membaca seluruh bobot model dari HBM. Anda tidak dapat membuat model lebih kecil tanpa mengubah keluarnya. Anda tidak dapat menambah ukuran batch melebihi batas memori. Anda mentok — kecuali Anda dapat membuat model menghasilkan lebih dari satu token per forward pass.

Generasi autoregresif terlihat bersifat serial secara inheren: `x_{t+1} = sample(p(· | x_{1:t}))`. Namun ada peluang konkurensi. Jika Anda memiliki prediktor murah yang mengatakan "4 token berikutnya kemungkinan [a, b, c, d]", Anda dapat memverifikasi kelima posisi dalam **satu forward pass model besar** dan menerima prefiks paling panjang yang cocok.

Leviathan, Kalai, Matias (2023, "Fast Inference from Transformers via Speculative Decoding") membuat ini eksak melalui aturan terima/tolak yang cerdas yang mempertahankan distribusi pengambilan sampel model target. Distribusi keluaran yang sama, 2-4× lebih cepat.

## Konsepnya

### Penyiapan Dua Model

- **Model target** `M_p`: model besar, lambat, berkualitas tinggi yang benar-benar ingin Anda ambil sampelnya. Distribusi: `p(x)`.
- **Model draft** `M_q`: model kecil, cepat, berkualitas lebih rendah. Distribusi: `q(x)`. 5-30× lebih kecil.

Per langkah:

1. Model draft mengusulkan `K` token secara autoregresif: `x_1, x_2, ..., x_K ~ q`.
2. Model target menjalankan SATU forward pass atas seluruh `K+1` posisi secara paralel, menghasilkan `p(x_k)` untuk setiap token yang diusulkan.
3. Terima/tolak setiap token dari kiri ke kanan melalui aturan rejection sampling (pengambilan sampel penolakan) yang dimodifikasi di bawah. Terima prefiks paling panjang yang cocok.
4. Jika ada token yang ditolak, ambil sampel penggantinya dari distribusi yang dikoreksi lalu berhenti. Jika tidak, ambil satu token bonus dari `p(· | x_1...x_K)`.

Jika draft cocok sempurna dengan target, Anda mendapat K+1 token per forward target. Jika draft salah di posisi 1, Anda hanya mendapat 1 token.

### Aturan Eksaknya

Dekode spekulatif **secara terbukti ekuivalen dalam distribusi dengan pengambilan sampel dari p**. Aturan penolakannya:

```
For each drafted token x_t:
    r ~ Uniform(0, 1)
    if r < p(x_t) / q(x_t):
        accept x_t
    else:
        sample replacement from residual: (p - q)+ / ||(p - q)+||_1
        stop
```

dengan `(p - q)+` menunjukkan bagian positif dari selisih pointwise. Saat draft dan target sepakat (`p ≈ q`), penerimaan mendekati 1. Saat keduanya tidak sepakat, distribusi residual dibentuk sedemikian rupa sehingga sampel keseluruhan tetap tepat `p`.

**Kasus greedy.** Untuk pengambilan sampel temperature=0, cukup periksa `argmax(p) == x_t`. Jika ya, terima; jika tidak, keluarkan `argmax(p)` lalu berhenti.

### Akselerasi yang Diharapkan

Jika tingkat penerimaan model draft per token adalah `α`, jumlah token yang diharapkan dihasilkan per forward pass target adalah:

```math
\mathbb{E}[\text{tokens}] = \frac{1 - \alpha^{K+1}}{1 - \alpha} \qquad K = \text{draft length}, \ \alpha \in [0, 1]
```

Pada `α = 0.8, K = 4`: `(1 - 0.8^5)/(1 - 0.8) = 3.36` token per forward. Satu forward target menelan biaya sekitar `cost_q * K + cost_p` (K langkah draft ditambah satu verifikasi target). Jika `cost_p >> cost_q * K`, rasio akselerasinya adalah `3.36× / 1 = 3.36×` pada throughput.

Satu-satunya parameter yang benar-benar penting adalah `α`, yang sepenuhnya bergantung pada keselarasan draft-target. Draft yang baik adalah segalanya.

### Melatih Draft: Distilasi

Model kecil acak menghasilkan draft yang buruk. Resep standarnya adalah mendistilasi dari target:

1. Pilih arsitektur kecil (~1B untuk target 70B, ~500M untuk target 7B).
2. Jalankan model target pada korpus teks besar; simpan distribusi token berikutnya.
3. Latih draft dengan divergensi KL terhadap distribusi target (bukan terhadap token ground truth).

Hasilnya: `α` biasanya 0.6-0.8 pada kode, 0.7-0.85 pada obrolan bahasa alami. Akselerasi 2-3× dalam produksi.

### EAGLE: Drafting Pohon + Penggunaan Ulang Fitur

Li, Wei, Zhang, Zhang (2024, "EAGLE: Speculative Sampling Requires Rethinking Feature Uncertainty") mengamati dua ketidakefisienan dalam dekode spekulatif standar:

1. Draft melakukan K langkah serial, masing-masing dengan tumpukan penuh. Padahal draft dapat menggunakan kembali fitur (hidden state) target dari verifikasi terakhir — target sudah menghitung representasi kaya yang sedang diturunkan ulang draft dari awal.
2. Draft menghasilkan rantai linier. Jika draft dapat menghasilkan *pohon* kandidat (setiap node memiliki beberapa tebakan), satu forward pass target dapat memverifikasi beberapa jalur kandidat secara paralel melalui masker atensi pohon, dan memilih cabang terpanjang yang diterima.

Perubahan pada EAGLE-1:
- Input draft = hidden state terakhir target pada posisi t, bukan token mentah.
- Arsitektur draft = 1 lapisan dekoder transformer (bukan model kecil terpisah).
- Keluaran = pohon dengan K = 4-8 kandidat per kedalaman, kedalaman 4-6.

EAGLE-2 (2024) menambahkan topologi pohon dinamis: pohon melebar saat draft tidak yakin dan tetap sempit saat draft yakin. Menaikkan `α_effective` tanpa menambah biaya verifikasi.

EAGLE-3 (Li dkk. 2025, "EAGLE-3: Scaling up Inference Acceleration of Large Language Models via Training-Time Test") menghilangkan ketergantungan fitur tetap pada lapisan teratas dan melatih draft dengan fungsi rugi "simulasi waktu uji" baru — draft dilatih pada keluaran yang sesuai dengan distribusi waktu uji target, bukan distribusi pelatihan teacher-forcing. Tingkat penerimaan naik dari 0.75 (EAGLE-2) menjadi 0.82 (EAGLE-3), dan rata-rata token/verifikasi dari 3.0 menjadi 4.5. Angka-angka ini berasal dari satu konfigurasi referensi. Penerimaan bergantung pada pasangan model, pohon draft, dan lalu lintas: Fase 10 · 15 mengutip 0.88 hingga 0.92 per token jalur yang diterima pada benchmark obrolan, dan Fase 17 · 05 merencanakan 0.6 hingga 0.8 pada obrolan umum produksi.

### Verifikasi Atensi Pohon

Saat draft menghasilkan pohon, model target memverifikasinya dalam satu forward pass menggunakan **masker atensi pohon** — masker kausal yang mengodekan topologi pohon, bukan sekadar garis lurus. Setiap token hanya menghadiri leluhurnya dalam pohon. Pass verifikasi tetap satu forward, satu matmul; masker topologis hanya menambah beberapa entri KV.

```
        root
       /    \
      a      b
     / \    / \
    c  d   e   f
```

Jika `a, b` adalah kandidat token pertama yang saling bersaing dan `c, d, e, f` adalah kandidat token kedua, keenam posisi diverifikasi dalam satu forward pass. Keluarannya adalah prefiks terpanjang sepanjang jalur mana pun yang diterima.

### Kapan Menang, Kapan Tidak

**Menang:**
- Obrolan / pelengkapan dengan teks yang dapat diprediksi (kode, bahasa Inggris umum, keluaran terstruktur). `α` tinggi.
- Pengaturan dengan komputasi GPU yang tidak terpakai saat dekode (fase terikat memori). Drafting pohon memanfaatkan FLOP yang tersedia.

**Kalah / tak ada keuntungan:**
- Keluaran yang sangat stokastik (penulisan kreatif dengan suhu tinggi). `α` turun menuju `1/|vocab|`.
- Penyajian batch dengan konkurensi sangat tinggi — batching sudah memenuhi FLOP, sehingga sedikit ruang untuk verifikasi pohon.
- Model target yang sangat kecil sehingga draft tidak jauh lebih kecil.

Tim produksi biasanya melaporkan akselerasi waktu nyata 2-3× pada obrolan, 3-5× pada generasi kode, dan hampir nol pada penulisan kreatif.

```figure
speculative-decoding
```

## Bangunlah

`code/main.py`:

- Referensi `speculative_decode(target, draft, prompt, K, temperature)` yang menerapkan aturan penolakan eksak dan memverifikasi bahwa ia mempertahankan distribusi target (KL empiris < 0.01 dibanding pengambilan sampel target biasa).
- Draft pohon bergaya EAGLE yang membangun pohon kedalaman-K dengan percabangan top-p.
- Pembangun masker atensi pohon yang menghasilkan pola kausal yang tepat untuk verifier.
- Kerangka penguji tingkat penerimaan yang menjalankan keduanya pada LM mini (distilasi GPT-2-small dari target GPT-2-medium).

```python
def speculative_step(p_target, q_draft, K, temperature=1.0):
    """Satu putaran dekode spekulatif. Mengembalikan daftar token yang diterima."""
    # 1. Draft K token
    draft_tokens = []
    q_dists = []
    state = draft_state_init()
    for _ in range(K):
        probs = softmax(q_draft(state) / temperature)
        t = np.random.choice(len(probs), p=probs)
        draft_tokens.append(t)
        q_dists.append(probs)
        state = draft_step(state, t)

    # 2. Target menghitung p di setiap posisi yang di-draft + 1 ekstra
    p_probs_all = target_forward_batched(p_target, draft_tokens, temperature)

    # 3. Terima/tolak dari kiri ke kanan
    accepted = []
    for k, tok in enumerate(draft_tokens):
        r = np.random.uniform()
        if r < p_probs_all[k][tok] / q_dists[k][tok]:
            accepted.append(tok)
        else:
            residual = np.maximum(p_probs_all[k] - q_dists[k], 0)
            residual /= residual.sum()
            accepted.append(np.random.choice(len(residual), p=residual))
            return accepted
    # 4. Semua K diterima → ambil sampel token bonus dari target
    accepted.append(np.random.choice(len(p_probs_all[-1]), p=p_probs_all[-1]))
    return accepted
```

## Gunakanlah

- **vLLM** dan **SGLang** menyediakan dekode spekulatif sebagai fitur utama. Di vLLM, berikan `--speculative-config` sebuah objek JSON dengan `method`, `model`, dan `num_speculative_tokens`; EAGLE-3 adalah `"method": "eagle3"`.
- **NVIDIA TensorRT-LLM** mendukung pohon Medusa dan EAGLE secara native.
- **Model draft referensi**: `Qwen/Qwen3-0.6B` (draft untuk Qwen3-32B), `meta-llama/Llama-3.2-1B-Instruct` (draft untuk Llama 3.x 70B).
- **Medusa heads** (Cai dkk. 2024, "Medusa: Simple LLM Inference Acceleration Framework with Multiple Decoding Heads"): alih-alih model draft, tambahkan K head prediksi paralel pada target itu sendiri. Lebih sederhana untuk deploi, penerimaan sedikit lebih rendah daripada EAGLE.

## Rilislah

Pelajaran ini menghasilkan `outputs/skill-speculative-tuning.md` — sebuah skill yang memprofilkan beban kerja model target dan memilih: model draft, K (panjang draft), lebar pohon, suhu, dan kapan harus kembali ke dekode biasa.

## Latihan

1. Terapkan aturan penolakan eksak dan verifikasi secara empiris. Jalankan 10K sampel melalui `speculative_decode` dan melalui pengambilan sampel target biasa; hitung jarak TV antara kedua distribusi keluaran. Seharusnya < 0.01.

2. Hitung rumus akselerasi. Dengan `α` dan `K` tetap, plot token yang diharapkan per forward target. Temukan $K$ optimal untuk $\alpha \in \lbrace 0.5, 0.7, 0.9 \rbrace$.

3. Latih draft mini. Ambil target GPT-2 124M dan distilasi draft GPT-2 30M pada 100M token dengan fungsi rugi KL. Ukur `α` pada teks tersisih. Diharapkan: 0.6-0.7.

4. Terapkan drafting pohon bergaya EAGLE. Alih-alih rantai, buat draft menghasilkan 3 cabang teratas di setiap kedalaman. Bangun masker atensi pohon. Verifikasi bahwa target menerima cabang benar terpanjang.

5. Ukur mode kegagalan. Jalankan dekode spekulatif pada temperature=1.5 (stokastisitas tinggi). Tunjukkan α runtuh dan algoritma menjadi lebih lambat daripada dekode biasa akibat overhead draft.

## Istilah Kunci

| Istilah | Yang orang katakan | Arti sebenarnya |
|------|-----------------|------------------------|
| Model target | "Model besar" | Model lambat berkualitas tinggi yang ingin Anda ambil sampelnya (distribusi p) |
| Model draft | "Speculator" | Prediktor kecil dan cepat (distribusi q); 5-30x lebih kecil |
| K / panjang draft | "Look-ahead" | Jumlah token yang dispekulasi per verifikasi |
| α / tingkat penerimaan | "Hit rate" | Probabilitas per token bahwa usulan draft diterima |
| Aturan penolakan eksak | "Tes penerimaan" | Perbandingan r < p/q yang mempertahankan distribusi target |
| Distribusi residual | "p-q terkoreksi" | (p - q)+ / ||(p - q)+||_1, distribusi yang diambil saat penolakan |
| Drafting pohon | "Spekulasi bercabang" | Draft menghasilkan pohon kandidat, diverifikasi dalam satu pass dengan masker atensi berstruktur pohon |
| Masker atensi pohon | "Masker topologis" | Masker kausal yang mengodekan topologi pohon sehingga setiap node hanya menghadiri leluhurnya |
| Medusa heads | "Head paralel" | K head prediksi tambahan pada target sendiri; tanpa model draft terpisah |
| Penggunaan ulang fitur EAGLE | "Draft hidden-state" | Input draft adalah hidden state terakhir target, bukan token mentah, sehingga draft lebih kecil |
| Fungsi rugi simulasi waktu uji | "Pelatihan EAGLE-3" | Latih draft pada keluaran yang cocok dengan distribusi waktu uji target, bukan teacher forcing |

## Bacaan Lanjutan

- [Leviathan, Kalai, Matias, 2023 — "Fast Inference from Transformers via Speculative Decoding"](https://arxiv.org/abs/2211.17192) — aturan penolakan eksak dan analisis akselerasi teoretis
- [Chen, Borgeaud, Irving et al., 2023 — "Accelerating Large Language Model Decoding with Speculative Sampling"](https://arxiv.org/abs/2302.01318) — makalah paralel tentang speculative sampling dari DeepMind
- [Cai, Li, Geng, Wang, Wang, Zhu, Dao, 2024 — "Medusa: Simple LLM Inference Acceleration Framework with Multiple Decoding Heads"](https://arxiv.org/abs/2401.10774) — alternatif head paralel pengganti model draft
- [Li, Wei, Zhang, Zhang, 2024 — "EAGLE: Speculative Sampling Requires Rethinking Feature Uncertainty"](https://arxiv.org/abs/2401.15077) — penggunaan ulang fitur dan drafting pohon
- [Li et al., 2024 — "EAGLE-2: Faster Inference of Language Models with Dynamic Draft Trees"](https://arxiv.org/abs/2406.16858) — topologi pohon dinamis
- [Li et al., 2025 — "EAGLE-3: Scaling up Inference Acceleration of Large Language Models via Training-Time Test"](https://arxiv.org/abs/2503.01840) — pencocokan waktu uji saat pelatihan
- [Fu, Haotian, Peng et al., 2024 — "Break the Sequential Dependency of LLM Inference Using Lookahead Decoding"](https://arxiv.org/abs/2402.02057) — dekode Jacobi/lookahead, alternatif tanpa speculator
