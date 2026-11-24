---
name: spec-decode-picker
description: Pilih strategi dekode spekulatif (vanilla / Medusa / EAGLE / lookahead) dan parameter penyetelan untuk beban kerja inferensi LLM baru.
version: 1.0.0
phase: 7
lesson: 16
tags: [inference, decoding, latency, speculative, optimization]
---

# Pemilih Dekode Spekulatif

Bantu insinyur memilih antara spekulatif vanilla, Medusa, EAGLE, atau dekode lookahead, dan menyetel `N` (panjang draf) untuk beban kerja tertentu.

## Input yang dikumpulkan

1. **Model verifikator** — LLM mana yang menghasilkan keluaran akhir. Ukuran penting (biaya draf harus < biaya verifikator agar ada percepatan).
2. **Jenis beban kerja** — kode, chat, keluaran terstruktur, ringkasan. Menentukan tingkat penerimaan.
3. **Strategi pengambilan sampel** — greedy, T-rendah, T-tinggi, beam. Pengambilan sampel T-tinggi menurunkan penerimaan.
4. **Perangkat keras target** — anggaran memori menentukan apakah Anda dapat memuat model draf terpisah.
5. **Anggaran rekayasa** — Medusa dan EAGLE memerlukan fine-tuning; vanilla dan lookahead tidak.
6. **Target latensi** — chat interaktif (<500ms TTFT, <50ms per token) vs batch (throughput diutamakan).

## Aturan keputusan

- **Mulai cepat, tanpa pelatihan**: draf vanilla dengan model 1B–3B dari keluarga yang sama. 2× tipikal.
- **Anda dapat menyetel halus**: EAGLE-2 atau EAGLE-3 menggunakan hidden state verifikator. 3–4× tipikal.
- **Anda dapat menyetel halus tetapi tidak dapat menjalankan dua model**: Medusa (head ekstra pada verifikator). 2–3×.
- **Tanpa anggaran pelatihan, tanpa model draf tersedia**: dekode lookahead. 1,3–1,6×.
- **Penyajian berat-batch**: continuous batching lebih penting; keuntungan spekulatif memudar saat batch membesar karena verifikator sudah jenuh.
- **Suhu tinggi atau pengambilan sampel stokastik**: penerimaan turun tajam. Pertimbangkan N lebih kecil (2–3) atau menonaktifkannya.
- **Keluaran terstruktur (JSON, kode)**: penerimaan tinggi. Dorong N ke 7+ untuk percepatan maksimal.

## Penyetelan

- **N (panjang draf)**: mulai dari 5. Ukur penerimaan. Jika α > 0,9, dorong ke 7. Jika α < 0,6, turunkan ke 3.
- **Suhu draf**: samakan dengan suhu verifikator. Sampel draf yang tidak cocok kehilangan α.
- **Kedalaman pohon (EAGLE-2 / Medusa)**: 3–5 cabang; pohon yang lebih lebar hanya membantu pada α > 0,8.
- **Ukuran model draf**: yang terkecil yang mencapai α > 0,7. Draf 1B untuk verifikator 70B adalah hal yang umum; jangan turun di bawah kompatibilitas tokenizer / embedding verifikator.

## Selalu tandai

- Periksa bahwa draf dan verifikator berbagi tokenizer. Pemisahan BPE yang berbeda merusak jaminan spekulatif.
- Dekode spekulatif berinteraksi dengan continuous batching di vLLM: percepatan per permintaan turun saat batch sudah jenuh.
- Input hidden state EAGLE memerlukan internals verifikator; tidak selalu diekspos melalui API HF. Utamakan runtime vLLM atau SGLang.
- Head Medusa memerlukan fine-tuning terawasi pada keluaran verifikator sendiri. Langkah pengumpulan data sering menjadi biaya dominan.

## Format keluaran

Kembalikan:

1. **Rekomendasi** — satu nama strategi dan parameter penyetelan (mis. "EAGLE-2, N=5, tree_depth=4").
2. **Percepatan ekspektasi** — dengan asumsi α yang eksplisit.
3. **Pemeriksaan kompatibilitas** — kecocokan tokenizer, dukungan runtime, dukungan rollback KV cache.
4. **Rencana cadangan** — jika strategi utama kurang baik, apa yang dicoba berikutnya.
5. **Rencana pengukuran** — bagaimana memvalidasi tingkat penerimaan dan percepatan pada sampel yang representatif.
