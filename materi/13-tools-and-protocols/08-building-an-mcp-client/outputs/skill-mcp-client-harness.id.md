---
name: mcp-client-harness
description: Buat kerangka klien MCP multi-server dengan metadata modern, negosiasi era yang aman, penggabungan deterministik, dan perutean.
version: 2.1.0
phase: 13
lesson: 08
tags: [mcp, client, stateless, compatibility, routing]
---

Dengan daftar transpor server MCP, hasilkan sebuah harness klien yang mengutamakan MCP `2026-07-28` dan mengisolasi kompatibilitas lama.

Hasilkan:

1. Konfigurasi rekanan (peer). Petakan nama server yang stabil ke perintah atau endpoint yang dikunci, argumen, daftar izin lingkungan, konteks otorisasi, jenis transpor, dan flag `allow_legacy` eksplisit yang bernilai false secara bawaan.
2. Pembangun permintaan modern. Cap versi protokol, kapabilitas klien terkini, dan identitas klien yang disarankan ke dalam setiap `params._meta` tepat sebelum serialisasi.
3. Probe era stdio. Kirim `server/discover` terlebih dahulu. Terima `DiscoverResult` yang valid, coba ulang `-32022` pada versi modern yang didukung kedua belah pihak, dan perlakukan `-32020` serta `-32021` sebagai kesalahan modern yang dapat diperbaiki.
4. Probe kompatibilitas lama. Perlakukan kesalahan yang tidak dikenal, waktu habis, koneksi tertutup, atau respons kosong sebagai ambigu. Kirim satu `initialize` dengan batas waktu hanya ketika rekanan itu memiliki `allow_legacy: true`. Pilih era lama hanya setelah keberhasilan JSON-RPC yang berkorelasi memuat revisi lama yang dikonfigurasi, kapabilitas objek, dan identitas server yang tidak kosong. Jika tidak, gagal dengan aman (*fail closed*).
5. Tembolok alat. Taatkan `ttlMs` dan `cacheScope` dalam konteks otorisasi yang disepakati. Perlakukan `resultType` lama yang tidak ada sebagai `"complete"`.
6. Penggabungan ruang nama. Urutkan rekanan dan alat. Beri awalan atau tolak tabrakan. Larang penimpaan senyap.
7. Perute. Petakan nama alat kanonik ke rekanan beserta nama lokalnya, buat id permintaan baru, kirim permintaan yang sesuai era, dan verifikasi id respons.
8. Pemulihan. Saat transpor terputus, gagalkan pekerjaan yang sedang berjalan, mulai ulang atau sambung ulang, ulangi penemuan dan daftar, buka kembali langganan, dan coba ulang hanya operasi yang diizinkan oleh kebijakan keamanan.

Penolakan keras:

- Mengirim permintaan modern tanpa `_meta` terkini.
- Beralih ke inisialisasi setelah kesalahan modern yang dikenali.
- Mengirim `initialize` ke rekanan yang tidak secara eksplisit masuk daftar izin kompatibilitas lama.
- Menganggap waktu habis, koneksi tertutup, respons kosong, kesalahan yang tidak dikenal, hasil yang malformed, atau revisi yang tidak didukung sebagai bukti perilaku lama.
- Menganggap proses, koneksi, atau `Mcp-Session-Id` sebagai keadaan protokol modern.
- Membagikan daftar tembolok privat lintas konteks otorisasi.
- Menimpa nama alat ganda secara senyap.
- Menerima keberhasilan modern tanpa `resultType`.

Aturan penolakan:

- Tolak menjalankan perintah di luar daftar izin yang dikunci.
- Tolak merutekan alat ketika pemiliknya ambigu.
- Tolak mencoba ulang panggilan non-idempoten secara otomatis tanpa kunci idempotensi aplikasi atau keputusan pengguna.

Hasilkan harness Python yang lengkap, minimal enam uji kepatuhan, dan laporan awal yang mencantumkan rekanan, era terpilih, versi terpilih, cakupan tembolok, dan nama alat kanonik.
