---
name: sampling-loop-designer
description: Migrasikan alat MCP berbantuan model ke inferensi langsung atau MRTR `2026-07-28` stateless dengan Sampling kompatibilitas yang terbatas.
version: 2.0.0
phase: 13
lesson: 11
tags: [mcp, mrtr, sampling, stateless, migration]
---

Rancang perilaku berbantuan model untuk server MCP yang menargetkan revisi protokol `2026-07-28`.

Mulailah dari satu keputusan: apakah server dapat berintegrasi langsung dengan penyedia model? Sampling tidak disarankan untuk desain baru. Utamakan integrasi langsung kecuali memakai model dan kredensial klien merupakan kebutuhan produk yang eksplisit.

Hasilkan:

1. Keputusan arsitektur. Pilih inferensi langsung atau Sampling kompatibilitas dan nyatakan alasannya.
2. Kontrak penemuan. Tampilkan `server/discover` dengan `supportedVersions` yang persis, kapabilitas yang diiklankan, `ttlMs`, dan `cacheScope`. Jika alat diiklankan, sertakan descriptor `tools/list` deterministik yang wajib dengan `inputSchema` objek valid, `resultType: "complete"`, metadata identitas server, dan petunjuk tembolok.
3. Amplop permintaan. Sertakan versi protokol dan kapabilitas klien di `_meta` pada setiap permintaan. Pakai `-32602` untuk versi yang hilang atau bukan string, `-32022` dengan data `supported` dan `requested` yang persis untuk versi yang tidak didukung, serta `-32021` dengan objek `requiredCapabilities` ketika Sampling tidak ada. Perlakukan metadata identitas klien hanya sebagai informasi. Jangan pernah mengeluarkan respons JSON-RPC untuk notifikasi tanpa id; notifikasi HTTP yang diterima mendapat `202` tanpa badan.
4. Tabel putaran. Untuk setiap putaran MRTR, sebutkan kunci `inputRequests`, metode permintaan tersemat, skema respons yang diharapkan, validasi, dan anggaran.
5. Kontrak percobaan ulang. Wajibkan metode dan argumen awal, id JSON-RPC baru, `inputResponses` putaran terkini, dan `requestState` yang persis byte demi byte.
6. Perlindungan keadaan. Ikat HMAC atau enkripsi terautentikasi pada prinsipal terautentikasi, metode, digest argumen, fase, dan masa kedaluwarsa singkat.
7. Kebijakan keamanan. Tentukan persetujuan, putaran maksimum, batas token dan byte, validasi respons, pencatatan, dan perilaku penolakan.
8. Rencana penghapusan. Jika Sampling tetap ada, sebutkan kondisi dan tanggal penggantiannya dengan integrasi langsung.

Penolakan keras:

- Desain baru yang mengadopsi Sampling yang tidak disarankan tanpa kebutuhan yang terdokumentasi.
- Server 2026-07-28 yang mengirim `sampling/createMessage` sebagai permintaan langsung server-ke-klien.
- Penggunaan `initialize`, `notifications/initialized`, `Mcp-Session-Id`, atau keadaan sesi protokol tersembunyi apa pun.
- `requestState` tanpa tanda tangan yang memengaruhi otorisasi, akses sumber daya, atau logika bisnis.
- Percobaan ulang yang memakai ulang id JSON-RPC awal atau mengubah argumen awal.
- Loop model klien tanpa pemeriksaan kapabilitas, kebijakan persetujuan, validasi, dan batas putaran keras.
- `includeContext: "allServers"` atau konteks lintas server implisit.

Aturan penolakan:

- Tolak panggilan model secara diam-diam atau desain apa pun yang menyembunyikan maksud server dari pengguna.
- Tolak keluaran model sebagai bukti identitas, otorisasi, atau persetujuan pengguna.
- Tolak desain multi-putaran ketika satu panggilan alat deterministik sudah cukup.
- Tolak menyebut metadata klien dan server sebagai identitas terautentikasi.

Hasilkan arsitektur satu halaman berisi keputusan, alur kabel, tabel putaran, isi keadaan bertanda tangan, anggaran keamanan, kasus kegagalan, dan rencana migrasi. Akhiri dengan putusan: `direct inference`, `temporary MRTR compatibility`, atau `no model required`.
