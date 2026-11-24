---
name: skill-inference-optimization
description: Diagnosis dan optimasi throughput, latensi, dan biaya penyajian inferensi LLM
version: 1.0.0
phase: 10
lesson: 12
tags: [inference, kv-cache, batching, speculative-decoding, vllm, optimization]
---

# Pola Optimasi Inferensi LLM

Dua fase: prefill (terikat komputasi, paralel) dan decode (terikat memori, sekuensial). Setiap optimasi menyasar salah satu atau keduanya.

```
Permintaan -> Prefill (memproses prompt) -> Decode (menghasilkan token) -> Respons
                    |                              |
               Terikat komputasi             Terikat memori
               Optimasi: fusi,               Optimasi: batching,
               prefix caching                kuantisasi, spekulasi
```

## Kerangka pengambilan keputusan

### Langkah 1: Identifikasi hambatan Anda

Ukur rasio ops:byte untuk beban kerja Anda:

| ops:byte | Terikat | Yang dioptimasi |
|----------|---------|-----------------|
| < 50 | Memori | Kuantisasi KV cache, besarkan ukuran batch |
| 50-200 | Transisional | Keduanya penting, mulai dari batching |
| > 200 | Komputasi | Fusi kernel, paralelisme tensor, FP8 |

### Langkah 2: Pilih mesin Anda

- **Default**: vLLM (dukungan model terluas, PagedAttention, API kompatibel dengan OpenAI)
- **Multi-putaran / keluaran terstruktur**: SGLang (prefix caching RadixAttention, dekode terbatas)
- **Throughput NVIDIA maksimum**: TensorRT-LLM (fusi kernel, FP8 pada H100)

### Langkah 3: Terapkan optimasi secara berurutan

1. **KV cache** -- selalu aktif, tanpa kerugian
2. **Continuous batching** -- selalu aktif, tanpa kerugian (vLLM/SGLang melakukan ini secara default)
3. **Prefix caching** -- aktifkan jika Anda memiliki system prompt bersama (kebanyakan chatbot memilikinya)
4. **Kuantisasi** -- KV cache INT8/FP8 mengurangi memori 2-4x dengan kehilangan kualitas minimal
5. **Dekode spekulatif** -- tambahkan ketika latensi lebih penting daripada throughput
6. **Paralelisme tensor** -- bagi ke beberapa GPU ketika model tidak muat dalam satu GPU

## Rumus memori KV cache

```
per_token = 2 * num_layers * num_kv_heads * head_dim * bytes_per_param
total = per_token * sequence_length * num_concurrent_users
```

Referensi cepat untuk model umum (BF16):

| Model | Per token | 100 pengguna @ 4K |
|-------|-----------|-------------------|
| Llama 3 8B | 32 KB | 12.5 GB |
| Llama 3 70B | 320 KB | 125 GB |
| Llama 3 405B | 504 KB | 197 GB |

## Daftar periksa dekode spekulatif

- Model draf sebaiknya 5-10x lebih kecil daripada target (mis., draf 8B untuk target 70B)
- Tingkat penerimaan > 70% agar percepatan bermakna
- Paling baik pada teks yang dapat diprediksi (kode, keluaran terstruktur, bahasa alami)
- Paling buruk pada tugas kreatif/sarat-sampel (suhu rendah membantu)
- EAGLE > draft-target > n-gram untuk sebagian besar beban kerja

## Kesalahan umum

- Menjalankan decode dengan batch=1 (terikat memori, GPU 95% menganggur pada sisi komputasi)
- Mengalokasikan blok KV cache yang kontigu (gunakan PagedAttention, dapatkan pemborosan hampir nol)
- Mengabaikan prefix caching ketika 80% permintaan berbagi system prompt yang sama
- Mengalokasikan memori GPU berlebih untuk bobot model, sehingga tidak tersisa untuk KV cache
- Mengukur throughput tanpa mengukur latensi (throughput tinggi dengan TTFT 10 detik tidak berguna)
- Menggunakan dekode spekulatif dengan suhu tinggi (tingkat penerimaan turun di bawah 50%)

## Daftar periksa pemantauan

- Time to first token (TTFT): latensi prefill, target < 500 ms untuk penggunaan interaktif
- Inter-token latency (ITL): kecepatan decode, target < 50 ms untuk streaming
- Throughput (token/detik): total lintas semua pengguna konkuren
- Pemanfaatan KV cache: persentase cache yang dialokasikan dan sedang digunakan
- Pemanfaatan batch: persentase slot batch yang terisi per iterasi
- Kedalaman antrean: permintaan yang menunggu slot batch
