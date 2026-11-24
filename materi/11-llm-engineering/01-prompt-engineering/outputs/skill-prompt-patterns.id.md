---
name: skill-prompt-patterns
description: Kerangka keputusan untuk memilih pola prompt yang tepat berdasarkan jenis tugas, persyaratan keandalan, dan model target
version: 1.0.0
phase: 11
lesson: 01
tags: [prompt-engineering, patterns, llm, temperature, cross-model, few-shot, chain-of-thought]
---

# Panduan Pemilihan Pola Prompt

Saat membangun fitur bertenaga LLM, pilih pola prompt sebelum menulis promptnya. Pola menentukan strukturnya. Konten mengisinya.

## Matriks Keputusan Pola

| Jenis Tugas | Pola Utama | Pola Sekunder | Suhu | Perlu Few-Shot? |
|-----------|----------------|-------------------|-------------|-----------------|
| Ekstraksi data | Isian Templat | Few-Shot | 0.0 | Ya (2-3 contoh) |
| Klasifikasi | Few-Shot | Guardrail | 0.0 | Ya (3-5 contoh) |
| Ringkasan | Persona + Templat | Adaptasi Audiens | 0.3 | Tidak |
| Generasi kode | Persona | Chain-of-Thought | 0.0 | Opsional |
| Penulisan kreatif | Persona | Kritik | 0.7-1.0 | Tidak |
| Penalaran multilangkah | Chain-of-Thought | Dekomposisi | 0.3 | Opsional |
| Tanya jawab | Persona + Guardrail | Batas | 0.3 | Tidak |
| Generasi prompt | Meta-Prompt | Kritik | 0.7 | Ya (1-2 contoh) |
| Moderasi konten | Guardrail + Batas | Few-Shot | 0.0 | Ya (5+ contoh) |
| Terjemahan/adaptasi | Adaptasi Audiens | Few-Shot | 0.3 | Ya (2-3 contoh) |

## Kapan Menggunakan Setiap Pola

**Pola Persona**: gunakan untuk setiap prompt sebagai dasar. Satu-satunya pertanyaan adalah seberapa spesifik peran itu dibuat. Untuk tugas umum, peran yang luas sudah cukup. Untuk tugas khusus domain, peran harus menyebut domain, tingkat senioritas, dan konteks.

**Pola Few-Shot**: gunakan ketika format keluaran lebih penting daripada konten. Jika model perlu menghasilkan bentuk JSON tertentu, format CSV, atau label klasifikasi, contoh lebih efektif daripada instruksi. Aturan praktis: 2-3 contoh untuk format sederhana, 5+ untuk format kompleks atau ambigu.

**Pola Chain-of-Thought**: gunakan untuk matematika, logika, analisis multilangkah, dan tugas apa pun yang memerlukan model "menunjukkan langkah kerjanya." Meningkatkan akurasi 10-40% pada tugas penalaran (Wei dkk., 2022). JANGAN gunakan untuk pencarian fakta sederhana atau ekstraksi -- itu membuang-buang token.

**Pola Isian Templat**: gunakan untuk ekstraksi terstruktur yang setiap keluarannya harus berbentuk sama. Bekerja terbaik dengan temperature=0.0 dan penanganan eksplisit "N/A" untuk bidang yang kosong.

**Pola Kritik**: gunakan ketika kualitas lebih penting daripada kecepatan. Model menghasilkan, mengkritik, lalu memperbaiki. Biaya token kira-kira dua kali lipat, tetapi akurasi dan kelengkapan meningkat signifikan. Terbaik untuk keluaran berisiko tinggi (laporan, rekomendasi, konten yang menghadap publik).

**Pola Guardrail**: gunakan untuk sistem apa pun yang menghadap pengguna. Selalu sertakan: batas ruang lingkup, perilaku penolakan untuk permintaan di luar ruang lingkup, dan penanganan eksplisit "Saya tidak tahu". Gabungkan dengan validasi masukan di sisi aplikasi.

**Pola Meta-Prompt**: gunakan untuk menghasilkan prompt bagi tugas baru. Alih-alih menulis prompt dari awal, jelaskan tugasnya dan biarkan model yang menulis promptnya. Lalu uji dan iterasi. Menghemat waktu pada pengembangan prompt awal.

**Pola Dekomposisi**: gunakan untuk masalah kompleks yang untung dengan strategi bagi-bagi-taklukkan. Model memecah masalah menjadi bagian-bagian, menyelesaikan masing-masing, lalu menggabungkannya. Paling efektif untuk tugas dengan 3-7 submasalah.

**Pola Adaptasi Audiens**: gunakan ketika konten yang sama perlu melayani audiens berbeda. Sebut audiens secara eksplisit -- jangan bergantung pada model yang menebak dari konteks.

**Pola Batas**: gunakan untuk sistem produksi yang TIDAK PERNAH boleh menjawab jenis pertanyaan tertentu. Lebih kuat daripada guardrail karena menetapkan ruang lingkup keras dengan pesan penolakan yang tepat. Esensial untuk domain yang sensitif terhadap kepatuhan.

## Kompatibilitas Lintas Model

Pola diurutkan berdasarkan konsistensinya di GPT-4o, Claude 3.5 Sonnet, Gemini 1.5 Pro, dan Llama 3:

| Pola | Konsistensi Lintas Model | Catatan |
|---------|------------------------|-------|
| Few-Shot | Sangat tinggi | Contoh berpindah dengan baik ke semua model |
| Isian Templat | Sangat tinggi | Struktur eksplisit menyisakan sedikit ruang untuk perbedaan |
| Chain-of-Thought | Tinggi | Semua model utama mendukung "berpikir langkah demi langkah" |
| Persona | Tinggi | Bekerja di mana saja, tetapi model berbeda merespons tingkat spesifisitas peran yang berbeda |
| Guardrail | Sedang | Claude mengikuti guardrail paling ketat; GPT-4o kadang menyimpang dalam percakapan panjang |
| Kritik | Sedang | Kualitas swakritik sangat bervariasi antarmodel |
| Meta-Prompt | Sedang | GPT-4o dan Claude menghasilkan gaya prompt yang berbeda |
| Batas | Rendah-Sedang | Perilaku penolakan bervariasi; uji per model |

## Kesalahan Umum

1. **Menggunakan Chain-of-Thought untuk segalanya**: CoT menambah token dan latensi. Gunakan hanya ketika langkah penalaran diperlukan.
2. **Terlalu banyak batasan**: lebih dari 5-7 batasan dan model mulai mengabaikan sebagian. Prioritaskan 3 yang paling penting.
3. **Persona + batasan yang bertentangan**: "Anda adalah penulis kreatif" + "Jangan pernah menggunakan metafora" membingungkan model.
4. **Tanpa spesifikasi suhu**: membiarkan suhu pada nilai bawaan (biasanya 1.0) saat Anda memerlukan keluaran deterministik.
5. **Menyalin-tempel prompt antarmodel**: selalu uji. Prompt yang disetel untuk GPT-4o bisa lebih buruk pada Claude, dan sebaliknya.
6. **Mengabaikan pesan system**: memasukkan semuanya ke pesan user alih-alih menggunakan pesan system untuk aturan yang persisten.
7. **Terlalu bergantung pada batasan negatif**: "JANGAN lakukan X, Y, Z, A, B, C" kurang efektif daripada "Lakukan HANYA W." Pembingkaian positif memberi model target yang jelas.

## Target Keandalan

| Kasus Penggunaan | Kombinasi Pola | Akurasi yang Diharapkan | Biaya Token |
|----------|-------------------|-------------------|------------|
| Ekstraksi produksi | Templat + Few-Shot | 95%+ | Rendah (500-1K) |
| Tanya jawab untuk pengguna | Persona + Guardrail + Batas | 90%+ | Sedang (1-2K) |
| Generasi kode | Persona + Chain-of-Thought | 85%+ | Sedang (1-3K) |
| Generasi konten | Persona + Kritik | Kualitas 90%+ | Tinggi (2-4K, dua pass) |
| Klasifikasi | Few-Shot + Guardrail | 95%+ | Rendah (300-800) |
| Analisis kompleks | Dekomposisi + Chain-of-Thought | 85%+ | Tinggi (3-5K) |
