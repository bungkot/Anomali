---
name: skill-fine-tuning-guide
description: Pohon keputusan kapan dan bagaimana melakukan fine-tuning LLM dengan LoRA dan QLoRA
version: 1.0.0
phase: 11
lesson: 8
tags: [fine-tuning, lora, qlora, peft, llm-engineering]
---

# Panduan Keputusan Fine-Tuning

Sebelum melakukan fine-tuning, coba hal-hal ini secara berurutan:

```
1. Rekayasa prompt (menit, $0)
2. Contoh few-shot dalam prompt (menit, $0)
3. RAG untuk pengambilan pengetahuan (hari, $10-100/bulan)
4. Fine-tuning dengan LoRA/QLoRA (hari, $5-50 per eksperimen)
5. Fine-tuning penuh (minggu, $100-10,000 per proses)
```

Lanjut ke langkah berikutnya hanya jika langkah sebelumnya secara terukur tidak memadai.

## Kapan melakukan fine-tuning

- Model memerlukan gaya atau format keluaran konsisten yang tidak bisa dicapai dengan prompting
- Anda mendistilasi model yang lebih besar (kualitas GPT-4 dari model 8B)
- Latensi penting dan contoh few-shot menambah terlalu banyak token
- Anda perlu model mengikuti pola penalaran kompleks secara andal
- Anda memiliki 1,000+ contoh berkualitas tinggi tentang perilaku masukan-keluaran yang diinginkan

## Kapan TIDAK melakukan fine-tuning

- Model sudah melakukan yang Anda inginkan dengan prompt yang tepat
- Anda perlu model mengetahui fakta (gunakan RAG sebagai gantinya)
- Anda memiliki kurang dari 500 contoh pelatihan (kemungkinan besar overfit)
- Tugas berubah dengan cepat (pelatihan ulang mahal)
- Anda perlu mengaudit data mana yang memengaruhi keluaran tertentu (fine-tuning adalah kotak hitam)

## Pemilihan metode

| VRAM GPU | Model 7B | Model 13B | Model 70B |
|----------|----------|-----------|-----------|
| 16GB (T4) | QLoRA | Tidak layak | Tidak layak |
| 24GB (3090/4090) | QLoRA atau LoRA | QLoRA | Tidak layak |
| 40GB (A100) | LoRA atau Penuh | QLoRA atau LoRA | QLoRA |
| 80GB (A100/H100) | Penuh | LoRA atau Penuh | QLoRA atau LoRA |

## Daftar pemeriksaan konfigurasi LoRA

1. Mulai dengan r=16, alpha=32 (default aman untuk sebagian besar tugas)
2. Targetkan q_proj dan v_proj terlebih dahulu (LoRA minimum yang layak)
3. Gunakan laju pembelajaran 2e-4 untuk QLoRA, 5e-5 untuk LoRA fp16
4. Tetapkan lora_dropout=0.05
5. Latih selama 1-3 epoch (lebih banyak berisiko overfit)
6. Evaluasi setiap 100 langkah pada himpunan tersisih
7. Simpan checkpoint dan pilih yang terbaik berdasarkan loss evaluasi

## Kesalahan umum

- Melatih terlalu banyak epoch (overfit setelah epoch 2-3 pada dataset kecil)
- Menggunakan laju pembelajaran yang sama dengan fine-tuning penuh (LoRA memerlukan LR lebih tinggi)
- Lupa menetapkan pad token (menyebabkan loss NaN pada model Llama)
- Tidak membekukan model dasar (menggagalkan tujuan LoRA)
- Hanya mengevaluasi pada data pelatihan (selalu sisihkan 10-20% untuk evaluasi)
- Melewati baseline rekayasa prompt (fine-tuning pada masalah yang sudah terselesaikan dengan prompt)

## Verifikasi kualitas

Setelah pelatihan, bandingkan pada 200+ contoh tersisih:
1. Model dasar dengan prompt terbaik (baseline)
2. Model dasar dengan adapter LoRA (model hasil fine-tuning Anda)
3. GPT-4 atau Claude dengan prompt yang sama (batas atas)

Jika model LoRA tidak mengalahkan baseline berprompt, data pelatihan atau konfigurasi Anda yang perlu diperbaiki, bukan menambah daya komputasi.

## Pengelolaan adapter

- Simpan adapter terpisah untuk penyajian multi-tugas (ganti adapter per permintaan)
- Gabungkan adapter ke bobot dasar untuk penerapan satu tugas
- Simpan adapter di Hugging Face Hub (10-100MB, mudah diversionkan dan dibagikan)
- Uji bahwa keluaran model hasil penggabungan sama dengan keluaran sebelum digabungkan sebelum menerapkannya
- Gunakan TIES-Merging atau DARE untuk menggabungkan beberapa adapter menjadi satu

## Debug pelatihan

Jika loss tidak turun:
1. Periksa laju pembelajaran (terlalu rendah untuk LoRA, coba 2e-4)
2. Verifikasi lapisan LoRA benar-benar menerima gradien
3. Pastikan bobot model dasar dibekukan
4. Periksa format data (tokenizer harus sesuai dengan format yang diharapkan model)

Jika loss turun tetapi kualitas evaluasi buruk:
1. Masalah kualitas data pelatihan (garbage in, garbage out)
2. Overfit (kurangi epoch, tingkatkan dropout, tambahkan data)
3. Modul target salah (tambahkan lapisan MLP untuk tugas kompleks)
4. Rank terlalu rendah (coba r=32 atau r=64)
