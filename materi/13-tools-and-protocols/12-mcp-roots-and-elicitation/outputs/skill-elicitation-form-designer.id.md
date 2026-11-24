---
name: elicitation-form-designer
description: Rancang cakupan sumber daya eksplisit dan elicitation MCP 2026-07-28 tanpa status dengan otorisasi, formulir aman, dan state percobaan ulang bertanda tangan.
version: 2.0.0
phase: 13
lesson: 12
tags: [mcp, elicitation, mrtr, scope, authorization]
---

Rancang langkah masukan pengguna untuk operasi MCP yang menargetkan revisi protokol `2026-07-28`.

Hasilkan:

1. Kontrak cakupan. Letakkan URI ruang kerja, direktori, atau sumber daya pada argumen alat yang terlihat atau konfigurasi server. Nyatakan prinsipal terautentikasi mana yang boleh menggunakannya.
2. Pemeriksaan batas. Tentukan normalisasi URI, pembatasan komponen jalur, kebijakan symbolic link, dan sandbox sistem operasi.
3. Kondisi pemicu. Sebutkan secara tepat ambiguitas, konfirmasi, atau interaksi eksternal yang memerlukan masukan pengguna.
4. Discovery dan gerbang kapabilitas. Kembalikan `supportedVersions`, kapabilitas, `ttlMs`, dan `cacheScope` yang tepat dari `server/discover`. Jika alat diiklankan, sertakan deskriptor `tools/list` deterministik yang wajib dengan `inputSchema` objek valid, metadata identitas server, dan petunjuk cache. Perlakukan `elicitation: {}` dan `elicitation.form` eksplisit sebagai dukungan formulir. Tolak dukungan yang hilang atau khusus URL dengan `-32021` dan `data.requiredCapabilities.elicitation.form`; gunakan `-32022` dengan data `supported` dan `requested` yang tepat untuk versi yang tidak didukung.
5. Hasil MRTR. Kembalikan `resultType: "input_required"` dengan kunci `inputRequests` yang stabil dan permintaan `elicitation/create`.
6. Desain interaksi. Untuk mode formulir, berikan pesan sederhana dan skema datar terbatas. Untuk mode URL, tampilkan tujuan HTTPS dan aturan penyelesaian di luar jalur.
7. Kontrak percobaan ulang. Wajibkan id JSON-RPC baru, metode dan argumen asli, `inputResponses` terkini, `_meta` per permintaan, dan gaungan `requestState` yang tepat.
   Notifikasi tanpa id tidak pernah menerima hasil maupun error JSON-RPC; notifikasi Streamable HTTP yang diterima mendapat `202` tanpa badan.
8. Penanganan cabang. Petakan `accept`, `decline`, dan `cancel` ke hasil aman yang berbeda.
9. Perlindungan state. Ikat HMAC atau enkripsi terautentikasi ke prinsipal terautentikasi, digest argumen asli, himpunan kandidat, fase operasi, masa berlaku, dan nonce sekali pakai. Konsumsi nonce secara atomik dalam penyimpanan replay terbatas yang dipangkas berdasarkan TTL dan dibagikan oleh setiap instance handler.
10. Validasi ulang akhir. Periksa ulang otorisasi, state rekaman aktual, dan pembatasan tepat sebelum mutasi.

Penolakan tegas:

- Memperlakukan Roots yang usang sebagai otorisasi, pembatasan, atau sandboxing.
- Menggunakan `roots/list` atau `notifications/roots/list_changed` dalam desain 2026-07-28 baru.
- Mengirim permintaan `elicitation/create` balik alih-alih mengembalikannya melalui MRTR.
- Mengumpulkan kata sandi, kunci API, token akses, atau kredensial pembayaran dalam mode formulir.
- Mengirim mode elicitation yang tidak ada dalam kapabilitas per permintaan saat ini.
- Memperlakukan `clientInfo` sebagai identitas pengguna terautentikasi.
- Melakukan tindakan destruktif sebelum penerimaan tervalidasi dan pemeriksaan otorisasi akhir.
- `requestState` tanpa tanda tangan yang membawa kandidat atau data yang relevan dengan izin.

Aturan penolakan:

- Tolak prompt berulang setelah penolakan eksplisit.
- Tolak elicitation untuk nilai yang dapat diturunkan atau divalidasi server tanpa pengguna.
- Tolak URL yang berisi kredensial, rahasia pengguna, atau nilai bearer terautentikasi sebelumnya.
- Tolak permintaan yang menggunakan state sesi protokol tersembunyi, `initialize`, atau `Mcp-Session-Id`.

Hasilkan desain satu halaman dengan cakupan, otorisasi, pembatasan, mode interaksi, skema atau URL, bentuk kabel MRTR, field state, cabang respons, kebijakan replay, dan daftar periksa validasi ulang akhir.
