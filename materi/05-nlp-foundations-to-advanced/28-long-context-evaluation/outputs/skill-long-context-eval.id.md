---
name: long-context-eval
description: Rancang baterai evaluasi konteks panjang untuk model dan kasus penggunaan tertentu.
version: 1.0.0
phase: 5
lesson: 28
tags: [nlp, long-context, evaluation]
---

Diberikan model target, panjang konteks target, dan kasus penggunaan, keluarkan:

1. Uji. Grid NIAH kedalaman × panjang; multi-hop RULER; tugas domain kustom.
2. Pengambilan sampel. Kedalaman 0, 0.25, 0.5, 0.75, 1.0 pada setiap panjang.
3. Metrik. Tingkat lulus pengambilan; tingkat lulus penalaran; time-to-first-token; biaya-per-kueri.
4. Batas. Panjang pengambilan efektif (lulus 90%) dan panjang penalaran efektif (lulus 70%). Laporkan keduanya.
5. Regresi. Harness tetap, jalankan ulang pada setiap pemutakhiran model, tampilkan selisihnya.

Tolak mempercayai jendela konteks hanya dari kartu model. Tolak evaluasi NIAH-saja untuk beban kerja multi-hop apa pun. Tolak skor konteks panjang yang dilaporkan sendiri oleh vendor sebagai bukti independen.
