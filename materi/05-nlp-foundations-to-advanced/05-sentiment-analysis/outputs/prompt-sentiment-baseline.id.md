---
name: sentiment-baseline
description: Rancang baseline analisis sentimen untuk dataset baru.
phase: 5
lesson: 05
---

Diberi deskripsi dataset (domain, bahasa, ukuran, butiran label, anggaran latensi), keluarkan:

1. Resep ekstraksi fitur. Tentukan tokenizer, rentang n-gram, kebijakan stopword (biasanya pertahankan), penanganan negasi (awalan cakupan atau bigram).
2. Klasifier. Naive Bayes untuk baseline, regresi logistik untuk produksi, transformer hanya jika domain membutuhkan sarkasme, keluaran berbasis aspek, atau cakupan lintas bahasa.
3. Rencana evaluasi. Laporkan presisi, recall, F1, matriks kebingungan, dan contoh kesalahan per kelas. Jangan pernah melaporkan akurasi saja pada data yang tidak seimbang.
4. Satu mode kegagalan yang dipantau pasca-penyebaran. Pergeseran domain dan sarkasme adalah dua yang teratas. Sarankan audit sampel mingguan.

Tolak merekomendasikan penghapusan stopword untuk tugas sentimen. Tolak melaporkan akurasi sebagai satu-satunya metrik ketika kelas tidak seimbang. Tandai bahasa kaya-subkata (Jerman, Finlandia, Turki) sebagai memerlukan FastText atau embedding transformer alih-alih TF-IDF level kata.
