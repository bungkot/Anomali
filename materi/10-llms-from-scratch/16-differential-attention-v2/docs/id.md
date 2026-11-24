# Atensi Diferensial (V2)

> Atensi softmax menyebarkan sedikit probabilitas ke setiap token yang tidak cocok. Pada 100k token, derau itu terakumulasi dan menenggelamkan sinyal. Differential Transformer (Ye et al., ICLR 2025) memperbaikinya dengan menghitung atensi sebagai selisih dua softmax, mengurangi lantai derau bersama. DIFF V2 (Microsoft, Januari 2026) adalah penulisan ulang untuk tumpukan produksi: latensi dekode setara Transformer baseline, tanpa kernel kustom, kompatibel dengan FlashAttention. Pelajaran ini membahas V1 hingga V2 dari awal sampai akhir, dengan implementasi mainan yang berfungsi untuk operasi selisih yang dapat Anda jalankan dengan Python stdlib.

**Jenis:** Membangun
**Bahasa:** Python (stdlib), Rust
**Prasyarat:** Fase 7 · 02 (self-attention), Fase 7 · 15 (varian atensi), Fase 10 · 14 (penelusuran arsitektur)
**Waktu:** ~60 menit

## Tujuan Pembelajaran

- Menyatakan secara tepat mengapa atensi softmax memiliki lantai derau dan mengapa lantai itu bertambah seiring panjang konteks.
- Menurunkan rumus atensi diferensial dan menjelaskan mengapa pengurangan menghilangkan komponen derau bersama sambil mempertahankan sinyal.
- Menelusuri diff V1 ke V2: apa yang menjadi lebih cepat, apa yang menjadi lebih sederhana, apa yang menjadi lebih stabil, dan mengapa setiap perubahan diperlukan untuk pra-pelatihan produksi.
- Mengimplementasikan atensi diferensial dari nol dalam Python murni dan memverifikasi secara empiris sifat pembatalan derau pada query sintetik berupa sinyal-plus-derau.

## Masalah

Atensi softmax standar memiliki sifat matematis yang berubah menjadi sakit kepala operasional pada skala besar. Untuk sebuah query `q`, bobot atensinya adalah `softmax(qK^T / sqrt(d))`. Softmax tidak pernah menghasilkan nol persis: setiap token yang tidak cocok mendapat massa positif. Massa residu itu adalah derau, dan ia bertambah seiring panjang konteks. Pada 128k token, bahkan jika setiap token yang tidak cocok hanya mendapat 0,0001% dari probabilitas, 127.999 di antaranya secara gabungan menyumbang sekitar 13% dari total. Model harus belajar menghindari lantai derau yang tumbuh bersama konteks.

Secara empiris, ini muncul sebagai interferensi head atensi: sitasi halusinatif pada RAG konteks panjang, kegagalan lost-in-the-middle pada tugas pengambilan informasi 100k token, dan penurunan akurasi halus pada benchmark needle-in-haystack di atas 32k. Makalah Differential Transformer (arXiv:2410.05258, ICLR 2025) mengukur selisihnya: Transformers DIFF mencapai perplexity lebih rendah, akurasi konteks panjang lebih tinggi, dan lebih sedikit halusinasi daripada baseline ukuran sama.

DIFF V1 memiliki tiga masalah yang menghalanginya masuk ke pipeline pra-pelatihan terdepan. Cache value-nya harus dimuat dua kali per langkah dekode, ia memerlukan kernel CUDA kustom yang memutus kompatibilitas FlashAttention, dan RMSNorm per-head-nya membuat pelatihan jangka panjang tidak stabil pada skala 70B ke atas. DIFF V2 (blog unilm Microsoft, 20 Januari 2026) memperbaiki ketiganya. Pelajaran ini membahas kedua versi, membangun operator selisih, dan mengukur pembatalan derau pada query mainan.

## Konsep

### Lantai derau softmax

Untuk sebuah query `q` dan kunci `K = [k_1, ..., k_N]`, bobot atensinya adalah:

```math
w_i = \frac{\exp(q \cdot k_i / \sqrt{d})}{\sum_j \exp(q \cdot k_j / \sqrt{d})}
```

Tidak ada `w_i` yang pernah nol. Jika `k_i` benar-benar tidak berkaitan dengan `q`, skor `q . k_i` bukan 0 — ia berfluktuasi di sekitar nol dengan varians `||q||^2 / d`. Setelah normalisasi softmax, setiap token yang tidak berkaitan tetap menyumbang `O(1/N)` pada jumlah berbobot. Sumbangan total token yang tidak berkaitan adalah `O((N-1)/N) = O(1)` — bukan jumlah yang kecil.

Yang diinginkan model adalah sesuatu seperti top-k keras: bobot tinggi pada token yang cocok, bobot hampir nol di tempat lain. Softmax terlalu halus untuk melakukan itu secara langsung.

### Ide diferensial

Bagi proyeksi Q dan K setiap head menjadi dua: $Q = (Q_1, Q_2)$ dan $K = (K_1, K_2)$. Hitung dua peta atensi:

```math
\begin{aligned}
A_1 &= \mathop{\mathrm{softmax}}(Q_1 K_1^T / \sqrt{d}) \\
A_2 &= \mathop{\mathrm{softmax}}(Q_2 K_2^T / \sqrt{d})
\end{aligned}
```

Keluaran:

```math
\mathop{\mathrm{DiffAttn}} = (A_1 - \lambda \cdot A_2) V
```

Pengurangan membatalkan distribusi derau apa pun yang dimiliki bersama oleh kedua peta. Jika kedua peta memiliki bobot yang kira-kira seragam pada 127k token yang tidak berkaitan (dan memang demikian pada inisialisasi acak), derau itu saling menghilang. Sinyal — bobot yang memuncak pada beberapa token yang benar-benar relevan — hanya akan hilang jika muncul di kedua peta dengan magnitudo yang sama, yang tidak akan terjadi setelah model berlatih.

`lambda` adalah skalar yang dapat dipelajari per head, diparameterisasi sebagai `lambda = exp(lambda_q1 dot lambda_k1) - exp(lambda_q2 dot lambda_k2) + lambda_init`. Nilainya boleh negatif. `lambda_init` secara bawaan adalah bilangan positif kecil seperti 0,8.

### Mengapa ini menyerupai mikrofon peredam derau

Bayangkan dua mikrofon berderau yang merekam suara yang sama. Keduanya menangkap pembicara plus derau latar yang berkorelasi. Kurangkan salah satu dari yang lain dan derau bersama hilang. Suara tetap ada karena kedua sinyal berbeda dalam fase atau amplitudo cukup jauh untuk mencegah pembatalan penuh. `lambda` per-head mempelajari keseimbangan ini secara tepat.

### V1 versus V2: diff-nya

V1 menjaga jumlah parameter tetap sama dengan Transformer baseline. Untuk mendapat dua query per head, ia membelah dimensi head menjadi dua. Ini mengorbankan ekspresivitas head dan — yang lebih menyakitkan — membelah cache value per head menjadi dua. Dekode harus memuat cache value dua kali per langkah (sekali per cabang softmax). Hasilnya: dekode lebih lambat daripada baseline meskipun jumlah parameter sama.

V2 menggandakan jumlah query head dan mempertahankan head KV tetap sama (meminjam parameter dari proyeksi up). Dimensi head tetap sama seperti baseline. Setelah pengurangan, dimensi ekstra diproyeksikan kembali ke bawah agar cocok dengan proyeksi $O_W$ Transformer baseline. Tiga hal terjadi sekaligus:

1. Kecepatan dekode setara baseline (KV cache dimuat sekali).
2. FlashAttention berjalan tanpa perubahan (tanpa kernel kustom).
3. Intensitas aritmetika saat dekode naik (lebih banyak komputasi per byte yang dimuat dari HBM).

V2 juga menghapus RMSNorm per-head yang digunakan V1 untuk menstabilkan pengurangan. Pada skala pra-pelatihan kelas 70B, RMSNorm itu membuat pelatihan akhir tidak stabil. V2 menggantinya dengan skema inisialisasi yang lebih sederhana yang menjaga pelatihan tetap stabil tanpa modul tambahan.

### Kapan menggunakannya

| Beban kerja | Manfaat |
|----------|---------|
| RAG konteks panjang (64k+) | Peta atensi lebih bersih, lebih sedikit sitasi halusinatif |
| Benchmark needle-in-haystack | Kenaikan akurasi substansial di atas 32k |
| QA multi-dokumen | Interferensi antardokumen lebih sedikit |
| Pelengkapan kode pada 8k | Marjinal, tidak sepadan dengan perubahan arsitektur |
| Chat pendek (< 4k) | Pada dasarnya tak terbedakan dari baseline |

Nilainya bertambah seiring panjang konteks. Pada 4k token, lantai derau cukup kecil sehingga atensi standar sudah memadai. Pada 128k, derau itu benar-benar merugikan Anda.

### Bagaimana ia berpadu dengan tombol 2026 lainnya

| Fitur | Kompatibel dengan DIFF V2? |
|---------|------------------------|
| GQA | Ya (V2 menambah head Q, bukan head KV) |
| MLA (DeepSeek) | Secara prinsip ya, belum ada makalah yang menggabungkan keduanya |
| MoE | Ya (atensi independent dari blok MLP) |
| RoPE | Ya (tidak berubah) |
| YaRN / penskalaan konteks panjang | Ya (tepat di situlah DIFF paling membantu) |
| FlashAttention | Ya pada V2 (tidak pada V1) |
| Dekode spekulatif | Ya (perubahan atensi tak terlihat oleh loop spec-decode) |

```figure
differential-attention
```

## Membangunnya

`code/main.py` mengimplementasikan atensi diferensial dalam Python murni. Query mainan dengan struktur sinyal-plus-derau yang diketahui memungkinkan Anda mengukur rasio pembatalan derau secara langsung.

### Langkah 1: atensi softmax standar

Operasi matriks stdlib: list of lists, matmul manual, softmax dengan pengurangan nilai maksimum untuk stabilitas numerik.

```python
def softmax(row):
    m = max(row)
    exps = [math.exp(x - m) for x in row]
    s = sum(exps)
    return [e / s for e in exps]
```

### Langkah 2: bagi Q, K menjadi dua bagian

Gaya V1: belah dimensi head menjadi dua. Gaya V2: pertahankan dimensi head dan gandakan jumlah head. Implementasi mainan menggunakan V1 demi kejelasan pedagogis — matematikanya identik, hanya pembukuannya yang berbeda.

### Langkah 3: dua cabang softmax + pengurangan

```python
A1 = [softmax([dot(q1, k) / scale for k in K1]) for q1 in Q1]
A2 = [softmax([dot(q2, k) / scale for k in K2]) for q2 in Q2]
diff_weights = [[a1 - lam * a2 for a1, a2 in zip(r1, r2)] for r1, r2 in zip(A1, A2)]
out = [[sum(w * v[j] for w, v in zip(row, V)) for j in range(d_v)] for row in diff_weights]
```

Perhatikan: bobot keluaran boleh negatif. Tidak apa-apa — cache value tetap menangani kontribusi bertanda. Proyeksi V berikutnya menyerap tandanya.

### Langkah 4: pengukuran pembatalan derau

Bangun urutan sintetik sepanjang 1024. Tempatkan token sinyal pada posisi yang diketahui, isi sisanya dengan derau. Hitung (a) bobot atensi softmax standar pada posisi sinyal dan (b) bobot atensi diferensial. Ukur rasio sinyal terhadap derau pada masing-masing. Atensi DIFF secara andal menghasilkan rasio sinyal-per-derau lebih tinggi sebesar faktor 3x–10x, tergantung seberapa jauh kedua cabang telah dilatih untuk berbeda.

### Langkah 5: pembukuan parameter V1 versus V2

Diberikan konfigurasi (hidden=4096, heads=32, d_head=128), cetak:

- Transformer baseline: Q, K, V masing-masing berukuran `hidden * hidden`, MLP pada 4 * hidden.
- DIFF V1: Q, K masing-masing berukuran `hidden * hidden`, V berukuran `hidden * hidden` (tidak berubah), dimensi head dibelah dua secara internal. Menambahkan parameter `lambda` per-head $(O(\text{heads} \cdot d_{\text{head}}))$.
- DIFF V2: Q berukuran `2 * hidden * hidden`, K berukuran `hidden * hidden`, V berukuran `hidden * hidden`. Dimensi ekstra diproyeksikan kembali sebelum $O_W$. Menambahkan parameter `lambda` yang sama.

Mainan mengukur biaya parameter tambahan untuk V2 (kurang lebih `hidden * hidden` ekstra per blok atensi) dan mencetaknya.

## Menggunakannya

Hingga April 2026, DIFF V2 belum dikirimkan di setiap server inferensi produksi, tetapi integrasinya sedang berjalan di vLLM dan SGLang. Sementara itu, pola ini muncul pada:

- Model produksi internal Microsoft untuk konteks panjang.
- Replikasi riset dalam beberapa proses pelatihan model terbuka yang menargetkan konteks 256k ke atas.
- Arsitektur hibrida yang menggabungkan atensi DIFF dengan atensi jendela geser pada lapisan bergantian.

Kapan Anda akan menggunakannya pada 2026:

- Melatih model baru dari nol dengan target konteks efektif 64k ke atas. Tambahkan atensi diferensial sejak awal; melatih ulang kemudian mahal.
- Menyetel halus model konteks panjang ketika kegagalan lost-in-the-middle mendominasi evaluasi Anda. LoRA pada proyeksi Q dapat mendekati struktur DIFF.

Kapan tidak menggunakannya:

- Anda melayani model padat terlatih dengan kinerja konteks panjang yang stabil. Biaya latih ulang jarang terbayar pada bobot yang sudah ada.
- Konteks Anda selalu di bawah 16k. Lantai derau tidak signifikan.

## Merilisnya

Pelajaran ini menghasilkan `outputs/skill-diff-attention-integrator.md`. Dengan arsitektur model, panjang konteks target, profil halusinasi, dan anggaran pelatihan, skill ini menghasilkan rencana integrasi untuk menambahkan atensi diferensial ke proses pra-pelatihan baru atau fine-tune LoRA.

## Latihan

1. Jalankan `code/main.py`. Verifikasi bahwa rasio sinyal-per-derau yang dilaporkan untuk atensi diferensial lebih tinggi daripada atensi softmax standar pada query sintetik. Variasikan amplitudo derau dan tunjukkan titik persilangan ketika atensi standar menjadi tak terpakai.

2. Hitung delta jumlah parameter dari baseline ke DIFF V1 dan dari baseline ke DIFF V2 untuk model kelas 7B (hidden=4096, heads=32, d_head=128, 32 lapisan). Tunjukkan komponen mana yang bertambah parameternya dan mana yang tetap.

3. Bacalah Bagian 3 makalah DIFF V1 (arXiv:2410.05258) dan Bagian 2 blog Hugging Face DIFF V2. Dalam dua kalimat, jelaskan mengapa RMSNorm per-head V1 diperlukan dan mengapa V2 dapat menghapusnya tanpa menyebabkan pelatihan divergen.

4. Implementasikan ablasi: hitung atensi diferensial dengan `lambda = 0` (softmax pertama murni) dan `lambda = 1` (pengurangan penuh). Pada query sintetik, ukur bagaimana rasio sinyal-per-derau berubah sepanjang sapuan. Identifikasi `lambda` yang memaksimalkan rasio sinyal-per-derau.

5. Kembangkan mainan menjadi GQA + DIFF V2. Pilih 8 head KV dan 32 head Q. Tunjukkan bahwa ukuran KV cache cocok dengan model GQA baseline pada konfigurasi (8, 32) yang sama.

## Istilah Kunci

| Istilah | Yang orang katakan | Yang sebenarnya dimaksud |
|------|----------------|------------------------|
| Atensi diferensial | "Dua softmax yang dikurangkan" | Membagi Q, K menjadi dua bagian, menghitung dua peta softmax, mengurangi yang kedua (diskalakan $\lambda$) dari yang pertama, lalu mengalikannya dengan V |
| Lantai derau | "Ekor softmax yang tidak nol" | Bobot $O(1/N)$ yang ditempatkan softmax pada setiap token yang tidak berkaitan, yang berjumlah $O(1)$ pada konteks panjang |
| lambda | "Skala pengurangan" | Skalar yang dapat dipelajari per-head, diparameterisasi sebagai `exp(lq1.lk1) - exp(lq2.lk2) + lambda_init`; boleh negatif |
| DIFF V1 | "Versi ICLR 2025" | Differential Transformer awal; membelah dimensi head untuk menjaga jumlah parameter, memerlukan kernel kustom, dekode lebih lambat |
| DIFF V2 | "Perbaikan Januari 2026" | Menggandakan head Q dengan head KV tetap; menyamai kecepatan dekode baseline dan bekerja dengan FlashAttention |
| RMSNorm per-head | "Penstabil V1" | Norm tambahan yang diterapkan V1 setelah pengurangan; V2 menghapusnya untuk mencegah ketidakstabilan pelatihan akhir |
| Rasio sinyal-per-derau | "Seberapa banyak atensi terbuang" | Perbandingan bobot pada posisi sinyal sejati terhadap bobot rata-rata pada posisi yang tidak berkaitan |
| Lost in the middle | "Mode gagal konteks panjang" | Fenomena empiris ketika akurasi pengambilan informasi turun untuk dokumen di tengah konteks panjang — atensi DIFF menguranginya |
| Intensitas aritmetika | "FLOP per byte yang dimuat" | Rasio yang dinaikkan V2 saat dekode dengan menggandakan query per pemuatan KV; penting untuk dekode terikat memori |

## Bacaan Lanjutan

- [Ye et al. — Differential Transformer (arXiv:2410.05258, ICLR 2025)](https://arxiv.org/abs/2410.05258) — makalah asli dengan teori pembatalan derau dan ablasi konteks panjang
- [Microsoft unilm — Differential Transformer V2 (blog Hugging Face, Januari 2026)](https://huggingface.co/blog/microsoft/diff-attn-v2) — penulisan ulang untuk tumpukan produksi, menyamai dekode baseline, kompatibel dengan FlashAttention
- [Understanding Differential Transformer Unchains Pretrained Self-Attentions (arXiv:2505.16333)](https://arxiv.org/abs/2505.16333) — analisis teoretis mengapa pengurangan memulihkan struktur atensi pra-terlatih
- [Shared DIFF Transformer (arXiv:2501.17900)](https://arxiv.org/html/2501.17900) — varian pembagian parameter
- [Vaswani et al. — Attention Is All You Need (arXiv:1706.03762)](https://arxiv.org/abs/1706.03762) — Transformer baseline yang dikurangi DIFF
- [Liu et al. — Lost in the Middle (arXiv:2307.03172)](https://arxiv.org/abs/2307.03172) — benchmark konteks panjang yang menjadi sasaran atensi DIFF
