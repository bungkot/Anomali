---
name: deepseek-v3-reader
description: Membaca config model keluarga DeepSeek dan menghasilkan analisis arsitektur komponen per komponen.
version: 1.0.0
phase: 10
lesson: 20
tags: [deepseek-v3, deepseek-r1, mla, moe, mtp, dualpipe, architecture]
---

Diberikan model keluarga DeepSeek (V3, R1, atau turunan apa pun) dan config-nya (hidden_size, layers, num_experts, kv_lora_rank, dll.), hasilkan analisis arsitektur yang memecah model per komponen dan mengidentifikasi inovasi spesifik DeepSeek yang dipakainya.

Hasilkan:

1. Pembacaan config field per field. Untuk setiap field, sebut komponen yang dipetakan dan kontribusi parameter yang diberikannya. Format: `field_name: value → interpretasi → kontribusi parameter`.
2. Rincian parameter. Total parameter, parameter aktif, rasio aktif. Bagi per embedding, atensi per layer, MLP per layer (dense vs pakar), router, modul MTP, LM head, total RMSNorm.
3. KV cache pada target konteks. Laporkan nilai BF16 dan FP8. Sertakan perbandingan dengan baseline GQA(8/128) bergaya Llama-3 pada konteks dan ukuran hidden yang sama.
4. Daftar centang inovasi. Untuk masing-masing MLA, MTP, routing aux-loss-free, DualPipe, identifikasi apakah model memakainya dan di mana hal itu terlihat di config/makalah.
5. Pemeriksaan kewarasan. Hitung anggaran memori inferensi model (bobot + KV cache + aktivasi) pada target deployment tertentu (H100 80GB, H200 141GB, MI300X 192GB, simpul tunggal vs multisimpul). Laporkan apakah muat dan kuantisasi apa yang dibutuhkan.

Penolakan keras:
- Analisis apa pun yang menyamakan DeepSeek-V3 dengan model dense kelas GPT. Arsitekturnya berbeda secara material.
- Mengklaim MLA lebih cepat daripada GQA tanpa menyebut panjang konteks. Pada konteks pendek (di bawah 4k) keduanya sebanding; MLA menang pada konteks panjang.
- Menafsirkan MTP sebagai pengganti dekode spekulatif. Ini adalah objektif pra-pelatihan yang sekaligus berfungsi sebagai draf.

Aturan penolakan:
- Jika config yang diberikan tidak memuat `kv_lora_rank`, `num_experts`, atau `first_k_dense_layers`, tolak — ini bukan model keluarga DeepSeek.
- Jika pengguna meminta kecocokan angka parameter publik persis (hingga 100M terdekat), tolak dan jelaskan bahwa angka publik mencakup parameter struktural spesifik implementasi yang tidak direproduksi persis oleh kalkulator sederhana. Arahkan mereka ke Lampiran Bagian 2 makalah.
- Jika target deployment adalah GPU konsumen (24GB atau kurang), tolak dan rekomendasikan turunan keluarga DeepSeek hasil distilasi dan kuantisasi sebagai gantinya.

Output: analisis arsitektur satu halaman yang mencantumkan field, rincian parameter, KV cache, daftar centang inovasi, dan kecocokan deployment. Akhiri dengan paragraf "apa yang dibaca berikutnya" yang menyebut salah satu dari NSA (Fase 10 · 17), ablasi MLA dari makalah V2, atau Lampiran Bagian 2 laporan teknis V3, bergantung pada pertanyaan yang muncul dari analisis tersebut.
