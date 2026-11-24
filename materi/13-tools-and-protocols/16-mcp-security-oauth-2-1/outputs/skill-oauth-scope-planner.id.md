---
name: oauth-scope-planner
description: Rancang otorisasi MCP 2026-07-28 dengan CIMD, isolasi penerbit, indikator sumber daya, dan cakupan step-up.
version: 2.0.0
phase: 13
lesson: 16
tags: [mcp, oauth, cimd, pkce, issuer, resource-indicators]
---

Dengan diberikan server MCP HTTP jarak jauh serta daftar alatnya, rancang batas otorisasi lengkap.

## Masukan wajib

- URI sumber daya MCP kanonik dan lokasi metadata sumber daya terlindungi.
- Penerbit server otorisasi yang diizinkan.
- Runtime klien: native atau web, dengan URI redirect yang tepat.
- Pemetaan alat ke cakupan dan operasi berdampak.
- Batasan token, penyegaran, dan penyimpanan kredensial.
- Server otorisasi lawas yang tidak memiliki CIMD, jika ada.

## Hasilkan

1. Metadata sumber daya. Susun `resource`, `authorization_servers`, dan `scopes_supported` sesuai RFC 9728. Pertahankan jalur sumber daya setelah segmen well-known, misalnya `https://notes.example.com/.well-known/oauth-protected-resource/mcp` untuk `https://notes.example.com/mcp`.
2. Kebijakan penerbit. Nyatakan penerbit yang tepat diizinkan, validasi metadata, penanganan perubahan, dan perbandingan `iss` RFC 9207.
3. Pendaftaran. Gunakan pra-pendaftaran bila tersedia; jika tidak, utamakan Client ID Metadata Document. URL HTTPS beserta jalurnya adalah `client_id`; wajibkan URI redirect yang tepat dan perlakukan metadata tampilan sebagai tidak tepercaya. `application_type` bersifat opsional di sini.
4. Fallback DCR. Jika diperlukan, beri label tidak digunakan lagi, nyatakan `application_type`, dan tetapkan kondisi persis yang mengizinkan fallback. Jangan menurunkan ke jalur lain hanya karena kegagalan keamanan CIMD umum.
5. Kunci kredensial. Simpan kredensial pra-terdaftar dan DCR berdasarkan penerbit, serta token berdasarkan `(issuer, resource)`. Larang penggunaan ulang lintas penerbit. Nyatakan bahwa URL CIMD yang di-host sendiri bersifat portabel dan tidak memerlukan pendaftaran ulang DCR ketika penerbit tepercaya berubah.
6. Alur PKCE. Wajibkan S256, URI redirect yang tepat, validasi penerbit pada respons otorisasi, serta sumber daya yang sama dalam permintaan otorisasi dan token.
7. Model cakupan. Petakan setiap alat ke cakupan minimumya. Anggap tantangan cakupan `WWW-Authenticate` terkini sebagai otoritatif.
8. Pengalaman step-up. Identifikasi cakupan tambahan, penjelasan kepada pengguna, titik persetujuan, otorisasi baru, dan percobaan ulang dengan id permintaan MCP yang segar.
9. Pemeriksaan server sumber daya. Implementasikan `tools/list` yang diiklankan dengan skema berakar objek yang valid, urutan deterministik, jenis hasil, identitas server, dan petunjuk tembolok. Validasi penerbit, audiens, kedaluwarsa, cakupan, header MCP terkini, dan metadata permintaan sebelum mendispatching alat.
10. Kebersihan token. Hanya header bearer, tanpa token di query, tanpa penerusan token, penyimpanan penyegaran yang rahasia, dan rencana rotasi.
11. Kontrak galat. Pertahankan setiap id permintaan dalam amplop galat JSON-RPC, termasuk kegagalan OAuth. Wajibkan HTTP 400 `-32020` untuk ketidaksesuaian header sebelum pemeriksaan dukungan versi HTTP 400 `-32022`, data yang didukung dan diminta secara tepat, HTTP 404 `-32601` untuk metode yang tidak dikenal, serta 202 dengan badan kosong untuk notifikasi yang diterima.
12. Batas transpor. Beri label contoh badan yang telah diurai sebagai model protokol dalam proses dan lampirkan ke adapter Streamable HTTP lengkap pada Pelajaran 09 untuk validasi Content-Type JSON serta Accept JSON plus SSE.

## Penolakan keras

- DCR disajikan sebagai mekanisme pendaftaran baru yang diutamakan.
- DCR tanpa `application_type`.
- Menggunakan kembali kredensial pendaftaran yang diterbitkan penerbit, token akses, atau token penyegaran setelah penerbit berubah. URL CIMD yang di-host sendiri adalah pengecualian portabel, bukan rahasia yang diterbitkan penerbit.
- Menormalkan `iss` respons otorisasi sebelum perbandingan.
- Tidak ada PKCE S256 atau tidak ada `resource` dalam permintaan otorisasi dan token.
- Menerima token untuk audiens lain atau meneruskan token MCP ke hilir.
- Menggunakan `clientInfo`, `serverInfo`, kapabilitas, atau sesi protokol yang dihapus sebagai autentikasi.
- Menambahkan OAuth pada stdio lokal hanya untuk meniru HTTP jarak jauh.
- Menghilangkan jalur sumber daya terlindungi saat menyusun URL metadata RFC 9728.
- Mengembalikan objek teks biasa atau objek ad hoc untuk galat permintaan MCP alih-alih amplop JSON-RPC dengan id yang sama.

## Format keluaran

Kembalikan bagian-bagian bernama Sumber Daya, Penerbit, Pendaftaran, Penyimpanan Kredensial, Alur PKCE, Matriks Cakupan, Step-Up, Validasi Server, Kebersihan Token, dan Kompatibilitas. Akhiri dengan peristiwa tepat yang memaksa peninjauan penerbit dan, untuk klien yang diterbitkan penerbit, pendaftaran ulang.
