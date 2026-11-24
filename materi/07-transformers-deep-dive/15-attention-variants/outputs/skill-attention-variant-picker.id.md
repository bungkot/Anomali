---
name: attention-variant-picker
description: Pilih topologi atensi penuh / jendela geser / jarang / diferensial untuk model baru diberikan panjang konteks, tuntutan pengambilan, dan profil komputasi.
version: 1.0.0
phase: 7
lesson: 15
tags: [attention, transformer, long-context, inference, memory]
---

# Pemilih Varian Atensi

Bantu pengembang memilih dan membenarkan topologi atensi untuk transformer baru, atau untuk transformer yang sedang diperpanjang konteksnya.

## Input yang dikumpulkan

1. **Panjang konteks target** saat pelatihan dan saat inferensi (sering berbeda — banyak model dilatih pada 16K dan diperpanjang saat inferensi).
2. **Tuntutan pengambilan** pada skala 1–5: 1 = chat murni, 5 = needle-in-haystack / RAG / kode dengan konteks repositori panjang.
3. **Anggaran memori inferensi** — toleransi KV cache per permintaan (byte per token per lapisan adalah satuan yang tepat).
4. **Toleransi biaya pelatihan** — melatih SWA dari awal murah; memasang atensi diferensial ke model yang sudah terlatih mahal.
5. **Perangkat keras target** — Hopper+ memiliki FlashAttention-3 penuh, Ada memiliki FA2, GPU lama terbatas oleh mask.

## Aturan keputusan

- **Konteks ≤ 16K dan pengambilan ≤ 3**: atensi penuh dengan FlashAttention. Jangan mengoptimalkan terlalu dini.
- **Konteks 16–128K dan pengambilan ≤ 3**: campuran SWA + global 5:1, jendela 1024 (bentuk Gemma 3). Menjaga pengambilan tetap layak sambil menyusutkan KV.
- **Konteks > 128K**: SWA penuh dengan satu lapisan global setiap 4–6 lapisan, ditambah interpolasi posisi / penskalaan YaRN (Pelajaran 04).
- **Pengambilan = 5 dan anggaran pelatihan mengizinkan**: pertimbangkan atensi diferensial hanya pada 4 lapisan teratas (separuh dari KV yang dua kali lipat, sebagian besar manfaat penghilangan sink).
- **Anda merilis API publik**: utamakan pola stabil (penuh, SWA, campuran Gemma-3). Lewati native-sparse / DIFF kecuali Anda memiliki insinyur kernel.
- **Anda tidak bisa mengubah model dasar**: SWA dapat dipasang saat inferensi melalui mask; diferensial dan jarang tidak bisa.

## Selalu tandai

- Model SWA-murni di bawah 7B sering mengalami penurunan terukur pada benchmark penalaran. Sarankan untuk tidak menggunakannya.
- Ukuran jendela < 512 hampir tidak pernah tepat. Perbesar atau gunakan topologi lain.
- Hasil atensi diferensial dalam makalah diuji pada model kecil (3–7B). Bukti penskalaan masih tipis pada awal 2026.
- Setiap varian berinteraksi dengan penskalaan RoPE / YaRN (Pelajaran 04). Nyatakan skema posisi secara eksplisit.

## Format keluaran

Kembalikan:

1. **Rekomendasi** — satu topologi bernama (mis. "campuran Gemma-3, W=1024, 5:1 SWA:global").
2. **Justifikasi** — petakan setiap input ke aturan keputusan di atas.
3. **Estimasi KV cache** — pada konteks target, dalam byte per token per lapisan dan GB pada batch 1.
4. **Jalur migrasi** — jika model dasar sudah terlatih, bagaimana memasangnya.
5. **Risiko yang diketahui** — benchmark / beban kerja apa yang mungkin mundur.
