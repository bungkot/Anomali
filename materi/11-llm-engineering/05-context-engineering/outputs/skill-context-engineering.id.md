---
name: skill-context-engineering
description: Kerangka pengambilan keputusan untuk merancang pipeline perakitan konteks berdasarkan jenis tugas, ukuran jendela, dan anggaran latensi
version: 1.0.0
phase: 11
lesson: 05
tags: [context-engineering, context-window, rag, memory, tool-selection, lost-in-the-middle]
---

# Rekayasa Konteks

Saat membangun aplikasi LLM, terapkan kerangka ini untuk merancang pipeline perakitan konteks.

## Prinsip inti

1. **Konteks itu langka.** Jendela 128K terdengar besar, tetapi cepat penuh. Alokasikan anggaran untuk setiap komponen secara eksplisit.
2. **Atensi tidak merata.** Model memusatkan atensi lebih besar pada awal dan akhir. Letakkan informasi penting di sana. Bagian tengah adalah zona mati.
3. **Dinamis mengalahkan statis.** Kueri berbeda membutuhkan konteks berbeda. Rakit konteks per kueri, bukan sekali saat startup.
4. **Lebih sedikit lebih baik.** Konteks 10K yang kurasi lebih unggul daripada konteks 100K yang ditumpahkan begitu saja. Rasio sinyal terhadap derau lebih penting daripada total informasi.
5. **Ukur semuanya.** Anda tidak dapat mengoptimalkan apa yang tidak Anda ukur. Hitung token per komponen pada setiap permintaan.

## Pedoman anggaran konteks

| Komponen | Rentang tipikal | Prioritas | Strategi kompresi |
|----------|-----------------|-----------|-------------------|
| System prompt | 200-1,000 token | Tetap, tinggi | Tulis ringkas, hilangkan redundansi |
| Definisi alat | 500-3,000 token | Dinamis, sedang | Pangkas berdasarkan niat kueri |
| Konteks hasil pengambilan | 1,000-5,000 token | Dinamis, tinggi | Peringkat ulang + ambang batas + deduplikasi |
| Riwayat percakapan | 500-5,000 token | Dinamis, sedang | Ringkas giliran lama |
| Contoh few-shot | 500-2,000 token | Dinamis, tinggi | Pilih berdasarkan kesamaan tugas |
| Kueri pengguna | 50-500 token | Tetap, tertinggi | N/A |
| Cadangan generasi | 2,000-8,000 token | Tetap | Sesuaikan dengan panjang keluaran yang diharapkan |

## Kapan menggunakan tiap jenis memori

**Jangka pendek (riwayat percakapan):** Sesi berjalan. Dikelola dengan peringkasan. Kompres giliran yang lebih tua dari 5-10 pertukaran. Simpan 3-4 giliran terakhir kata demi kata.

**Jangka panjang (basis data fakta):** Preferensi dan fakta proyek yang bertahan lintas sesi. Ambil saat sesi dimulai. Contoh: "pengguna lebih suka Python", "proyek menggunakan PostgreSQL", "tim mengikuti pengembangan berbasis trunk". Simpan di CLAUDE.md, basis data, atau sistem memori terstruktur.

**Episodik (interaksi masa lalu):** Percakapan masa lalu tertentu yang relevan dengan tugas saat ini. Simpan sebagai embedding, ambil berdasarkan kesamaan. "Minggu lalu kita men-debug masalah autentikasi yang serupa" adalah memori episodik.

## Strategi pemilihan alat

Jangan sertakan semua alat pada setiap permintaan. Ini membuang-buang token dan membingungkan model.

1. Klasifikasikan niat kueri (kode, email, kalender, riset, data)
2. Petakan niat ke kategori alat
3. Sertakan hanya alat yang cocok
4. Jika niat ambigu, sertakan alat dari 2 kategori teratas
5. Selalu sertakan satu alat "umum" (seperti pencarian web) sebagai cadangan

Penghematan yang diharapkan: 60-80% token definisi alat pada kueri dengan niat yang jelas.

## Praktik terbaik pengambilan (retrieval)

- **Peringkat ulang setelah pengambilan.** Kesamaan vektor adalah filter kasar. Peringkat ulang (cross-encoder atau berbasis LLM) meningkatkan presisi secara signifikan.
- **Tetapkan ambang relevansi.** Jangan sertakan potongan (chunk) dengan kesamaan cosinus di bawah 0.3. Potongan itu hanya menambah derau.
- **Lakukan deduplikasi.** Jika dua potongan memiliki kesamaan konten 80% atau lebih, simpan hanya yang skornya lebih tinggi.
- **Terapkan pengurutan lost-in-the-middle.** Letakkan potongan paling relevan di awal dan di akhir.
- **Batasi total token hasil pengambilan.** 3-5 potongan yang sangat relevan lebih baik daripada 15 potongan biasa saja.

## Pengelolaan riwayat

- Simpan 3-4 giliran terakhir kata demi kata (model membutuhkan konteks terkini)
- Ringkas giliran lama menjadi ringkasan ("Kita membahas X, memutuskan Y, dan terhambat pada Z")
- Buang giliran yang dihasilkan sistem dan tidak menambah informasi (pemanggilan alat tanpa konten yang ditujukan kepada pengguna)
- Picu kompresi ketika riwayat melebihi 30% dari anggaran yang tersedia

## Tanda bahaya

- System prompt melebihi 2,000 token: kemungkinan memuat informasi yang seharusnya dinamis
- Semua alat disertakan pada setiap permintaan: terapkan pemilihan berbasis niat
- Tidak ada filter relevansi pada pengambilan: Anda membuang derau ke dalam jendela
- Riwayat tumbuh tanpa batas: peringkasan belum diterapkan
- Tidak ada cadangan generasi: model memotong responsnya
- Informasi yang sama di 3 tempat (system prompt, dokumen hasil pengambilan, riwayat): lakukan deduplikasi
- Pemanfaatan konteks di atas 60%: Anda menyisakan terlalu sedikit ruang bagi model untuk "berpikir"
