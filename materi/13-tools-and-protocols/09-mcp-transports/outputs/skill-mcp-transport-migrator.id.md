---
name: mcp-transport-migrator
description: Migrasikan transpor HTTP MCP lama ke kontrak `2026-07-28` yang stateless dan khusus POST.
version: 2.0.0
phase: 13
lesson: 09
tags: [mcp, streamable-http, stateless, migration, headers]
---

Dengan server Streamable HTTP berbasis sesi atau HTTP+SSE, hasilkan buku panduan migrasi untuk MCP `2026-07-28`.

Hasilkan:

1. Peta endpoint. Tentukan satu endpoint MCP modern yang menerima POST. Setiap permintaan atau notifikasi JSON-RPC mendapat POST baru.
2. Peta respons. Gunakan `application/json` untuk satu respons atau `text/event-stream` cakupan permintaan untuk notifikasi terkait yang diikuti respons akhir.
3. Perilaku yang dihapus. Kembalikan `405` untuk GET dan DELETE modern. Abaikan `Mcp-Session-Id` dan `Last-Event-ID`; jangan pernah menerbitkan, menggaungkan, mencabut, atau melanjutkannya.
4. Metadata permintaan. Wajibkan versi protokol dan kapabilitas klien di setiap `_meta` badan, disertai identitas klien yang disarankan.
5. Validasi header. Wajibkan `MCP-Protocol-Version`, `Mcp-Method`, dan `Mcp-Name` bersyarat. Dekode penanda Base64 dan bandingkan header dengan badan. Kembalikan `-32020` saat tidak cocok. Kembalikan `-32022` untuk versi yang cocok tetapi tidak didukung dengan kunci data persis `supported` dan `requested`.
6. Migrasi langganan. Ganti GET mandiri, `resources/subscribe`, dan `resources/unsubscribe` dengan POST `subscriptions/listen`. Tandai pengakuan, setiap notifikasi, dan hasil akhir dengan `io.modelcontextprotocol/subscriptionId` yang sama dengan id permintaan listen.
7. Migrasi keadaan. Ganti afinitas koneksi dengan pegangan aplikasi yang eksplisit dan opak, terikat pada prinsipal terautentikasi.
8. Jendela kompatibilitas. Jaga endpoint lama tetap terpisah dan diberi label jelas. Kesalahan POST modern harus diperiksa sebelum tindakan lama apa pun. Jangan alihkan POST dengan `301` atau `302` karena pelestarian metode dan badan tidak aman.
9. Verifikasi. Uji penolakan Origin, negosiasi media POST, metadata badan, header yang dicerminkan, respons JSON, notifikasi `202` tanpa badan, metadata langganan SSE cakupan, GET dan DELETE `405`, header yang dihapus yang diabaikan, serta coba ulang aliran putus dengan id baru.

Penolakan keras:

- Menyajikan id sesi, GET mandiri, DELETE, atau replay sebagai perilaku modern.
- Membagikan kapabilitas per permintaan melalui memori proses atau koneksi.
- Mengirim permintaan JSON-RPC yang dimulai server.
- Melanjutkan aliran SSE modern dengan `Last-Event-ID`.
- Beralih ke era lama setelah kesalahan modern yang dikenali.
- Memakai pengalihan untuk memindahkan POST JSON-RPC selama migrasi.

Aturan penolakan:

- Tolak paparan publik tanpa autentikasi, otorisasi, dan kebijakan Origin yang persis.
- Tolak perutean sticky tersembunyi sebagai pengganti keadaan alur kerja eksplisit.
- Tolak percobaan ulang otomatis operasi non-idempoten tanpa kontrol idempotensi aplikasi.

Hasilkan tabel endpoint sebelum-dan-sesudah, peluncuran bertahap, batas pembatalan (*rollback*), dan daftar periksa kepatuhan yang dapat dijalankan. Akhiri dengan tanggal pasti saat rute lama akan dihapus.
