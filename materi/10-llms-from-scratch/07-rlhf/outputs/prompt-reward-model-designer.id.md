---
name: prompt-reward-model-designer
description: Rancang pipeline pelatihan model hadiah untuk penyelarasan RLHF
version: 1.0.0
phase: 10
lesson: 7
tags: [rlhf, reward-model, ppo, alignment, human-feedback, preference-learning]
---

# Perancang Model Hadiah

Saat membangun pipeline RLHF untuk menyelaraskan model bahasa ke arah perilaku target (kebergunaan, kemampuan pemrograman, keamanan, kejujuran), gunakan kerangka kerja ini untuk merancang protokol pengumpulan data, melatih model hadiah, dan mengonfigurasi PPO.

## Persyaratan Masukan

Berikan:

- **Perilaku target** (mis. "asisten yang berguna dan tidak berbahaya", "programmer Python ahli", "tanya jawab medis dengan keamanan")
- **Model dasar** (mis. Llama 3 8B setelah SFT, Mistral 7B Chat)
- **Ukuran model hadiah** (biasanya sama besar atau lebih besar daripada model kebijakan)
- **Anggaran anotasi** (jam kerja manusia atau jumlah pasangan perbandingan yang tersedia)
- **Anggaran komputasi** (jam GPU untuk pelatihan model hadiah + PPO)

## Langkah 1: Pengumpulan Data Preferensi

### Protokol Anotasi

1. **Seleksi prompt**: Ambil sampel dari distribusi pelatihan SFT ditambah prompt di luar distribusi (10-20% baru)
2. **Pembangkitan respons**: Hasilkan 2-4 respons per prompt menggunakan model SFT dengan suhu berbeda (0,3, 0,7, 1,0)
3. **Format perbandingan**: Tunjukkan tepat 2 respons kepada anotator dan tanyakan "Respons mana yang lebih baik?"
4. **Rubrik kriteria**: Tentukan arti "lebih baik" untuk kasus penggunaan Anda

### Template Rubrik

| Kriteria | Bobot | Deskripsi |
|----------|-------|-----------|
| Kebergunaan | 40% | Apakah jawaban ini menjawab pertanyaan secara lengkap dan benar? |
| Keamanan | 25% | Apakah jawaban ini menghindari konten berbahaya, bias, atau menyesatkan? |
| Kejujuran | 20% | Apakah jawaban ini mengakui ketidakpastian alih-alih berhalusinasi? |
| Keringkasan | 15% | Apakah panjang respons sesuai untuk pertanyaannya? |

Sesuaikan bobot untuk kasus penggunaan Anda. Asisten pemrograman mungkin memberikan bobot 60% pada kebenaran dan 20% pada keringkasan.

### Pedoman Ukuran Data

| Skala | Pasangan Perbandingan | Jam Anotator | Akurasi RM yang Diharapkan |
|-------|-----------------------|--------------|----------------------------|
| Minimum layak | 5.000-10.000 | 400-800 | 60-65% |
| Produksi v1 | 20.000-50.000 | 1.600-4.000 | 65-72% |
| Produksi v2 | 100.000-500.000 | 8.000-40.000 | 72-78% |

InstructGPT menggunakan 33.000 perbandingan dari 40 kontraktor. Makalah awal Anthropic menggunakan 22.000 dari 20 anotator. Kesepakatan antar-anotator biasanya 70-75% -- model hadiah tidak dapat melebihi tingkat kesepakatan manusia.

### Kontrol Kualitas

- **Penyaringan kesepakatan**: Buang pasangan yang disetujui oleh kurang dari 70% anotator
- **Kalibrasi anotator**: Jalankan putaran kalibrasi dengan pasangan yang sudah diketahui baik sebelum anotasi sesungguhnya
- **Deteksi bias**: Pantau apakah anotator secara konsisten lebih menyukai respons yang lebih panjang, bahasa formal, atau pola tertentu
- **Contoh adversarial**: Sertakan 5-10% contoh yang dirancang untuk menjaring anotator yang tidak membaca dengan saksama

## Langkah 2: Arsitektur Model Hadiah

### Keputusan Arsitektur

| Keputusan | Rekomendasi | Alasan |
|-----------|-------------|--------|
| Arsitektur dasar | Transformer yang sama dengan kebijakan | Inisialisasi bobot dari checkpoint SFT memberikan fitur awal yang kuat |
| Head keluaran | Satu proyeksi linier dari keadaan tersembunyi terakhir | Hadiah skalar dari representasi posisi paling lengkap |
| Ukuran model | >= ukuran model kebijakan | RM yang lebih kecil menghasilkan sinyal yang tidak andal dan mengganggu stabilitas PPO |
| Inisialisasi | Checkpoint SFT dengan head keluaran baru | Fitur yang telah dilatih sebelumnya sudah menangkap kualitas bahasa |

### Konfigurasi Pelatihan

| Parameter | Rentang | Catatan |
|-----------|---------|---------|
| Laju pembelajaran | 1e-5 sampai 5e-5 | Lebih rendah daripada SFT karena tugasnya lebih sederhana |
| Epoch | 1-3 | Overfitting merupakan risiko besar dengan data perbandingan yang terbatas |
| Ukuran batch | 64-256 | Setiap "contoh" adalah satu pasangan, sehingga data efektifnya 2x |
| Fungsi rugi | Bradley-Terry: -log(sigmoid(r_preferred - r_rejected)) | Standar untuk perbandingan berpasangan |
| Pemisahan validasi | 10-20% | Pantau akurasi pada pasangan yang tidak dipakai saat pelatihan |

### Metrik Evaluasi

1. **Akurasi berpasangan**: Berapa bagian dari pasangan preferensi yang tidak dipakai saat pelatihan yang berhasil diurutkan dengan benar oleh RM? Target: > 65%
2. **Distribusi margin**: Plot distribusi dari (r_preferred - r_rejected). Seharusnya terpusat di atas 0 dengan sedikit nilai negatif.
3. **Kalibrasi**: Apakah sigmoid(r_preferred - r_rejected) mendekati probabilitas preferensi manusia yang sebenarnya?
4. **Generalisasi OOD**: Uji pada prompt yang berasal dari distribusi berbeda dengan data pelatihan. Penurunan akurasi seharusnya < 10%.

## Langkah 3: Konfigurasi PPO

### Hiperparameter

| Parameter | Nilai Khas | Efek Jika Terlalu Tinggi | Efek Jika Terlalu Rendah |
|-----------|------------|--------------------------|--------------------------|
| Koefisien KL (beta) | 0,01-0,05 | Model hampir tidak belajar, terlalu dekat dengan SFT | Reward hacking, keluaran yang merusak |
| Laju pembelajaran | 5e-6 sampai 3e-5 | Pelatihan tidak stabil, divergensi | Konvergensi lambat, komputasi terbuang |
| Rasio klip (epsilon) | 0,1-0,3 | Pembaruan besar yang berpotensi menidakstabilkan | Pembaruan sangat konservatif, pembelajaran lambat |
| Epoch PPO per batch | 1-4 | Overfitting pada batch saat ini | Batch tidak dimanfaatkan secara optimal |
| Ukuran batch pembangkitan | 128-512 | Masalah memori | Estimasi gradien yang bising |
| Panjang respons maksimum | 256-1024 | Pembangkitan lambat, masalah memori | Memotong respons yang berguna |

### Dasbor Pemantauan

Lacak metrik-metrik ini selama pelatihan PPO:

1. **Hadiah rata-rata**: Seharusnya meningkat seiring pelatihan. Plot yang mendatar tidak apa-apa; penurunan berarti ketidakstabilan.
2. **Divergensi KL**: Seharusnya tetap di bawah 10-20 nats. Lonjakan = reward hacking.
3. **Panjang respons**: Seharusnya tetap stabil. Kenaikan monoton = reward hacking berupa verbositas.
4. **Entropi**: Entropi distribusi token seharusnya menurun secara perlahan. Penurunan cepat = mode collapse.
5. **Kesepakatan model hadiah**: Beri skor respons PPO dengan model hadiah; kesepakatan seharusnya membaik.

### Tanda Bahaya Selama PPO

| Gejala | Penyebab yang Mungkin | Perbaikan |
|--------|-----------------------|-----------|
| Hadiah naik tetapi keluaran memburuk | Reward hacking | Naikkan koefisien KL, latih ulang RM dengan contoh adversarial |
| Divergensi KL meledak | Laju pembelajaran terlalu tinggi atau koefisien KL terlalu rendah | Turunkan lr, naikkan beta |
| Panjang respons bertambah secara monoton | RM memberi hadiah pada verbositas | Tambahkan penalti panjang pada hadiah, latih ulang RM dengan pasangan yang dikontrol panjangnya |
| Semua respons menjadi identik | Mode collapse | Naikkan suhu pembangkitan, kurangi epoch PPO |
| Hadiah berayun liar | Ketidakstabilan PPO | Turunkan laju pembelajaran, naikkan rasio klip |

## Langkah 4: Validasi Ujung ke Ujung

Sebelum menerapkan model yang telah dilatih dengan RLHF:

1. **Uji A/B terhadap SFT**: Jalankan model SFT dan RLHF pada 200+ prompt uji. Minta 3+ evaluator membandingkan respons. Model RLHF seharusnya menang > 60% dari kasus.
2. **Evaluasi keamanan**: Uji pada prompt adversarial yang diketahui (jailbreak, permintaan berbahaya). Model RLHF seharusnya menolak dengan tepat.
3. **Pemeriksaan regresi**: Jalankan benchmark standar (MMLU, HumanEval, MT-Bench) untuk memastikan model RLHF tidak kehilangan kemampuan inti.
4. **Pemeriksaan lupa**: Ukur perplexity pada korpus teks umum. Kenaikan seharusnya < 10% dibandingkan model SFT.
5. **Analisis panjang**: Bandingkan panjang respons rata-rata antara model SFT dan RLHF. Jika respons RLHF > 50% lebih panjang, model hadiah kemungkinan memiliki bias verbositas.
