---
name: dst-designer
description: Rancang pelacak status dialog — skema, ekstraktor, kebijakan pembaruan, evaluasi.
version: 1.0.0
phase: 5
lesson: 29
tags: [nlp, dialogue, task-oriented]
---

Diberikan sebuah kasus penggunaan (domain, bahasa, keterbukaan kosakata, kebutuhan kepatuhan), keluarkan:

1. Skema. Daftar domain, slot per domain, kosakata terbuka vs tertutup per slot.
2. Ekstraktor. Berbasis aturan / seq2seq / LLM-dengan-Pydantic. Sertakan alasan.
3. Kebijakan pembaruan. Bangkitkan-ulang-seluruh-status / inkremental; penanganan koreksi; penanganan negasi.
4. Evaluasi. Joint Goal Accuracy pada himpunan dialog yang disisihkan, presisi/recall tingkat slot, kebingungan pada slot tersulit.
5. Alur konfirmasi. Kapan secara eksplisit meminta konfirmasi kepada pengguna (tindakan destruktif, ekstraksi keyakinan rendah).

Tolak DST hanya-LLM untuk slot sensitif kepatuhan tanpa pemeriksaan sekunder berbasis aturan. Tolak DST apa pun yang tidak bisa memutar balik sebuah slot saat pengguna mengoreksi. Tandai skema tanpa tag versi.
