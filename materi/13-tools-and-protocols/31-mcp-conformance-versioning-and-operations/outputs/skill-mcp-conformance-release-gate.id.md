---
name: mcp-conformance-release-gate
description: Bangun matriks kepatuhan MCP serta keputusan promosi, penahanan, atau pemutaran balik yang didukung bukti.
version: 1.0.0
phase: 13
lesson: 31
tags: [mcp, conformance, versioning, transcripts, proxy, operations]
---

Dengan perubahan pada klien, server, gerbang, SDK, atau transpor MCP, hasilkan rangkaian kepatuhan tingkat kabel dan keputusan rilis dengan bukti yang telah diredact.

## Masukan wajib

- Versi protokol modern dan lawan yang didukung serta kebijakan untuk setiap target deployment.
- Titik tangkap permintaan dan respons mentah sebelum dekode SDK.
- Header HTTP cermin, body JSON-RPC, status, jenis konten, dan topologi perantara.
- Kapabilitas klien, server, dan ekstensi yang diiklankan.
- Nama SDK, versi, nilai ternormalisasi, dan galat.
- Ambang kesehatan, jendela canary, jumlah sampel minimum, dan pengukuran baseline.
- Versi pemutaran balik yang persis, digest bukti penerimaan, pin artefak dan deskriptor SHA-256, status Registry, kesehatan saat ini, penandatangan rilis tepercaya, serta attestation atas seluruh payload pemutaran balik.
- Kebijakan redaksi, retensi, dan akses bukti.

## Prosedur

1. Tentukan era protokol secara eksplisit. Wajibkan kunci `io.modelcontextprotocol/protocolVersion` dan `io.modelcontextprotocol/clientCapabilities` yang persis pada cabang modern. Tempatkan perilaku era inisialisasi hingga `2025-11-25` dalam adapter lawan terpisah.
2. Pilih kebijakan ketat atau fallback terbatas per target. `server/discover` yang berhasil atau galat modern yang dikenal membuktikan cabang modern. Waktu habis atau respons kosong tidak membuktikan apa pun. Izinkan probe lawan hanya untuk endpoint yang dikonfigurasi atau masuk daftar izin, lalu pilih lawan hanya setelah memvalidasi hasil `initialize` positif untuk revisi lawan yang dipasangi pin. Jangan pernah menurunkan versi setelah `-32020`, `-32021`, atau `-32022`.
3. Buat transkrip golden untuk permintaan yang diterima, hasil spesifik metode yang valid, hasil ekstensi yang diiklankan, perilaku lawan yang dipilih, dan invarian notifikasi tanpa respons.
4. Buat transkrip negatif untuk amplop malformed, ketidaksesuaian versi atau id respons, keeksklusifan hasil dan galat, galat malformed, pemetaan HTTP yang salah, metadata yang hilang, ketidaksesuaian header dan body, respons galat server yang hilang, versi yang tidak didukung, `resultType` modern yang hilang, payload metode malformed, jenis hasil yang tidak dikenal atau tidak diiklankan, serta respons notifikasi yang dilarang. Setiap kali validasi lokal melihat `HeaderMismatch`, wajibkan dan validasi secara struktural respons JSON-RPC `-32020` HTTP 400 yang sungguhan secara otomatis. Galat lokal saja tidak pernah membuat kasus itu lulus.
5. Validasi jenis metadata JSON-RPC terlebih dahulu. Cocokkan nama header HTTP tanpa memedulikan huruf besar-kecil, tolak duplikat yang bertentangan, dekode sentinel Base64 yang persis untuk nilai `Mcp-Name` yang tidak aman, lalu bandingkan `MCP-Protocol-Version`, `Mcp-Method`, dan nilai `Mcp-Name` yang berlaku dengan body sebelum memeriksa dukungan untuk nilai yang cocok. Perlakukan spasi di awal maupun di akhir sebagai tidak aman dan tolak bila dikirim mentah, meskipun nilainya sama dengan nilai body.
6. Terima hasil modern `complete` dan `input_required` yang dikenal. Terima diskriminator ekstensi hanya ketika kapabilitasnya diiklankan. Tolak setiap `resultType` yang tidak dikenal maupun tidak diiklankan. Lalu validasi payload metode, termasuk deskriptor alat lengkap untuk `tools/list`, kolom siklus hidup wajib untuk hasil task, serta objek completion bernilai string berbatas untuk `completion/complete`.
7. Pertahankan kolom hasil aditif dan `_meta` mentah dalam bukti. Putuskan secara eksplisit apakah setiap komponen boleh mengabaikannya atau wajib meneruskannya.
8. Jalankan setiap transkrip berisiko tinggi melalui tiap SDK yang disertakan. Bandingkan semantik kabel mentah dengan nilai kembalian ternormalisasi dan laporkan setiap kolom yang disintesiskan, diangkat, dibuang, atau diubah.
9. Jalankan rangkaian langsung dan melalui setiap perantara produksi. Tangkap bukti ingress, origin, dan egress yang telah diredact. Deteksi peruntuhan status, penulisan ulang body JSON-RPC, ketidaksesuaian header perutean, penyanggaan, dan perubahan negosiasi konten.
10. Terapkan redaksi sebelum serialisasi, hashing, logging, atau unggahan. Samakan huruf pada nama kolom dan header serta hapus pemisah agar varian camelCase, bertanda hubung, bergaris bawah, dan bertitik menggunakan daftar hitam yang sama, lalu hapus kredensial seperti `Authorization`, `Set-Cookie`, `X-Api-Key`, `accessToken`, `clientSecret`, dan `registrationAccessToken` ditambah argumen sensitif spesifik metode. Lakukan hash pada paket bukti yang telah diredact.
11. Evaluasi kandidat terhadap bukti transkrip deterministik, diferensial SDK, dan proksi yang tidak kosong, ditambah jendela kesehatan yang dideklarasikan sebelumnya dengan jumlah sampel minimum yang positif. Wajibkan digest bukti yang valid dari setiap sumber. Batas yang dihilangkan adalah gerbang yang gagal, bukan daftar kosong yang lulus.
12. Verifikasi target pemutaran balik yang persis, diterima, dipasangi pin, aktif, dan sehat sebelum promosi. Validasi jenis kolom dan digest SHA-256 yang persis, lalu verifikasi secara kriptografis attestation-nya terhadap identitas pengontrol rilis tepercaya. Promosikan hanya ketika bukti kepatuhan, SDK, proksi, kesehatan, dan kesiapan pemutaran balik lulus. Pada kegagalan kandidat, putar balik hanya ke target terverifikasi itu. Jika tidak, tahan.

## Matriks wajib

Untuk setiap kasus, laporkan:

- nama kasus stabil dan invarian normatif
- era protokol dan bukti pemilihan
- versi klien, server, SDK, proksi, dan build
- status, bentuk respons, jenis hasil, atau kode galat yang diharapkan
- digest bukti ingress, origin, dan egress yang teramati
- perbedaan normalisasi SDK
- lulus atau gagal pada setiap batas
- versi kebijakan redaksi
- kode alasan akhir

Sertakan setidaknya kasus-kasus ini:

- discovery atau panggilan metode modern golden
- hasil `complete` golden yang dikenal dengan kolom aditif
- `resultType` lawan golden yang hilang dalam era lawan yang dipilih
- hasil ekstensi golden yang diiklankan
- hasil `completion/complete` golden yang valid
- notifikasi golden tanpa respons JSON-RPC
- ketidaksesuaian header dan body negatif
- kapabilitas hilang negatif
- versi cocok yang tidak didukung negatif
- `resultType` modern hilang negatif
- `resultType` tidak dikenal atau tidak diiklankan negatif
- transformasi status atau body oleh proksi negatif
- kehilangan kolom semantik SDK negatif
- hasil `completion/complete` malformed negatif

## Penolakan keras

- Mengklaim kepatuhan dari nilai kembalian SDK tanpa bukti kabel mentah.
- Satu parser yang diam-diam menerima bentuk modern maupun lawan.
- Fallback lawan setelah galat modern yang dikenal.
- Fallback lawan hanya berdasarkan waktu habis, keheningan, penutupan koneksi, atau respons yang tidak dikenal.
- Meloloskan kasus permintaan negatif tanpa menangkap dan memvalidasi respons galat server.
- Menjadikan bukti JSON-RPC `-32020` HTTP 400 opsional setelah deteksi `HeaderMismatch` lokal.
- Menyimpulkan lengkap untuk `resultType` yang hilang sebelum memilih era lawan.
- Menganggap diskriminator yang tidak dikenal sebagai lengkap.
- Menolak semua kolom asing aditif tanpa alasan kolom yang dicadangkan.
- Mengirim respons JSON-RPC apa pun ke sebuah notifikasi.
- Menerima `resultType` yang dikenal tanpa memvalidasi payload spesifik metode.
- Mengotorisasi header cermin tanpa memeriksa kesetaraan dengan body.
- Memperlakukan nama kolom HTTP sebagai peka huruf besar-kecil atau membandingkan `Mcp-Name` tersandikan tanpa dekode sentinel yang persis.
- Menerima `Mcp-Name` mentah dengan spasi di awal atau di akhir alih-alih mensyaratkan penyandian sentinel.
- Menerima hasil completion tanpa objek completion bernilai string berbatas yang valid.
- Menganggap 500 yang dihasilkan proksi setara dengan galat protokol origin.
- Menulis token bearer, cookie, rahasia, atau argumen sensitif ke dalam bukti.
- Menggunakan normalisasi redaksi yang membiarkan varian camelCase atau pemisah melewati daftar hitam kredensial kanonik.
- Menyatakan canary tanpa sampel sebagai sehat.
- Menganggap bukti transkrip, SDK, atau proksi yang kosong sebagai batas yang lulus.
- Menganggap string truthy atau kamus pemutaran balik yang tidak terautentikasi sebagai bukti terverifikasi.
- Memutar balik ke versi tanpa bukti penerimaan, pin, status, dan kesehatan yang persis.
- Mempromosikan kandidat produksi sebelum membuktikan target pemutaran balik yang sehat.

## Hasil

Kembalikan bagian-bagian berikut:

1. Kebijakan Era: bukti modern, target ketat, pemicu fallback terbatas, dan sinyal penurunan versi yang dilarang.
2. Matriks Transkrip: kasus golden dan negatif dengan hasil kabel yang diharapkan.
3. Kompatibilitas Hasil: diskriminator inti, ekstensi yang diiklankan, kebijakan kolom aditif, dan batas inferensi lawan.
4. Diferensial SDK: digest mentah dan ternormalisasi serta kolom yang diangkat, dibuang, disintesiskan, dan diubah.
5. Bukti Proksi: hasil ingress, origin, dan egress dengan lompatan yang gagal secara persis.
6. Laporan Redaksi: versi kebijakan, kelas kolom yang dihapus, dan digest bukti yang diredact.
7. Jendela Kesehatan: sampel, laju galat, latensi, saturasi, durasi, ambang, dan perbandingan baseline.
8. Bukti Pemutaran Balik: target persis, digest penerimaan, pin, status Registry, kesehatan, identitas penandatangan, attestation terverifikasi, dan rencana pemulihan rute.
9. Keputusan: `promote`, `hold`, atau `rollback`, dengan kode alasan stabil dan digest bukti lengkap.

Akhiri dengan batas pertama yang gagal. Jika semua batas lulus, nyatakan kondisi penyelesaian canary yang mengizinkan promosi penuh.
