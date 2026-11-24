---
name: mcp-request-tracer
description: Audit transkrip MCP pesan demi pesan melintasi era protokol modern stateless dan legacy eksplisit.
version: 2.0.0
phase: 13
lesson: 06
tags: [mcp, json-rpc, stateless, metadata, compatibility]
---

Dengan diberikan urutan amplop JSON-RPC MCP, audit setiap pesan secara mandiri terhadap MCP `2026-07-28`. Deteksi lalu lintas legacy, tetapi jangan pernah mengasumsikan handshake atau sesi protokol ada.

Hasilkan:

1. Anotasi pesan. Nyatakan arah, jenis JSON-RPC, metode, primitif, id permintaan, dan era yang terdeteksi.
2. Pemeriksaan metadata modern. Untuk setiap permintaan, verifikasi `params._meta.io.modelcontextprotocol/protocolVersion` dan `params._meta.io.modelcontextprotocol/clientCapabilities`. Catat apakah `clientInfo` yang disarankan hadir.
3. Pemeriksaan hasil. Verifikasi setiap keberhasilan modern memiliki `resultType: "complete"` atau jenis hasil lain yang ditentukan, ditambah identitas server yang disarankan di `_meta` hasil.
4. Pemeriksaan penemuan dan versi. Verifikasi server modern mengimplementasikan `server/discover`. Artikan `-32022` sebagai bukti modern dan periksa `data.requested` plus `data.supported`.
5. Pemeriksaan cache. Untuk `server/discover`, metode daftar, dan `resources/read`, wajibkan `ttlMs` dan `cacheScope`. Tandai urutan daftar yang nondeterministik.
6. Pemeriksaan arah. Tolak permintaan JSON-RPC yang dimulai server dalam lalu lintas modern. Izinkan notifikasi yang terkait permintaan dan aliran `subscriptions/listen` yang dibuka klien.
7. Pemeriksaan kompatibilitas. Beri label `initialize` dan `notifications/initialized` hanya sebagai legacy. Jangan mewajibkannya dalam lalu lintas modern.

Penolakan keras:

- Memperlakukan proses stdio, koneksi HTTP, atau `Mcp-Session-Id` sebagai status protokol modern.
- Menyimpulkan kapabilitas klien dari permintaan sebelumnya.
- Beralih ke legacy setelah error modern yang dikenali seperti `-32020`, `-32021`, atau `-32022`.
- Menerima keberhasilan modern tanpa `resultType`.

Aturan penolakan:

- Jika transkrip bukan JSON-RPC 2.0, berhenti dan identifikasi amplop yang tidak kompatibel.
- Jika diminta menulis ulang bukti secara diam-diam, tolak. Pertahankan transkrip asli dan hasilkan contoh koreksi terpisah.

Keluarkan satu baris per pesan sesuai urutan kedatangan:

```text
[request/modern/tools] id=7 tools/list metadata=valid
```

Akhiri dengan jumlah pesan modern, legacy, tidak valid, dan ambigu, diikuti tindakan korektif pertama.
