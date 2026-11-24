---
name: sim2real-planner
description: Rencanakan pipeline transfer sim-to-real untuk robot + tugas tertentu, mencakup DR, SI, dan keselamatan.
version: 1.0.0
phase: 9
lesson: 11
tags: [rl, sim2real, robotics, domain-randomization]
---

Dengan platform robot, sebuah tugas, dan akses ke waktu perangkat keras sungguhan, keluarkan:

1. Inventarisasi jurang realitas. Sumber yang dicurigai diurutkan berdasarkan dampak yang diharapkan (kontak, sensing, keterlambatan aktuasi, visi).
2. Parameter DR. Daftar pasti, rentang, distribusi. Justifikasi setiap rentang terhadap pengukuran dunia nyata.
3. Langkah SI. Parameter mana yang diukur; metode pengukuran.
4. Pembagian guru/murid. Informasi istimewa apa yang dipakai guru; observasi apa yang dipakai murid.
5. Amplop keselamatan. Batas tingkat rendah, pemberhentian darurat, pengontrol cadangan.

Tolak penerapan tanpa (a) uji varian sim zero-shot, (b) pelindung keselamatan, (c) rencana rollback. Tandai rentang DR apa pun yang lebih lebar dari 3× variabilitas real terukur sebagai kemungkinan besar terlalu terandomisasi.
