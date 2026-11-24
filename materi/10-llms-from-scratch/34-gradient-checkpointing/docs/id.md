# Checkpointing Gradien dan Komputasi Ulang Aktivasi

> Propagasi balik menyimpan setiap aktivasi antara. Pada 70B parameter dan konteks 128K, itu adalah 3 TB aktivasi per rank. Checkpointing menukar FLOP dengan memori: komputasi ulang, alih-alih menyimpan. Pertanyaannya adalah segmen mana yang dibuang, dan jawabannya bukan "semuanya."

**Jenis:** Bangun
**Bahasa:** Python (dengan numpy, torch opsional)
**Prasyarat:** Fase 10 Pelajaran 04 (Pra-Pelatihan Mini-GPT), Fase 10 Pelajaran 05 (Penskalaan & Terdistribusi)
**Waktu:** ~70 menit

## Masalah

Melatih transformer menyimpan, untuk setiap layer, masukan ke setiap op yang terdiferensiasi pada backward: masukan atensi, proyeksi Q/K/V, keluaran softmax, masukan FFN, keluaran norm, dan residual stream. Untuk layer dengan hidden size `d`, panjang urutan `L`, batch `B`, ini sekitar `12 * B * L * d` float per layer.

Untuk `d=8192, L=8192, B=1`, itu sekitar 805M nilai, atau 1,6 GB/layer dalam BF16. Model 64 layer berisi sekitar 103 GB aktivasi, dan itu sebelum Anda kalikan dengan ukuran mikrobatch, sebelum Anda tambahkan antara softmax atensi (`L^2` per head), dan sebelum Anda hitung salinan parsial paralelisme tensor.

Tagihan dua sisi: bobot BF16 plus keadaan pengoptimal mungkin muat dalam 80GB, tetapi aktivasi mendorong Anda melewatinya. Checkpointing gradien (alias komputasi ulang aktivasi) adalah perbaikan standar. Buang sebagian besar aktivasi; jalankan ulang forward saat backward untuk mendapatkannya kembali. Biaya: FLOP ekstra. Manfaat: memori turun sesuai rasio segmen checkpoint terhadap total layer.

Jika dilakukan naif, checkpointing menambah sekitar 33% FLOP forward per langkah. Jika dilakukan dengan baik — checkpointing selektif menurut "pemilihan cerdas" Korthikanti et al. — Anda menghemat memori 5x dengan overhead FLOP di bawah 5%. Dan dengan matmul FP8, offload FSDP, dan MoE paralel-pakar, ini benar-benar penting: Anda tidak mampu membeli memori maupun komputasi yang terbuang.

## Konsep

### Apa yang Benar-benar Dibutuhkan Backward

`output = layer(input)`. Backward menginginkan `grad_input` dan `grad_params`. Untuk menghitungnya ia memerlukan:

- `input` (untuk menghitung `grad_params = input.T @ grad_output` pada layer linear)
- beberapa antara turunan aktivasi (turunan ReLU/GELU/softmax bergantung pada nilai aktivasi)

Forward menyimpan ini secara otomatis pada graf autograd. Setiap `tensor.retain_grad()` dan setiap op yang memerlukan masukannya menyimpan referensi.

### Checkpointing Penuh Naif

Bagi jaringan menjadi `N` segmen. Selama forward, simpan hanya *masukan* ke setiap segmen. Saat backward memerlukan antara, jalankan ulang forward segmen itu untuk mewujudkannya, lalu diferensiasi.

Contoh: transformer 32 layer dibagi menjadi 32 segmen masing-masing 1 layer.

- Memori: 32 masukan layer (kecil) vs 32 * (volume aktivasi per layer) (besar).
- Komputasi ekstra: 1 forward ekstra per segmen, yaitu ~33% lebih banyak FLOP forward total (karena backward adalah 2x forward, langkah penuh menjadi 1 + 1 + 2 = 4 unit alih-alih 1 + 2 = 3).

Ini resep asli Chen et al. 2016: satu checkpoint setiap `sqrt(L)` layer untuk menyeimbangkan memori dan komputasi. Untuk $L=64$, itu 8 checkpoint.

### Checkpointing Selektif (Korthikanti 2022)

Tidak semua aktivasi sama biayanya. Keluaran softmax atensi adalah `B*L*L*heads` dan bertumbuh *kuadratik* terhadap panjang urutan. Aktivasi tersembunyi FFN adalah `B*L*4d` dan bertumbuh linear. Untuk urutan panjang, softmax mendominasi.

Checkpointing selektif menyimpan aktivasi yang murah disimpan (proyeksi linear, residual) dan hanya mengomputasi ulang yang mahal (atensi). Anda membayar FLOP minimal untuk komputasi ulang, tetapi menghemat memori $O(L^2)$.

Megatron-Core mengimplementasikannya sebagai komputasi ulang aktivasi "selective". Dipakai pada kebanyakan proses pelatihan frontier 2024+.

### Offload

Alternatif komputasi ulang: kirim aktivasi ke RAM CPU antara forward dan backward. Membutuhkan bandwidth PCIe; menguntungkan ketika bandwidth menganggur melebihi biaya rematerialisasi. Strategi campuran umum: checkpoint sebagian layer, offload sisanya.

FSDP2 mengirim offload sebagai opsi kelas satu. Offload bersinar ketika GPU terhambat memori, tetapi transfer CPU-GPU masih punya ruang.

### Model Biaya Komputasi Ulang

FLOP per langkah dengan checkpointing naif setiap `k` layer dari `L`:

```
flops_fwd_normal = L * f_layer
flops_bwd_normal = 2 * L * f_layer
flops_total_normal = 3 * L * f_layer

flops_fwd_ckpt = L * f_layer
flops_recompute = L * f_layer  # satu forward ekstra per layer dalam segmen
flops_bwd_ckpt = 2 * L * f_layer
flops_total_ckpt = 4 * L * f_layer
overhead = 4 / 3 - 1 = 0.33 = 33%
```

Dengan checkpointing selektif Anda hanya mengomputasi ulang kernel atensi, bukan seluruh layer:

```
flops_recompute_selective = L * f_attention ~= L * f_layer * 0.15
overhead_selective = (3 + 0.15) / 3 - 1 = 0.05 = 5%
```

### Model Penghematan Memori

Volume aktivasi per layer: `A`. Untuk `L` layer, total memori aktivasi: `L * A`.

Checkpoint penuh (ukuran segmen 1): simpan hanya `L * input_volume` (~`L * 1/10 A` untuk transformer standar). Menghemat ~`9 * L * A * 1/10`.

Checkpoint setiap `k` layer: simpan `L/k * A` ditambah nilai `k-1` layer di dalam segmen aktif.

Pada `k = sqrt(L)`, memori dan biaya komputasi ulang sama-sama berskala `sqrt(L)` — trade-off optimal untuk layer dengan biaya seragam.

### Kapan Tidak Checkpoint

- Layer paling dalam pada tahap pipeline yang sudah in-flight. Mereka tetap harus selesai.
- Layer pertama dan terakhir jika mendominasi komputasi tahap (jarang pada transformer).
- Kernel atensi yang sudah memakai FlashAttention — Flash sudah mengomputasi ulang softmax dengan cepat, sehingga checkpointing tingkat layer tambahan sedikit manfaatnya.

### Pola Implementasi

1. **Pembungkus fungsi:** bungkus segmen dengan `torch.utils.checkpoint.checkpoint(fn, input)`. PyTorch hanya menyimpan `input`, mengomputasi ulang sisanya saat backward.

2. **Berdasarkan dekorator:** tandai layer sebagai dapat-checkpoint; trainer memutuskan pada waktu konfigurasi segmen mana yang dibungkus.

3. **Komputasi ulang eksplisit manual:** tulis sendiri backward, memanggil `recompute_forward` kustom yang menduplikasi forward dengan masukan tersimpan.

Ketiganya memberikan hasil fungsional yang sama. Pembungkus adalah idiom standar.

### Interaksi dengan TP / PP / FP8

- **Paralelisme tensor:** masukan checkpoint harus dikumpulkan atau disebar ulang saat komputasi ulang; tangani biaya komunikasi.
- **Paralelisme pipeline:** pola umum adalah checkpoint forward setiap tahap pipeline agar mikrobatch urutan-terbalik dapat memakai ulang memori aktivasi.
- **Komputasi ulang FP8:** riwayat amax yang diperbarui saat komputasi ulang harus cocok dengan forward asli, atau skala FP8 bergeser (drift). Kebanyakan framework mengambil snapshot skala.

```figure
activation-recompute
```

## Bangun

### Langkah 1: Model Mainan Dengan Segmen

```python
import numpy as np


def linear_forward(x, w, b):
    return x @ w + b


def relu(x):
    return np.maximum(x, 0)


def layer_forward(x, w1, b1, w2, b2):
    h = relu(linear_forward(x, w1, b1))
    return linear_forward(h, w2, b2)


def model_forward(x, params):
    activations = [x]
    h = x
    for w1, b1, w2, b2 in params:
        h = layer_forward(h, w1, b1, w2, b2)
        activations.append(h)
    return h, activations
```

### Langkah 2: Backward Naif yang Membutuhkan Semua Aktivasi

```python
def model_backward(grad_output, activations, params):
    grads = [None] * len(params)
    g = grad_output
    for i in range(len(params) - 1, -1, -1):
        w1, b1, w2, b2 = params[i]
        x_in = activations[i]
        h_pre = linear_forward(x_in, w1, b1)
        h = relu(h_pre)
        gh = g @ w2.T
        gw2 = h.T @ g
        gb2 = g.sum(axis=0)
        g_pre = gh * (h_pre > 0)
        gx = g_pre @ w1.T
        gw1 = x_in.T @ g_pre
        gb1 = g_pre.sum(axis=0)
        grads[i] = (gw1, gb1, gw2, gb2)
        g = gx
    return g, grads
```

### Langkah 3: Memori Checkpoint-Setiap-k

```python
def model_forward_checkpointed(x, params, k=4):
    saved_inputs = [x]
    h = x
    for i, (w1, b1, w2, b2) in enumerate(params):
        h = layer_forward(h, w1, b1, w2, b2)
        if (i + 1) % k == 0:
            saved_inputs.append(h)
    return h, saved_inputs


def model_backward_checkpointed(grad_output, saved_inputs, params, k=4):
    grads = [None] * len(params)
    g = grad_output
    segments = [(j * k, min((j + 1) * k, len(params))) for j in range(len(saved_inputs))]
    for seg_idx in range(len(saved_inputs) - 1, -1, -1):
        start, end = segments[seg_idx]
        if start >= end:
            continue
        x_in = saved_inputs[seg_idx]
        _, seg_acts = model_forward(x_in, params[start:end])
        g, seg_grads = model_backward(g, seg_acts, params[start:end])
        for j, gr in enumerate(seg_grads):
            grads[start + j] = gr
    return g, grads
```

### Langkah 4: Model Biaya

```python
def checkpoint_cost(n_layers, segment_size, flops_per_layer=1.0):
    fwd = n_layers * flops_per_layer
    recompute = n_layers * flops_per_layer
    bwd = 2 * n_layers * flops_per_layer
    return {
        "fwd": fwd,
        "recompute": recompute,
        "bwd": bwd,
        "total": fwd + recompute + bwd,
        "overhead_vs_no_ckpt": (fwd + recompute + bwd) / (fwd + bwd) - 1.0,
    }


def selective_checkpoint_cost(n_layers, attention_fraction=0.15,
                              flops_per_layer=1.0):
    fwd = n_layers * flops_per_layer
    recompute = n_layers * attention_fraction * flops_per_layer
    bwd = 2 * n_layers * flops_per_layer
    return {
        "fwd": fwd,
        "recompute": recompute,
        "bwd": bwd,
        "total": fwd + recompute + bwd,
        "overhead_vs_no_ckpt": (fwd + recompute + bwd) / (fwd + bwd) - 1.0,
    }
```

### Langkah 5: Estimator Memori

```python
def activation_memory_mb(n_layers, hidden=8192, seq=8192,
                         batch=1, bytes_per_value=2):
    per_layer = 12 * batch * seq * hidden * bytes_per_value
    return n_layers * per_layer / 1e6


def memory_after_checkpoint(n_layers, segment_size, hidden=8192,
                            seq=8192, batch=1, bytes_per_value=2):
    n_seg = max(1, n_layers // segment_size)
    saved = (n_seg + segment_size) * 1 * batch * seq * hidden * bytes_per_value
    return saved / 1e6
```

### Langkah 6: Ukuran Segmen Optimal

```python
def optimal_segment(n_layers):
    return int(round(np.sqrt(n_layers)))
```

### Langkah 7: Keputusan Checkpoint Selektif

```python
def should_recompute(layer_type, activation_bytes, recompute_flops_ratio):
    if layer_type == "attention" and activation_bytes > 100 * 1e6:
        return True
    if layer_type == "ffn" and activation_bytes > 500 * 1e6:
        return recompute_flops_ratio < 0.1
    return False
```

## Gunakan

- **torch.utils.checkpoint**: `from torch.utils.checkpoint import checkpoint` — pembungkus kanonik di PyTorch. Membungkus fungsi; hanya menyimpan masukan, mengomputasi ulang saat backward.
- **Komputasi ulang aktivasi Megatron-Core**: mendukung mode `selective`, `full`, dan `block`. Standar pada pelatihan frontier 2024+.
- **Offload FSDP2**: `module.to_empty(device="cpu")` dengan `offload_policy` di FSDP2 menyebarkan aktivasi ke CPU alih-alih mengomputasi ulang.
- **DeepSpeed ZeRO-Offload**: offload CPU untuk keadaan pengoptimal dan aktivasi, melengkapi checkpointing.

## Kirim

Pelajaran ini menghasilkan `outputs/prompt-activation-recompute-policy.md` — sebuah prompt yang menerima konfigurasi model Anda (layer, hidden, seq, batch) dan memori GPU yang tersedia, lalu menghasilkan kebijakan komputasi ulang per layer (none / selective / full / offload).

## Latihan

1. Verifikasi kebenaran. Jalankan `model_forward` + `model_backward` (aktivasi penuh) vs `model_forward_checkpointed` + `model_backward_checkpointed` (segmen). Gradian parameter harus identik hingga presisi mesin.

2. Sapu ukuran segmen `k` dari 1 hingga `L`. Plot overhead FLOP dan memori. Temukan titik tekuk kurvanya.

3. Implementasikan checkpointing selektif: simpan masukan modul atensi, tetapi bukan antara-nya. Ukur overhead FLOP vs checkpointing seluruh layer untuk model 32 layer pada seq=8192.

4. Tambahkan offload. Simpan masukan segmen ke "buffer CPU" simulasi (daftar terpisah). Ukur "bandwidth PCIe" sebagai byte/waktu dan temukan titik impas antara offload dan komputasi ulang.

5. Benchmark transformer PyTorch nyata dengan dan tanpa `torch.utils.checkpoint`. Ukur memori (via `torch.cuda.max_memory_allocated`) dan waktu langkah.

## Istilah Kunci

| Istilah | Yang orang katakan | Arti sebenarnya |
|---------|--------------------|-----------------|
| Checkpointing gradien | "Hemat memori dengan mengulang forward" | Simpan hanya masukan segmen; komputasi ulang antara saat backward untuk mendapatkan tensor pendukung gradien |
| Komputasi ulang aktivasi | "Sama seperti checkpointing" | Nama bernuansa HPC untuk teknik yang sama |
| Ukuran segmen (k) | "Berapa layer per checkpoint" | Jumlah layer yang antara-nya dibuang dan diwujudkan kembali bersama-sama |
| Checkpointing selektif | "Triknya Korthikanti" | Komputasi ulang hanya aktivasi yang mahal disimpan (softmax atensi); simpan yang murah |
| Checkpointing penuh | "Versi naif" | Komputasi ulang antara setiap layer di setiap segmen |
| Checkpointing blok | "Berbutir kasar" | Checkpoint seluruh blok transformer; granularitas terbesar |
| Overhead FLOP | "Pajak komputasi" | FLOP ekstra per langkah = (FLOP komputasi ulang) / (FLOP fwd + bwd); 33% naif, 5% selektif |
| Offload aktivasi | "Kirim ke CPU" | Pindahkan aktivasi ke RAM CPU melintasi forward->backward; alternatif komputasi ulang |
| Aturan sqrt-L | "Optimum klasik" | Untuk layer dengan biaya seragam, jarak checkpoint optimal adalah $\sqrt{L}$ layer |
| Volume softmax atensi | "Masalah O(L^2)" | $L^2 \cdot \text{heads} \cdot \text{batch}$ float; mendominasi memori aktivasi pada konteks panjang |

## Bacaan Lanjutan

- [Chen et al., 2016 -- "Training Deep Nets with Sublinear Memory Cost"](https://arxiv.org/abs/1604.06174) -- makalah asli yang meresmikan checkpointing gradien
- [Korthikanti et al., 2022 -- "Reducing Activation Recomputation in Large Transformer Models"](https://arxiv.org/abs/2205.05198) -- komputasi ulang aktivasi selektif dan analisis biaya formal
- [Pudipeddi et al., 2020 -- "Training Large Neural Networks with Constant Memory using a New Execution Algorithm"](https://arxiv.org/abs/2002.05645) -- pendekatan memori-konstan alternatif lewat rematerialisasi mode-terbalik
- [Ren et al., 2021 -- "ZeRO-Offload: Democratizing Billion-Scale Model Training"](https://arxiv.org/abs/2101.06840) -- offload aktivasi pada skala besar
- [Dokumentasi PyTorch torch.utils.checkpoint](https://pytorch.org/docs/stable/checkpoint.html) -- API standar
- [Dokumentasi komputasi ulang aktivasi Megatron Bridge](https://docs.nvidia.com/nemo/megatron-bridge/latest/training/activation-recomputation.html) -- mode selective, full, dan block
