---
name: grammar-pipeline
description: Rancang pipeline POS + dependensi klasik untuk sebuah tugas NLP hilir.
version: 1.0.0
phase: 5
lesson: 07
tags: [nlp, pos, parsing]
---

Diberikan sebuah tugas hilir (ekstraksi informasi, validasi penulisan ulang, dekomposisi kueri, lemmatisasi), Anda menghasilkan:

1. Himpunan tag. Penn Treebank untuk pipeline lawas khusus Inggris, Universal Dependencies untuk multilingual atau lintas bahasa.
2. Pustaka. spaCy untuk sebagian besar produksi (`en_core_web_sm` / `_lg` / `_trf`), stanza untuk multilingual setingkat akademik, trankit untuk akurasi UD tertinggi.
3. Cuplikan integrasi. 3-5 baris yang memanggil pustaka dan mengonsumsi `.pos_`, `.dep_`, `.head`.
4. Mode kegagalan yang diuji. Ambiguitas kata benda-kata kerja (`saw`, `book`, `can`) dan ambiguitas pelekatan PP adalah jebakan klasik. Ambil sampel 20 keluaran dan periksa secara visual.

Tolak merekomendasikan parser buatan sendiri. Membangun parser dari nol adalah proyek riset, bukan tugas aplikasi. Tandai sebagai rapuh setiap pipeline yang mengonsumsi tag POS tanpa menangani varian huruf kecil / huruf besar.
