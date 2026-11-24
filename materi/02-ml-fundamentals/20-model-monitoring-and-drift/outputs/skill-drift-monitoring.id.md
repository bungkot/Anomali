---
name: skill-drift-monitoring
description: Rencanakan monitor pergeseran untuk model yang telah dipasang dan tangani alarm pergeseran
version: 1.0.0
phase: 2
lesson: 20
tags: [monitoring, drift, psi, kolmogorov-smirnov, chi-square, alerting]
---

# Monitor Pergeseran

Gunakan skill ini dalam dua situasi. Gunakan Bagian A ketika sebuah model akan diluncurkan dan memerlukan monitor. Gunakan Bagian B ketika alarm pergeseran berbunyi.

## Bagian A: Rencanakan monitor

### Langkah 1: Cantumkan sinyal per lapisan

| Lapisan | Sinyal minimum |
|---|---|
| Kesehatan sistem | Latensi p99, tingkat galat, permintaan per menit |
| Masukan | Satu uji pergeseran per fitur penting, tingkat nilai hilang, kategori baru |
| Prediksi | Distribusi skor (PSI pada desil skor), tingkat prediksi positif |
| Hasil | Metrik offline dari desain, dihitung hanya pada peristiwa berlabel |

### Langkah 2: Pilih uji untuk setiap masukan

- Fitur numerik: PSI pada desil referensi, ditambah statistik KS `D`
- Fitur kategorikal: uji chi-kuadrat pada tabel dua baris, ditambah V Cramér
- Fitur dengan lima nilai berbeda atau lebih sedikit: perlakukan sebagai kategorikal

### Langkah 3: Tetapkan jendela

- Referensi: data latihan, atau periode produksi yang telah diperiksa seseorang
- Terkini: jendela yang cukup besar sehingga jendela bersih tetap berada di bawah level peringatan. Jalankan uji pada data lama yang bersih untuk mengukur hal ini.
- Cakup satu siklus mingguan penuh dalam referensi ketika lalu lintas berubah menurut hari kerja

### Langkah 4: Tetapkan kebijakan alarm

Tuliskan angka-angka ini sebelum peluncuran:

```text
family alpha          = 0.01
per-test alpha        = family alpha / number of tests
PSI warn / drift      = 0.10 / 0.25
KS D for drift        = 0.10 together with p < per-test alpha
Cramér's V for drift  = 0.10 together with p < per-test alpha
persistence           = 2 consecutive drift windows before an alert
outcome alert         = metric below reference metric - tolerance, for 2 windows
```

Kirim peringatan awal ke dasbor. Kirim alarm ke orang.

### Langkah 5: Catat keterlambatan label

Nyatakan berapa lama label membutuhkan waktu untuk tiba dan apa yang dipantau tim sampai label itu tiba. Jika keterlambatannya lebih dari satu minggu, tambahkan estimasi kinerja yang memakai probabilitas terkalibrasi. Estimasi semacam itu menangani pergeseran kovariat tetapi tidak dapat mendeteksi pergeseran konsep.

## Bagian B: Tangani alarm

### Langkah 1: Singkirkan cacat pipeline

Periksa hal-hal ini terlebih dahulu, karena tampak seperti pergeseran:

- Perubahan skema, kolom yang diganti nama, atau perubahan satuan
- Lonjakan nilai hilang atau nilai default
- Kategori baru yang berasal dari perubahan kode, bukan dari pengguna
- Penggabungan yang rusak sehingga mengisi fitur dengan null

### Langkah 2: Sebutkan jenis pergeseran

| Bukti | Kemungkinan pergeseran | Tindakan |
|---|---|---|
| Masukan bergeser, metrik hasil stabil | Pergeseran kovariat | Terus awasi. Latih ulang hanya jika lalu lintas bergerak ke wilayah dengan sedikit data. |
| Tingkat positif bergeser, masukan per kelas tampak sama | Pergeseran prior | Kalibrasi ulang skor atau geser ambang keputusan |
| Masukan stabil, metrik hasil turun | Pergeseran konsep | Kumpulkan label baru, lalu latih ulang pada data berlabel terbaru |
| Masukan bergeser dan metrik hasil turun | Pergeseran kovariat yang merugikan, atau kedua jenis sekaligus | Latih ulang pada data berlabel terbaru dan uji model baru pada periode paling terbaru |

### Langkah 3: Periksa besarnya efek

Nilai-p yang kecil pada jendela besar tidak banyak berarti. Laporkan PSI, `D`, atau V Cramér, serta perubahan pada metrik hasil. Urutkan fitur yang bergeser berdasarkan kepentingannya bagi model.

### Langkah 4: Tulis catatan penanganan

```text
Alarm: <sinyal, akhir jendela, nilai, ambang batas>
Pemeriksaan pipeline: <cacat ditemukan atau tidak ada>
Jenis pergeseran: kovariat | prior | konsep | tidak jelas
Dampak: <perubahan metrik hasil, atau "label menunggu hingga <tanggal>">
Tindakan: menunggu | kalibrasi ulang | mengumpulkan label | latih ulang | pulihkan
Pemilik dan tanggal tinjauan: <nama, tanggal>
```

Setelah pelatihan ulang apa pun, atur ulang jendela referensi ke data latihan yang baru.
