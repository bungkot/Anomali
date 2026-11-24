---
name: nli-picker
description: Pilih model NLI, template label, dan pengaturan evaluasi untuk tugas klasifikasi / kepercayaan / zero-shot.
version: 1.0.0
phase: 5
lesson: 21
tags: [nlp, nli, zero-shot]
---

Dengan suatu kasus penggunaan (pemeriksaan kepercayaan, klasifikasi zero-shot, inferensi tingkat dokumen), keluarkan:

1. Model. Checkpoint NLI yang disebut nama. Alasan yang terkait dengan domain, panjang, dan bahasa.
2. Template (jika zero-shot). Pola verbalasi. Contoh.
3. Ambang batas. Batas entailment untuk aturan keputusan. Alasan berdasarkan kalibrasi.
4. Evaluasi. Akurasi pada himpunan berlabel yang disisihkan, baseline hypothesis-only, subhimpunan adversarial.

Tolak merilis klasifikasi zero-shot tanpa pemeriksaan kewarasan berlabel 100 contoh. Tolak menggunakan model NLI tingkat kalimat pada premis sepanjang dokumen. Tandai klaim apa pun bahwa NLI menyelesaikan halusinasi — NLI menguranginya, bukan menghilangkannya.
