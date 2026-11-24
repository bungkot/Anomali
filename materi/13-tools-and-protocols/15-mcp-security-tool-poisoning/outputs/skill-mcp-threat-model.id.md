---
name: mcp-threat-model
description: Buat model ancaman untuk deployment MCP 2026-07-28 melintasi metadata, routing, otorisasi, MRTR, dan batas kompatibilitas.
version: 2.0.0
phase: 13
lesson: 15
tags: [mcp, security, stateless, tool-poisoning, mrtr]
---

Dengan diberikan sebuah deployment MCP, hasilkan model ancaman berbasis bukti. Anggap server, paket, tembolok, entri registri, atau rute gerbang mana pun dapat dikompromi.

## Masukan wajib

- Batas kepercayaan klien, gerbang, server, server otorisasi, dan registri.
- Deskriptor alat lengkap yang ternormalisasi serta intisari yang disetujui.
- Prinsipal terautentikasi, penerbit, audiens, cakupan, dan kebijakan alat.
- Revisi protokol terkini dan lawas yang diterima.
- Operasi MRTR, skema masukan, perlindungan keadaan, dan kebijakan replay.
- Cakupan tembolok, TTL, rute langganan, dan retensi audit.

## Hasilkan

1. Validasi kawat. Verifikasi versi dan kapabilitas per permintaan, lalu kesetaraan header routing sebelum dukungan versi. Wajibkan HTTP 400 `-32020` untuk ketidaksesuaian, HTTP 400 `-32022` dengan data yang didukung dan diminta secara tepat untuk versi yang cocok tetapi tidak didukung, HTTP 404 `-32601` untuk metode yang tidak dikenal, serta 202 dengan badan kosong untuk notifikasi yang diterima.
2. Tinjauan deskriptor. Laporkan indikator peracunan, perubahan intisari deskriptor lengkap, alat yang tidak dikenal, serta perubahan skema atau anotasi.
3. Peta namespace. Berikan satu nama publik berkualifikasi untuk setiap alat backend dan tolak resolusi tabrakan secara diam-diam.
4. Matriks otorisasi. Petakan prinsipal terautentikasi dan penerbit ke sumber daya, alat, batasan argumen, dan cakupan. Jangan gunakan `clientInfo` atau `serverInfo` sebagai identitas.
5. Tinjauan MRTR. Pastikan setiap entri `inputRequests` adalah permintaan tersemat lengkap yang didukung oleh kapabilitas yang dinyatakan klien. Anggap `elicitation: {}` sebagai dukungan formulir implisit dan `elicitation: {form: {}}` sebagai dukungan formulir eksplisit. Tolak elicitation hanya-URL dengan HTTP 400 `-32021` dan `data.requiredCapabilities.elicitation.form`. Ikat `requestState` yang dilindungi ke metode, alat, argumen yang tepat, prinsipal, tujuan, kedaluwarsa, dan nonce. Cocokkan dan validasi setiap entri `inputResponses` berdasarkan kunci sebelum mengonsumsi nonce secara atomik dalam penyimpanan replay terbatas yang dipangkas berdasarkan TTL dan bersama untuk setiap instance handler.
6. Tinjauan sumbu risiko. Tandai setiap langkah otomatis yang menggabungkan masukan tidak tepercaya, data sensitif, dan aksi berdampak.
7. Tinjauan tembolok dan langganan. Pastikan hasil yang bergantung pada pengguna bersifat privat dan notifikasi berumur panjang menggunakan `subscriptions/listen`.
8. Batas kompatibilitas. Isolasi jabat tangan, sesi, aliran GET, callback server, atau perilaku task eksperimental lawas di balik gating versi eksplisit.
9. Batas transpor. Identifikasi apakah implementasi merupakan adapter HTTP lengkap atau model protokol dalam proses. Hubungkan model ke Pelajaran 09 untuk validasi Content-Type JSON serta Accept JSON plus SSE.
10. Urutan perbaikan. Berikan tiga perbaikan dengan daya ungkit tertinggi beserta penanggung jawab dan bukti penerimaan.

## Penolakan keras

- Timpaan alat atau pemilihan rute secara diam-diam berdasarkan urutan penemuan.
- Memperbarui intisari deskriptor tanpa persetujuan ulang oleh manusia atau kebijakan.
- Menganggap informasi klien atau server yang dilaporkan sendiri sebagai autentikasi.
- Menganggap kapabilitas yang dinyatakan sebagai izin.
- Memercayai `requestState` plaintext atau tanpa tanda tangan untuk aksi berdampak.
- Menyimpan satu-satunya buku besar replay di dalam satu instance gerbang atau server.
- Membatasi laju atau status persetujuan hanya berdasarkan `Mcp-Session-Id`.
- Menyajikan Sampling, Roots, Logging, atau HTTP plus SSE lawas yang tidak digunakan lagi sebagai jalur implementasi baru.

## Format keluaran

Kembalikan bagian-bagian bernama Batas Kepercayaan, Temuan Kawat, Temuan Deskriptor, Peta Rute, Matriks Otorisasi, Temuan MRTR, Temuan Kompatibilitas, dan Perbaikan. Pisahkan bukti yang terkonfirmasi dari asumsi. Akhiri dengan satu jalur serangan yang saat ini melintasi paling banyak batas.
