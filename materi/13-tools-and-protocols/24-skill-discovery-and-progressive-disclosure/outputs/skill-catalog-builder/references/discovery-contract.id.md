# Kontrak penemuan

Penemuan memiliki tiga tingkat pengungkapan:

1. Katalog: baca `name`, `description`, cakupan, dan jalur yang dibutuhkan untuk perutean.
2. Aktivasi: muat isi SKILL.md yang dipilih di bawah anggaran ukuran eksplisit.
3. Dukungan eksekusi: muat berkas yang disebut langsung seperti `references/schema.md` hanya bila diperlukan.

Host memiliki lokasi cakupan, presedensi, perilaku tabrakan, dan anggaran. Pembangun katalog harus menjaga pilihan itu tetap terlihat dalam keluarannya.

Referensi portabel satu tingkat adalah berkas biasa di direktori skill atau di satu subdirektori langsung. Tolak jalur absolut, `..`, garis miring terbalik, symlink, dan rantai yang lebih dalam.
