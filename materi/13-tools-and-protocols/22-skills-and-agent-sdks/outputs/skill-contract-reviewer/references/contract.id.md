# Daftar periksa kontrak portabel

- Bundel adalah direktori yang berisi berkas `SKILL.md` biasa.
- Frontmatter dimulai pada baris pertama dan memiliki penutup delimiter.
- `name` ada, panjangnya tidak lebih dari 64 karakter, dan memakai huruf kecil, digit, serta tanda hubung tunggal.
- `name` cocok dengan direktori bundel.
- `description` ada, panjangnya tidak lebih dari 1024 karakter, dan menyatakan kapan skill berguna.
- `compatibility` opsional berisi 1 sampai 500 karakter bila ada.
- `metadata` opsional memetakan kunci string ke nilai string.
- `allowed-tools` eksperimental opsional adalah string tidak kosong yang dipisah spasi, dengan perilaku yang diverifikasi pada host target.
- Kolom runtime yang tidak dikenal dipisahkan dari kontrak paket portabel dan ditangani oleh adapter eksplisit.
- Isi Markdown memuat prosedur.
- Metadata portabel opsional dan ekstensi spesifik host tetap dapat dibedakan.

Lulus daftar periksa ini membuat paket dapat dimuat secara struktural. Ini tidak memberikan wewenang atas sistem berkas, jaringan, rahasia, subproses, maupun alat.
