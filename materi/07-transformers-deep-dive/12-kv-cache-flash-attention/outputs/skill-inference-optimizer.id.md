---
name: inference-optimizer
description: Pilih implementasi atensi, strategi KV cache, kuantisasi, dan dekode spekulatif untuk sebuah deployment inferensi baru.
version: 1.0.0
phase: 7
lesson: 12
tags: [transformers, inference, flash-attention, kv-cache]
---

Diberikan sebuah deployment inferensi (nama model + parameter, perangkat keras target, konkurensi, panjang konteks maksimum, SLO latensi, target throughput), keluarkan:

1. Stack penyajian. vLLM (default produksi), SGLang (latensi terendah per token), TensorRT-LLM (optimal NVIDIA), llama.cpp (edge/CPU), MLX (silikon Apple). Alasan satu kalimat.
2. Implementasi atensi. Flash Attention 2 (default Ampere/Ada), Flash Attention 3 (Hopper), Flash Attention 4 (Blackwell, forward-only). Tentukan fallback-nya.
3. KV cache. Dtype (fp16 default, fp8 jika didukung), paged vs contiguous, penembolokan prefiks (prefix caching) aktif/nonaktif, KV bersama untuk pengambilan sampel paralel.
4. Kuantisasi. fp16 / bf16 (default), int8 (hanya bobot), AWQ / GPTQ / GGUF untuk bobot. Kuantisasi aktivasi hanya jika sudah di-benchmark.
5. Akselerasi tambahan. Dekode spekulatif (EAGLE 2 / Medusa / model draf), continuous batching (selalu aktif), chunked prefill (beban kerja prompt panjang), penembolokan prefiks jika prompt berulang.

Tolak untuk deploy Flash Attention 4 bagi pelatihan — ia hanya forward-only saat diluncurkan. Tolak menyarankan KV cache fp8 tanpa mem-benchmark dampak kualitasnya pada tugas target. Tandai model 70B+ apa pun tanpa GQA sebagai memiliki KV cache yang tidak terkelola pada konteks 32K+. Wajibkan penembolokan prefiks aktif untuk deployment agen/pemanggilan alat apa pun dengan system prompt yang berulang.
