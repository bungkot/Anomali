---
name: text-encoder-picker
description: Pilih arsitektur encoder teks untuk sekumpulan kendala tertentu.
phase: 5
lesson: 08
---

Diberikan kendala (tugas, volume data, anggaran latensi, target deploy, anggaran komputasi), hasilkan:

1. Arsitektur encoder: TextCNN, BiLSTM, BiLSTM-CRF, fine-tuning transformer, atau "transformer terlatih sebagai encoder beku + head kecil".
2. Masukan embedding: inisialisasi acak, GloVe atau fastText beku, atau embedding transformer terkontekstualisasi.
3. Resep pelatihan dalam 5 baris: pengoptimal, laju pembelajaran, ukuran batch, epoch, regularisasi.
4. Satu sinyal pemantauan. Model RNN/CNN: periksa akurasi per panjang sekuens untuk kegagalan ketergantungan jarak jauh. Fine-tuning transformer: waspadai keruntuhan fine-tuning jika LR terlalu tinggi; periksa loss pelatihan dalam 100 langkah pertama.

Tolak merekomendasikan fine-tuning transformer ketika pengguna memiliki di bawah ~500 contoh berlabel tanpa terlebih dahulu menunjukkan bahwa baseline TextCNN / BiLSTM sudah mencapai plateau. Tandai deploy di tepi (ponsel, mikrokontroler, browser) sebagai kebutuhan yang menuntut keputusan arsitektur sebelum hal lain.
