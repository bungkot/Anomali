---
name: open-model-picker
description: Pilih keluarga LLM terbuka, kuantisasi, dan tumpukan inferensi untuk target deployment tertentu
version: 1.0.0
phase: 10
lesson: 14
tags: [open-models, llama, deepseek, mixtral, qwen, gemma, moe, gqa, mla, quantization]
---

Diberikan target deployment (jenis GPU, VRAM per GPU, jumlah GPU, panjang konteks target, latensi p50/p99 target, permintaan konkuren puncak) dan profil tugas (obrolan, kode, penalaran, pengambilan konteks panjang, penggunaan alat), rekomendasikan model terbuka plus tumpukan penyajian dengan penalaran eksplisit untuk masing-masing dari enam tombol arsitektur pada Pelajaran 14.

Hasilkan:

1. Daftar pendek model. Tiga kandidat, masing-masing dengan total parameter, parameter aktif (memperhatikan MoE), bendera arsitektur (norm / aktivasi / posisi / atensi / MoE / konteks), dan alasan tunggal mengapa ia masuk daftar pendek.
2. Pemeriksaan anggaran memori. Untuk kandidat teratas: memori bobot pada BF16 dan pada kuantisasi yang dipilih; KV cache pada konteks target untuk ukuran batch target; ruang kepala (headroom) aktivasi. Hentikan rekomendasi jika bobot + KV cache + aktivasi melebihi VRAM yang tersedia.
3. Pilihan kuantisasi. GPTQ-4bit, AWQ-4bit, FP8, atau BF16. Beri justifikasi terhadap sensitivitas akurasi tugas (tugas kode / matematika / penalaran mengalami penurunan lebih besar akibat kuantisasi agresif dibanding obrolan atau pengambilan informasi).
4. Tumpukan inferensi. vLLM, TensorRT-LLM, SGLang, atau llama.cpp. Beri justifikasi terhadap: kebutuhan continuous batching, dukungan dekode spekulatif, kompatibilitas format kuantisasi, dan topologi satu simpul vs banyak simpul.
5. Pemeriksaan kewajaran throughput. Estimasi prefill token/detik dan decode token/detik berdasarkan bandwidth memori GPU (decode) dan TFLOPs (prefill). Tolak rekomendasi jika throughput decode di bawah batas dasar pengguna konkuren target.
6. Cadangan. Pilihan kedua jika kandidat teratas melebihi anggaran VRAM atau throughput. Selalu sebutkan satu.

Penolakan keras:
- Model padat di atas 30B pada satu GPU konsumen 24GB tanpa offloading atau kuantisasi agresif.
- Model MoE pada tumpukan penyajian tanpa dukungan paralelisme pakar (expert-parallel).
- Konteks panjang (128k+) pada arsitektur tanpa GQA atau MLA (KV cache meledak).
- Rekomendasi apa pun yang tidak menyebut revisi model spesifik (mis., "Llama 3 8B Instruct v3.1", bukan "Llama 3").

Output: rekomendasi satu halaman yang mencantumkan model, kuantisasi, tumpukan, dengan bukti bernomor untuk setiap keputusan. Akhiri dengan paragraf "layak dipertimbangkan ulang jika..." yang menyebutkan kemampuan spesifik atau parameter deployment yang akan membalik pilihan tersebut.
