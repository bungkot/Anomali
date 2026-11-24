---
name: structured-output-picker
description: Pilih pendekatan keluaran terstruktur, desain skema, dan rencana validasi.
version: 1.0.0
phase: 5
lesson: 20
tags: [nlp, llm, structured-output]
---

Dengan suatu kasus penggunaan (penyedia, anggaran latensi, kompleksitas skema, toleransi kegagalan), keluarkan:

1. Mekanisme. Keluaran terstruktur native vendor, percobaan ulang Instructor, FSM Outlines, atau CFG XGrammar. Alasan satu kalimat.
2. Desain skema. Urutan field (penalaran terlebih dahulu, jawaban terakhir), field nullable untuk "tidak diketahui", enum versus regex, field wajib.
3. Strategi kegagalan. Jumlah percobaan ulang maksimum, model cadangan, penanganan `null` yang anggun, penolakan terhadap data di luar distribusi.
4. Rencana validasi. Tingkat kepatuhan skema (target 100%), validitas semantik (LLM-judge), tingkat cakupan field, latensi p50/p99.

Tolak desain apa pun yang menaruh `answer` atau `decision` sebelum field penalaran. Tolak menggunakan mode JSON polos tanpa skema. Tandai skema rekursif yang berada di balik pustaka yang hanya mendukung FSM.
