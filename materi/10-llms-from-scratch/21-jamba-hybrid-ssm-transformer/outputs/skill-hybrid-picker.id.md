---
name: hybrid-picker
description: Pilih antara Transformer-murni, hibrida gaya Jamba, dan SSM-murni untuk beban kerja tertentu.
version: 1.0.0
phase: 10
lesson: 21
tags: [jamba, mamba, ssm, hybrid, long-context, memory-budget, architecture]
---

Diberikan spesifikasi beban kerja (profil panjang konteks p50/p99, campuran tugas, anggaran memori per GPU, throughput target, prioritas kualitas vs kecepatan), rekomendasikan antara Transformer-murni (+MoE +MLA), hibrida gaya Jamba, dan model Mamba-murni.

Hasilkan:

1. Bucket panjang konteks. Pendek (di bawah 16k), menengah (16k-64k), panjang (64k-256k), atau sangat panjang (256k-plus). Menentukan keputusan pertama.
2. Rekomendasi arsitektur. Pilih salah satu Transformer-murni, hibrida 1:7, hibrida 1:3, hibrida 1:15, atau Mamba-murni. Justifikasi memakai bucket konteks plus tuntutan pengambilan dalam konteks dari tugas.
3. Pemeriksaan anggaran memori. Hitung KV cache + keadaan SSM pada konteks target. Konfirmasi bahwa ia muat pada akselerator target setelah memperhitungkan bobot dan memori aktivasi (biasanya 10-20 GB di atas bobot dan KV cache).
4. Pengungkapan trade-off kualitas. Dokumentasikan biaya kualitas dari tingkat kejarangan yang dipilih. Hibrida di bawah rasio 1:7 merosot pada pengambilan dalam konteks dalam jumlah terukur; Mamba-murni gagal pada beberapa tugas pelacakan keadaan.
5. Kompatibilitas tumpukan inferensi. Konfirmasi arsitektur yang dipilih didukung tumpukan target (vLLM, TensorRT-LLM, SGLang, llama.cpp). Hibrida memiliki cakupan perkakas lebih tipis daripada Transformer-murni.

Penolakan keras:
- Hibrida gaya Jamba untuk konteks di bawah 16k. Overhead arsitektur tidak sepadan.
- Mamba-murni untuk tugas penalaran berat atau referensi silang banyak dokumen. Batasan pelacakan keadaan menggigit.
- Rasio hibrida di bawah 1:15. Di bawah itu, pengambilan dalam konteks tidak andal.
- Rekomendasi apa pun yang tidak muat pada anggaran memori terhitung di akselerator yang ditentukan.

Aturan penolakan:
- Jika beban kerja benar-benar campuran konteks pendek dan panjang, tolak rekomendasi hibrida dan rekomendasikan Transformer-murni (dengan MLA jika memungkinkan) — hibrida unggul khusus pada beban kerja konteks panjang.
- Jika akselerator kelas konsumen (24GB atau kurang), tolak model berukuran hibrida dan rekomendasikan hibrida kecil hasil distilasi atau Transformer-murni terkuantisasi.
- Jika beban kerja adalah generasi batch-1 sensitif-latensi dan modelnya baru (belum ada jalur deploy), tolak dan rekomendasikan Transformer-murni yang didukung baik dengan dekode spekulatif (Fase 10 · 15) sebagai jalur lebih sederhana.

Keluaran: rekomendasi satu halaman yang mencantumkan bucket konteks, pilihan arsitektur, KV cache pada konteks target, pengungkapan trade-off kualitas, dan kompatibilitas tumpukan inferensi. Akhiri dengan paragraf "apa yang dipantau" yang menyebut evaluasi konteks panjang spesifik (RULER, LongBench, needle-in-haystack) yang akan mengonfirmasi rekomendasi dalam 10k permintaan produksi pertama.
