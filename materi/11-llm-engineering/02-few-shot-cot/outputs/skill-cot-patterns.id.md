---
name: skill-cot-patterns
description: Kerangka keputusan untuk memilih teknik penalaran yang tepat berdasarkan kompleksitas tugas, kebutuhan akurasi, dan batasan biaya
version: 1.0.0
phase: 11
lesson: 02
tags: [chain-of-thought, few-shot, self-consistency, tree-of-thought, react, reasoning, prompting]
---

# Panduan Pemilihan Teknik Penalaran

Saat Anda memerlukan LLM untuk menalar melalui suatu masalah, pilih tekniknya terlebih dahulu sebelum menulis prompt. Teknik yang dipilih menentukan arsitektur penalaran. Prompt mengisinya.

## Pohon Keputusan Singkat

1. Apakah tugasnya berupa pencarian fakta sederhana atau klasifikasi selangkah?
   - Ya: gunakan **zero-shot**. CoT hanya menambah biaya tanpa kenaikan akurasi.
   - Tidak: lanjutkan.

2. Apakah tugas memerlukan penalaran multilangkah (matematika, logika, perencanaan)?
   - Ya: gunakan **Chain-of-Thought**. Lanjutkan ke langkah 3.
   - Tidak: gunakan **few-shot** jika format penting, zero-shot jika tidak.

3. Apakah satu kesalahan penalaran masih dapat diterima?
   - Ya: gunakan **few-shot CoT** (satu sampel, temperature 0,0).
   - Tidak: gunakan **self-consistency** (N=5, temperature 0,7). Lanjutkan ke langkah 4.

4. Apakah masalahnya berupa masalah pencarian/perencanaan dengan banyak jalur yang mungkin?
   - Ya: gunakan **Tree-of-Thought**.
   - Tidak: self-consistency sudah cukup.

5. Apakah tugas memerlukan informasi atau komputasi eksternal?
   - Ya: gunakan **ReAct** (penalaran + panggilan alat/tool).
   - Tidak: teknik penalaran murni sudah cukup.

## Matriks Teknik

| Teknik | Kenaikan Akurasi | Pengali Biaya | Latensi | Paling Cocok Untuk |
|--------|------------------|---------------|---------|--------------------|
| Zero-shot | Dasar | 1x | ~1 dtk | Tugas sederhana, tanya-jawab faktual |
| Few-shot | +5-15% | 1,2x | ~1 dtk | Pencocokan format, klasifikasi |
| Zero-shot CoT | +10-20% | 1,3x | ~1,5 dtk | Dorongan penalaran cepat |
| Few-shot CoT | +15-25% | 1,5x | ~2 dtk | Matematika, logika, multilayer |
| Self-Consistency (N=5) | +2-5% atas CoT | 5x | ~5 dtk | Penalaran berisiko tinggi |
| Self-Consistency (N=10) | +1-2% atas N=5 | 10x | ~10 dtk | Hanya untuk keputusan kritis |
| Tree-of-Thought | Bergantung tugas | 10-40x | ~30 dtk+ | Pencarian, perencanaan, teka-teki |
| ReAct | Bergantung tugas | 3-10x | ~5-15 dtk | Tugas berbasis pengetahuan |
| Prompt Chaining | +5-10% atas tunggal | 2-5x | ~5-10 dtk | Tugas kompleks berbagian banyak |

## Panduan Khusus Model

### GPT-4o / GPT-4.1
- Dasar penalaran yang kuat. Zero-shot CoT sering sudah cukup.
- Few-shot CoT dengan 3 contoh mencapai 95% pada GSM8K.
- Self-consistency memberi perbaikan marjinal (95% menjadi 97%) -- hanya layak untuk tugas kritis.
- Mendukung keluaran terstruktur secara native untuk ekstraksi jawaban.

### Claude 3.5 Sonnet / Claude 3.7 Sonnet
- Sangat baik mengikuti format prompt terstruktur (tag XML).
- Few-shot CoT dengan contoh yang dibatasi tag XML bekerja paling baik.
- Extended thinking (Claude 3.7) adalah CoT native -- tidak perlu memintanya lewat prompt.
- Self-consistency efektif karena penalaran Claude sangat bervariasi pada temperature 0,7.

### Llama 3.1/3.3 70B
- Paling banyak mendapat manfaat dari few-shot CoT (jurang akurasi lebih besar dibanding zero-shot).
- Self-consistency dengan N=5 disarankan untuk tugas penalaran.
- Membutuhkan instruksi format yang lebih eksplisit dibanding model komersial.
- ToT mahal pada inferensi lokal -- pertimbangkan hanya untuk pemrosesan batch.

### Gemini 2.5 Pro
- Kuat dalam penalaran multilangkah sejak awal.
- Mode thinking menyediakan CoT bawaan tanpa rekayasa prompt.
- Contoh few-shot lebih membantu konsistensi format daripada akurasi.
- Jendela konteks besar (1M) membuat few-shot yang banyak contoh menjadi praktis.

## Anti-Pola

**CoT untuk tugas sederhana**: bertanya "Berapa 2+2? Mari kita berpikir selangkah demi selangkah" hanya membuang token. Model sudah benar untuk aritmetika sederhana tanpa jejak penalaran. CoT membantu ketika ada 3+ langkah.

**Self-consistency pada temperature 0,0**: semua N sampel akan identik. Anda harus memakai temperature > 0 (0,5-0,8 disarankan) agar jalur penalaran beragam.

**ToT untuk segala hal**: ToT membutuhkan panggilan LLM sejumlah O(b^d) dengan b=faktor percabangan dan d=kedalaman. Pohon dengan b=3, d=3 membutuhkan hingga 39 panggilan. Simpan hanya untuk masalah yang gagal dengan teknik yang lebih murah.

**Few-shot dengan contoh buruk**: contoh yang mengandung kesalahan penalaran mengajari model melakukan kesalahan itu. Setiap contoh harus diverifikasi. Satu contoh yang salah dapat menurunkan akurasi lebih parah daripada tanpa contoh sama sekali.

**Mengekstrak jawaban tanpa format yang konsisten**: self-consistency memerlukan perbandingan jawaban lintas sampel. Jika format jawaban berubah ("$18", "18 dolar", "delapan belas"), voting gagal. Selalu wajibkan: "Jawabannya adalah [angka]."

## Optimasi Biaya

Untuk sistem produksi yang melayani 10.000 permintaan/hari dengan harga GPT-4o ($2,50/1M token masukan, $10/1M token keluaran):

| Teknik | Rata-rata Token/Kueri | Biaya Harian | Akurasi |
|--------|-----------------------|--------------|---------|
| Zero-shot | ~200 | ~$5 | 78% |
| Few-shot CoT | ~600 | ~$15 | 95% |
| Self-Consistency (N=5) | ~3.000 | ~$75 | 97% |
| ToT (b=3, d=2) | ~6.000 | ~$150 | Bergantung tugas |

Strategi paling optimal dari sisi biaya untuk sebagian besar aplikasi: mulailah dengan few-shot CoT. Tambahkan self-consistency hanya untuk kueri yang kepercayaannya rendah (pola eskalasi dari bagian Build It).

## Integrasi dengan Prompt Chaining

Teknik penalaran dapat disusun dengan prompt chaining:

**Langkah Rantai 1** (Ekstrak): zero-shot, temperature 0,0
**Langkah Rantai 2** (Nalar): few-shot CoT, temperature 0,0
**Langkah Rantai 3** (Verifikasi): self-consistency dengan N=3, temperature 0,7

Rantai tiga langkah ini biayanya ~3x satu panggilan CoT, tetapi menangkap kesalahan ekstraksi, kesalahan penalaran, dan memberikan skor kepercayaan dari langkah verifikasi.

## Kapan Harus Beralih dari Prompting

Jika Anda menghabiskan lebih banyak waktu untuk rekayasa prompt daripada menulis kode aplikasi, pertimbangkan:

1. **Fine-tuning**: jika Anda memiliki 500+ contoh berlabel dan tugasnya sempit
2. **Kompilasi DSPy**: jika Anda ingin optimasi prompt otomatis
3. **Kerangka kerja agen**: jika tugas memerlukan penggunaan alat multi-giliran (Fase 14)
4. **RAG**: jika model memerlukan akses ke pengetahuan pribadi/terkini (Pelajaran 06-07)

Teknik prompting adalah fondasi. Teknik ini bekerja dengan model apa pun, penyedia apa pun, dan tidak memerlukan data pelatihan. Namun teknik ini punya batas. Mengetahui kapan harus naik ke level berikutnya sama pentingnya dengan menguasai tekniknya sendiri.
