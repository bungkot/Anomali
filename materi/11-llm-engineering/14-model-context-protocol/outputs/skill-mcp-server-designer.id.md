---
name: mcp-server-designer
description: Rancang server MCP 2026-07-28 stateless dengan penemuan, keadaan, transpor, dan kontrak keamanan yang eksplisit.
version: 2.0.0
phase: 11
lesson: 14
tags: [llm-engineering, mcp, stateless, tool-use]
---

Diberikan sebuah domain (API internal, basis data, sumber berkas) dan host yang akan memasang server, keluarkan:

1. Peta primitif. Kemampuan mana yang menjadi `tools` (tindakan), mana yang menjadi `resources` (data baca-saja), mana yang menjadi `prompts` (templat yang dipanggil pengguna). Satu baris per primitif.
2. Kontrak penemuan. Rancang `server/discover` dengan versi yang tepat yang didukung implementasi, kapabilitas, identitas server, instruksi, `ttlMs`, dan `cacheScope`.
3. Kontrak permintaan. Wajibkan string versi protokol dan objek kapabilitas klien dalam `params._meta` pada setiap permintaan. Rekomendasikan identitas klien. Kembalikan Invalid Params (`-32602`) untuk metadata wajib yang hilang atau bertipe salah. Kembalikan `UnsupportedProtocolVersionError` (`-32022`) dengan `data.supported` dan `data.requested` hanya untuk string versi yang diberikan yang tidak diterapkan server.
4. Kontrak hasil. Tambahkan `resultType`, metadata identitas server, pengurutan daftar deterministik, dan kebijakan penembolokan pada setiap hasil yang relevan.
5. Rencana MRTR. Gunakan `input_required` hanya untuk `tools/call`, `resources/read`, atau `prompts/get`. Sertakan setidaknya salah satu dari `inputRequests` atau `requestState` opak; coba ulang metode asli dengan ID JSON-RPC baru, respons masukan yang sesuai saat diminta, dan nilai keadaan persis saat ada.
6. Rencana keadaan. Untuk setiap alur kerja multi-panggilan, definisikan gagangan opak yang dicetak server dan diteruskan sebagai argumen alat biasa. Jangan sembunyikan keadaan di balik koneksi atau sesi protokol.
7. Rencana transpor dan autentikasi. Pilih stdio atau endpoint POST Streamable HTTP 2026-07-28. Untuk HTTP, definisikan validasi Origin dan otorisasi per permintaan. Wajibkan `MCP-Protocol-Version` pada permintaan POST, `Mcp-Method` pada permintaan JSON-RPC, dan `Mcp-Name` hanya untuk `tools/call`, `resources/read`, dan `prompts/get`. POST notifikasi yang diterima mengembalikan HTTP 202 tanpa isi.
8. Draf skema. Tulis JSON Schema untuk setiap parameter alat, dengan deskripsi yang disetel untuk pemilihan model dan batas eksplisit untuk masukan yang tidak tepercaya.
9. Daftar tindakan destruktif. Tandai setiap alat yang mengubah keadaan dengan `destructiveHint: true` dan wajibkan persetujuan manusia.
10. Rencana verifikasi. Cakup notifikasi yang tidak menghasilkan respons JSON-RPC, amplop dan ID permintaan yang malformed, penolakan metadata, penemuan, daftar deterministik, ketidaksesuaian versi, kolom tembolok, ketidaksesuaian header-isi, otorisasi, persetujuan, dan satu kasus injeksi prompt.

Tolak rancangan yang menggunakan `initialize`, `notifications/initialized`, `Mcp-Session-Id`, HTTP GET mandiri, HTTP DELETE, atau `Last-Event-ID` sebagai jalur modernya. Izinkan mekanisme itu hanya di dalam adapter yang terisolasi jelas untuk versi protokol hingga 2025-11-25. Jangan tambahkan Roots, Sampling, atau Logging yang sudah usang ke implementasi baru; dukungan kompatibilitas harus diberi label dan masukan Roots atau Sampling harus menggunakan MRTR. Tolak server yang menulis ke disk atau memanggil API eksternal tanpa otorisasi, validasi, dan jalur persetujuan.
