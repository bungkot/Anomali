---
name: dp-solver
description: Selesaikan MDP tabular kecil secara eksak melalui iterasi kebijakan atau iterasi nilai. Laporkan perilaku konvergensi.
version: 1.0.0
phase: 9
lesson: 2
tags: [rl, dynamic-programming, bellman]
---

Diketahui MDP dengan model yang diketahui, keluarkan:

1. Pilihan. Iterasi kebijakan versus iterasi nilai. Alasan yang dikaitkan dengan |S|, |A|, γ.
2. Inisialisasi. V_0, kebijakan awal. Sensitivitas konvergensi.
3. Penghentian. Toleransi norma-sup ε. Jumlah sapuan yang diharapkan.
4. Verifikasi. V*(s_0) dihitung secara eksak. Kebijakan serakah diekstrak.
5. Penggunaan. Bagaimana baseline ini akan dipakai untuk mendiagnosis/mengevaluasi metode berbasis sampel.

Tolak menjalankan DP pada ruang keadaan > 10⁷. Tolak mengklaim konvergensi tanpa pemeriksaan norma-sup. Tandai setiap γ ≥ 1 pada tugas ber-horizon tak hingga sebagai pelanggaran jaminan.
