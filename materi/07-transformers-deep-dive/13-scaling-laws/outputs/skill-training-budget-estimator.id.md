---
name: training-budget-estimator
description: Estimasi (N, D, jam, jumlah GPU) untuk sebuah pelatihan transformer baru diberikan anggaran komputasi dan kendala deploy.
version: 1.0.0
phase: 7
lesson: 13
tags: [scaling-laws, training, chinchilla]
---

Diberikan sasaran pelatihan (rugi target / MMLU target / metrik hilir target), anggaran komputasi (dolar atau FLOPs), volume inferensi (token/bulan), dan kendala (perangkat target, memori, latensi), keluarkan:

1. Rezim komputasi. Chinchilla-optimal, pelatihan berlebih (dioptimalkan untuk inferensi), kurang terlatih (prototipe). Alasan satu kalimat yang terkait dengan volume inferensi.
2. N dan D. Nilai konkret. Cetak rasio `D/N`. Jika pelatihan berlebih, catat penalti rugi dibanding Chinchilla-optimal.
3. Waktu pelatihan tembok (wall-clock). Jam × jumlah GPU dengan asumsi throughput pelatihan (MFU ≈ 40% untuk dense, ~30% untuk MoE). Anggarkan presisi (bf16 / fp8) dan pengoptimal (AdamW / Muon).
4. Sumber data. Korpus bernama atau anggaran sintetis. Tandai jika `D` yang dibutuhkan melampaui token berkualitas tinggi yang tersedia.
5. Catatan risiko. Satu mode kegagalan spesifik: kontaminasi data, ketidakstabilan pengoptimal pada skala, ketidaksesuaian tokenizer dengan panjang konteks, saturasi rangkaian evaluasi.

Tolak melatih model dense >8B di bawah Chinchilla-optimal jika model itu akan melayani volume inferensi tinggi — biaya inferensi berlipat ganda. Tolak menetapkan rugi target tanpa rangkaian evaluasi tersisih yang didefinisikan. Tandai rencana apa pun yang membelanjakan >1% anggaran untuk pencarian arsitektur, bukan kurasi data — pengembaliannya diketahui kecil. Wajibkan uji coba 1% dari anggaran pada skala untuk memvalidasi asumsi sebelum mengucurkan anggaran penuh.
