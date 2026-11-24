---
name: skill-llm-evaluation
description: Kerangka keputusan untuk memilih strategi evaluasi LLM yang tepat berdasarkan jenis tugas, anggaran, dan kebutuhan
version: 1.0.0
phase: 10
lesson: 10
tags: [evaluation, evals, benchmarks, llm-as-judge, elo, metrics]
---

# Strategi Evaluasi LLM

Saat mengevaluasi sebuah sistem LLM, terapkan kerangka keputusan ini untuk memilih pendekatan yang tepat.

## Kapan menggunakan tiap jenis evaluasi

**Benchmark (MMLU, HumanEval, SWE-bench):** Anda sedang melakukan seleksi model awal. Anda perlu menyempitkan 10 kandidat menjadi 3. Benchmarkmemberikan peringkat kasar tanpa biaya. Jangan gunakan benchmark sebagai evaluasi akhir Anda.

**Evaluasi khusus (custom evals):** Anda membangun untuk produksi. Anda memiliki tugas spesifik dengan mode kegagalan yang spesifik. Evaluasi khusus adalah satu-satunya evaluasi yang memprediksi kinerja dunia nyata. Minimum 50 kasus uji untuk prototipe, 200+ untuk produksi.

**LLM-as-judge:** Tugas Anda bersifat terbuka (ringkasan, penulisan, percakapan). Exact match dan metrik tumpang tindih token terlalu kaku. LLM-as-judge berbiaya ~$0,01 per penilaian dan setuju dengan manusia sekitar 80% waktu. Selalu gunakan rubrik, bukan prompt yang kabur.

**Evaluasi manusia:** Taruhannya tinggi dan metrik otomatis tidak sepakat. Evaluasi manusia adalah kebenaran dasar (ground truth), tetapi berbiaya $0,10-$2,00 per penilaian. Simpan untuk kasus ambigu dan kalibrasi berkala metrik otomatis.

**ELO dari perbandingan berpasangan:** Anda membandingkan beberapa model pada tugas yang sama. Berpasangan lebih andal daripada penilaian absolut karena manusia (dan juri LLM) lebih baik dalam penilaian relatif.

## Pemilihan fungsi penilaian

- **Exact match**: klasifikasi, ekstraksi entitas, keluaran terstruktur dengan jawaban yang diketahui
- **Token F1**: tugas ekstraksi di mana kredit parsial penting
- **ROUGE-L**: ringkasan, terjemahan
- **BLEU**: terjemahan mesin
- **LLM-as-judge**: generasi terbuka, kualitas percakapan, kebergunaan
- **Berbasis eksekusi**: generasi kode (jalankan kodenya, periksa apakah tes lulus)
- **Kepatuhan skema**: keluaran terstruktur (apakah JSON sesuai skema?)

## Tanda bahaya dalam desain evaluasi

- Himpunan evaluasi kurang dari 50 kasus: hasilnya tidak bermakna secara statistik
- Tidak ada kasus tepi: Anda mengukur kinerja jalur normal, yang selalu lebih tinggi daripada dunia nyata
- Metrik tunggal: metrik berbeda menceritakan kisah berbeda, gunakan setidaknya dua
- Tanpa versi: Anda tidak dapat melacak perbaikan tanpa himpunan evaluasi berversi
- Kontaminasi himpunan evaluasi: jangan sertakan contoh evaluasi dalam data fine-tuning atau prompt few-shot
- Hanya menguji satu model: Anda memerlukan baseline (bahkan heuristik sederhana) untuk perbandingan

## Daftar periksa pipeline evaluasi

1. Tentukan tugas secara presisi (bukan "menjawab pertanyaan" melainkan "mengklasifikasikan tiket dukungan ke dalam 5 kategori")
2. Buat kasus uji yang mencakup jalur normal, kasus tepi, dan regresi yang diketahui
3. Pilih 2-3 fungsi penilaian yang sesuai dengan jenis tugas
4. Tetapkan ambang lulus/gagal berdasarkan kebutuhan produksi
5. Otomatisasi eksekusi: satu perintah menjalankan seluruh rangkaian
6. Beri versi pada semuanya: kasus uji, fungsi penilaian, prompt, versi model
7. Jalankan pada setiap perubahan: pembaruan prompt, penggantian model, penerapan kode
8. Lacak tren: satu skor adalah derau, garis tren adalah sinyal
9. Kalibrasi terhadap penilaian manusia setiap kuartal
10. Tambahkan kasus regresi setiap kali ada kegagalan produksi yang ditemukan
