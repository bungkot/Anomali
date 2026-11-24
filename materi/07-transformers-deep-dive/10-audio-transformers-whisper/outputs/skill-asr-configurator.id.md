---
name: asr-configurator
description: Pilih model ASR (varian Whisper / Moonshine / faster-whisper) dan parameter dekode untuk pipeline suara baru.
version: 1.0.0
phase: 7
lesson: 10
tags: [transformers, whisper, asr, speech]
---

Diberikan tugas suara (transkripsi / terjemahan / streaming / on-device), bahasa(-bahasa), karakteristik audio (kebisingan, aksen, durasi), serta target latensi/kualitas, keluarkan:

1. Pilihan model. Salah satu dari: faster-whisper large-v3-turbo (default produksi), whisper large-v3 (kualitas tertinggi, multibahasa), whisper medium (kelas menengah), Moonshine base (edge), distil-whisper (2× lebih cepat untuk bahasa Inggris). Sertakan alasan satu kalimat.
2. Kuantisasi. int8_float16 (default CPU), float16 (default GPU), fp32 (riset). Tandai dampaknya terhadap VRAM.
3. Dekode. Lebar beam (5 lazim, 1 untuk streaming), jadwal fallback temperature, ambang log-prob, ambang no-speech, gerbang VAD aktif/nonaktif.
4. Pemotongan (chunking). Jendela tetap 30 s versus potongan streaming (biasanya 10 s dengan tumpang tindih 2 s) + segmentasi berbasis VAD. Dokumentasikan strategi penggabungan ulang untuk area tumpang tindih.
5. Pasca-pemrosesan. Penyelarasan timestamp (forced alignment WhisperX), restorasi tanda baca, diarisasi (pyannote). Tandai mana yang diperlukan oleh tugas.

Tolak merekomendasikan OpenAI Whisper biasa (implementasi referensi) untuk produksi — `faster-whisper` 4× lebih cepat dengan keluaran identik. Tolak mengirim ASR streaming tanpa VAD kecuali ada alasan terdokumentasi. Tandai setiap asumsi satu pembicara ketika input kemungkinan berasal dari banyak pembicara.
