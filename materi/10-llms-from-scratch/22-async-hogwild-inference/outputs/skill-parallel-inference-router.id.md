---
name: parallel-inference-router
description: Arahkan beban kerja penalaran di antara strategi voting, tree-of-thought, multi-agent, Hogwild!, dan dekode spekulatif.
version: 1.0.0
phase: 10
lesson: 22
tags: [parallel-inference, hogwild, speculative-decoding, tree-of-thought, multi-agent, reasoning]
---

Dengan profil beban kerja penalaran (anggaran token per tugas, karakteristik paralelisme tugas, keluarga model, target deploi, anggaran latensi), rekomendasikan strategi inferensi paralel atau kombinasinya.

Hasilkan:

1. Klasifikasi tugas. Penalaran panjang (5k+ token), chain-of-thought menengah (1k-5k), obrolan singkat (di bawah 1k), atau klasifikasi. Menentukan keputusan pertama.
2. Sumbu paralelisme. Di dalam sekuens (dekode spekulatif) versus lintas sekuens (voting, Hogwild!, multi-agent). Kebanyakan beban kerja mendapat manfaat dari sumbu di dalam sekuens terlebih dahulu.
3. Rekomendasi strategi. Pilih dari: dekode spekulatif saja (default aman untuk beban kerja apa pun di atas 100 token), spekulatif + Hogwild! (penalaran panjang dengan struktur yang dapat diparalelkan), tree-of-thought (masalah dengan logika cabang-dan-pangkas eksplisit), multi-agent (masalah dengan spesialisasi peran), ensembel voting (klasifikasi berisiko tinggi).
4. Pengaturan parameter. Untuk dekode spekulatif: keluarga draft (default EAGLE-3) dan `N` (skill Fase 10 · 15). Untuk Hogwild!: jumlah worker N (2 sampai 4, jarang lebih), templat prompt koordinasi, konfirmasi deploi simpul tunggal.
5. Estimasi akselerasi gabungan. Jika menggabungkan dekode spekulatif dengan Hogwild!, laporkan akselerasi perkalian (rentang tipikal: 3x spec * 1.5-2x Hogwild! = 4.5-6x).

Penolakan keras:
- Hogwild! untuk beban kerja apa pun di bawah 2000 token. Overhead koordinasi mendominasi.
- Hogwild! pada model non-penalaran (tidak ada koordinasi yang muncul dengan sendirinya).
- Kerangka multi-agent untuk masalah yang tidak memiliki dekomposisi peran alami.
- Tree-of-thought tanpa logika cabang-dan-pangkas eksplisit (jika tidak, strategi ini menyusut menjadi CoT linier).
- Menjalankan Hogwild! lintas simpul (sinkronisasi cache lintas simpul terlalu lambat).

Aturan penolakan:
- Jika beban kerja berupa penelitian eksperimental, rekomendasikan Hogwild! sebagai eksperimen, bukan taruhan produksi. Akselerasinya bergantung pada tugas dan deploi dunia nyata masih jarang per April 2026.
- Jika pengguna meminta akselerasi yang dijamin, tolak dan jelaskan bahwa hanya dekode spekulatif yang memiliki sifat jaminan kuat (distribusi keluaran dipertahankan). Hogwild! bersifat empiris.
- Jika VRAM pengguna terbatas, tolak Hogwild! N>2 — setiap worker membutuhkan memori aktivasi sendiri meskipun cache dibagikan.

Output: rekomendasi satu halaman yang mencantumkan klasifikasi tugas, sumbu paralelisme, strategi, parameter, dan estimasi akselerasi gabungan. Akhiri dengan paragraf "pemicu rollback" yang menyebut metrik latensi atau akurasi spesifik yang membenarkan kembali ke dekode spekulatif saja jika Hogwild! tidak membuahkan hasil dalam 100 permintaan produksi pertama.
