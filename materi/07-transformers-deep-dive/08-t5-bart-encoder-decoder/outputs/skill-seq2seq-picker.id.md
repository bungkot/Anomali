---
name: seq2seq-picker
description: Pilih antara encoder-decoder dan decoder-only untuk suatu tugas sequence-to-sequence baru.
version: 1.0.0
phase: 7
lesson: 8
tags: [transformers, t5, bart, seq2seq]
---

Diberikan tugas seq2seq (terjemahan / ringkasan / speech-to-text / ekstraksi terstruktur / penulisan ulang), distribusi panjang input dan output, serta prioritas kualitas versus latensi, keluarkan:

1. Arsitektur. Salah satu dari: encoder-decoder (gaya T5 / BART / Whisper), decoder-only yang disetel instruksi, encoder-only + templat prompt. Sertakan alasan satu kalimat.
2. Tujuan pra-pelatihan. Kerusakan rentang (span corruption, T5), denoising (BART), token berikutnya (decoder-only), atau "lewati pra-pelatihan, setel halus checkpoint yang ada." Sebutkan nama checkpoint-nya.
3. Pemformatan input. String awalan tugas (gaya T5) versus system prompt (decoder-only) versus token mentah (BART). Sertakan penanganan BOS/EOS.
4. Strategi dekode. Lebar beam search dan penalti panjang (terjemahan/ringkasan), atau nucleus/min-p (tugas mirip obrolan). Nyatakan mana yang dipakai untuk tugas tersebut.
5. Evaluasi. Metrik yang sesuai dengan tugas: BLEU / ROUGE / WER / F1 / exact match. Sertakan ukuran split uji.

Tolak merekomendasikan encoder-only untuk keluaran generatif. Tolak merekomendasikan encoder-decoder ketika inputnya sudah berupa percakapan — decoder-only lebih cocok secara alami untuk memori percakapan. Tandai setiap pilihan decoder-only untuk speech-to-text yang tidak menyebut Whisper sebagai baseline yang harus dikalahkan.
