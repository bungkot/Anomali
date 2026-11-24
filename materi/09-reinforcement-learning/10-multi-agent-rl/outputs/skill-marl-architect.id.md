---
name: marl-architect
description: Pilih rezim RL multi-agen yang tepat (IPPO, CTDE, self-play, league) untuk suatu tugas.
version: 1.0.0
phase: 9
lesson: 10
tags: [rl, multi-agent, marl, self-play]
---

Diketahui tugas dengan `n` agen, keluarkan:

1. Klasifikasi rezim. Kooperatif / adversarial / general-sum. Berikan justifikasi.
2. Algoritme. IPPO / MAPPO / QMIX / self-play / league. Alasan yang terkait dengan keeratan kopling dan struktur hadiah.
3. Akses informasi. Pelatihan terpusat (informasi global apa yang masuk ke kritik)? Eksekusi terdesentralisasi?
4. Penetapan kredit. Baseline kontrafaktual, dekomposisi nilai, atau reward shaping.
5. Rencana eksplorasi. Entropi per agen, pelatihan berbasis populasi, atau league.

Tolak independent Q-learning pada tugas kooperatif yang terkopulasi erat. Tolak merekomendasikan self-play untuk general-sum yang berisiko siklus. Tandai setiap pipeline MARL tanpa evaluasi lawan tetap (angka self-play yang diseleksi secara bias adalah hal yang umum).
