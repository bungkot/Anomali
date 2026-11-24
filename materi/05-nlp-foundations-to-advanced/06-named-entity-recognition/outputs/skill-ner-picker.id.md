---
name: ner-picker
description: Pilih pendekatan NER yang tepat untuk tugas ekstraksi tertentu.
version: 1.0.0
phase: 5
lesson: 06
tags: [nlp, ner, extraction]
---

Diberi deskripsi tugas (domain, himpunan label, bahasa, latensi, volume data), keluarkan:

1. Pendekatan. Berbasis aturan + gazetteer, CRF, BiLSTM-CRF, atau fine-tuning transformer.
2. Model awal. Sebutkan namanya (ID model spaCy seperti `en_core_web_sm` / `en_core_web_trf`, ID checkpoint Hugging Face seperti `dslim/bert-base-NER`, atau "kustom, dilatih dari awal").
3. Strategi pelabelan. BIO, BILOU, atau berbasis span. Beri alasan satu kalimat.
4. Evaluasi. Pakai `seqeval`. Selalu laporkan F1 level entitas, jangan pernah level token.

Tolak merekomendasikan fine-tuning transformer untuk kurang dari 500 contoh berlabel, kecuali pengguna sudah punya model domain pra-terlatih (mis. BioBERT untuk bidang medis). Tandai entitas bersarang sebagai memerlukan model berbasis span atau multi-lintasan. Wajibkan audit gazetteer jika pengguna menyebut "skala produksi" sambil memakai label CoNLL-2003 apa adanya.
