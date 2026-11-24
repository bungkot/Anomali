# Model ancaman

Tinjau batas-batas ini secara independen:

- Otoritas: instruksi tidak dapat menulis ulang izin host.
- Sistem berkas: selesaikan target dan jaga agar tetap berada di dalam akar ruang kerja; tolak pelarian melalui symlink.
- Perintah: terima array `argv`, tolak meta-karakter shell dan executable destruktif, serta wajibkan daftar izin (allowlist) executable.
- Jaringan: wajibkan HTTPS dan daftar izin asal (origin) yang persis. Normalisasi port efektif, sehingga `https://api.example.test` dan `https://api.example.test:443` cocok, sementara port `8443` memerlukan entri tersendiri. Jangan menerima kredensial di userinfo URL.
- Konten eksternal: perlakukan teks yang diambil sebagai data, sama sekali bukan sebagai kebijakan atau persetujuan.
- Rahasia: deteksi payload yang kemungkinan memuat rahasia tanpa mencatat nilainya.
- Tindakan destruktif: tolak atau wajibkan persetujuan manusia yang tercatat sesuai kebijakan host.

Verdict `allow` hanya berarti bahwa permintaan simulasi memenuhi kebijakan yang diberikan. Bundel ini tidak menjalankan tindakan apa pun.
