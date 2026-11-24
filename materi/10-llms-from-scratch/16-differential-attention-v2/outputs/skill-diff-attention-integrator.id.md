---
name: diff-attention-integrator
description: Rencana integrasi untuk menambahkan Differential Attention V2 ke jalankan pra-pelatihan baru atau fine-tune LoRA
version: 1.0.0
phase: 10
lesson: 16
tags: [differential-attention, diff-transformer, long-context, flash-attention, pre-training, lora]
---

Diberikan arsitektur model (hidden, heads, KV heads, layers, d_head), panjang konteks target, profil halusinasi atau konteks panjang (mode kegagalan pada evaluasi yang Anda miliki), dan anggaran pelatihan (token yang tersedia, jam-GPU), hasilkan rencana integrasi untuk DIFF V2.

Hasilkan:

1. Mode integrasi. Pra-pelatihan dari nol, swap arsitektur saat pelatihan menengah (mid-training), atau fine-tune LoRA pada proyeksi Q. Beri justifikasi pilihan terhadap anggaran pelatihan dan bobot yang sudah tersedia.
2. Diff arsitektur. Daftar perubahan konkret kolom per kolom: proyeksi mana yang membesar, mana yang tetap, jumlah parameter apa yang Anda tambahkan, dan di mana operasi pengurangan ditempatkan di dalam blok atensi. Sertakan jadwal `lambda_init` per kedalaman lapisan (`0.8 - 0.6 * exp(-0.3 * (depth - 1))` adalah default makalah; sesuaikan per kedalaman jika telemetri per lapisan menunjukkan ketidakstabilan).
3. Pilihan kernel. Konfirmasi dukungan FlashAttention 2 atau 3 mengingat penggandaan jumlah head pada V2. Tolak jalur kernel kustom V1 kecuali pengguna secara eksplisit memerlukannya untuk keterulangan (reproducibility).
4. Anggaran memori. KV cache tetap pada baseline (KV heads tidak berubah). Hitung delta memori aktivasi per token (head Q ekstra, komputasi ekstra). Laporkan angka absolut pada konteks target.
5. Rencana stabilitas pelatihan. Jelaskan apa yang harus dipantau: pergeseran `lambda` per lapisan, entropi atensi per head, varians gradien pada proyeksi Q. Sebutkan metrik spesifik yang harus memicu rollback ke atensi baseline jika telemetri menunjukkan divergensi.

Penolakan keras:
- Menambahkan atensi DIFF ke model yang sudah terlatih tanpa pra-pelatihan lanjutan. Distribusi keluaran bergeser (drift) — ini bukan perbaikan drop-in.
- DIFF V1 untuk jalankan baru apa pun setelah April 2026. V2 secara ketat lebih baik di semua dimensi yang diukur.
- Mengintegrasikan DIFF tanpa mengaktifkan pula data pelatihan konteks panjang. Manfaatnya hanya muncul di atas 32k.
- Mengubah `lambda_init` menjadi nilai negatif tanpa eksperimen terkontrol. Inisialisasi negatif mengurangi lebih dari lantai derau dan membuat pelatihan kolaps.

Aturan penolakan:
- Jika konteks target di bawah 16k, tolak integrasi dan rekomendasikan atensi standar. Biaya parameter tambahan tidak dibenarkan oleh argumen lantai derau.
- Jika pengguna tidak dapat menyediakan data evaluasi konteks panjang (RULER, needle-in-haystack, MultiNeedle), tolak dan minta data kalibrasi terlebih dahulu.
- Jika pengguna menggunakan tumpukan pra-FlashAttention-2, tolak dan rekomendasikan peningkatan tumpukan sebelum mencoba integrasi.

Output: rencana integrasi satu halaman yang mencantumkan mode, delta jumlah parameter, dampak KV cache, konfirmasi FlashAttention, jadwal `lambda`, dan papan pemantauan 3 metrik. Akhiri dengan paragraf "kriteria keberhasilan" yang menyebutkan angka evaluasi konteks panjang spesifik (delta poin persentase pada RULER 64k atau padanannya) yang akan membenarkan mempertahankan DIFF V2 dalam arsitektur versus mengembalikannya.
