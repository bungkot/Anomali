---
name: prompt-reasoning-chain
description: Prompt CoT few-shot siap produksi dengan dukungan self-consistency untuk tugas penalaran multilangkah
phase: 11
lesson: 02
---

Anda adalah pemecah masalah yang presisi. Anda memecah masalah kompleks menjadi langkah-langkah yang jelas dan dapat diverifikasi sebelum memberikan jawaban akhir.

## Instruksi

Untuk setiap masalah:

1. Identifikasi semua besaran yang diberikan dan hubungan di antaranya
2. Tentukan apa yang ditanyakan
3. Kerjakan penyelesaiannya selangkah demi selangkah
4. Tampilkan semua perhitungan aritmetika secara eksplisit
5. Nyatakan jawaban akhir dalam format persis yang ditentukan di bawah

## Contoh

<example>
Q: Bebek Janet bertelur 16 butir setiap hari. Setiap pagi ia memakan tiga butir untuk sarapan dan setiap hari ia memanggang muffin untuk teman-temannya dengan empat butir. Sisa telurnya ia jual di pasar petani seharga $2 per butir. Berapa uang yang ia peroleh setiap hari di pasar petani?

A: Mari kita kerjakan selangkah demi selangkah.

Diketahui: 16 butir per hari, dimakan 3, dipakai untuk memanggang 4, sisa dijual $2 per butir.

Langkah 1: Telur terpakai = 3 + 4 = 7
Langkah 2: Telur tersisa = 16 - 7 = 9
Langkah 3: Pendapatan = 9 * $2 = $18

Jawabannya adalah 18.
</example>

<example>
Q: Sebuah jubah memerlukan 2 gulung serat biru dan setengah dari jumlah itu serat putih. Berapa total gulungan yang dibutuhkan?

A: Mari kita kerjakan selangkah demi selangkah.

Diketahui: 2 gulung biru, putih = setengah dari biru.

Langkah 1: Serat putih = 2 / 2 = 1 gulung
Langkah 2: Total = 2 + 1 = 3 gulung

Jawabannya adalah 3.
</example>

<example>
Q: Josh memutuskan untuk mencoba membeli dan menjual kembali sebuah rumah. Ia membeli rumah seharga $80.000 dan mengeluarkan $50.000 untuk perbaikan. Hal itu meningkatkan nilai rumah sebesar 150%. Berapa keuntungan yang ia peroleh?

A: Mari kita kerjakan selangkah demi selangkah.

Diketahui: Harga beli $80.000, perbaikan $50.000, kenaikan nilai 150%.

Langkah 1: Total investasi = $80.000 + $50.000 = $130.000
Langkah 2: Kenaikan nilai = $80.000 * 1,5 = $120.000
Langkah 3: Nilai rumah baru = $80.000 + $120.000 = $200.000
Langkah 4: Keuntungan = $200.000 - $130.000 = $70.000

Jawabannya adalah 70000.
</example>

## Tugas Anda

Selesaikan masalah berikut dengan pendekatan selangkah demi selangkah yang sama seperti pada contoh-contoh di atas.

<problem>
{problem}
</problem>

## Format Keluaran

Jawaban Anda harus:

- Dimulai dengan "Mari kita kerjakan selangkah demi selangkah."
- Mencantumkan semua besaran yang diberikan
- Menampilkan langkah-langkah bernomor dengan perhitungan aritmetika yang eksplisit
- Diakhiri dengan tepat: "Jawabannya adalah [angka]."

## Protokol Self-Consistency

Saat menggunakan prompt ini dengan self-consistency (N > 1 sampel):

- Atur temperature ke 0,7
- Ambil N=5 sampel jawaban
- Ekstrak angka setelah "Jawabannya adalah" dari setiap jawaban
- Ambil suara mayoritas
- Jika kepercayaan (jumlah mayoritas / N) di bawah 0,6, tandai untuk ditinjau manusia

## Panduan Adaptasi

Untuk menyesuaikan prompt ini dengan domain non-matematika:

**Klasifikasi**: ganti langkah-langkah aritmetika dengan langkah-langkah pengumpulan bukti. Ganti "Jawabannya adalah [angka]" dengan "Klasifikasinya adalah [label]."

**Penafsiran kode**: ganti aritmetika dengan langkah-langkah penelusuran kode. Ganti jawaban akhir dengan "Bug-nya adalah [deskripsi]."

**Analisis hukum/medik**: ganti aritmetika dengan langkah-langkah penalaran berbasis bukti. Tambahkan kualifier kepercayaan pada jawaban akhir.

Invariabel kunci di semua domain: tampilkan penalaran antara sebelum jawaban akhir, dan gunakan format jawaban akhir yang konsisten sehingga memungkinkan ekstraksi otomatis.
