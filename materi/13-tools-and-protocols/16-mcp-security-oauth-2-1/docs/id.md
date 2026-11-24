# Otorisasi MCP: CIMD, Ikatan Penerbit, PKCE, dan Step-Up

> Permintaan MCP jarak jauh tidak memiliki keadaan, tetapi otorisasinya tidak anonim. Ikatkan setiap kredensial ke penerbit yang menciptakannya dan setiap token ke sumber daya yang menerimanya.

**Jenis:** Bangun
**Bahasa:** Python, TypeScript
**Prasyarat:** Fase 13 · 09 (transpor), Fase 13 · 15 (keamanan)
**Waktu:** ~90 menit

## Tujuan Pembelajaran

- Temukan server otorisasi melalui metadata sumber daya terlindungi.
- Utamakan Client ID Metadata Documents daripada Dynamic Client Registration yang tidak digunakan lagi.
- Nyatakan `application_type` yang benar ketika jalur kompatibilitas DCR tidak terelakkan.
- Validasi `iss` respons otorisasi dan isolasi kredensial menurut penerbit.
- Gunakan PKCE, indikator sumber daya, validasi audiens, dan cakupan inkremental.
- Kirim permintaan MCP 2026-07-28 terotorisasi tanpa sesi protokol.

## Masalahnya

Server MCP jarak jauh dapat membaca catatan pribadi, menulis sistem eksternal, atau memicu pekerjaan yang mahal. Autentikasi memberi tahu server siapa yang menyajikan kredensial. Otorisasi juga harus menjawab:

- Server otorisasi mana yang menerbitkan kredensial itu?
- Untuk sumber daya MCP mana token itu berlaku?
- Klien dan URI redirect mana yang menyelesaikan alur itu?
- Operasi apa yang disetujui pengguna?
- Apakah permintaan ini masih sesuai dengan persetujuan itu?

Profil otorisasi 2026-07-28 memperkeras pendaftaran klien dan penanganan penerbit. Profil ini mengutamakan Client ID Metadata Documents, tidak digunakan lagi Dynamic Client Registration, mewajibkan `application_type` yang tepat pada DCR, memvalidasi respons penerbit RFC 9207, dan melarang penggunaan ulang kredensial lintas penerbit.

Aturan ini melengkapi inti tanpa keadaan. Aturan ini tidak memulihkan jabat tangan inti maupun `Mcp-Session-Id`.

## Konsepnya

### Kenali tiga peran itu

- **Klien MCP:** mengirim permintaan atas nama pemilik sumber daya.
- **Server sumber daya MCP:** menerima token akses dan menyajikan endpoint MCP.
- **Server otorisasi:** mengautentikasi pemilik sumber daya, mengumpulkan persetujuan, dan menerbitkan token.

Server sumber daya dan server otorisasi dapat dioperasikan bersama, tetapi identifier dan tanggung jawab validasinya harus tetap terpisah.

### Otorisasi berlaku untuk HTTP

Spesifikasi otorisasi MCP berlaku untuk transpor berbasis HTTP. Server stdio lokal berada di dalam batas kepercayaan proses dan sistem operasi. Jangan menambahkan alur OAuth peramban palsu pada stdio hanya demi kesimetrisan.

Untuk Streamable HTTP jarak jauh, kirim token bearer di header `Authorization` pada setiap permintaan. Jangan pernah menaruhnya di URL.

### Mulai dari metadata sumber daya terlindungi

Server sumber daya menerbitkan metadata RFC 9728:

```json
{
  "resource": "https://notes.example.com/mcp",
  "authorization_servers": ["https://auth.example.com"],
  "scopes_supported": ["notes:delete", "notes:read", "notes:write"]
}
```

Klien berangkat dari URL sumber daya MCP, mengambil dokumen ini, memilih server otorisasi yang diiklankan, lalu mengambil metadata OAuth atau OpenID Connect server itu.

Pertahankan jalur sumber daya saat menyusun URL well-known RFC 9728. Untuk sumber daya `https://notes.example.com/mcp`, pelajaran ini menggunakan `https://notes.example.com/.well-known/oauth-protected-resource/mcp`. Menghilangkan akhiran `/mcp` dapat memilih metadata untuk sumber daya terlindungi lain pada origin yang sama.

Jangan menebak server otorisasi dari hostname. Jangan mengikuti penerbit yang ditemukan dari badan galat yang belum divalidasi. Pertahankan kebijakan tentang penerbit mana yang bersifat dipercaya klien.

### Verifikasi metadata server otorisasi

Metadata sebaiknya menyingkap endpoint dan kontrol yang didukung:

```json
{
  "issuer": "https://auth.example.com",
  "authorization_endpoint": "https://auth.example.com/authorize",
  "token_endpoint": "https://auth.example.com/token",
  "code_challenge_methods_supported": ["S256"],
  "authorization_response_iss_parameter_supported": true,
  "client_id_metadata_document_supported": true
}
```

Wajibkan S256 untuk PKCE. Rekam string penerbit yang tepat. Nilai persis itu menjadi kunci penyimpanan pendaftaran dan token.

### Ikuti prioritas pendaftaran

Gunakan informasi klien pra-terdaftar ketika klien sudah memiliki hubungan eksplisit dengan penerbit yang dipilih. Jika tidak, utamakan Client ID Metadata Documents ketika server otorisasi mengiklankan dukungannya. Gunakan DCR hanya sebagai fallback kompatibilitas yang tidak digunakan lagi, lalu minta informasi klien jika tidak ada mekanisme tersebut.

### Utamakan Client ID Metadata Documents

Client ID Metadata Document memberi server otorisasi sebuah URL HTTPS yang sekaligus menjadi identifier klien dan lokasi metadata klien:

```json
{
  "client_id": "https://client.example.com/oauth/metadata.json",
  "client_name": "Notes desktop client",
  "application_type": "native",
  "redirect_uris": ["http://127.0.0.1:8765/callback"],
  "grant_types": ["authorization_code"],
  "response_types": ["code"]
}
```

Server otorisasi mengambil dan memvalidasi dokumen itu. `client_id` harus berupa URL HTTPS yang memiliki jalur, dan nilai di dalam dokumen harus sama persis dengan URL itu. Field dokumen yang wajib adalah `client_id`, `client_name`, dan `redirect_uris`. `application_type` muncul dalam contoh ini, tetapi bukan keharusan CIMD. Penggunaan wajib yang baru justru khusus untuk jalur DCR.

Perlakukan pengambilan dokumen sebagai operasi sensitif SSRF. Selesaikan dan validasi tujuan, tolak alamat loopback, privat, link-local, dan alamat yang tidak diizinkan lainnya, periksa ulang setelah redirect dan perubahan DNS, batasi redirect, byte, dan waktu, wajibkan JSON, serta tembolokan hanya sesuai kontrol tembolok HTTP yang tervalidasi. Perlakukan `client_name` dan field tampilan lain sebagai teks yang tidak tepercaya.

CIMD menghilangkan kebutuhan menerbitkan identifier dinamis baru pada setiap kontak pertama. CIMD tidak menghilangkan validasi URI redirect, kebijakan penerbit, maupun persetujuan pengguna.

### DCR adalah jalur kompatibilitas

Dynamic Client Registration tetap tersedia untuk server otorisasi lawas, tetapi tidak digunakan lagi untuk implementasi MCP baru.

Saat menggunakan DCR, nyatakan `application_type`:

```json
{
  "client_name": "Notes desktop client",
  "application_type": "native",
  "redirect_uris": ["http://127.0.0.1:8765/callback"],
  "grant_types": ["authorization_code"],
  "response_types": ["code"]
}
```

- Klien desktop, seluler, baris perintah, dan loopback menggunakan `native`.
- Aplikasi peramban yang dihost jarak jauh menggunakan `web` dan redirect HTTPS jarak jauh.

Menghilangkan field dapat membuat implementasi pendaftaran OpenID Connect beralih ke `web` dan menyebabkan redirect loopback yang sah gagal.

Simpan kode DCR di balik keputusan fallback eksplisit. Jangan melakukan fallback diam-diam setelah kegagalan validasi CIMD sewenang-wenang. Hal itu dapat mengubah kegagalan keamanan menjadi jalur pendaftaran yang lebih lemah.

### Ikatkan kredensial ke penerbit

Simpan material pendaftaran yang diterbitkan penerbit di bawah penerbit yang tepat:

```text
issuer_credentials[issuer] = pre_registered_or_dcr_client
tokens[(issuer, resource)] = access_token
```

Jika penemuan sumber daya terlindungi berubah dari `https://auth-one.example` ke `https://auth-two.example`, evaluasi ulang kepercayaan. Jangan pernah mengirim client secret, client id DCR, registration access token, refresh token, atau access token penerbit pertama ke penerbit kedua. Klien pra-terdaftar dan DCR harus menggunakan kredensial yang diterbitkan untuk penerbit baru.

Client id CIMD berbeda karena merupakan URL HTTPS yang di-host sendiri, bukan kredensial yang diterbitkan server otorisasi. URL CIMD yang sama bersifat portabel: penerbit tepercaya baru mengambil dan memvalidasi dokumen tanpa pendaftaran ulang DCR. Respons otorisasi dan token tetap divalidasi serta disimpan di bawah penerbit baru.

### Kode otorisasi dengan PKCE

Alur interaktif adalah:

1. Hasilkan `code_verifier` dengan entropi tinggi.
2. Turunkan `code_challenge` S256.
3. Kirim permintaan otorisasi dengan `client_id`, `redirect_uri`, `scope`, `code_challenge`, dan `resource` yang tepat.
4. Terima respons otorisasi yang memuat `code` dan, bila disediakan, `iss`.
5. Validasi `iss` terhadap penerbit yang tercatat tepat sebelum menggunakan field respons apa pun.
6. Tukarkan kode dengan `code_verifier`, URI redirect yang sama, dan `resource` yang sama.
7. Simpan token yang dihasilkan di bawah `(issuer, resource)`.

Parameter `resource` dari RFC 8707 muncul baik dalam permintaan otorisasi maupun token. Parameter itu mengidentifikasi URI server MCP kanonik.

### Validasi `iss` secara tepat

RFC 9207 mencegah respons otorisasi dari satu penerbit tertukar dengan respons dari penerbit lain.

Ketika `iss` hadir, bandingkan dengan penerbit yang tercatat tanpa pelipatan huruf, perubahan garis miring akhir, penghapusan port bawaan, maupun normalisasi pengkodean persen. Pada ketidaksesuaian, jangan bertindak berdasarkan kode itu atau bahkan menampilkan detail galat yang dikendalikan penyerang dari respons itu.

Server otorisasi yang menyertakan `iss` mengiklankan `authorization_response_iss_parameter_supported: true`. Klien terkini tetap memvalidasi `iss` yang hadir meskipun iklan itu tidak ada.

### Validasi audiens di server MCP

Server sumber daya hanya menerima token yang diterbitkan untuk dirinya:

```text
token.issuer == configured_authorization_server
token.audience == canonical_mcp_resource
```

Token yang tidak valid, kedaluwarsa, salah penerbit, atau salah audiens mendapat 401. Server MCP tidak boleh menerima maupun meneruskan token yang ditujukan untuk layanan lain.

### Minta cakupan terkini terkecil

Mulailah dengan cakupan yang dibutuhkan saat ini. Jika alat berikutnya memerlukan lebih banyak, server mengembalikan 403 dengan tantangan cakupan yang otoritatif:

```text
WWW-Authenticate: Bearer error="insufficient_scope",
  scope="notes:delete",
  resource_metadata="https://notes.example.com/.well-known/oauth-protected-resource/mcp"
```

Klien menjelaskan izin baru, memperoleh persetujuan, melakukan alur otorisasi baru dengan gabungan cakupan, dan mencoba ulang permintaan MCP dengan id JSON-RPC baru.

Jangan menganggap cakupan yang ditantangan merupakan himpunan bagian dari `scopes_supported`. Tantangan bersifat otoritatif untuk operasi saat ini.

### Otorisasi dan kawat MCP tanpa keadaan

Panggilan alat terotorisasi tetap membawa amplop permintaan terkini lengkap:

```text
POST /mcp
Authorization: Bearer <access-token>
MCP-Protocol-Version: 2026-07-28
Mcp-Method: tools/call
Mcp-Name: notes.delete
```

```json
{
  "jsonrpc": "2.0",
  "id": 12,
  "method": "tools/call",
  "params": {
    "name": "notes.delete",
    "arguments": {"id": "note-7"},
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientCapabilities": {},
      "io.modelcontextprotocol/clientInfo": {
        "name": "oauth-lesson-client",
        "version": "1.0.0"
      }
    }
  }
}
```

Token mengotorisasi prinsipal. Metadata permintaan menegosiasi perilaku protokol. Keduanya tidak dapat saling menggantikan.

Validasi kawat dalam urutan tetap: tipe JSON-RPC dan metadata, kesetaraan header dan badan, lalu dukungan protokol. Ketidaksesuaian routing atau header versi mengembalikan HTTP 400 dengan `-32020`. Jika header dan badan sepakat pada versi yang tidak didukung, kembalikan HTTP 400 dengan `-32022` dan `data` tepat `{"supported":["2026-07-28"],"requested":"<actual>"}`. Metode yang tidak dikenal mengembalikan HTTP 404 dengan `-32601`.

Setiap galat permintaan, termasuk 401 token tidak valid dan 403 cakupan tidak cukup, adalah amplop galat JSON-RPC dengan `id` permintaan asli. Informasi pemulihan terstruktur berada di `data` galat opsional; `WWW-Authenticate` tetap menjadi header respons HTTP. Notifikasi tidak memiliki `id`, sehingga tidak menerima badan JSON-RPC. Notifikasi HTTP yang diterima mengembalikan 202 dengan badan kosong.

Server mengimplementasikan `server/discover` dan mengiklankan alat, sehingga server juga mengimplementasikan metode `tools/list` yang wajib. Deskriptor alatnya memiliki nama, deskripsi, dan nilai `inputSchema` berakar objek yang stabil. Daftar bersifat deterministik dan mengembalikan `resultType`, metadata identitas server, `ttlMs` terbatas, dan `cacheScope`. Penemuan serta daftar alat yang tidak bergantung pada pengguna dapat tersedia sebelum otorisasi. Terapkan kebijakan biasa dan penembolokan privat jika salah satunya berbeda menurut prinsipal.

### Tidak ada penerusan token

Server MCP tidak boleh meneruskan token akses MCP klien ke API hilir. Peroleh token hilir terpisah dengan audiens yang tepat atau gunakan desain pertukaran token yang eksplisit. Validasi audiens hanya berfungsi ketika layanan menolak token yang diterbitkan untuk pihak lain.

### Token penyegaran

Token penyegaran bersifat opsional. Saat diterbitkan, simpan secara rahasia dan kunci berdasarkan penerbit serta sumber daya. Jangan menganggap token itu pasti ada. Putar token itu ketika server otorisasi mendukung rotasi dan deteksi penggunaan ulang nilai yang tidak berlaku.

```figure
t3-scope-stepup
```

## Bangunlah

`code/main.py` adalah simulator protokol dan otorisasi dalam proses. Simulator ini mengimplementasikan penemuan sumber daya terlindungi, metadata server otorisasi, pendaftaran CIMD, fallback DCR yang dijaga versi, pemeriksaan jenis aplikasi, PKCE, validasi penerbit, token terikat sumber daya, step-up cakupan, `server/discover`, `tools/list`, serta permintaan alat tanpa keadaan.

Model menerima badan permintaan yang telah diurai dan header routing. Model ini bukan adapter HTTP lengkap dan tidak mengurai `Content-Type` maupun `Accept`. Hubungkan ke adapter Streamable HTTP Pelajaran 09, yang mewajibkan `Content-Type: application/json` dan nilai `Accept` yang memuat baik `application/json` maupun `text/event-stream`.

Jalankan:

```bash
cd phases/13-tools-and-protocols/16-mcp-security-oauth-2-1
python3 code/main.py
npx tsx code/main.ts
python3 -m unittest discover code/tests -v
```

Keluaran menampilkan penemuan terlebih dahulu, pendaftaran CIMD, satu pembacaan biasa, dua step-up cakupan terpisah, dan penyimpanan kredensial berdasarkan kunci penerbit.

## Gunakanlah

Petakan objek simulator ke komponen produksi:

- `ResourceServer.protected_resource_metadata` menjadi endpoint RFC 9728.
- `AuthorizationServer.metadata` menjadi penemuan RFC 8414 atau OpenID Connect.
- `Client.enroll` menjadi resolusi CIMD plus cabang kompatibilitas DCR yang eksplisit.
- Kredensial klien yang diterbitkan penerbit dan `tokens_by_issuer_resource` menjadi catatan terenkripsi. URL CIMD boleh tetap portabel sementara hasil otorisasinya tetap terikat penerbit.
- `ResourceServer.handle` menjadi middleware yang memvalidasi header MCP terkini, token, dan cakupan alat sebelum dispatch, sekaligus menjaga setiap galat permintaan tetap berada dalam amplop JSON-RPC yang sesuai.

## Kirimkan

Pelajaran ini mengirimkan `outputs/skill-oauth-scope-planner.md`. Berkas itu kini merancang prioritas pendaftaran, penyimpanan kredensial terikat penerbit, jenis aplikasi, PKCE, indikator sumber daya, tantangan cakupan, dan batas permintaan tanpa keadaan terkini.

## Latihan

1. Tambahkan rotasi token penyegaran dan tolak penggunaan ulang token penyegaran sebelumnya.
2. Tambahkan daftar penerbit yang diizinkan. Saat penerbit berubah, gunakan hanya URL CIMD yang portabel; tolak semua kredensial dan token sebelumnya yang diterbitkan penerbit lama.
3. Tambahkan kedaluwarsa pada kode otorisasi dan pastikan penukaran yang terlambat gagal.
4. Bangun varian klien web dengan redirect HTTPS jarak jauh dan bandingkan metadata DCR-nya dengan klien native.
5. Tambahkan sumber daya kedua di bawah penerbit yang sama. Pastikan token aksesnya tidak dapat digunakan di sumber daya pertama.

## Istilah Kunci

| Istilah | Makna |
|---------|-------|
| Metadata sumber daya terlindungi | Dokumen RFC 9728 yang mengidentifikasi sumber daya dan server otorisasi |
| CIMD | Dokumen metadata HTTPS yang URL-nya adalah identifier klien OAuth |
| DCR | Pendaftaran klien dinamis yang tidak digunakan lagi tetapi dipertahankan untuk kompatibilitas |
| `application_type` | `native` atau `web`, dipakai untuk memvalidasi aturan URI redirect |
| PKCE | Verifier dan tantangan S256 yang melindungi kode otorisasi yang disadap |
| `iss` | Identifier penerbit respons otorisasi RFC 9207 |
| Indikator sumber daya | Parameter RFC 8707 yang mengikat permintaan token ke sumber daya MCP |
| Audiens | Sumber daya tempat token berlaku |
| Step-up | Persetujuan dan penerbitan token baru untuk cakupan operasi terkini tambahan |
| Kredensial terikat penerbit | Catatan pendaftaran dan token yang diisolasi berdasarkan penerbit server otorisasi yang tepat |

## Bacaan Lanjutan

- [Spesifikasi otorisasi MCP 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization)
- [RFC 9728: OAuth 2.0 Protected Resource Metadata](https://www.rfc-editor.org/rfc/rfc9728)
- [RFC 8707: Resource Indicators for OAuth 2.0](https://www.rfc-editor.org/rfc/rfc8707)
- [RFC 9207: OAuth 2.0 Authorization Server Issuer Identification](https://www.rfc-editor.org/rfc/rfc9207)
- [Draf OAuth Client ID Metadata Document](https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document/)
