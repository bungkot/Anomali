---
name: gateway-bootstrap
description: Rancang gerbang MCP tanpa keadaan 2026-07-28 dengan penerimaan registri, kebijakan, routing, dan batas kompatibilitas.
version: 2.0.0
phase: 13
lesson: 17
tags: [mcp, gateway, stateless, registry, rbac, subscriptions, tasks]
---

Dengan diberikan klien, backend, persyaratan otorisasi, dan batasan kepatuhan, hasilkan desain gerbang.

## Masukan wajib

- URI sumber daya gerbang publik, revisi protokol yang diterima, dan transpor.
- Prinsipal terautentikasi dan model peran.
- Endpoint backend, penerbit, sumber daya, catatan registri, bukti penerbit, dan deskriptor yang disetujui.
- Visibilitas alat, kebijakan argumen, kelas biaya, dan sensitivitas data.
- Persyaratan streaming, notifikasi perubahan, MRTR, dan Tasks.
- Persyaratan audit, retensi, trace, dan redaksi.

## Hasilkan

1. Ingress tanpa keadaan. Satu endpoint POST, versi dan kapabilitas per permintaan, header metode dan nama yang cocok, JSON atau SSE cakupan permintaan, serta 405 untuk GET dan DELETE modern. Validasi kesetaraan header sebelum dukungan versi. Tentukan HTTP 400 `-32020`, HTTP 400 `-32022` dengan data yang didukung dan diminta secara tepat, HTTP 404 `-32601`, serialisasi `data` galat opsional, dan penanganan notifikasi 202 dengan badan kosong.
2. Rencana penemuan. Implementasikan `server/discover` gerbang, temukan setiap backend, ekspos hanya irisan kapabilitas aman ujung-ke-ujung, dan sertakan `resultType`, `ttlMs`, `cacheScope`, serta metadata identitas server terkini.
3. Tabel penerimaan. Validasi bentuk publikasi `server.json` Registri resmi dan gaya nama `com.example/*` secara terpisah dari penerimaan keamanan. Untuk setiap backend, gabungkan catatan dengan namespace penerbit terverifikasi eksternal, sumber provenans, endpoint, kebijakan versi, intisari deskriptor, penerbit, sumber daya, persetujuan, dan status kedaluwarsa.
4. Peta namespace. Berikan setiap alat backend nama publik berkualifikasi yang stabil dan pertahankan `inputSchema` berakar objek yang valid pada setiap deskriptor `tools/list`. Tolak tabrakan berdasarkan urutan.
5. Matriks otorisasi. Petakan prinsipal dan peran ke alat publik, sumber daya, argumen, dan cakupan. Jaga kredensial luar dan backend tetap terpisah serta terikat pada penerbit.
6. Kontrak penerusan. Bangun permintaan backend mandiri yang segar, iklankan hanya kapabilitas klien yang dimediasi, validasi hasil backend, dan pertahankan korelasi trace.
7. Rencana tembolok. Jadikan penemuan dan daftar yang bergantung pada prinsipal bersifat privat. Tetapkan TTL terbatas dan perilaku invalidasi.
8. Kebijakan laju dan audit. Kunci batas berdasarkan prinsipal, penerbit, sumber daya, alat, kelas biaya, dan waktu. Redaksi kredensial serta argumen sensitif yang tidak perlu.
9. Routing interaksi. Jelaskan SSE cakupan permintaan, acknowledgment dan perilaku reconnect `subscriptions/listen`, penerusan keadaan MRTR yang tepat byte, serta routing Tasks berdasarkan id task di `Mcp-Name`.
10. Adapter transpor. Jika gerbang menerima permintaan dan header yang telah diurai, sebut sebagai model protokol dalam proses dan hubungkan ke Pelajaran 09 untuk penerapan Content-Type JSON serta Accept JSON plus SSE.
11. Adapter kompatibilitas. Pagar inisialisasi lawas, id sesi, aliran GET, langganan sumber daya, dan metode task eksperimental menjauh dari inti gerbang modern.

## Penolakan keras

- Afinitas sesi, penyimpanan sesi, atau penulisan ulang id sesi yang disajikan sebagai keharusan untuk 2026-07-28.
- Memercayai keberadaan di registri atau nama tampilan tanpa bukti penerimaan.
- Tabrakan alat diam-diam atau pembaruan pin deskriptor tanpa persetujuan ulang.
- Menggunakan token bearer luar di backend atau token backend di penerbit atau sumber daya lain.
- Penembolokan publik atas daftar yang difilter berdasarkan prinsipal.
- Aliran peristiwa GET mandiri modern, putar ulang Last-Event-ID, atau metode langganan sumber daya.
- Perilaku `tasks/list` atau `tasks/result` baru.
- Batas laju yang hanya dikunci berdasarkan sesi protokol yang dihapus.
- Verifikasi keamanan yang direkayasa di dalam `server.json`, alih-alih keadaan penerimaan dan provenans terverifikasi terpisah.
- Deskriptor alat bers namespace yang menghilangkan `inputSchema`.

## Format keluaran

Kembalikan bagian-bagian bernama Ingress, Penemuan, Penerimaan, Peta Namespace, Otorisasi, Penerusan, Tembolok, Batas Laju, Audit, Interaksi, dan Adapter Lawas. Akhiri dengan satu rute yang memerlukan uji penerimaan paling kuat.
