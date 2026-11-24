---
name: seq2seq-design
description: Rancang pipeline sequence-to-sequence untuk sebuah tugas tertentu.
phase: 5
lesson: 09
---

Diberikan sebuah tugas (terjemahan, peringkasan, parafrasa, penulisan ulang kueri), hasilkan:

1. Arsitektur. Encoder-decoder transformer terlatih (BART, T5, mBART, NLLB) adalah standar. Seq2seq berbasis RNN hanya untuk kendala tertentu (streaming, inferensi di tepi, pengajaran).
2. Checkpoint awal. Sebutkan namanya (`facebook/bart-base`, `google/flan-t5-base`, `facebook/nllb-200-distilled-600M`). Sesuaikan checkpoint dengan tugas dan cakupan bahasa.
3. Strategi dekode. Greedy untuk keluaran deterministik, beam search (lebar 4-5) untuk kualitas, sampling dengan temperatur untuk keberagaman. Satu kalimat justifikasi.
4. Satu mode kegagalan yang diverifikasi sebelum kirim. Bias paparan muncul sebagai pergeseran generasi pada keluaran yang lebih panjang; ambil sampel 20 keluaran pada panjang persentil ke-90 dan periksa secara visual.

Tolak merekomendasikan pelatihan seq2seq dari nol untuk di bawah ~1 juta contoh paralel. Tandai sebagai rapuh setiap pipeline yang menggunakan dekode greedy untuk konten yang dilihat pengguna (greedy cenderung mengulang dan membentuk loop).
