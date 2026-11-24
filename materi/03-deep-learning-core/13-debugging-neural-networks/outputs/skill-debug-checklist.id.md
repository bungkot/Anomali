---
name: skill-debug-checklist
description: Daftar periksa pohon keputusan untuk debugging kegagalan pelatihan jaringan saraf
version: 1.0.0
phase: 3
lesson: 13
tags: [debugging, neural-networks, training, diagnostics, deep-learning]
---

# Daftar Periksa Debug Jaringan Saraf

Protokol debugging sistematis untuk saat pelatihan berjalan salah. Kerjakan secara berurutan -- sebagian besar bug terdeteksi dalam 3 langkah pertama.

## Sebelum pelatihan (mencegah bug)

1. Cetak arsitektur model dan jumlah parameter. Apakah ukurannya masuk akal untuk data Anda?
2. Jalankan satu forward pass dengan masukan acak. Apakah bentuk keluaran cocok dengan bentuk target?
3. Cek bahwa label bertipe benar (CrossEntropyLoss butuh Long, BCELoss butuh Float)
4. Verifikasi normalisasi data: masukan seharusnya memiliki rata-rata mendekati 0 dan std mendekati 1
5. Cetak 5 pasangan (masukan, label) acak. Apakah label sesuai dengan yang Anda harapkan?
6. Pastikan pembagian train/test tidak memiliki sampel duplikat

## Uji overfit-satu-batch (60 detik, menangkap 80% bug)

1. Ambil 8-32 sampel dari himpunan pelatihan Anda
2. Latih selama 200 langkah dengan laju pembelajaran yang wajar
3. Loss seharusnya mendekati 0. Akurasi pelatihan seharusnya mencapai 100%
4. Jika gagal: bug ada pada model, fungsi rugi, atau loop pelatihan -- bukan pada data atau hiperparameter
5. Jika lolos: lanjut ke pelatihan penuh

## Loss tidak menurun

1. Cek laju pembelajaran. Coba 3 nilai: current/10, current, current*10
2. Cetak norma gradien per lapisan. Semua nol berarti jaringan mati atau graf terputus
3. Cek `requires_grad=True` pada parameter. Pastikan `loss.backward()` dipanggil
4. Pastikan `optimizer.zero_grad()` dipanggil sebelum `loss.backward()`
5. Pastikan `optimizer.step()` dipanggil setelah `loss.backward()`
6. Verifikasi parameter model diteruskan ke optimizer: `optimizer = Adam(model.parameters())`

## Loss bernilai NaN atau Inf

1. Turunkan laju pembelajaran 10x
2. Tambahkan epsilon pada semua pemanggilan log(): `torch.log(x + 1e-7)`
3. Tambahkan epsilon pada semua pembagian: `x / (y + 1e-8)`
4. Jepit prediksi: `torch.clamp(pred, 1e-7, 1 - 1e-7)` sebelum loss BCE
5. Gunakan `torch.autograd.detect_anomaly()` untuk menemukan operasi yang tepat
6. Cek NaN pada data masukan: `assert not torch.isnan(x).any()`

## Loss berosilasi

1. Turunkan laju pembelajaran 3-10x
2. Besarkan ukuran batch (mengurangi noise gradien)
3. Tambahkan pemotongan gradien: `torch.nn.utils.clip_grad_norm_(model.parameters(), 1.0)`
4. Ganti SGD dengan Adam (LR adaptif per parameter)
5. Tambahkan pemanasan laju pembelajaran untuk 5-10% pertama dari pelatihan

## Overfitting (akurasi train tinggi, test rendah)

1. Tambahkan dropout (mulai dari p=0.1, naikkan ke 0.5)
2. Tambahkan penurunan bobot ke optimizer: `Adam(params, weight_decay=1e-4)`
3. Perkecil ukuran model (lebih sedikit lapisan atau lapisan lebih sempit)
4. Tambahkan augmentasi data
5. Gunakan early stopping: berhenti saat loss validasi meningkat selama 5+ epoch
6. Cek kebocoran data antara himpunan train dan test

## Underfitting (akurasi train dan test sama-sama rendah)

1. Besarkan kapasitas model (lebih banyak lapisan, lapisan lebih lebar)
2. Latih lebih banyak epoch
3. Naikkan laju pembelajaran (dengan hati-hati)
4. Hapus regularisasi sementara untuk memverifikasi model bisa belajar
5. Cek apakah model cukup ekspresif untuk tugasnya

## Neuron ReLU mati

1. Cek fraksi aktivasi nol per lapisan. >50% adalah masalah
2. Ganti ke LeakyReLU(0.01) atau GELU
3. Gunakan inisialisasi Kaiming untuk bobot
4. Turunkan laju pembelajaran (pembaruan besar bisa mendorong neuron ke zona mati)
5. Tambahkan batch normalization sebelum fungsi aktivasi

## Referensi cepat: titik awal laju pembelajaran

| Pengoptimal | Tugas | LR awal |
|-------------|-------|---------|
| Adam | Pelatihan dari awal | 1e-3 |
| Adam | Fine-tuning model pra-pelatihan | 1e-5 |
| SGD + momentum | Pelatihan dari awal | 1e-1 |
| SGD + momentum | Fine-tuning model pra-pelatihan | 1e-3 |
| AdamW | Pelatihan Transformer | 3e-4 |

## Referensi cepat: efek ukuran batch

| Ukuran batch | Noise gradien | Memori | Generalisasi |
|--------------|---------------|--------|--------------|
| 8-16 | Tinggi (noisy) | Rendah | Sering lebih baik |
| 32-64 | Sedang | Sedang | Default yang baik |
| 128-256 | Rendah (mulus) | Tinggi | Mungkin perlu pemanasan |
| 512+ | Sangat rendah | Sangat tinggi | Perlu penskalaan LR |

## Saat tidak ada yang berhasil

1. Sederhanakan model menjadi 1 lapisan tersembunyi. Apakah model belajar?
2. Sederhanakan data menjadi 100 sampel. Apakah terjadi overfit?
3. Ganti fungsi rugi dengan MSE. Apakah konvergen?
4. Ganti optimizer dengan SGD(lr=0.01). Apakah ada kemajuan?
5. Ganti data dengan data sintetis (mis. y = x[0] > 0). Apakah model belajar?
6. Jika tidak ada yang berhasil: bug ada pada kode yang tidak Anda periksa (pemuatan data, praproses, bentuk tensor)
