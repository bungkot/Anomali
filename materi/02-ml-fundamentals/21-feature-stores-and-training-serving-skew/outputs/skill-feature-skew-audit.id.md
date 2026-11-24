---
name: skill-feature-skew-audit
description: Audit pipeline fitur untuk perjalanan waktu dan ketimpangan latihan-penyajian
version: 1.0.0
phase: 2
lesson: 21
tags: [feature-store, point-in-time, training-serving-skew, leakage, backfill]
---

# Audit Ketimpangan Fitur

Gunakan skill ini ketika sebuah model mencetak skor bagus secara offline dan lebih buruk di produksi. Gunakan juga ketika sebuah tim menambahkan feature store, dan sebelum fitur baru đi live. Kerjakan langkah-langkah secara berurutan dan catat setiap temuan.

## Langkah 1: Petakan kedua jalur

Untuk setiap fitur, tuliskan dua baris:

```text
jalur latihan: <tabel sumber> -> <kode transformasi> -> <penggabungan dengan label>
jalur penyajian: <sumber> -> <kode transformasi> -> <penyimpanan atau komputasi saat permintaan>
```

Jika kedua baris menyebut kode transformasi yang berbeda, tandai fitur itu sebagai "dua jalur kode". Itu adalah risiko ketimpangan pertama.

## Langkah 2: Periksa penggabungan latihan untuk perjalanan waktu

- Penggabungan memakai cap waktu label, bukan hanya kunci entitas.
- Penggabungan mengambil baris terakhir yang waktu peristiwanya tidak lebih lambat dari waktu label.
- Penggabungan membuang baris yang lebih tua dari TTL.
- Baris hasil backfill atau koreksi membawa cap waktu pembuatan, dan penggabungan mengabaikan baris yang dibuat setelah waktu label.
- Jendela agregasi berakhir pada waktu label. Jendela seperti "30 hari di sekitar label" bocor.

Uji cepat: hitung baris latihan yang baris fiturnya lebih lambat dari label. Jumlah berapa pun di atas nol adalah kebocoran.

## Langkah 3: Bandingkan definisi

Untuk setiap fitur, bandingkan kedua jalur pada hal-hal ini:

| Hal | Pertanyaan |
|---|---|
| Tepi jendela | Apakah setiap tepi jendela inklusif atau eksklusif pada kedua jalur? |
| Nilai hilang | Apakah kedua jalur memakai default yang sama untuk tanpa riwayat dan untuk nilai yang kedaluwarsa? |
| Satuan dan tipe | Mata uang sama, zona waktu sama, tipe integer atau float sama? |
| Pembulatan | Pembulatan sama, pada tahap yang sama? |
| Pemetaan kategori | Kosakata sama, penanganan nilai tak dikenal sama? |

## Langkah 4: Bandingkan kesegaran

- Berapa usia nilai latihan pada waktu labelnya, rata-rata dan maksimum?
- Berapa usia nilai penyajian pada saat permintaan, rata-rata dan maksimum?
- Apa yang terjadi ketika materialisasi gagal? TTL harus mengubah nilai usang menjadi default, bukan menyajikannya diam-diam.

Model yang dilatih pada nilai yang usianya hingga satu minggu dapat berbeda perilakunya pada nilai yang segar. Samakan kesegarannya, atau latih ulang pada nilai dengan kesegaran penyajian.

## Langkah 5: Ukur ketimpangan

1. Catat nilai fitur yang dipakai jalur penyajian, beserta entitas dan waktu permintaan.
2. Hitung ulang kunci yang sama dengan jalur latihan.
3. Per fitur, laporkan laju ketidaksesuaian dan selisih absolut rata-rata.
4. Pisahkan penyebabnya dengan tiga perbandingan: pembacaan store versus latihan, komputasi segar versus latihan, jalur kedua versus komputasi segar.

Tandai sebuah fitur ketika lebih dari 1 persen nilainya berbeda, kecuali selisih itu adalah jurang kesegaran yang terdokumentasi.

## Langkah 6: Tulis hasil audit

```text
Fitur: <nama, versi>
Jalur: definisi bersama | dua jalur kode
Perjalanan waktu: tidak ditemukan | <baris terdampak dan penyebab>
Celah definisi: <tepi jendela, default, satuan, pembulatan>
Kesegaran: latihan <usia>, penyajian <usia>, TTL <hari>
Ketimpangan terukur: <laju ketidaksesuaian, selisih |diff| rata-rata>
Perbaikan: <pindahkan kedua jalur ke satu definisi di registri | perbaiki penggabungan | samakan default | samakan TTL>
```

Urutan perbaikan: singkirkan perjalanan waktu terlebih dahulu, lalu pindahkan kedua jalur ke satu definisi, lalu samakan kesegaran.
