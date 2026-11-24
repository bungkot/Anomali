---
name: task-store-designer
description: Rancang pekerjaan MCP yang tahan lama dengan ekstensi Tasks terkini, permintaan tanpa keadaan, kepemilikan eksplisit, polling, pembaruan masukan, dan pembatalan.
version: 2.0.0
phase: 13
lesson: 13
tags: [mcp, tasks, extension, durable-state, stateless]
---

Rancang pekerjaan MCP yang berjalan lama terhadap ekstensi `io.modelcontextprotocol/tasks`.

Hasilkan:

1. Keputusan kelayakan. Jelaskan mengapa operasi ini memerlukan task alih-alih `tools/call` sinkron.
2. Kontrak kapabilitas. Tampilkan `supportedVersions`, kapabilitas, `ttlMs`, dan `cacheScope` yang tepat di `server/discover`, plus ekstensi Tasks dalam kapabilitas klien per permintaan. Jika alat diiklankan, sertakan deskriptor `tools/list` deterministik wajib dengan `inputSchema` berupa objek yang valid, metadata identitas server, dan petunjuk tembolok. Gunakan `-32021` dengan objek `requiredCapabilities` ketika ekstensi tidak ada, dan `-32022` dengan data `supported` serta `requested` yang persis untuk versi yang tidak didukung.
3. Transaksi pembuatan. Pertahankan task sampai `tasks/get` dapat menyelesaikannya, lalu kembalikan `resultType: "task"` yang diarahkan oleh server.
4. Bentuk keadaan. Sertakan `taskId`, `status`, `statusMessage`, stempel waktu ISO, `ttlMs`, `pollIntervalMs`, pemilik yang berwenang, referensi operasi asli, hasil atau galat, permintaan masukan yang masih tertunda, dan semua kunci masukan yang pernah diterbitkan. `CallToolResult` bersarang dari task yang selesai memiliki `resultType: "complete"` wajib dan SEBAIKNYA menyertakan metadata `io.modelcontextprotocol/serverInfo`-nya sendiri.
5. Metode terkini. Definisikan `tasks/get`, `tasks/update`, dan `tasks/cancel`. Untuk Streamable HTTP, setiap permintaan menetapkan `Mcp-Name` ke `params.taskId`. Jangan memperkenalkan `tasks/status`, `tasks/result`, atau `tasks/list`.
6. Kelanjutan masukan. Pisahkan MRTR sebelum pembuatan dari `tasks/get` plus `tasks/update` setelah pembuatan. Wajibkan kunci masukan yang unik seumur hidup dan penanganan respons parsial.
7. Rencana durabilitas. Pilih penyimpanan sistem berkas atomik, basis data transaksional, atau antrean plus penyimpanan bersama. Sertakan leasing pekerja dan perilaku saat restart.
8. Kebijakan kepemilikan. Otorisasi setiap metode task dan langganan berdasarkan tenant serta prinsipal. Jangan pernah menganggap pengetahuan tentang id task sebagai izin.
9. Kontrak pembatalan. Nyatakan bahwa acknowledgment bersifat kooperatif dan mungkin tidak menghasilkan `cancelled`.
10. Opsi notifikasi. Gunakan `subscriptions/listen` pada aliran SSE respons POST dan `notifications/tasks`, dengan polling sebagai dasar. Letakkan `io.modelcontextprotocol/subscriptionId`, yang sama dengan id permintaan listen, dalam acknowledgment dan setiap notifikasi task. Notifikasi tanpa id tidak menerima respons JSON-RPC; notifikasi HTTP yang diterima mendapat `202` tanpa badan.
11. Kebijakan kedaluwarsa. Artikan `ttlMs` sejak pembuatan, definisikan perilaku pembersihan, dan hindari membocorkan apakah task tenant lain ada.
12. Peta migrasi. Ganti flag task yang diminta klien dan metode eksperimental yang dihapus dengan alur ekstensi terkini.

Penolakan keras:

- Mengembalikan pegangan task sebelum visibilitas baca yang tahan lama.
- Mengembalikan `resultType: "task"` kepada permintaan yang tidak mengiklankan ekstensi.
- Menggunakan `params._meta.task.required`, `tasks/status`, `tasks/result`, atau `tasks/list` sebagai API terkini.
- Menggunakan `initialize`, `Mcp-Session-Id`, routing lengket, atau keadaan sesi transpor tersembunyi sebagai penyimpanan task.
- Menganggap acknowledgment `tasks/cancel` sebagai bukti bahwa pekerja telah berhenti.
- Menggunakan kembali kunci `inputRequests` dalam satu masa hidup task.
- Mengembalikan task kepada pemanggil yang bukan pemilik berwenangnya.
- Mengimplementasikan pengiriman notifikasi melalui GET mandiri, SSE sesi, atau putar ulang `Last-Event-ID`.

Aturan penolakan:

- Tolak task untuk pencarian deterministik cepat kecuali pemanggil memberikan persyaratan durabilitas yang konkret.
- Tolak penyimpanan produksi yang hanya berada dalam memori ketika pekerjaan harus bertahan setelah proses restart.
- Tolak muatan hasil yang tidak terbatas; simpan artefak besar secara eksternal dan kembalikan pegangan sumber daya yang terotorisasi.
- Tolak endpoint riwayat tanpa kepemilikan tenant, pemfilteran, paginasi, dan kebijakan retensi yang eksplisit.

Keluarkan desain satu halaman dengan tabel siklus hidup, metode kawat, transaksi persistensi, aturan kepemilikan, alur masukan, ritme polling, semantik pembatalan, opsi langganan, pembersihan kedaluwarsa, model kegagalan, dan peta migrasi lawas.
