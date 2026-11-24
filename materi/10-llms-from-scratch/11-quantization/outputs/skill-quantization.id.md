---
name: skill-quantization
description: Pilih strategi kuantisasi yang tepat untuk penerapan LLM berdasarkan batasan perangkat keras, kualitas, dan latensi
version: 1.0.0
phase: 10
lesson: 11
tags: [quantization, inference, deployment, optimization, fp8, int4, int8, gptq, awq, gguf]
---

# Kerangka Keputusan Kuantisasi

Saat menerapkan sebuah model bahasa, gunakan kerangka ini untuk memilih format angka, metode kuantisasi, dan strategi validasi kualitas yang tepat.

## Persyaratan Masukan

Berikan:
- **Model** (nama, jumlah parameter, presisi asli)
- **Perangkat keras target** (model/VRAM GPU, CPU, Apple Silicon, perangkat edge)
- **Target latensi** (token/detik, waktu hingga token pertama)
- **Lantai kualitas** (kenaikan maksimum perplexity yang dapat diterima, delta benchmark)
- **Pola penyajian** (ukuran batch, panjang konteks maksimum, pengguna bersamaan)

## Pemilihan Cepat

| Situasi Anda | Format | Metode | Kehilangan Kualitas yang Diperkirakan |
|---------------|--------|--------|----------------------|
| GPU H100, throughput maksimum | FP8 E4M3 | Konversi native H100 | < 0,1% |
| A100/A10, butuh 2x throughput | INT8 | LLM.int8() atau SmoothQuant | < 0,5% |
| Satu GPU 24GB, model 70B | INT4 | AWQ atau GPTQ | 1-3% |
| MacBook / Apple Silicon | INT4 GGUF | Q4_K_M via llama.cpp | 1-2% |
| Perangkat seluler / edge | INT4 atau INT3 | QAT + khusus perangkat | 2-5% |
| Kompresi maksimum, kehilangan masih OK | INT2 | QuIP# atau AQLM | 5-15% |
| Pelatihan (presisi campuran) | BF16 + akumulasi FP32 | Dukungan native framework | 0% |

## Pemilihan Presisi per Komponen

Tidak semua tensor harus diperlakukan sama.

| Komponen | Minimum Aman | Direkomendasikan | Hindari |
|-----------|-------------|-------------|-------|
| Bobot FFN | INT4 | INT4 (AWQ/GPTQ) | INT2 tanpa QAT |
| Bobot atensi | INT4 | INT8 atau FP8 | INT2 |
| Lapisan embedding | INT8 | FP16 (biarkan asli) | INT4 |
| Head keluaran | INT8 | FP16 (biarkan asli) | INT4 |
| KV cache | FP8 | FP8 atau INT8 | INT4 pada konteks panjang |
| Logit atensi | FP16 | FP16 atau BF16 | INT8 |
| Aktivasi (inferensi) | INT8 | FP8 atau INT8 | INT4 |

## Perbandingan Metode

### GPTQ
- **Kapan:** Inferensi GPU, Anda menginginkan model yang kompatibel dengan Hugging Face
- **Data kalibrasi:** 128 contoh, masing-masing 2048 token
- **Waktu:** 30-60 menit untuk 70B pada A100
- **Alat:** `gptqmodel`, `exllama`, `exllamav2`
- **Keunggulan:** Teruji dengan baik, kumpulan model besar di Hugging Face
- **Kelemahan:** Lebih lambat diterapkan daripada AWQ, kualitasnya sedikit lebih rendah pada beberapa model

### AWQ
- **Kapan:** Inferensi GPU, Anda menginginkan kualitas terbaik per bit
- **Data kalibrasi:** 128 contoh
- **Waktu:** 15-30 menit untuk 70B pada A100
- **Alat:** `llmcompressor`, `vLLM` (dukungan native)
- **Keunggulan:** Kualitas INT4 terbaik, cepat diterapkan, integrasi vLLM
- **Kelemahan:** Kumpulan model lebih kecil daripada GPTQ

### GGUF
- **Kapan:** Inferensi CPU, Apple Silicon, ekosistem llama.cpp
- **Varian:** Q2_K, Q3_K_S/M/L, Q4_K_S/M, Q5_K_S/M, Q6_K, Q8_0, F16
- **Default yang direkomendasikan:** Q4_K_M (keseimbangan kualitas/ukuran terbaik)
- **Alat:** `llama.cpp`, `ollama`, `LM Studio`
- **Keunggulan:** Berkas mandiri, presisi campuran, ekosistem besar
- **Kelemahan:** Tidak optimal untuk GPU (dirancang untuk CPU/Metal)

### SmoothQuant
- **Kapan:** INT8 pada GPU, memerlukan kuantisasi bobot dan aktivasi
- **Ide utama:** Memindahkan kesulitan kuantisasi dari aktivasi ke bobot melalui penskalaan per kanal
- **Alat:** `smoothquant`, `TensorRT-LLM`
- **Keunggulan:** Memungkinkan W8A8 (bobot dan aktivasi sama-sama INT8) untuk akselerasi 2x
- **Kelemahan:** Hanya INT8, tidak dapat diperluas ke INT4

## Protokol Validasi Kualitas

Setelah kuantisasi, validasi sebelum penerapan:

1. **Uji perplexity.** Hitung pada WikiText-2 atau korpus domain Anda. Delta < 0,5 sangat baik, 0,5-1,0 baik, > 2,0 bermasalah.

2. **Sapuan benchmark.** Jalankan MMLU (umum), GSM8K (matematika), HumanEval (kode). Matematika dan kode paling sensitif terhadap kehilangan presisi.

3. **Perbandingan keluaran.** Hasilkan 100 respons dari model asli dan model terkuantisasi. Gunakan LLM-as-judge untuk menghitung tingkat kemenangan. Target: model terkuantisasi menang atau seri pada > 90% prompt.

4. **Pengukuran latensi.** Ukur token/detik pada ukuran batch 1 dan ukuran batch target Anda. Pastikan akselerasi membenarkan biaya kualitasnya.

5. **Uji konteks panjang.** Jika melayani konteks panjang (> 4K token), uji pada panjang konteks maksimum Anda. Kesalahan kuantisasi KV cache bertambah seiring panjang urutan.

## Kalkulator Anggaran Memori

```
Memori bobot (GB) = parameter (B) * bit / 8 / 1.073741824
KV cache per token (MB) = 2 * num_layers * d_model * bits / 8 / 1048576
KV cache untuk konteks (GB) = kv_per_token * max_context_length / 1024
Memori aktivasi (GB) ~ 1-4 GB (relatif konstan, tergantung ukuran batch)
Total = weight_memory + kv_cache + activation_memory + overhead (10-20%)
```

Contoh untuk Llama 3 70B pada INT4, konteks 32K:
- Bobot: 70B * 4 / 8 / 1,07 = 32,6 GB
- KV cache (FP16): 2 * 80 * 8192 * 16 / 8 / 1e9 * 32768 = ~40 GB
- KV cache (FP8): ~20 GB
- Total dengan KV FP8: ~55 GB (muat dalam satu A100 80GB)

## Kesalahan Umum

| Kesalahan | Mengapa Gagal | Perbaikan |
|---------|-------------|-----|
| Mengkuantisasi lapisan embedding ke INT4 | Lapisan pertama memperkuat kesalahan ke seluruh model | Biarkan embedding pada FP16 atau INT8 |
| Menggunakan skala per tensor untuk INT4 | Satu baris outlier merusak presisi semua baris | Gunakan skala per kanal atau per grup |
| Tidak melakukan kalibrasi GPTQ/AWQ | Faktor skala salah tanpa data representatif | Gunakan 128 contoh dari domain Anda |
| Bit-width sama untuk semua lapisan | Lapisan pertama/terakhir lebih sensitif | Presisi campuran: bit lebih tinggi untuk lapisan pertama/terakhir |
| Mengkuantisasi KV cache pada konteks sangat panjang | Kesalahan bertambah kuadratik dengan panjang urutan | Gunakan FP8 untuk KV cache, bukan INT4 |
| Melewati validasi kualitas | Beberapa model kuantisasinya buruk (terutama di batas) | Selalu jalankan perplexity + evaluasi tugas |

## Resep Penerapan

### Resep 1: vLLM dengan AWQ (server GPU)
```
pip install vllm
vllm serve model-awq --quantization awq --dtype half --max-model-len 8192
```

### Resep 2: llama.cpp dengan GGUF (MacBook)
```
./llama-server -m model.Q4_K_M.gguf -c 4096 -ngl 99
```

### Resep 3: TensorRT-LLM dengan FP8 (H100)
```
trtllm-build --model_dir model --output_dir engine --dtype float16 --use_fp8
```
