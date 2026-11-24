---
name: sampling-tuner
description: Pilih strategi dekode (greedy / temperature / top-k / top-p / min-p / spekulatif) untuk suatu tugas generasi tertentu.
version: 1.0.0
phase: 7
lesson: 7
tags: [gpt, sampling, decoding, inference]
---

Diberikan suatu tugas generasi (kode, tulisan kreatif, penalaran, dialog, keluaran terstruktur) serta target latensi/kualitas, keluarkan:

1. Metode pengambilan sampel. Salah satu dari: greedy, temperature-only, top-k, top-p, min-p, beam-k, spekulatif. Sertakan alasan satu kalimat.
2. Nilai parameter. Temperature, top-k, top-p, min-p, repetition penalty — angka konkret yang dikaitkan dengan jenis tugas. (mis. temperature 0.2 + top-p 1.0 untuk kode; min-p 0.1 + temperature 0.7 untuk obrolan.)
3. Kondisi berhenti. `max_new_tokens`, daftar token henti, penghentian berbasis pola (mis. `</tool_call>` penutup).
4. Tombol determinisme. Seed tetap untuk keterulangan; nyatakan apakah kasus penggunaan (evaluasi, hukum) memerlukannya.
5. Pemeriksaan kualitas. Uji satu baris terhadap sasaran tugas (kompilasi/lulus unit test, faktualitas, keabsahan format, dll.).

Tolak merekomendasikan temperature > 1.0 untuk keluaran terstruktur atau pelengkapan kode — risiko halusinasi meningkat tajam. Tolak merekomendasikan greedy murni untuk dialog terbuka — model akan berulang. Tolak untuk mengirim konfigurasi pengambilan sampel tanpa daftar token henti yang ditentukan ketika model dapat menghasilkan templat/alat.
