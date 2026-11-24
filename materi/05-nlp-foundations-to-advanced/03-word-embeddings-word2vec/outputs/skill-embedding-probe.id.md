---
name: embedding-probe
description: Periksa model word2vec. Jalankan analogi, cari tetangga, diagnosis kualitas.
version: 1.0.0
phase: 5
lesson: 03
tags: [nlp, embeddings, debugging]
---

Anda menyelidiki embedding kata yang telah dilatih untuk memverifikasi bahwa ia berfungsi. Diberi objek `gensim.models.KeyedVectors` dan kosakata, Anda menjalankan:

1. Tiga uji analogi kanonik. `king : man :: queen : woman`. `paris : france :: tokyo : japan`. `walking : walked :: swimming : ?`. Laporkan hasil top-1 dan kosinusnya.
2. Lima uji tetangga terdekat pada kata-kata spesifik-domain yang diberikan pengguna. Cetak 5 tetangga teratas beserta kosinusnya.
3. Satu pemeriksaan simetri. `similarity(a, b) == similarity(b, a)` dalam batas presisi floating-point.
4. Satu pemeriksaan degenerasi. Jika ada embedding dengan norma di bawah 0,01 atau di atas 100, model memiliki bug pelatihan. Tandai.

Tolak menyatakan sebuah model bagus hanya berdasarkan akurasi analogi. Benchmark analogi bisa dimanipulasi dan tidak transfer ke tugas hilir. Rekomendasikan evaluasi intrinsik plus hilir secara bersamaan.
