---
name: mcp-server-scaffolder
description: Rancang server MCP 2026-07-28 tanpa keadaan dengan penemuan, validasi permintaan, dan primitif deterministik.
version: 2.0.0
phase: 13
lesson: 07
tags: [mcp, server, stateless, discovery, scaffold]
---

Dengan sebuah domain tertentu, hasilkan rencana server MCP modern. Jaga keadaan aplikasi tetap eksplisit dan jaga perilaku protokol tetap tanpa keadaan.

Hasilkan:

1. Pemisahan primitif. Tentukan alat atomik, sumber daya (resource) beralamat URI, dan prompt yang berguna. Hilangkan sebuah primitif bila domain tidak memiliki penggunaan yang jujur untuknya.
2. Hasil penemuan. Sertakan `supportedVersions`, kemampuan server, instruksi opsional, `resultType: "complete"`, petunjuk penembolokan, dan identitas server di `_meta` hasil.
3. Pemvalidasi permintaan. Wajibkan versi protokol dan kemampuan klien pada setiap `params._meta`. Validasi identitas klien yang disarankan bila tersedia. Kembalikan `-32022` beserta versi yang diminta dan versi yang didukung saat terjadi ketidaksesuaian.
4. Pembungkus hasil. Tambahkan `resultType: "complete"` dan identitas server pada setiap keberhasilan. Tambahkan `ttlMs` dan `cacheScope` pada penemuan, daftar, templat, dan pembacaan sumber daya.
5. Kebijakan pengurutan. Tentukan kunci sort yang stabil untuk setiap respons daftar.
6. Kebijakan keadaan. Simpan keadaan persisten dalam basis data atau kembalikan pegangan (handle) yang eksplisit dan buram sebagai argumen alat biasa. Jangan pernah menyembunyikan keadaan dalam sesi protokol.
7. Batas kompatibilitas. Bila dukungan lama diperlukan, isolasi adaptor `initialize` `2025-11-25`. Pilih adaptor itu hanya untuk lalu lintas lama dan uji kedua era secara independen.

Penolakan tegas:

- Server modern yang metode valid pertamanya harus `initialize`.
- Menggunakan kembali kemampuan, identitas, atau versi dari permintaan sebelumnya.
- Mengembalikan `Mcp-Session-Id` pada lalu lintas HTTP modern.
- Mengembalikan hasil daftar atau pembacaan sumber daya tanpa petunjuk penembolokan.
- Memperlakukan anotasi sebagai kontrol otorisasi.
- Mengirim permintaan JSON-RPC independen dari server.

Aturan penolakan:

- Bila sumber daya yang diminta akan menyingkap rahasia tanpa otorisasi, hentikan dan minta kebijakan akses.
- Bila domain tidak memiliki data hanya-baca, hilangkan sumber daya daripada mengarangnya.
- Bila domain tidak memiliki templat yang dapat digunakan ulang, hilangkan prompt daripada mengirimkan pengisi.

Keluarkan arsitektur satu halaman, tabel metode, pseudokode validasi, contoh hasil, aturan pengurutan deterministik, dan sedikitnya enam uji kepatuhan (conformance). Akhiri dengan batas antara keadaan aplikasi dan keadaan protokol.
