---
name: speculative-tuning
description: Profilkan beban kerja dekode dan pilih model draft, panjang draft K, gerbang suhu, dan kebijakan fallback untuk dekode spekulatif.
version: 1.0.0
phase: 10
lesson: 25
tags: [speculative-decoding, draft-model, alpha, throughput, inference, decode-latency]
---

Dengan model target (ukuran, keluarga, tokenizer), telemetri beban kerja (campuran tugas, rasio token prompt versus dekode, latensi dekode p50/p99, ruang kosong akselerator dan HBM, ukuran batch rata-rata, distribusi suhu pengambilan sampel), dan checkpoint draft yang tersedia, keluarkan:

1. Pilihan draft. Pilih dari model kecil sekeluarga (Llama-3.2-1B untuk Llama-70B), draft terdistilasi yang dilatih pada keluaran target, head Medusa yang dipasang pada target, atau "tanpa dekode spec" jika tidak ada draft yang rasio biaya FLOP-nya mendekati dalam 30 persen. Konfirmasi kecocokan tokenizer dengan target byte-per-byte; tolak tokenizer yang tidak cocok.
2. Panjang draft K. Argmax dari E[tokens] / (1 + K x c) dengan c sebagai rasio biaya draft terhadap target. Tunjukkan perhitungannya untuk K pada 2, 3, 4, 5, 6 menggunakan alpha terukur dari lari kalibrasi pada 5_000 token data se-distribusi. Default K=4 untuk obrolan, K=6 untuk kode, K=2 untuk penulisan kreatif bersuhu tinggi.
3. Gerbang suhu. Tetapkan ambang suhu di atasnya dekode spec dinonaktifkan. Default 0.8; turunkan ke 0.6 jika kalibrasi menunjukkan alpha runtuh lebih awal. Tolak gerbang suhu apa pun yang bergantung pada inspeksi per permintaan yang menambah lebih dari 50 mikrodetik.
4. Anggaran pohon. Jika tumpukan penyajian mendukung drafting berbentuk pohon, pilih pohon tetap kecil (kedalaman 2, cabang 3-2) untuk batch di bawah 8; rantai datar untuk batch di atas 32. Nyatakan ukuran scratch KV verifier dalam byte dan konfirmasi bahwa ia muat dalam ruang kosong HBM.
5. Kebijakan fallback. Sebut metriknya (alpha terukur dengan jendela geser atas 1_000 verifikasi terakhir) dan ambangnya (alpha di bawah 0.4) saat server kembali ke dekode autoregresif biasa untuk aliran permintaan tersebut. Sertakan masa berlaku keputusan fallback per permintaan.

Tolak dekode spec pada ukuran batch di atas titik tempat verifier menjadi terikat komputasi. Di atas titik itu, FLOP tak terpakai yang seharusnya diserap oleh speculator tidak lagi ada; throughput turun. Tolak dekode spec untuk keluarga tugas apa pun dengan alpha terukur di bawah 0.4; overhead draft mendominasi dan latensi waktu nyata memburuk. Tolak draft yang belum divalidasi pada sampel tersisih 1_000 token terhadap target: draft yang tidak divalidasi adalah pergeseran KL yang diam-diam terjadi.

Contoh input: "Llama-3.3-70B pada 8xH100, beban kerja obrolan, batch 16, dekode p50 28 ms, p99 60 ms, distribusi suhu rata-rata 0.4 / maks 1.2, kalibrasi menunjukkan alpha 0.78 pada obrolan, 0.61 pada kode."

Contoh output:
- Draft: Llama-3.2-1B-Instruct. Tokenizer sama, keluarga sama, rasio c sekitar 0.03.
- K: 4. E[tokens/verify] = 3.4 obrolan, 2.5 kode. K=5 menambah 0.1 token obrolan dan membayar 0.03 c ekstra; tolak.
- Gerbang suhu: 0.8. Di atas 0.8 alpha turun di bawah 0.45 pada himpunan kalibrasi.
- Anggaran pohon: kedalaman 2 cabang (3, 2). Scratch KV 480 MB pada batch 16 muat.
- Fallback: alpha jendela geser atas 1_000 verifikasi terakhir di bawah 0.40 menonaktifkan dekode spec untuk aliran itu selama 30 s, lalu mencoba lagi.
