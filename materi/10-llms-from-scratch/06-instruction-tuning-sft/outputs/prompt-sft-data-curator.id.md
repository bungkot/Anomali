---
name: prompt-sft-data-curator
description: Rancang dan kurasi dataset instruksi untuk supervised fine-tuning
version: 1.0.0
phase: 10
lesson: 6
tags: [sft, instruction-tuning, fine-tuning, data-curation, alignment]
---

# Kurator Data SFT

Saat merancang dataset penyetelan instruksi untuk kapabilitas tertentu (generasi kode, matematika, percakapan, keamanan), gunakan kerangka kerja ini untuk merencanakan pengumpulan data, mendefinisikan kriteria kualitas, dan menyusun pipeline pelatihan.

## Persyaratan Masukan

Berikan:
- **Kapabilitas target** (misalnya "generasi kode Python", "tanya jawab medis", "percakapan multi-giliran")
- **Model dasar** (misalnya Llama 3 8B, Mistral 7B, Qwen 2.5 72B)
- **Anggaran** (jam anotasi, biaya API untuk generasi sintetis)
- **Preferensi format** (Alpaca, ShareGPT, ChatML)

## Langkah 1: Desain Dataset

### Pedoman Ukuran

| Tingkat Kualitas | Contoh Dibutuhkan | Hasil yang Diharapkan |
|------------------|-------------------|-----------------------|
| Prototipe riset | 1.000-5.000 | Kualitas LIMA: sejajar dengan dataset lebih besar jika contoh ditulis pakar |
| Produksi v1 | 10.000-50.000 | Level Stanford Alpaca: pengikutan instruksi solid pada tugas umum |
| Produksi v2 | 50.000-200.000 | Level Vicuna/Llama 2 Chat: multi-giliran tangguh, cakupan domain |

Kualitas selalu mengalahkan kuantitas. 1.000 contoh hasil tulisan pakar (LIMA, Mei 2023) sejajar dengan model yang dilatih pada 50.000+ contoh. Prioritaskan:

1. **Keragaman** -- cakup seluruh rentang kapabilitas target
2. **Keakuratan** -- setiap respons harus benar secara faktual
3. **Kejelasan** -- respons sebaiknya ringkas dan terstruktur dengan baik
4. **Gradien kesulitan** -- sertakan contoh mudah, sedang, dan sulit

### Daftar Periksa Keragaman

Untuk asisten serbaguna:
- Pertanyaan terbuka (20%)
- Tanya jawab faktual (20%)
- Tulisan kreatif (10%)
- Generasi kode (15%)
- Penalaran dan matematika (15%)
- Ringkasan (10%)
- Pengikutan instruksi dengan batasan (10%)

Sesuaikan persentase untuk model spesifik-domain. Asisten pemrograman mungkin mengalokasikan 60% untuk generasi kode dan 20% untuk penjelasan kode.

## Langkah 2: Format Data

### Format Alpaca (satu giliran)

```json
{
  "instruction": "Write a function that reverses a string in Python.",
  "input": "",
  "output": "def reverse_string(s):\n    return s[::-1]"
}
```

Pakai ketika: tugas satu giliran, pasangan instruksi-respons sederhana, prototipe cepat.

### Format ShareGPT (multi-giliran)

```json
{
  "conversations": [
    {"from": "system", "value": "You are a Python expert."},
    {"from": "human", "value": "How do I reverse a string?"},
    {"from": "gpt", "value": "Use slicing: s[::-1]"},
    {"from": "human", "value": "What about for a list?"},
    {"from": "gpt", "value": "Same syntax works: my_list[::-1]"}
  ]
}
```

Pakai ketika: aplikasi percakapan, konteks multi-giliran penting.

### Format ChatML (dengan token khusus)

```
<|im_start|>system
You are a Python expert.<|im_end|>
<|im_start|>user
How do I reverse a string?<|im_end|>
<|im_start|>assistant
Use slicing: s[::-1]<|im_end|>
```

Pakai ketika: menargetkan model yang memakai ChatML secara native (Qwen, Yi).

## Langkah 3: Kriteria Kualitas

### Pemeriksaan per Contoh

1. **Relevansi respons**: Apakah respons benar-benar menjawab instruksi?
2. **Keakuratan faktual**: Apakah semua klaim dapat diverifikasi dan benar?
3. **Kelengkapan**: Apakah respons sepenuhnya menangani instruksi?
4. **Keringkasan**: Apakah informasi yang sama dapat disampaikan dengan lebih sedikit kata?
5. **Konsistensi format**: Apakah respons mengikuti gaya yang diharapkan?

### Tanda Bahaya (tolak contoh)

- Respons bertentangan dengan dirinya sendiri
- Respons memuat konten berbahaya tanpa penolakan
- Respons menghalusinasi fakta atau sitasi
- Instruksi ambigu dan respons tidak memperjelasnya
- Respons hanya parafrasa dari instruksi

### Pemeriksaan Tingkat Dataset

- Tidak lebih dari 5% contoh berasal dari satu sumber/template
- Setidaknya 80% token respons bermakna (bukan pengisi)
- Panjang respons rata-rata 50-200 token (hindari yang sangat pendek atau sangat panjang)
- Keragaman system prompt: setidaknya 10 system prompt berbeda terwakili

## Langkah 4: Konfigurasi Pelatihan

| Parameter | Rentang yang Disarankan | Catatan |
|-----------|-------------------------|---------|
| Laju pembelajaran | 1e-5 sampai 5e-5 | Lebih rendah untuk model lebih besar (1e-5 untuk 70B, 5e-5 untuk 7B) |
| Epoch | 1-3 | Pantau loss validasi, berhenti pada tanda pertama kenaikan |
| Ukuran batch | 32-128 | Skalakan dengan akumulasi gradien jika GPU terbatas |
| Warmup | 0-5% dari langkah | Kurang krusial dibanding pra-pelatihan |
| Weight decay | 0,0-0,1 | Opsional untuk pelatihan singkat |
| Loss masking | Hanya token respons | Tutupi token instruksi dan system prompt |
| Pencampuran data pra-pelatihan | 2-5% | Campur teks mentah untuk mencegah catastrophic forgetting |

## Langkah 5: Protokol Evaluasi

Setelah pelatihan, evaluasi pada:

1. **Tingkat pengikutan instruksi**: Persentase prompt uji di mana model menghasilkan respons yang relevan dan lengkap
2. **Skor lupa**: Perplexity pada korpus teks umum yang disisihkan dibandingkan dengan model dasar
3. **Kepatuhan format**: Persentase respons yang mengikuti format chat yang diharapkan
4. **MT-Bench atau AlpacaEval**: Benchmark standar untuk model yang telah disetel mengikuti instruksi
5. **Evaluasi spesifik-domain**: Evaluasi khusus untuk kapabilitas target Anda

### Tanda Peringatan

- Loss validasi meningkat setelah epoch 1: Anda overfit, kurangi epoch atau tambah data
- Skor lupa meningkat > 15%: laju pembelajaran terlalu tinggi atau epoch terlalu banyak
- Model mereproduksi contoh pelatihan kata demi kata: overfitting parah, butuh data lebih beragam
- Model menolak instruksi yang tidak berbahaya: terlatih berlebih pada data keamanan, seimbangkan kembali dataset
