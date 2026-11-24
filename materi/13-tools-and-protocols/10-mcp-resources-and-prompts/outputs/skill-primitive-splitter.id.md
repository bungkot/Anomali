---
name: primitive-splitter
description: Tinjau desain server MCP dan pisahkan alat, sumber daya, prompt, tembolok, dan langganan memakai kontrak `2026-07-28`.
version: 2.0.0
phase: 13
lesson: 10
tags: [mcp, resources, prompts, subscriptions, caching]
---

Tinjau usulan server MCP dari sudut pandang konsumen.

Hasilkan:

1. Hasil `server/discover` yang mengiklankan revisi `2026-07-28` serta kapabilitas sumber daya dan prompt yang persis.
2. Tabel dengan kolom `name`, `chooser`, `primitive`, dan `reason`.
3. Skema URI sumber daya yang stabil dan templat sumber daya yang terbatas.
4. Nama prompt, deskripsi, serta argumen wajib atau opsional.
5. Aturan pengurutan deterministik untuk setiap metode daftar.
6. Kebijakan tembolok dengan `ttlMs` dan `cacheScope` untuk setiap hasil yang dapat ditembolokkan.
7. Filter `subscriptions/listen` untuk sumber daya atau perubahan daftar yang perlu pembaruan.
8. Satu contoh sumber daya tidak valid yang mengembalikan JSON-RPC `-32602`, plus satu contoh revisi yang tidak didukung yang mengembalikan `-32022` dengan `supported` dan `requested`.

Pakai aturan keputusan ini:

- Operasi yang dipilih model adalah alat.
- Konten yang dialamatkan URI dan dapat dibaca host adalah sumber daya.
- Alur kerja pesan yang dipilih pengguna adalah prompt.
- Aliran pembaruan dibuka klien melalui `subscriptions/listen`.
- Id permintaan listen menjadi `io.modelcontextprotocol/subscriptionId`.
- Pengakuan harus mendahului semua event pada langganan itu.
- Notifikasi tidak pernah melewati otorisasi untuk pembacaan berikutnya.
- `server/discover` wajib meskipun klien memilih memanggil metode lain terlebih dahulu.

Tolak desain ketika:

- Daftar berubah karena riwayat koneksi.
- Hasil privat ditempatkan di tembolok publik.
- URI sumber daya diterima tanpa penguraian, otorisasi, dan pemeriksaan batas.
- Desain memakai `resources/subscribe` atau memperlakukan langganan sebagai sesi protokol.
- Prompt diizinkan menggantikan instruksi host yang tepercaya.

Kembalikan tinjauan kontrak satu halaman. Akhiri dengan kesalahan primitif, tembolok, atau langganan berisiko tertinggi dan koreksi terkecilnya.
