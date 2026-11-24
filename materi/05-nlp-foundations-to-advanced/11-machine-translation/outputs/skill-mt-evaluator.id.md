---
name: mt-evaluator
description: Evaluasi keluaran terjemahan mesin untuk keperluan pengiriman.
version: 1.0.0
phase: 5
lesson: 11
tags: [nlp, translation, evaluation]
---

Diberikan teks sumber dan kandidat terjemahan, hasilkan:

1. Estimasi skor otomatis. Rentang BLEU dan chrF yang Anda perkirakan. Nyatakan apakah referensi tersedia.
2. Daftar periksa lima poin yang dapat diverifikasi manusia: keutuhan konten (tanpa halusinasi), bahasa target yang benar, kesesuaian register / formalitas, konsistensi terminologi dengan glosarium bila diberikan, tanpa pemotongan atau ledakan panjang.
3. Satu isu spesifik domain yang perlu diselidiki. Hukum: entitas bernama, kutipan pasal. Medis: nama obat, dosis. UI: variabel placeholder seperti `{name}`.
4. Bendera keyakinan. "Ship" / "Ship with review" / "Do not ship". Kaitkan dengan tingkat keparahan isu yang ditemukan.

Tolak mengirim tanpa pemeriksaan pengenal bahasa pada keluaran. Tolak mengevaluasi tanpa referensi kecuali pengguna secara eksplisit memilih penilaian tanpa referensi (COMET-QE, BLEURT-QE). Tandai konten apa pun di atas 1000 token sebagai kemungkinan besar memerlukan terjemahan per potongan.
