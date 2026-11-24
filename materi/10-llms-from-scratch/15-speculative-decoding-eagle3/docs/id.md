# Dekode Spekulatif dan EAGLE-3

> Fase 7 · Pelajaran 16 membuktikan matematikanya: aturan penolakan Leviathan mempertahankan distribusi verifier secara eksak. Pelajaran ini adalah pandangan tumpukan pelatihan dari dekode spekulatif produksi 2026. EAGLE-3 mengubah model draf dari perkiraan murah menjadi jaringan kecil yang dibangun khusus dan dilatih pada status tersembunyi verifier sendiri, lalu menambahkan loop uji saat pelatihan yang menyelaraskan distribusi pelatihan dan inferensinya. Hasilnya: percepatan ujung-ke-ujung 3× hingga 6,5×, tingkat penerimaan per token di atas 0,9 pada chat, tanpa trade-off distribusi. Setiap tumpukan inferensi produksi pada 2026 menyertakannya secara bawaan.

**Jenis:** Membangun
**Bahasa:** Python (stdlib), Rust
**Prasyarat:** Fase 7 · 16 (matematika dekode spekulatif), Fase 10 · 12 (optimasi inferensi)
**Waktu:** ~75 menit

## Tujuan Pembelajaran

- Menyatakan teorema Leviathan dalam satu kalimat dan membuktikan bahwa loop spekulatif menghasilkan sampel yang terdistribusi identik dengan verifier
- Menelusuri perkembangan dua tahun dari dekode spekulatif biasa (Leviathan 2023) melalui EAGLE, EAGLE-2, dan EAGLE-3 serta menyebut keterbatasan persis yang dihilangkan pada setiap langkah
- Menghitung percepatan ekspektasi dari tingkat penerimaan `α` dan rasio biaya draf-terhadap-verifier `c`, serta memilih panjang draf optimal `N` untuk setiap rezim
- Mengimplementasikan loop spekulatif lengkap dari nol: draf, verifikasi, sampel penolakan dari residual, gulung balik KV cache saat penolakan, dan keluarkan token bonus saat penerimaan penuh

## Masalah

Dekode autoregresif pada model 70B berjalan sekitar 35 token per detik di H100. GPU jauh dari jenuh. Bandwidth memori adalah langit-langitnya: setiap token memuat 70B bobot dari HBM, melakukan satu langkah aritmetika, dan menghasilkan satu bilangan floating point. Unit komputasi sebagian besar menganggur.

Dekode spekulatif mengubahnya menjadi masalah throughput yang benar-benar bisa Anda selesaikan. Draf yang murah mengusulkan `N` token dalam `N` lintasan maju kecil. Verifier berjalan sekali pada prefiks plus ke-`N` draf tersebut. Jika distribusi verifier pada posisi `i` setuju dengan draf (dalam arti statistik yang akan kita perjelas), kita terima; jika tidak, kita tolak dan ambil sampel koreksi dari distribusi residual. Satu lintasan maju model besar menghasilkan hingga `N+1` token yang diterima alih-alih satu.

Teorema yang penting adalah Leviathan, Kalman, Matias (ICML 2023): distribusi keluaran identik dengan apa yang akan dihasilkan oleh pengambilan sampel langsung dari verifier. Bukan mendekati. Identik. Inilah alasan penuh mengapa dekode spekulatif dapat diterima dalam produksi — ini adalah optimasi latensi murni tanpa trade-off kualitas.

Apa yang diberikan Fase 7 · Pelajaran 16 kepada Anda adalah matematikanya. Apa yang diberikan pelajaran ini adalah tumpukan pelatihannya. Draf yang baik bernilai 2× lebih banyak percepatan daripada draf yang murah. EAGLE, EAGLE-2, dan EAGLE-3 (Li et al., 2024–2025) mengubah "draf = versi lebih kecil dari model yang sama" menjadi disiplin rekayasa yang presisi. Server inferensi produksi 2026 beralih ke EAGLE-3 secara bawaan.

## Konsep

### Invariannya: Pengambilan Sampel Penolakan Leviathan

Misalkan `p(t)` adalah distribusi draf untuk token berikutnya dengan sebuah prefiks tertentu, dan `q(t)` adalah distribusi verifier. Ambil sampel token draf `d ~ p`. Terima dengan probabilitas `min(1, q(d) / p(d))`. Saat ditolak, ambil sampel dari distribusi residual `(q - p)_+ / ||(q - p)_+||_1`. Sampel yang dihasilkan terdistribusi sesuai `q`. Ini berlaku berapa pun buruknya `p` — semakin buruk, semakin sering Anda menolak, tetapi keluaran tetap eksak.

Tumpukkan `N` pemanggilan ini berurutan dengan satu lintasan maju verifier pada `prefix + d_1 + ... + d_N`. Verifier mengembalikan `q_1, q_2, ..., q_{N+1}` secara bersamaan. Jalan dari kiri ke kanan. Pada penolakan pertama di posisi `j`, ambil sampel dari `residual(q_j, p_j)` lalu berhenti. Pada penerimaan penuh, ambil satu token bonus dari `q_{N+1}`.

### Yang Menentukan Percepatan

Misalkan `α` adalah tingkat penerimaan ekspektasi per token yang didraf. Misalkan `c = cost(draft) / cost(verifier)` adalah rasio biaya. Jumlah ekspektasi token yang diterima per lintasan maju verifier adalah:

```math
\mathbb{E}[\text{accepted}] = \frac{1 - \alpha^{N+1}}{1 - \alpha}
```

Total waktu dinding ekspektasi per token yang diterima adalah `(N * c + 1) / E[accepted]`. Minimalkan terhadap `N` dan Anda mendapat titik manisnya. Untuk `α = 0.8, c = 0.05`: `N` optimal sekitar 5–7, percepatan 3,2×. Untuk `α = 0.95, c = 0.02`: `N` optimal sekitar 8–10, percepatan mendekati 5×.

Tuas terbesar adalah `α`. Beralih dari `α = 0.6` (draf biasa) ke `α = 0.9` (EAGLE-3) pada `N = 5` tetap membawa Anda dari 2,2 token yang diterima per lintasan verifier menjadi 4,1. Hampir 2× lipat throughput dari verifier yang sama.

### Perkembangan Dua Tahun

**Spekulatif biasa (Leviathan, 2023).** Model draf adalah LLM lebih kecil yang dilatih secara mandiri dari keluarga yang sama. Mudah dipasang, `α ≈ 0,6`, percepatan paling banter sekitar 2×.

**EAGLE-1 (Li et al., 2024).** Draf adalah transformer sangat kecil — biasanya satu atau dua lapisan — yang menerima status tersembunyi lapisan terakhir verifier sebagai masukan dan memprediksi token berikutnya secara langsung. Karena draf melihat representasi fitur verifier, distribusinya jauh lebih dekat ke verifier. `α` naik ke 0,7–0,8.

**EAGLE-2 (Li et al., 2024).** Menambahkan pohon draf dinamis: alih-alih mengusulkan satu urutan `N` token, teknik ini mengusulkan pohon kandidat kecil, menilai masing-masing dengan verifier dalam satu lintasan maju (tree attention), lalu menelusuri jalur berprobabilitas tertinggi. Panjang draf menjadi adaptif per langkah. `α` per token pada jalur yang diterima naik di atas 0,85.

**EAGLE-3 (Li et al., 2025, NeurIPS).** Dua perubahan lagi. Pertama, hilangkan seluruh loss prediksi fitur — EAGLE-1/2 melatih draf agar mencocokkan status tersembunyi verifier, yang membatasi seberapa besar data membantu. EAGLE-3 melatih langsung pada prediksi token. Kedua, training-time test (TTT): selama pelatihan draf, masukkan kembali prediksi draf sendiri sebagai input selama beberapa langkah, sama seperti cara ia beroperasi saat inferensi. Ini menyelaraskan distribusi pelatihan dan pengujian serta menghentikan akumulasi error. Percepatan terukur: hingga 6,5× pada chat, peningkatan throughput 38% pada batch 64 di SGLang atas H100.

### Gulung Balik KV Cache

Verifikasi memperpanjang KV cache verifier sebanyak `N` entri dalam satu lintasan. Jika penolakan terjadi pada posisi `j`, isi cache setelah posisi `j-1` menjadi salah. Dua implementasi umum: menulis ke buffer sementara lalu melakukan commit saat penerimaan (vLLM, TensorRT-LLM), atau menyimpan KV cache fisik plus panjang logis lalu memotongnya saat penolakan. Bagaimanapun, biaya gulung baliknya adalah byte per lapisan per head, yang sangat kecil dibanding biaya lintasan maju.

Untuk pencarian pohon EAGLE-2, verifier menjalankan atensi dengan masker non-kausal yang menghormati topologi pohon. Rekayasanya agak rumit, tetapi komputasinya adalah panggilan flash-attention standar dengan masker khusus.

### Arsitektur Draf pada 2026

| Strategi | Jenis draf | `α` | Percepatan | Biaya pelatihan |
|----------|-----------|-----|---------|---------------|
| Biasa | LLM kecil terpisah | 0.55-0.70 | 1.8-2.3× | Tidak ada (pakai ulang model kecil yang ada) |
| Medusa | Head LM tambahan pada verifier | 0.65-0.75 | 2-3× | ~1B token SFT |
| EAGLE-1 | Transformer 1 lapisan pada status tersembunyi | 0.70-0.80 | 2.5-3× | ~60B token |
| EAGLE-2 | EAGLE-1 + pohon draf dinamis | 0.80-0.88 | 3-4× | ~60B token |
| EAGLE-3 | Fusi fitur multi-lapisan + TTT | 0.88-0.92 | 3.5-6.5× | ~60-200B token |
| Lookahead | Tanpa draf (iterasi Jacobi) | N/A | 1.3-1.6× | Tidak ada |

Kolom `α` adalah tingkat penerimaan per token sepanjang jalur yang diterima, pada benchmark chat, dengan draf yang dilatih pada data chat yang cocok. Itu adalah batas atas. Fase 10 · 25 menyebut 0,82 untuk EAGLE-3 dari konfigurasi berbeda, dan Fase 17 · 05 merencanakan peluncuran produksi dengan 0,6 hingga 0,8 pada chat umum. Ukur `α` pada lalu lintas Anda sendiri sebelum memilih `N`.

Pada produksi 2026: vLLM dan SGLang beralih ke EAGLE-3 saat tersedia, jika tidak maka EAGLE-2. TensorRT-LLM memiliki jalur Medusa tercepat untuk model publik Meta dan NVIDIA. llama.cpp menyediakan draf biasa untuk deployment CPU.

```figure
l5-spec-decode-eagle
```

## Membangunnya

Lihat `code/main.py`. Ini adalah loop spekulatif Leviathan lengkap dengan semua bagiannya: draf-N, lintasan paralel verifier, penolakan per posisi, pengambilan sampel residual, token bonus, gulung balik KV, dan verifikasi empiris bahwa distribusi keluaran cocok dengan pengambilan sampel langsung dari `q`.

### Langkah 1: aturan penolakan

```python
def accept(q_prob, p_prob, u):
    if p_prob <= 0:
        return True
    return u < min(1.0, q_prob / p_prob)
```

### Langkah 2: distribusi residual

```python
def residual(q, p):
    raw = [max(0.0, qi - pi) for qi, pi in zip(q, p)]
    s = sum(raw)
    if s == 0:
        return list(q)
    return [r / s for r in raw]
```

### Langkah 3: satu langkah spekulatif penuh

Fungsi `spec_step` mendraf `N` token dari `p`, lalu memverifikasi semuanya dalam satu evaluasi `q` paralel. Untuk setiap token yang didraf, fungsi ini menerapkan aturan penolakan, dan pada penolakan pertama ia mengambil sampel koreksi dari residual. Jika semua diterima, ia mengeluarkan token bonus dari `q_{N+1}`.

### Langkah 4: pembukuan gulung balik KV

Simulator melacak `kv_length` logis per pekerja. Saat `k` draf diterima, `kv_length += k`. Pada penolakan di posisi `j`, cache sudah ditulis melewati `j`, tetapi panjang logis ditetapkan menjadi `prefix_length + j + 1` — satu lewat token koreksi. Pembacaan berikutnya dipotong sesuai panjang logis.

### Langkah 5: pemeriksaan Leviathan

Jalankan 50.000 langkah spekulatif. Hitung distribusi empiris token yang diterima. Bandingkan dengan 50.000 sampel langsung dari `q`. Statistik chi-square seharusnya jauh di bawah nilai kritis. Teorema lolos dalam praktik.

### Langkah 6: percepatan versus α

Sapu kualitas draf dengan mengganggu `p` menjauh dari `q` pada berbagai amplitudo. Ukur `α`, lalu plot token ekspektasi per panggilan verifier sebagai fungsi dari `α` dan `N`. Kode mencetak tabel yang menunjukkan bagaimana kualitas draf kelas EAGLE-3 (`α ≈ 0,9`) membuka 4–5 token per panggilan verifier.

## Menggunakannya

`vllm serve` level produksi dengan EAGLE-3:

```bash
vllm serve meta-llama/Llama-3.3-70B-Instruct \
  --speculative-config '{
    "model": "yuhuili/EAGLE3-LLaMA3.3-Instruct-70B",
    "num_speculative_tokens": 5,
    "method": "eagle3"
  }'
```

SGLang dengan EAGLE-3 pada batch 64 di H100: sekitar 1,38× lebih banyak throughput daripada dekode biasa pada batch 64, menurut makalah EAGLE-3.

Kapan menggunakan dekode spekulatif:

- Beban chat interaktif apa pun yang latensi p50-nya lebih penting daripada throughput puncak.
- Generasi kode dan keluaran terstruktur (JSON, SQL). `α` di atas 0,9 karena distribusi target sangat dapat diprediksi.
- Generasi panjang (ribuan token). Percepatan yang teramortisasi terus memberi manfaat.

Kapan tidak menggunakannya:

- Model sangat kecil (< 3B). Draf tidak jauh lebih murah daripada verifier.
- Deployment CPU sangat kecil dengan batch 1. Overhead memori model draf mungkin tidak sepadan.
- Pengambilan sampel kreatif bersuhu sangat tinggi tempat `α` anjlok.

## Merilisnya

Pelajaran ini menghasilkan `outputs/skill-eagle3-tuner.md`. Dengan beban kerja inferensi (model, ukuran batch, latensi target, profil tugas), skill ini merekomendasikan strategi dekode spekulatif dan parameter penyetelan (keluarga draf, `N`, kedalaman pohon, peralihan sadar-suhu).

## Latihan

1. Jalankan `code/main.py`. Pastikan statistik chi-square pada pemeriksaan distribusi Leviathan tetap di bawah nilai kritis 95% pada 50.000 sampel.

2. Sapu `N` dari 1 sampai 10 dengan `α` tetap 0,9 dan `c` tetap 0,04. Plot token ekspektasi per panggilan verifier dan waktu dinding aktual per token. Temukan `N` yang meminimalkan waktu dinding. Jelaskan bentuk kurvanya.

3. Ubah kode untuk menyimulasikan pencarian pohon EAGLE-2: pada setiap langkah, draf mengusulkan pohon berbentuk `[2, 2, 2]` (delapan jalur kandidat). Verifier berjalan sekali, dan jalur yang diterima dengan probabilitas tertinggi menang. Hitung `α` per daun dan total token per panggilan verifier. Bandingkan dengan dekode spekulatif rantai linier pada komputasi yang setara.

4. Implementasikan simulator gulung balik KV batch untuk dua urutan bersamaan. Urutan A memiliki semua draf yang diterima; urutan B menolak pada posisi 2. Tunjukkan bahwa `kv_length` diperbarui dengan benar per urutan dan tidak ada pekerjaan yang terbuang.

5. Bacalah Bagian 4 makalah EAGLE-3 (Training-Time Test). Jelaskan dalam dua kalimat mengapa pelatihan draf polos tanpa TTT mengalami exposure bias, dan mengapa memberi draf prediksinya sendiri selama pelatihan memperbaikinya. Hubungkan ini dengan literatur scheduled sampling dalam seq2seq.

## Istilah Kunci

| Istilah | Yang orang katakan | Yang sebenarnya dimaksud |
|------|----------------|------------------------|
| Aturan Leviathan | "min(1, q per p)" | Terima/tolak Bernoulli dengan probabilitas `min(1, q(d)/p(d))`, yang mempertahankan distribusi verifier secara eksak saat sampel diambil dari residual pada penolakan |
| Distribusi residual | "(q dikurangi p) plus, dinormalisasi" | `(q - p)_+` dibatasi di nol lalu dinormalisasi ulang — distribusi yang tepat untuk diambil sampelnya saat penolakan |
| Tingkat penerimaan α | "seberapa sering draf benar" | Probabilitas keberhasilan Bernoulli per token ekspektasi di bawah aturan penolakan; mengatur seluruh matematika percepatan |
| EAGLE-1 | "Draf status tersembunyi" | Draf transformer kecil yang dikondisikan pada status tersembunyi lapisan terakhir verifier (Li et al., 2024) |
| EAGLE-2 | "Pohon draf dinamis" | EAGLE-1 ditambah pohon kelanjutan kandidat yang dinilai dengan tree attention dalam satu lintasan verifier |
| EAGLE-3 | "Uji saat pelatihan" | Menghapus loss prediksi fitur, melatih langsung pada prediksi token dengan draf diberi keluarannya sendiri selama pelatihan |
| Uji saat pelatihan (TTT) | "Perbaikan exposure bias" | Menjalankan draf secara autoregresif saat pelatihan agar distribusi input pelatihan dan pengujian cocok — analog langsung dari scheduled sampling |
| Gulung balik KV | "Urungkan draf yang ditolak" | Pembukuan yang mengembalikan KV cache verifier ke panjang prefiks yang diterima setelah penolakan |
| Token bonus | "Yang gratis" | Saat ke-`N` draf semua diterima, ambil satu sampel tambahan dari `q_{N+1}` tanpa biaya verifier tambahan |
| Tree attention | "Verifikasi banyak kandidat sekaligus" | Atensi dengan masker non-kausal yang menghormati topologi pohon draf; menghitung `q_i` untuk setiap simpul dalam pohon dalam satu lintasan maju |

## Bacaan Lanjutan

- [Leviathan, Kalman, Matias — Fast Inference from Transformers via Speculative Decoding (arXiv:2211.17192, ICML 2023)](https://arxiv.org/abs/2211.17192) — makalah fondasional dan teorema ekivalensi
- [Chen et al. — Accelerating Large Language Model Decoding with Speculative Sampling (arXiv:2302.01318)](https://arxiv.org/abs/2302.01318) — pengenalan independen secara bersamaan dengan bukti yang rapi
- [Li et al. — EAGLE: Speculative Sampling Requires Rethinking Feature Uncertainty (arXiv:2401.15077)](https://arxiv.org/abs/2401.15077) — EAGLE-1, draf berkondisi status tersembunyi
- [Li et al. — EAGLE-2: Faster Inference of Language Models with Dynamic Draft Trees (arXiv:2406.16858)](https://arxiv.org/abs/2406.16858) — pencarian pohon dinamis
- [Li et al. — EAGLE-3: Scaling up Inference Acceleration via Training-Time Test (arXiv:2503.01840, NeurIPS 2025)](https://arxiv.org/abs/2503.01840) — bawaan produksi 2026
- [Cai et al. — Medusa: Multiple Decoding Heads (arXiv:2401.10774)](https://arxiv.org/abs/2401.10774) — pendekatan alternatif tanpa draf
- [Dokumentasi Speculative Decoding vLLM](https://docs.vllm.ai/en/latest/features/spec_decode.html) — referensi produksi kanonik dengan semua strategi terpasang
