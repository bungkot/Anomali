---
name: mcp-contract-reviewer
description: Tinjau deskriptor alat, hasil, paginasi, completion, dan kebijakan header parameter pada MCP sebelum mengekspos alat kepada model.
version: 1.0.0
phase: 13
lesson: 28
tags: [mcp, tools, json-schema, pagination, completion, security]
---

Tinjau permukaan alat MCP yang diberikan terhadap versi protokol `2026-07-28`.

Mintalah input berikut jika tidak ada:

1. Halaman deskriptor `tools/list` lengkap, termasuk setiap kolom `nextCursor`.
2. Setidaknya satu hasil sukses dan satu hasil gagal per alat.
3. Pemetaan header parameter Streamable HTTP, jika digunakan.
4. Referensi completion, kelas pemanggil, dan contoh saran.
5. Konteks otorisasi yang dapat mengubah kumpulan alat yang terlihat.

Hasilkan laporan ringkas dengan bagian-bagian ini.

## Penerimaan deskriptor

Untuk setiap alat:

- verifikasi nama yang stabil dan tidak kosong;
- wajibkan objek `inputSchema`;
- identifikasi dialek JSON Schema, dengan default 2020-12 bila dihilangkan;
- validasi `outputSchema` saat ada;
- cantumkan annotation sebagai petunjuk yang tidak tepercaya, bukan kebijakan;
- kembalikan `ADMIT`, `REJECT`, atau `CONDITIONAL` dengan satu alasan yang presisi.

Tolak satu deskriptor yang cacat tanpa menolak alat valid yang tidak terkait.

## Kontrak hasil

Untuk setiap hasil lengkap, termasuk hasil dengan `isError: true`:

- wajibkan `resultType: complete`;
- validasi setiap blok konten menurut jenisnya;
- perlakukan `structuredContent` sebagai nilai JSON apa pun, bukan hanya objek;
- wajibkan `structuredContent` dan kepatuhan terhadap `outputSchema` ketika skema itu ada;
- wajibkan blok teks kompatibilitas untuk hasil terstruktur;
- bedakan tautan sumber daya dari sumber daya tersemat;
- nyatakan batas ukuran dan tipe media.

Klasifikasikan permintaan yang cacat sebagai error JSON-RPC. Klasifikasikan kegagalan eksekusi yang dapat ditindaklanjuti sebagai hasil lengkap dengan `isError: true`, tanpa melewati kontrak keluaran yang dipublikasikan.

## Header parameter

Untuk setiap `x-mcp-header`:

- wajibkan token nama kolom HTTP yang valid dan tidak kosong;
- wajibkan keunikan tanpa memedulikan huruf besar/kecil;
- wajibkan bertipe string, integer, atau boolean;
- telusuri seluruh skema input, termasuk properti bersarang, kombinator, item array, dan definisi yang dipakai `$ref`;
- izinkan annotation hanya pada anggota langsung `inputSchema.properties`, dan tolak setiap annotation yang ditemukan di tempat lain atau di `outputSchema`;
- tolak `number` serta nilai integer di luar rentang `-9007199254740991` sampai `9007199254740991`;
- tolak kolom kredensial, token, rahasia, kata sandi, otorisasi, dan PII sesuai kebijakan deployment;
- kirimkan nilai secara polos hanya ketika nilai itu berupa ASCII tampak yang tidak kosong dan tidak diawali `=?base64?`;
- jika tidak, hasilkan tepat `=?base64?{Base64UTF8}?=` tanpa memangkas atau menormalkan nilai asli;
- enkodekan string Unicode, kosong, spasi, karakter kontrol, CR atau LF, ber-padding, dan yang tampak seperti sentinel, serta render boolean sebagai teks huruf kecil;
- pada batas HTTP, dekodekan nilai `Mcp-Param-*` yang dikenali, bandingkan nama header tanpa memedulikan huruf besar/kecil serta nilai hasil dekode secara persis dengan isi JSON, dan tolak salinan yang hilang, ganda, tak terduga, cacat, atau tidak cocok sebagai HTTP `400` plus JSON-RPC `-32020`;
- catat nama header akhir dan kategori penolakan, jangan pernah nilai argumen atau payload hasil enkode.

## Paginasi

Telusuri setiap permintaan daftar. Lanjutkan selama `nextCursor` ada dan tidak null, termasuk ketika nilainya string kosong. Jangan pernah mendekode, mengubah, menaikkan, mengurutkan, atau menyimpulkan makna dari sebuah cursor. Laporkan alat ganda, halaman yang hilang, dan urutan yang tidak stabil.

## Completion

Untuk setiap referensi prompt atau sumber daya:

- validasi referensi dan argumen;
- saring saran melalui otorisasi pemanggil;
- batasi hasil hingga 100 nilai;
- tetapkan debounce klien dan batas laju server;
- uji bahwa nama tenant, sumber daya, dan lingkungan yang tersembunyi tidak bocor.

## Matriks verifikasi

Kembalikan setidaknya pemeriksaan berikut:

| Pemeriksaan | Fixture | Hasil yang diharapkan |
|------|---------|-----------------|
| Keluaran terstruktur non-objek | Skema array, skalar, atau null yang valid | Diterima jika sesuai |
| Ketidaksesuaian keluaran | Jenis JSON salah atau properti hilang | Ditolak sebelum dipakai model |
| Ketidaksesuaian keluaran error | `isError: true` dengan konten terstruktur hilang atau tidak valid | Ditolak sebelum dipakai model |
| Cursor kosong | `nextCursor: ""` | Permintaan lanjutan mengirim cursor persis itu |
| Header tidak aman | Token atau nama kolom tidak valid | Deskriptor ditolak |
| Annotation header bersarang | `oneOf`, `items`, objek bersarang, atau definisi `$ref` | Deskriptor ditolak saat penerimaan pohon penuh |
| Nilai header terenkode | Unicode, baris baru, padding, atau teks tampak seperti sentinel | Sentinel base64 UTF-8 persis mengembalikan nilai asli |
| Nilai header integer | Kedua batas aman serta satu nilai di luar masing-masing batas | Batas aman lolos; nilai tidak aman ditolak |
| Kesetaraan header dan isi | Variasi huruf, salinan hilang, dan ketidaksesuaian hasil dekode | Variasi huruf lolos; salinan hilang atau tidak cocok menghasilkan HTTP 400 dan JSON-RPC -32020 |
| Konten campuran | Teks, media, tautan, sumber daya tersemat | Setiap blok divalidasi secara independen |
| Isolasi completion | Pemanggil hak rendah | Tidak ada saran istimewa yang dikembalikan |
| Pelapisan error | Alat tidak dikenal dan kegagalan bisnis | Error JSON-RPC dan `isError: true` tetap berbeda |

Tolak persetujuan ketika bukti hanya berupa satu pemanggilan alat yang berhasil. Wajibkan halaman penemuan, keputusan penerimaan, dan fixture hasil yang tervalidasi.
