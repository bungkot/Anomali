---
name: mcp-apps-spec
description: Rancang dan tinjau kontrak MCP App pada protokol tanpa keadaan 2026-07-28.
version: 2.0.0
phase: 13
lesson: 14
tags: [mcp, apps, stateless, ui-resources, csp, sandbox]
---

Dengan diberikan sebuah alat MCP yang mungkin memerlukan tampilan interaktif, hasilkan kontrak yang netral terhadap kerangka kerja.

## Masukan wajib

- Nama alat, argumen, hasil teks biasa, dan hasil terstruktur.
- Interaksi pengguna yang harus didukung tampilan.
- Sensitivitas data dan apakah respons berbeda berdasarkan konteks otorisasi.
- Izin peramban dan origin eksternal yang dibutuhkan tampilan.
- Perilaku teks-saja untuk host tanpa dukungan Apps.

## Hasilkan

1. Amplop inti terkini. Tampilkan `2026-07-28`, `protocolVersion` per permintaan, `clientCapabilities`, `clientInfo` yang disarankan, header `Mcp-Method` dan `Mcp-Name` yang cocok, serta respons `resultType`.
2. Entri penemuan. Iklankan `io.modelcontextprotocol/ui` di `server/discover`, dengan `ttlMs` dan `cacheScope` yang konservatif.
3. Deklarasi alat. Letakkan `_meta.ui.resourceUri` bersarang pada alat yang dikembalikan oleh `tools/list`. Jangan menunggu `tools/call` untuk menyingkap UI.
4. Kontrak sumber daya. Sertakan metadata `resources/list` deterministik sebelum `resources/read`. Berikan satu URI `ui://` kanonik, nama dan deskripsi yang stabil, `text/html;profile=mcp-app`, petunjuk tembolok, daftar domain CSP (`connectDomains`, `resourceDomains`, `frameDomains`, `baseUriDomains`), dan objek izin minimum.
5. Kontrak hasil. Kembalikan teks dan data terstruktur yang berguna, baik host merender App maupun tidak.
6. Kontrak jembatan. Cantumkan setiap metode Apps `ui/*` atau terproksi, origin pesan yang tepat, skema argumen, skema hasil, dan pemeriksaan persetujuan di sisi host.
7. Fallback. Jelaskan alat dan hasil ketika klien menghilangkan kapabilitas ekstensi Apps.
8. Tabel verifikasi. Cakup HTTP 400 `-32020` untuk ketidaksesuaian header sebelum routing, HTTP 400 `-32022` dengan data versi yang didukung dan diminta secara tepat, HTTP 400 `-32021` dengan `data.requiredCapabilities`, HTTP 404 `-32601`, notifikasi 202 dengan badan kosong, pelanggaran CSP, konten tidak tepercaya, panggilan jembatan tanpa izin, dan fallback teks.
9. Batas transpor. Jika implementasi menerima permintaan dan header yang telah diurai, sebut sebagai model protokol dalam proses dan hubungkan dengan adapter Streamable HTTP lengkap pada Pelajaran 09. Adapter nyata harus mewajibkan Content-Type JSON dan nilai Accept yang memuat JSON plus SSE.

## Penolakan keras

- Jalur `initialize`, `notifications/initialized`, atau `Mcp-Session-Id` inti yang disajikan sebagai MCP terkini.
- Target origin `postMessage` berupa wildcard atau penerima yang melewati validasi `event.origin`.
- Binding UI yang hanya diungkap setelah alat dijalankan.
- Daftar domain CSP wildcard, origin jaringan tanpa batas, atau izin tanpa fitur yang terlihat.
- HTML yang dikendalikan pengguna disisipkan tanpa batas sanitasi yang ditetapkan.
- Aksi UI berdampak yang menganggap klik iframe sebagai otorisasi host.
- Server yang mengiklankan sumber daya tetapi menghilangkan `resources/list`.
- Badan respons JSON-RPC apa pun untuk notifikasi tanpa `id`.

## Batas kompatibilitas

Metadata UI datar lawas boleh dibaca sebagai fallback, tetapi keluaran baru menggunakan `_meta.ui.resourceUri` bersarang. `ui/initialize` hanya diizinkan bila diidentifikasi sebagai jabat tangan postMessage Apps. Metode ini tidak pernah menggantikan inisialisasi inti MCP yang telah dihapus.

## Format keluaran

Kembalikan desain ringkas dengan judul-judul berikut: Kawat Inti, Penemuan, Alat, Sumber Daya, Hasil, Jembatan, Keamanan, Fallback, Verifikasi. Akhiri dengan satu asumsi origin, izin, atau persetujuan yang paling berisiko.
