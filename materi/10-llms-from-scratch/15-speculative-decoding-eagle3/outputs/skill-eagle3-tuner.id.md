---
name: eagle3-tuner
description: Pilih dan setel strategi dekode spekulatif (vanilla / Medusa / EAGLE-1/2/3 / lookahead) untuk beban kerja inferensi baru
version: 1.0.0
phase: 10
lesson: 15
tags: [speculative-decoding, eagle, eagle-3, medusa, inference, vllm, sglang, tensorrt-llm]
---

Diberikan target inferensi produksi (model verifier, ukuran batch, profil panjang sekuens, latensi decode p50/p99 target, akselerator, rentang alpha yang diharapkan dari telemetri, campuran tugas), rekomendasikan strategi dekode spekulatif dan parameter penyetelan. Rekomendasi harus mempertahankan distribusi keluaran verifier persis sama — tidak ada kompromi kualitas yang diterima tanpa persetujuan eksplisit.

Hasilkan:

1. Keluarga draf. Pilih dari vanilla, Medusa, EAGLE-1, EAGLE-2, EAGLE-3, atau lookahead. Beri justifikasi menggunakan telemetri alpha (atau estimasi terkalibrasi), biaya pelatihan yang tersedia (tidak ada, SFT kecil, atau jalankan penuh 60B+ token), dan apakah verifier dikirim dengan draf yang telah dipublikasikan (checkpoint EAGLE-3 tersedia untuk Llama 3.1/3.3, DeepSeek-V3, Qwen 2.5, Qwen 3).
2. Panjang draf N. Pilih bilangan bulat N yang meminimalkan waktu dinding (wall time) yang diharapkan per token dengan alpha dan rasio biaya draf-terhadap-verifier c: minimalkan (1 + N*c) / ((1 - alpha^(N+1)) / (1 - alpha)). Tunjukkan perhitungan untuk tiga nilai N kandidat di sekitar titik optimum.
3. Parameter pencarian pohon jika EAGLE-2/3. Pilih kedalaman pohon dan faktor percabangan agar tetap dalam anggaran memori. Default: kedalaman 3, percabangan (4, 2, 2) untuk batch <=8; kedalaman 2 (4, 2) untuk batch 16-64; dan tanpa pohon untuk batch >64.
4. Gerbang suhu. Ketika suhu > 0,8, alpha anjlok. Rekomendasikan menonaktifkan spec decode di atas ambang terkalibrasi, atau beralih ke pohon yang lebih lebar dengan percabangan per node lebih rendah.
5. Rencana rollback KV. Sebutkan implementasi cache KV spesifik (buffer scratch milik vLLM vs panjang-logical per-sekuens milik TensorRT-LLM) dan pastikan ia mendukung penolakan (rejection) berganda pada konkurensi target.

Penolakan keras:
- Rekomendasi apa pun yang mengubah distribusi keluaran verifier (mis., spec-decode aproksimasi, relaksasi penolakan).
- Spec decode pada batch 1 untuk satu model kecil ketika biaya draf melebihi biaya verifier yang dihemat.
- EAGLE dengan checkpoint draf yang dilatih terhadap tokenizer atau revisi model dasar yang berbeda dari verifier.
- Menjalankan spec decode tanpa rollback KV — akan merusak token berikutnya secara diam-diam.

Aturan penolakan:
- Jika telemetri alpha tidak tersedia DAN campuran tugas adalah penulisan kreatif bersuhu tinggi, tolak rekomendasi dan minta jalankan kalibrasi terlebih dahulu.
- Jika verifier lebih kecil dari 7B parameter padat, rekomendasikan menonaktifkan spec decode alih-alih memilih strategi.
- Jika tumpukan penyajian tidak mendukung keluarga draf yang dipilih (mis., versi vLLM tanpa EAGLE-3), turunkan ke EAGLE-2 alih-alih meminta pengguna membangun ulang tumpukan.

Output: rekomendasi satu halaman yang mencantumkan keluarga draf, N, bentuk pohon (jika ada), konfirmasi rollback KV, dan rentang percepatan yang diharapkan. Akhiri dengan paragraf "rencana telemetri alpha" yang menyebutkan hook logging tepat yang harus ditambahkan pengguna ke server inferensi mereka untuk memverifikasi rekomendasi pada minggu pertama produksi.
