---
name: entity-linker
description: Rancang pipeline penautan entitas — KB, pembangkit kandidat, disambiguator, evaluasi.
version: 1.0.0
phase: 5
lesson: 25
tags: [nlp, entity-linking, knowledge-graph]
---

Dengan kasus penggunaan (KB domain, bahasa, volume, anggaran latensi), hasilkan:

1. Basis pengetahuan. Wikidata / Wikipedia / KB khusus. Tanggal versi. Kadenesi penyegaran.
2. Pembangkit kandidat. Indeks alias, embedding, atau hibrida. Target recall penyebutan @ K.
3. Disambiguator. Prior + konteks, berbasis embedding, generatif, atau berbasis prompt LLM.
4. Strategi NIL. Ambang batas pada skor teratas, classifier, atau kandidat NIL eksplisit.
5. Evaluasi. Recall penyebutan @ 30, akurasi top-1, F1 deteksi NIL pada himpunan tersembunyi.

Tolak pipeline EL apa pun tanpa baseline recall penyebutan (Anda tidak dapat mengevaluasi disambiguator tanpa mengetahui apakah pembangkit kandidat menemukan entitas yang benar). Tolak pipeline apa pun yang menggunakan EL berbasis prompt LLM tanpa keluaran terbatas pada id KB yang valid. Tandai sistem yang bias popularitasnya merugikan entitas minoritas (misalnya bentrok nama) tanpa fine-tuning domain.
