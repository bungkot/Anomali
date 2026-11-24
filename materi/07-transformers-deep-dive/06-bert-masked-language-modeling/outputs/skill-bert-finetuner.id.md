---
name: bert-finetuner
description: Tentukan ruang lingkup fine-tuning BERT untuk tugas klasifikasi, ekstraksi, atau pengambilan baru.
version: 1.0.0
phase: 7
lesson: 6
tags: [bert, fine-tuning, nlp]
---

Dengan tugas hilir (klasifikasi / NER / pengambilan / peringkasan ulang / NLI), ukuran data berlabel, dan batasan deployment (latensi, perangkat), keluarkan:

1. Pilihan backbone. Nama model (ModernBERT-base / large, DeBERTa-v3, multilingual-e5, dll.) dengan alasan satu kalimat. Utamakan ModernBERT untuk tugas bahasa Inggris yang memerlukan konteks ≤8K.
2. Spesifikasi head. Klasifikasi: `[CLS]` → dropout → linear(num_classes). NER: linear per token + CRF opsional. Pengambilan: mean-pool + contrastive loss.
3. Resep pelatihan. Pengoptimal (AdamW, lr 2e-5 umumnya), persentase warmup (6–10%), epoch (3–5), ukuran batch, fp16/bf16.
4. Rencana evaluasi. Metrik yang sesuai tugas (akurasi + F1 untuk klasifikasi, F1 tingkat entitas untuk NER, MRR/NDCG untuk pengambilan). Ukuran split data uji.
5. Pemeriksaan mode kegagalan. Satu risiko yang disebutkan: kebocoran label, ketidakseimbangan kelas, pemotongan konteks, ketidaksesuaian tokenizer antara korpus pra-pelatihan dan fine-tuning.

Tolak melakukan fine-tuning BERT untuk keluaran generatif (pembuatan teks) — rekomendasikan decoder-only sebagai gantinya. Tolak merilis fine-tuning tanpa evaluasi berstrata kelas ketika kelas minoritas di bawah 10%. Tandai fine-tuning mana pun yang membuka seluruh backbone dengan <1.000 contoh berlabel sebagai kemungkinan overfit.
