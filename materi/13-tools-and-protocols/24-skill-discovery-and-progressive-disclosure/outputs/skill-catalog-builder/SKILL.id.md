---
name: skill-catalog-builder
description: Bangun katalog Agent Skill yang terbatas lintas cakupan penemuan eksplisit dan laporkan tabrakan sebelum memuat isi instruksi.
license: MIT
metadata:
  lesson: "24"
---

# Pembangun katalog skill

Pakai skill ini ketika host agen membutuhkan penemuan yang deterministik lintas lebih dari satu direktori skill.

1. Baca `references/discovery-contract.md`.
2. Tinjau contoh kebijakan host di `assets/scope-policy.json`; jangan mengira urutannya bersifat universal.
3. Jalankan `python3 scripts/build_catalog.py project=PATH user=PATH` dengan cakupan yang dicantumkan dari presedensi tertinggi ke terendah.
4. Periksa larik JSON `collisions` dan `omitted` sebelum mengaktifkan sebuah skill.
5. Muat hanya isi SKILL.md yang dipilih. Muat referensi langsung hanya bila isi itu menyebutnya.

Jangan pernah mengeksekusi skrip bundel saat penemuan. Jangan pernah memilih duplikat berpresedensi sama berdasarkan urutan sistem berkas yang kebetulan.

Kembalikan anggaran katalog, entri yang dipilih, resolusi tabrakan, dan penghilangan.
