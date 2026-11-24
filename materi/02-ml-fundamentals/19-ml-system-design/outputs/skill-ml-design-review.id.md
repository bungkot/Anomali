---
name: skill-ml-design-review
description: Tinjau dokumen desain sistem ML sebelum pelatihan dimulai, dengan celah pemblokir terlebih dahulu
version: 1.0.0
phase: 2
lesson: 19
tags: [ml-system-design, design-review, baselines, serving, rollout]
---

# Tinjauan Desain ML

Gunakan skill ini ketika seseorang membagikan dokumen desain, proposal, atau rencana untuk sebuah model yang akan mengambil keputusan nyata. Tinjau dokumen dengan urutan di bawah ini. Laporkan lebih dulu celah-celah yang memblokir, lalu celah lainnya, lalu pertanyaan-pertanyaan.

## Langkah 1: Periksa enam butiran pemblokir

Hentikan tinjauan dan minta perbaikan apabila salah satu butiran ini tidak ada:

1. **Keputusan.** Dokumen menyebutkan tindakan yang menggunakan setiap prediksi serta siapa atau apa yang mengambilnya.
2. **Metrik bisnis.** Dokumen menyebutkan hasil perusahaan yang harus diubah oleh keputusan itu dan bagaimana sebuah eksperimen mengukurnya.
3. **Baseline yang kuat.** Dokumen membandingkan model dengan aturan heuristik atau dengan sistem saat ini. Baseline kelas mayoritas saja tidak cukup.
4. **Tidak ada kebocoran.** Dokumen memastikan bahwa setiap fitur tersedia pada saat prediksi dibuat.
5. **Peluncuran dan pemulihan.** Rencana memuat tahap shadow atau canary serta pemulihan (rollback) yang memakan waktu beberapa menit.
6. **Nilai default saat gagal.** Setiap mode kegagalan memiliki tindakan default, misalnya aturan heuristik atau skor terakhir yang baik.

## Langkah 2: Periksa perumusan masalah

- Prediksi menyebutkan entitas, target, dan horizon, misalnya "pengguna membatalkan dalam 30 hari ke depan".
- Dokumen menyatakan mengapa sebuah aturan atau tabel pencarian tidak cukup.
- Jenis tugas ML sesuai dengan keputusannya: klasifikasi, regresi, ranking, peramalan, atau deteksi anomali.

## Langkah 3: Periksa metrik

| Lapisan | Pertanyaan |
|---|---|
| Offline | Apakah metriknya sesuai dengan ambang keputusan? Pakai presisi dalam 5 persen teratas jika pekerjaan bertindak pada 5 persen teratas. |
| Online | Sinyal langsung apa yang menunjukkan model bekerja: tingkat penerimaan, bagian permintaan yang diberi skor, distribusi skor? |
| Bisnis | Eksperimen apa yang mengukur hasilnya, dan berapa lama sampai hasilnya tersedia? |
| Pagar pengaman | Metrik apa yang tidak boleh memburuk: biaya, latensi, keluhan, pembatalan langganan? |

## Langkah 4: Periksa data dan label

- Dari mana label berasal dan siapa pemilik tabel itu
- Keterlambatan label dalam hari, dan apa yang dipantau tim sampai label tiba
- Jumlah contoh positif
- Himpunan uji tertahan berbasis waktu, dengan periode paling terbaru yang ditahan

## Langkah 5: Periksa pilihan penyajian dengan aritmetika

Mintalah lima angka: jumlah entitas, permintaan per hari, QPS puncak, keusangan maksimum dalam jam, dan anggaran latensi p99. Lalu hitung biaya kedua mode:

```text
batch runs per day   = 24 / (max_staleness_hours - batch_job_hours)
batch cost per day   = entities * runs per day * price per prediction + lookups * price per lookup
online replicas      = max(2, ceil(peak_qps / qps per replica))
online cost per day  = replicas * price per replica hour * 24
```

- Batch tidak dapat bekerja ketika batas keusangan lebih pendek dari satu pekerjaan batch.
- Online tidak dapat bekerja ketika latensi model berada di atas anggaran p99.
- Jika keduanya bekerja, pilih yang lebih murah, dan periksa berapa banyak skor batch yang tidak pernah dibaca siapa pun.
- Entitas baru tanpa riwayat memerlukan jalur online atau sebuah aturan.

## Langkah 6: Periksa loop umpan balik

Tanyakan bagaimana keputusan model mengubah data latihan masa depannya:

- Apakah sebuah intervensi, seperti penawaran atau pemblokiran, menyembunyikan hasil yang sebenarnya?
- Apakah hanya item yang ditampilkan model yang mengumpulkan umpan balik?
- Apakah ada kelompok uji acak atau bagian eksplorasi yang menjaga data tetap bersih?
- Apakah sistem mencatat skor dan keputusan bersama setiap contoh?

## Langkah 7: Tulis tinjauan

Gunakan format ini:

```text
Keputusan: siap ditinjau | revisi
Pemblokir: <daftar, atau "tidak ada">
Celah: <daftar celah non-pemblokir>
Pemeriksaan penyajian: <batch $/hari, online $/hari, mode terpilih, lulus atau gagal>
Pertanyaan: <paling banyak lima pertanyaan untuk penulis>
```

Untuk menilai versi JSON dari dokumen, jalankan `python3 code/main.py path/to/design_doc.json` dari folder pelajaran.
