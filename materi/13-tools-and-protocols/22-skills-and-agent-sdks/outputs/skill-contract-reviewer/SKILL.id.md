---
name: skill-contract-reviewer
description: Validasi paket Agent Skill dan pilih primitif instruksi, kapabilitas, atau siklus hidup yang tepat sebelum implementasi.
license: MIT
metadata:
  lesson: "22"
---

# Pemeriksa kontrak skill

Pakai skill ini ketika sebuah alur kerja hendak menjadi artefak agen yang dapat dipakai ulang.

1. Tetapkan `SKILL_ROOT` sebagai direktori absolut yang berisi `SKILL.md` yang
   terpasang ini. Jangan mengira direktori kerja proses adalah bundel itu.
2. Tetapkan `TARGET_ROOT` sebagai direktori kerja ruang kerja asli yang absolut
   dan selesaikan direktori skill yang diusulkan di bawah root itu.
3. Baca `$SKILL_ROOT/references/contract.md` dan validasi kolom identitas
   `SKILL.md` portabel.
4. Baca `$SKILL_ROOT/references/decision-model.md` dan pisahkan konteks
   repositori, metode yang dapat dipakai ulang, kapabilitas eksternal,
   penentuan waktu siklus hidup, logika deterministik, dan delegasi terisolasi.
5. Sebelum eksekusi, tampilkan vektor argumen yang telah diselesaikan persisnya.
   Jalankan `python3 "$SKILL_ROOT/scripts/check_skill.py" "$TARGET_SKILL"`,
   dengan `TARGET_SKILL` adalah direktori skill yang diusulkan secara absolut
   di bawah `TARGET_ROOT`.
6. Periksa laporan JSON. Perbaiki setiap error sebelum mendiskusikan ekstensi
   spesifik host.
7. Bandingkan artefak yang diusulkan dengan
   `$SKILL_ROOT/assets/task-shapes.json` dan kembalikan himpunan primitif
   terkecil yang dapat dikomposisikan.

Jangan klaim bahwa ekstensi runtime adalah bagian dari kontrak portabel. Jangan memperlakukan skill yang valid sebagai izin untuk menjalankan skrip atau mengakses alat.

Kembalikan laporan validasi, primitif yang dipilih, dan satu kalimat yang
menjelaskan setiap pilihan. Sertakan bukti eksekusi berupa jalur skrip yang
telah diselesaikan, jalur target yang telah diselesaikan, cwd, argv yang
persis, dan kode keluar. Bila host tidak dapat menampilkan salah satu pengamatan
itu, tandai sebagai tidak terverifikasi alih-alih mengarangnya.
