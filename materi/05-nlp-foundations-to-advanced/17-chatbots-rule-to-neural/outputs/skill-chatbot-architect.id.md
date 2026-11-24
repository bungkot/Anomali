---
name: chatbot-architect
description: Rancang tumpukan chatbot untuk suatu kasus penggunaan tertentu.
version: 1.0.0
phase: 5
lesson: 17
tags: [nlp, agents, chatbot]
---

Dengan konteks produk tertentu (kebutuhan pengguna, batasan kepatuhan, alat yang tersedia, volume data), keluarkan:

1. Arsitektur. Berbasis aturan, berbasis pengambilan (retrieval), neural, agen LLM, atau hibrida (tentukan jalur mana yang menuju ke mana).
2. Pilihan LLM jika berlaku. Sebutkan keluarga model (Claude, GPT-4, Llama-3.1, Mixtral). Sesuaikan dengan kualitas penggunaan alat dan biaya.
3. Strategi penumpuan (grounding). Sumber RAG, metode pengambilan (pelajaran 14), kontrak alat.
4. Rencana evaluasi. Tingkat keberhasilan tugas, kebenaran panggilan alat, tingkat keluar dari topik, tingkat halusinasi pada dialog yang disisihkan.

Tolak merekomendasikan agen LLM murni untuk tindakan destruktif apa pun (pembayaran, penghapusan akun, modifikasi data) tanpa alur konfirmasi terstruktur. Tolak melewati audit prompt injection jika agen memiliki akses tulis ke apa pun.
