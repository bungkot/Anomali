# Auth MCP di Produksi: Pendaftaran dan Token yang Terikat Penerbit

> Pelajaran 16 membangun mesin keadaan OAuth 2.1. Pelajaran ini memperkeras batas produksinya untuk MCP 2026-07-28: Client ID Metadata Documents terlebih dahulu, pendaftaran dinamis yang tidak digunakan lagi hanya untuk kompatibilitas, validasi penerbit respons otorisasi, kredensial klien berbasis kunci penerbit, penyegaran JWKS, serta token yang dikunci ke satu sumber daya pada setiap permintaan tanpa keadaan.
>
> **Catatan spesifikasi (2026-07-28):** Dynamic Client Registration tidak digunakan lagi demi Client ID Metadata Documents. DCR tetap menjadi mekanisme kompatibilitas. Saat digunakan, klien menyatakan `application_type` yang benar. Klien memvalidasi nilai `iss` RFC 9207 yang hadir dan tidak pernah menggunakan ulang kredensial lintas penerbit server otorisasi.

**Jenis:** Bangun
**Bahasa:** Python (stdlib), TypeScript
**Prasyarat:** Fase 13 · 16 (mesin keadaan OAuth 2.1), Fase 13 · 17 (gerbang)
**Waktu:** ~90 menit

## Tujuan Pembelajaran

- Temukan server otorisasi melalui metadata RFC 8414 dan verifikasi kontraknya.
- Daftar melalui Client ID Metadata Document dan isolasi DCR yang tidak digunakan lagi sebagai fallback.
- Validasi `iss` RFC 9207, kunci pendaftaran berdasarkan penerbit server otorisasi, serta kunci token terikat sumber daya berdasarkan penerbit plus sumber daya.
- Tembolokkan dan segarkan kunci JWKS sesuai jadwal agar verifikasi tanda tangan bertahan saat kunci berganti.
- Kunci token ke satu sumber daya MCP menggunakan indikator sumber daya RFC 8707 dan tolak penggunaan ulang confused-deputy.
- Pilih validasi JWT atau introspection token, tetapkan kesegaran pencabutan, dan gagal dengan aman ketika dependensi identitas tidak tersedia.
- Pisahkan server otorisasi, server sumber daya, dan klien agar masing-masing menegakkan hanya pemeriksaan sendiri.
- Audit server otorisasi terhadap daftar periksa deployment dan tolak pendaftaran maupun penggunaan ulang token yang tidak aman.

## Masalahnya

Simulator Pelajaran 16 menjalankan OAuth 2.1 dalam memori. Produksi memiliki tiga celah operasional yang tidak terlihat oleh simulator yang hanya berada dalam memori.

Celah pertama adalah pendaftaran dan isolasi kredensial. Sebuah organisasi nyata mungkin menjalankan ratusan server MCP dan ribuan klien MCP. Revisi 2026-07-28 mengutamakan **Client ID Metadata Document**: klien menggunakan URL HTTPS dengan jalur yang dikendalikannya sebagai identifier, dan server otorisasi menarik metadata itu. RFC 7591 dynamic registration tetap hanya sebagai jalur kompatibilitas yang tidak digunakan lagi. Ketika DCR tidak terelakkan, permintaan menyatakan `application_type` yang benar. Klien menyimpan pendaftaran di bawah penerbit server otorisasi dan token akses di bawah pasangan `(issuer, resource)`. Penerbit yang berubah berarti pendaftaran baru, dan sumber daya yang berbeda berarti token dengan ikatan audiens tersendiri.

Celah kedua adalah rotasi kunci. Validasi JWT bergantung pada kunci penandatanganan server otorisasi, yang diterbitkan sebagai JSON Web Key Set (JWKS). Server otorisasi memutar kunci itu sesuai jadwal (sering setiap jam, kadang lebih cepat saat respons insiden). Server MCP yang mengambil JWKS sekali saat boot akan tervalidasi dengan baik sampai jendela rotasi — lalu setiap permintaan gagal sampai restart. Produksi menghubungkan JWKS sebagai nilai yang ditembolokkan dengan pekerjaan penyegaran yang menimpa tembolok sebelum kunci lama kedaluwarsa, plus pengambilan ulang fallback saat tembolok miss untuk kasus ketika token yang ditandatangani dengan kunci yang lebih baru daripada tembolok tiba.

Celah ketiga adalah ikatan audiens. Pelajaran 16 memperkenalkan indikator sumber daya RFC 8707. Di produksi, indikator itu menjadi pemeriksaan klaim yang keras pada setiap permintaan. Server MCP membandingkan `token.aud` dengan URL sumber daya kanoniknya sendiri dan menolak ketidaksesuaian dengan HTTP 401. Ini adalah satu-satunya pertahanan terhadap server MCP upstream (atau klien berbahaya yang memegang token untuk satu server) yang memutar ulang token itu ke server lain dalam jaring kepercayaan yang sama.

Pelajaran ini memetakan setiap celah ke bagian konkret dari permukaan. Dokumen metadata adalah endpoint HTTP. Penyegaran tembolok JWKS adalah pekerjaan terjadwal plus tembolok key-value. Validasi JWT adalah rutin yang dijalankan server sumber daya sebelum mendispatch alat apa pun. Jaga ketiga peran tetap terpisah dan masing-masing menegakkan hanya pemeriksaan yang menjadi miliknya: server otorisasi menerbitkan dan memutar kunci, server sumber daya menembolokkan dan memvalidasi, klien menemukan dan mendaftar.

## Cakupan: Penegakan Produksi Setelah Pelajaran 16

[Pelajaran 16: Keamanan MCP dengan OAuth 2.1](../../16-mcp-security-oauth-2-1/docs/en.md) memiliki mesin keadaan kode otorisasi, PKCE, penemuan sumber daya terlindungi, indikator sumber daya, dan keputusan cakupan. Pelajaran ini tidak mendefinisikan alur OAuth kedua. Pelajaran ini dimulai setelah kontrak-kontrak itu ada dan menanyakan bagaimana server sumber daya yang dideploy terus menegakkannya selama rotasi kunci, validasi token buram, pencabutan, kegagalan dependensi, rollout, dan respons insiden.

Batas produksinya lebih sempit dan lebih operasional:

- Jalur JWT memverifikasi penerbit, algoritme, kunci tanda tangan, audiens, klaim waktu, dan cakupan yang dikunci pada setiap permintaan sambil menyegarkan JWKS dengan aman.
- Jalur token buram memanggil endpoint introspection terautentikasi milik penerbit dan memvalidasi keadaan active yang dikembalikan, audiens atau sumber daya, kedaluwarsa, subjek, dan cakupan.
- Kebijakan pencabutan menentukan seberapa cepat kredensial harus berhenti berfungsi dan tembolok mana yang dapat menunda fakta itu.
- Kebijakan kegagalan memutuskan apa yang terjadi ketika penemuan, JWKS, introspection, atau infrastruktur pencabutan tidak tersedia.
- Bukti merekam metadata penerbit, kumpulan kunci atau respons introspection, klaim token, versi kebijakan, dan alasan penolakan apa yang mendorong hasil, tanpa menyimpan tokennya.

Pembedaan ini menjaga pelajaran tetap dapat disusun. Pelajaran 16 membuktikan alurnya. Pelajaran 18 membuktikan bahwa token tetap dapat dipercaya, atau ditolak, setelah mencapai jalur permintaan MCP yang nyata.

## Konsepnya

### RFC 8414 — Metadata Server Otorisasi OAuth

Dokumen di `/.well-known/oauth-authorization-server` menjelaskan semua yang dibutuhkan klien:

```json
{
  "issuer": "https://auth.example.com",
  "authorization_endpoint": "https://auth.example.com/authorize",
  "token_endpoint": "https://auth.example.com/token",
  "jwks_uri": "https://auth.example.com/.well-known/jwks.json",
  "client_id_metadata_document_supported": true,
  "registration_endpoint": "https://auth.example.com/register",
  "authorization_response_iss_parameter_supported": true,
  "response_types_supported": ["code"],
  "grant_types_supported": ["authorization_code", "refresh_token"],
  "code_challenge_methods_supported": ["S256"],
  "scopes_supported": ["mcp:tools.read", "mcp:tools.invoke"],
  "token_endpoint_auth_methods_supported": ["none", "private_key_jwt"]
}
```

Klien yang diberi URL sumber daya MCP merangkai penemuan: `oauth-protected-resource` dari RFC 9728 (dokumen server sumber daya) menyebut penerbit, lalu `oauth-authorization-server` (RFC ini) menyebut setiap endpoint. Klien tidak pernah mengkodekan URL otorisasi secara tetap.

Untuk identifier sumber daya yang memiliki jalur, sisipkan segmen well-known sebelum jalur itu. Misalnya, `https://mcp.example.com/team/server` menyelesaikan metadata sumber daya terlindungi di `https://mcp.example.com/.well-known/oauth-protected-resource/team/server`. Menambahkan `/.well-known/...` setelah jalur sumber daya adalah kesalahan.

Kontrak yang Anda verifikasi sebelum memercayai IdP untuk MCP:

- `code_challenge_methods_supported` menyertakan `S256` (PKCE sesuai RFC 7636). Spesifikasi bersifat eksplisit: jika field ini **tidak ada**, server otorisasi tidak mendukung PKCE dan klien **WAJIB** menolak untuk melanjutkan.
- `grant_types_supported` menyertakan `authorization_code` serta menolak `password` dan `implicit`.
- Minimal satu jalur pendaftaran tersedia: `client_id_metadata_document_supported: true` (CIMD, diutamakan), klien pra-terdaftar, atau `registration_endpoint` (kompatibilitas RFC 7591 yang tidak digunakan lagi).
- Jika `authorization_response_iss_parameter_supported` bernilai true, klien mewajibkan `iss` RFC 9207 yang dikembalikan dan membandingkannya secara tepat dengan penerbit yang tercatat sebelum redirect.
- `response_types_supported` tepat `["code"]` untuk OAuth 2.1.

Jika `S256` tidak ada, server MCP menolak deploy terhadap IdP itu — tidak ada mode degradasi untuk PKCE. Jika *tidak ada* jalur pendaftaran yang diiklankan dan Anda tidak memiliki `client_id` pra-terdaftar, Anda juga tidak dapat mendaftar; manifes deployment yang salah, bukan kodenya.

### RFC 9728 (ringkasan) — Metadata Sumber Daya Terlindungi

Pelajaran 16 membahas RFC 9728. Yang berubah di produksi: dokumen ini adalah satu-satunya tempat klien mencari server otorisasi yang dipercaya oleh *server MCP ini*. Satu server MCP boleh menerima token dari beberapa IdP (satu untuk karyawan, satu untuk mitra). RFC 9728 menyatakan himpunan itu; RFC 8414 mendokumentasikan apa yang didukung masing-masing IdP.

```json
{
  "resource": "https://notes.example.com",
  "authorization_servers": ["https://auth.example.com", "https://partners.example.com"],
  "scopes_supported": ["mcp:tools.invoke"],
  "bearer_methods_supported": ["header"],
  "resource_documentation": "https://notes.example.com/docs"
}
```

### Client ID Metadata Documents (nilai bawaan yang disarankan)

CIMD membalik pendaftaran dari *push* menjadi *pull*. Alih-alih meminta server otorisasi menerbitkan `client_id`, klien menggunakan URL HTTPS yang dikendalikannya **sebagai** `client_id`. URL itu mengarah ke dokumen metadata JSON; server otorisasi mengambilnya sesuai permintaan selama alur OAuth. Kepercayaan berakar pada DNS: jika operator server memercayai `app.example.com`, server itu memercayai klien yang disajikan dari `https://app.example.com/client.json`. Tanpa putaran pendaftaran, tanpa ruang nama `client_id` yang habis, tanpa keadaan per server yang harus disinkronkan.

Dokumen metadata yang dihost klien:

```json
{
  "client_id": "https://app.example.com/oauth/client.json",
  "client_name": "Example MCP Client",
  "client_uri": "https://app.example.com",
  "application_type": "native",
  "redirect_uris": ["http://127.0.0.1:7333/callback", "http://localhost:7333/callback"],
  "grant_types": ["authorization_code", "refresh_token"],
  "response_types": ["code"],
  "token_endpoint_auth_method": "none"
}
```

Nilai `client_id` dalam dokumen **WAJIB** sama dengan URL tempat dokumen itu disajikan (server otorisasi memverifikasi ini; ketidaksesuaian ditolak). Server otorisasi mengiklankan dukungan dengan `client_id_metadata_document_supported: true` dalam metadata RFC 8414-nya.

Untuk kontrak CIMD terkini, `client_id`, `client_name`, dan array `redirect_uris` yang tidak kosong bersifat wajib. Identifier klien adalah URL HTTPS absolut yang memiliki jalur. `application_type` boleh disertakan, tetapi bukan field CIMD wajib. Jangan menyalin keharusan `application_type` dari DCR ke jalur CIMD yang diutamakan.

Dua fakta keamanan yang dinyatakan spesifikasi dengan terus terang:

- **SSRF.** Server otorisasi mengambil URL yang disediakan penyerang. Server itu harus bertahan terhadap server-side request forgery (tidak boleh mengambil endpoint internal/admin).
- **Peniruan localhost.** CIMD sendiri tidak dapat mencegah penyerang lokal mengklaim URL metadata klien yang sah dan mengikat redirect `localhost` sembarangan. Server otorisasi **WAJIB** menampilkan hostname URI redirect secara jelas saat persetujuan dan **SEBAIKNYA** memberi peringatan pada redirect yang hanya `localhost`.

Karena CIMD tidak memerlukan keadaan sisi server, tidak ada registrar yang perlu disiapkan seperti yang diperlukan DCR. Sisi klien bersifat baca-saja: sajikan dokumen metadata Anda dari endpoint HTTPS statis dan biarkan server otorisasi menarik itu.

Jika operator server otorisasi telah menyediakan identifier klien, gunakan pendaftaran yang dibatasi penerbit itu sebelum mencoba pendaftaran otomatis. Jika tidak, utamakan CIMD. Gunakan DCR yang tidak digunakan lagi hanya ketika penerbit tidak dapat memakai pra-pendaftaran maupun CIMD.

### RFC 7591: pendaftaran kompatibilitas yang tidak digunakan lagi

DCR tidak digunakan lagi dalam revisi 2026-07-28. Pertahankan hanya untuk server otorisasi yang tidak dapat mengonsumsi CIMD dan ketika pra-pendaftaran tidak praktis. Klien kompatibilitas mengirim:

```json
POST /register
Content-Type: application/json

{
  "application_type": "native",
  "redirect_uris": ["http://127.0.0.1:7333/callback"],
  "grant_types": ["authorization_code", "refresh_token"],
  "response_types": ["code"],
  "token_endpoint_auth_method": "none",
  "scope": "mcp:tools.invoke",
  "client_name": "Cursor",
  "software_id": "com.cursor.cursor",
  "software_version": "0.42.0"
}
```

Server merespons dengan `client_id` dan `registration_access_token` untuk pembaruan berikutnya:

```json
{
  "client_id": "c_3e7f1a",
  "client_id_issued_at": 1769472000,
  "redirect_uris": ["http://127.0.0.1:7333/callback"],
  "grant_types": ["authorization_code", "refresh_token"],
  "registration_access_token": "regt_b2...",
  "registration_client_uri": "https://auth.example.com/register/c_3e7f1a"
}
```

`application_type` bukan sekadar hiasan. Klien desktop loopback menyatakan `native`; klien yang dihost server menyatakan `web` dan memakai URI redirect HTTPS. `token_endpoint_auth_method: none` adalah nilai bawaan yang tepat untuk klien native publik. Klien itu hanya mendapat `client_id`, dengan PKCE sebagai bukti kepemilikan.

Tiga jebakan produksi:

- Endpoint pendaftaran harus membatasi laju berdasarkan IP sumber. Tanpa itu, aktor bermusuhan dapat menulis jutaan pendaftaran palsu dan menguras ruang nama `client_id`. Jalankan pemeriksaan batas laju sebelum registrar menangani permintaan.
- `software_statement` (JWT bertanda tangan yang menjamin klien) diwajibkan oleh beberapa IdP enterprise. Mock pelajaran melewatkannya; produksi memasang langkah verifikasi yang menolak pendaftaran tanpa tanda tangan dari apa pun selain URI redirect localhost.
- `registration_access_token` harus disimpan sebagai hash, bukan plaintext. Pencurian token ini berarti penyerang dapat menulis ulang URI redirect klien.

### RFC 8707 (ringkasan) — Indikator Sumber Daya

Pelajaran 16 menetapkan bentuknya. Aturan produksi: setiap permintaan token memuat `resource=<canonical-mcp-url>`, dan server MCP memverifikasi bahwa `token.aud` cocok dengan URL sumber dayanya sendiri pada setiap panggilan. URI kanonik adalah identifier **paling spesifik** untuk server: menggunakan skema dan host huruf kecil, tanpa fragmen, dan secara konvensional tanpa garis miring akhir. Komponen jalur **tidak** dihapus berdasarkan aturan — spesifikasi mempertahankannya ketika diperlukan untuk mengidentifikasi satu server MCP. `https://mcp.example.com`, `https://mcp.example.com/mcp`, `https://mcp.example.com:8443`, dan `https://mcp.example.com/server/mcp` semuanya adalah URI kanonik yang valid. Pilih satu per server dan kunci `aud` tepat pada URI itu. (Mock pelajaran ini memakai audiens host-polos seperti `https://notes.example.com` demi ringkas; deployment yang menghost beberapa server MCP di satu origin membedakannya berdasarkan jalur.)

### RFC 7636 (ringkasan) — PKCE

PKCE bersifat wajib dalam OAuth 2.1. Alur kode otorisasi pelajaran ini selalu membawa `code_challenge` dan `code_verifier`. Server menolak permintaan token apa pun tanpa verifier atau dengan verifier yang tidak meng-hash menjadi challenge yang tersimpan.

### Profil otorisasi MCP 2026-07-28

Revisi MCP terkini mempertahankan batas server sumber daya OAuth sembari membuat transpor MCP tanpa keadaan. Tidak ada sesi protokol untuk menembolokkan keputusan identitas. Karena itu, lapisan otorisasi memvalidasi setiap permintaan secara mandiri:

- Implementasikan metadata sumber daya terlindungi RFC 9728, dan sediakan lokasinya baik melalui header `WWW-Authenticate: Bearer resource_metadata="..."` pada 401 **maupun** URI well-known `/.well-known/oauth-protected-resource` (SEP-985 membuat header opsional dengan fallback well-known). Field `authorization_servers` dalam metadata **WAJIB** menyebut minimal satu server.
- Terima token hanya lewat `Authorization: Bearer ...` pada **setiap** permintaan — jangan pernah di query string, jangan pernah divalidasi hanya saat awal sesi.
- Validasi `aud`, `iss`, `exp`, dan cakupan yang dibutuhkan per permintaan. Server **WAJIB** memvalidasi bahwa token diterbitkan secara khusus untuknya (audiens); `aud` yang tidak ada atau tidak cocok ditolak, tidak pernah dianggap wildcard.
- Pada 401/403, kembalikan `WWW-Authenticate: Bearer` yang membawa `error=...`, parameter `resource_metadata="<PRM-URL>"` (URL dokumen metadata, *bukan* sumber daya polos), dan `scope="..."` pada `insufficient_scope` (403). Catatan: parameternya adalah `resource_metadata`, penunjuk penemuan — tidak ada parameter `resource` dalam tantangan itu.
- Penemuan server otorisasi menerima **baik** metadata OAuth RFC 8414 **maupun** OpenID Connect Discovery 1.0; klien harus mencoba kedua akhiran well-known sesuai urutan prioritas.
- Klien (bukan server) yang bertahan terhadap **serangan mix-up**: klien merekam `issuer` yang diharapkan sebelum redirect dan memvalidasi nilai `iss` yang dikembalikan dalam respons otorisasi aktual (RFC 9207) sebelum menebus kode. PKCE saja tidak menghentikan mix-up, karena klien menyerahkan `code_verifier`-nya ke endpoint token mana pun yang diarahkan kepadanya.
- Kredensial klien milik satu penerbit server otorisasi. Jika penemuan mengarah ke penerbit yang berbeda, klien mendaftar ulang alih-alih menyajikan `client_id`, token pendaftaran, atau token akses lama.
- CIMD adalah mekanisme pendaftaran yang diutamakan. DCR tidak digunakan lagi; permintaan DCR kompatibilitas tetap menyatakan `application_type` yang benar.

Draf OAuth 2.1 adalah landasannya; RFC 8414/7591/8707/9728/9207 plus RFC 7636 dan CIMD adalah permukaannya; spesifikasi MCP adalah profilnya.

### Daftar periksa kapabilitas deployment

Tabel fitur vendor cepat usang. Periksa metadata yang dikembalikan oleh server otorisasi yang benar-benar akan Anda deploy. Gerbangnya bersifat mekanis:

| Pemeriksaan | Keputusan yang dibutuhkan |
|-------------|---------------------------|
| Penerbit yang ditemukan | Penerbit HTTPS tepat yang diharapkan kebijakan |
| PKCE | `S256` diiklankan; jika tidak, berhenti |
| Pendaftaran | CIMD diutamakan, pra-pendaftaran diterima, DCR hanya sebagai kompatibilitas yang tidak digunakan lagi |
| Respons otorisasi | Validasi `iss` RFC 9207 bila hadir atau diiklankan |
| Ikatan sumber daya | Permintaan token membawa `resource`; server sumber daya mewajibkan `aud` yang cocok |
| Penyimpanan kredensial | Kunci client id dan kredensial pendaftaran berdasarkan penerbit; kunci token akses berdasarkan penerbit plus sumber daya |
| Kompatibilitas DCR | Nyatakan `native` atau `web`; tolak URI redirect yang tidak sesuai dengan jenis aplikasi yang dinyatakan |

Jangan menyimpulkan dukungan dari nama produk atau tingkat harga. Rekam dokumen yang ditemukan sebagai bukti deployment dan gagal tertutup ketika field wajib tidak ada.

### Pola penyegaran JWKS (rotasi di AS, penyegaran di server sumber daya)

Jaga agar dua kata kerja tetap terpisah, karena menggabungkannya adalah bug produksi nyata:

- **Rotasi** adalah yang dilakukan *server otorisasi*: menerbitkan kunci penandatanganan baru, memuatnya ke JWKS, lalu memensiunkan yang lama kemudian. Server sumber daya tidak berperan dan tidak dapat melakukannya — server itu tidak memiliki kunci privat IdP.
- **Penyegaran** adalah yang dilakukan *server sumber daya*: meng-`GET` ulang JWKS yang diterbitkan ke temboloknya. Itu adalah satu-satunya aksi JWKS yang pernah dilakukan server sumber daya.

Mode kegagalan produksinya adalah tembolok basi. Selesaikan dengan pekerjaan penyegaran terjadwal plus tembolok key-value. Server sumber daya menjalankan pekerjaan (cron, timer, atau apa pun yang ditawarkan runtime Anda) yang pada interval tetap mengambil `<issuer>/.well-known/jwks.json` dan menimpa `cache[issuer] = {keys, fetched_at}`. Validator membaca dari tembolok itu. Token yang `kid`-nya tidak ada dalam tembolok memicu **satu** penyegaran sinkron sebagai fallback, lalu memeriksa ulang. Ini menangani dua kasus sekaligus: penyegaran terjadwal dan jendela tumpang tindih kunci ketika token yang ditandatangani kunci baru tiba sebelum penyegaran terjadwal berikutnya.

Fallback **harus** berupa pengambilan ulang, bukan rotasi. Jika Anda menghubungkan jalur tembolok miss ke rotate-and-mint, dua hal rusak: (1) menerbitkan kunci baru menghasilkan `kid` yang *tetap* tidak cocok dengan token, jadi pencarian gagal juga; dan (2) penyerang yang menyebarkan token dengan nilai `kid` acak memaksa rentetan pembuatan kunci tanpa batas — DoS yang ditimbulkan sendiri. Pengambilan ulang bersifat idempoten, sehingga `kid` palsu paling banyak membiayai satu pengambilan yang sia-sia.

Bentuk tembolok:

```json
{
  "https://auth.example.com": {
    "keys": [
      {"kid": "k_2026_03", "kty": "RSA", "n": "...", "e": "AQAB", "alg": "RS256", "use": "sig"},
      {"kid": "k_2026_04", "kty": "RSA", "n": "...", "e": "AQAB", "alg": "RS256", "use": "sig"}
    ],
    "fetched_at": 1772668800
  }
}
```

Dua kunci sekaligus adalah keadaan mantap. Server otorisasi berrotasi dengan memperkenalkan kunci berikutnya (`k_2026_04`) sebelum memensiunkan yang sebelumnya (`k_2026_03`), agar token yang diterbitkan dengan kunci lama tetap valid sampai kedaluwarsa. Tembolok menyimpan gabungannya; validator memilih berdasarkan `kid`.

### Rutin validasi

Server MCP menjalankan validasi sebelum mendispatch alat apa pun. Bentuk yang dipakai `code/main.py`:

```python
result = server.validate(bearer_token, required_scope="mcp:tools.invoke")
if not result["valid"]:
    return {"status": result["status"], "WWW-Authenticate": result["www_authenticate"]}
```

`validate` mendekode JWT, menyelesaikan kunci penandatanganan dari tembolok JWKS (menyegarkan sekali saat miss), memverifikasi tanda tangan, lalu memeriksa `iss` terhadap daftar yang diizinkan, `aud` terhadap sumber daya kanonik server ini, `exp`, dan cakupan yang dibutuhkan — mengembalikan tantangan `WWW-Authenticate` pada kegagalan pertama. Menjadikannya satu rutin di server sumber daya berarti setiap titik masuk (setiap panggilan alat, setiap transpor) melewati pemeriksaan yang sama; tidak ada jalur yang mencapai alat tanpa validasi terlebih dahulu.

### Token buram memakai introspection, bukan tebakan

Tidak semua token akses adalah JWT. Jika penerbit mendokumentasikan token buram, server sumber daya tidak dapat mendekodenya menjadi klaim yang dapat dipercaya. Server itu mengirim token ke endpoint introspection RFC 7662 milik penerbit melalui backchannel terautentikasi dan mewajibkan `active: true`, konteks penerbit yang diharapkan, audiens atau sumber daya MCP yang tepat, klaim waktu yang belum kedaluwarsa, serta cakupan yang dibutuhkan alat konkret.

Tembolokkan introspection berdasarkan penerbit, intisari token satu arah, dan sumber daya MCP. Jangan pernah memakai token jernih sebagai label log atau tembolok. Batasi entri tembolok positif pada waktu paling awal di antara kedaluwarsa token, petunjuk tembolok penerbit, dan sasaran kesegaran pencabutan deployment. Jaga tembolok negatif tetap pendek agar token yang baru diterbitkan tidak tetap terasa tidak aktif. Hasil untuk satu sumber daya tidak dapat mengotorisasi sumber daya lain meskipun string token buramnya identik.

Jangan memilih mode validasi dari isi token yang dikendalikan penyerang. Kunci perilaku JWT versus introspection pada metadata penerbit yang tervalidasi dan konfigurasi deployment. Pada jalur JWT, kunci algoritme yang diterima serta `jwks_uri` tepercaya; jangan pernah mengikuti URL kunci atau algoritme yang hanya dipilih oleh header token.

### Pencabutan adalah kontrak kesegaran

RFC 7009 memungkinkan klien meminta server otorisasi mencabut sebuah token. Permintaan itu tidak menghapus salinan yang telah ditembolokkan oleh setiap server sumber daya. Tetapkan penundaan pencabutan maksimum yang dapat diterima dan pastikan setiap tembolok mematuhinya.

Deployment token buram dapat mencapai pencabutan yang lebih ketat dengan melakukan introspection pada setiap panggilan berisiko tinggi atau memakai tembolok positif yang pendek. Deployment JWT mandiri biasanya menggabungkan masa hidup token akses yang pendek dengan pencabutan token penyegaran, pensiunan kunci untuk insiden lingkup penerbit, serta daftar hitam subjek, sesi, atau id token opsional untuk penolakan lokal darurat. JWT bertanda tangan tetap sah secara kriptografis sampai kedaluwarsa kecuali server sumber daya memiliki bukti pencabutan eksternal terkini.

Logout, penonaktifan akun, pencabutan persetujuan, dan respons insiden adalah pemicu berbeda, tetapi harus bertemu pada satu pernyataan yang terukur: setelah paling lama jendela pencabutan yang dinyatakan, setiap replika menolak kredensial itu. Uji pernyataan itu melalui penyeimbang beban, bukan hanya terhadap satu proses hangat.

### Kegagalan dependensi memerlukan keputusan yang dinyatakan

Jangan pernah mengimprovisasi kebijakan ketersediaan di dalam handler exception.

| Kegagalan | Perilaku produksi yang aman |
|-----------|-----------------------------|
| Penyegaran JWKS terjadwal gagal, `kid` yang dikenal tetap ada dalam tembolok terbatas yang masih berlaku | Lanjutkan hanya dalam jendela stale-on-error yang dinyatakan dan pancarkan bukti kesehatan yang menurun |
| Token memiliki `kid` yang tidak dikenal dan satu penyegaran yang diizinkan gagal | Tolak; jangan pernah menerima tanda tangan yang tidak dapat diverifikasi |
| Introspection tidak tersedia | Gagal tertutup untuk panggilan terlindungi; jangan mengubah kegagalan jaringan menjadi `active: true` |
| Metadata sumber daya terlindungi atau penerbit berubah secara tak terduga | Hentikan pendaftaran dan perolehan token baru; pertahankan hanya konfigurasi yang dikunci secara eksplisit dan belum kedaluwarsa di bawah kebijakan insiden terbatas |
| Endpoint pencabutan tidak tersedia | Laporkan logout atau pencabutan sebagai belum selesai, tahan kredensial secara lokal sebagai tidak dapat dipakai bila memungkinkan, dan jangan mengklaim pencabutan global berhasil |
| Sumber jam atau tipe klaim tidak valid | Tolak alih-alih memperlebar skew sampai token lolos |

Klasifikasikan kegagalan secara terpisah dari kredensial yang tidak valid. Pemadaman dependensi adalah galat operasional dengan kebijakan kesehatan dan percobaan ulang. Tanda tangan, penerbit, audiens, kedaluwarsa, atau cakupan yang buruk adalah penolakan otorisasi. Keduanya tidak boleh mencapai handler alat, dan keduanya tidak boleh membocorkan isi token ke bukti audit.

### Penelusuran replay audiens (pembatasan hak token akses)

Server A (`notes.example.com`) dan Server B (`tasks.example.com`) keduanya mendaftar ke server otorisasi yang sama. Server A dikompromi. Penyerang mengambil token notes milik pengguna dan memutarnya ulang ke Server B.

Validator Server B:

1. Dekode JWT, ambil JWKS berdasarkan `kid`, verifikasi tanda tangan.
2. Periksa `iss` terhadap `authorization_servers` dalam metadata sumber daya terlindunginya. (Lolos — IdP sama.)
3. Periksa `aud == "https://tasks.example.com"`. (Gagal — `aud` token adalah `https://notes.example.com`.)
4. Kembalikan 401 dengan `WWW-Authenticate: Bearer error="invalid_token", error_description="audience mismatch", resource_metadata="https://tasks.example.com/.well-known/oauth-protected-resource"`.

Klaim audiens adalah satu-satunya pertahanan terhadap serangan ini pada lapisan protokol. Melewatkannya demi performa adalah kesalahan produksi paling umum; validator harus berjalan pada setiap permintaan, bukan hanya saat awal sesi. Spesifikasi menyebut ini **pembatasan hak token akses**: server MCP **WAJIB** menolak token apa pun yang tidak menyebutnya dalam audiens.

> **Catatan penamaan.** Spesifikasi memakai istilah *confused deputy* untuk masalah yang terkait tetapi berbeda: server MCP yang bertindak sebagai **proxy** OAuth ke API pihak ketiga, memakai client id statis, dan meneruskan token tanpa memperoleh persetujuan pengguna per klien. Ikatan audiens memperbaiki replay di atas; perbaikan confused deputy adalah persetujuan per klien **plus** tidak pernah meneruskan token masuk ke API upstream (server MCP **WAJIB** memperoleh token upstream tersendiri).

### Serangan mix-up (pertahanan sisi klien yang tidak dapat diberikan server)

Klien berbicara dengan banyak server otorisasi sepanjang hidupnya. AS berbahaya dapat mencoba membuat klien menebus kode otorisasi AS jujur di endpoint token penyerang. Ikatan audiens tidak membantu di sini — serangan terjadi sebelum token apa pun ada. Pertahanannya ada di klien (RFC 9207):

1. Sebelum redirect, klien merekam `issuer` yang diharapkan dari metadata AS yang tervalidasi.
2. Pada respons otorisasi, klien membandingkan parameter `iss` yang dikembalikan dengan penerbit yang tercatat itu (perbandingan string sederhana, tanpa normalisasi) sebelum mengirim kode ke mana pun.
3. Ketidaksesuaian (atau `iss` tidak ada saat AS mengiklankan `authorization_response_iss_parameter_supported`) → tolak, dan jangan tampilkan field `error` sama sekali.

PKCE saja tidak menghentikan mix-up, karena klien menyerahkan `code_verifier`-nya ke endpoint token mana pun yang diarahkan kepadanya. Itulah sebabnya spesifikasi merekam penerbit per permintaan di samping verifier PKCE dan `state`.

### Mode kegagalan

- **JWKS basi.** Validator menolak token yang valid setelah AS memutar kunci. Perbaikannya adalah pola cron-refresh + cache-miss-refetch di atas. Jangan pernah menembolokkan JWKS tanpa pekerjaan penyegaran.
- **Rotasi sebagai fallback.** Menghubungkan jalur tembolok miss ke rotate-and-mint alih-alih pengambilan ulang adalah bug nyata: tidak pernah menghasilkan `kid` yang hilang, dan mengubah nilai `kid` yang dikendalikan penyerang menjadi DoS pembuatan kunci. Fallback harus berupa `refresh-jwks` yang idempoten.
- **Klaim `aud` tidak ada.** Beberapa IdP secara bawaan menghilangkan `aud` kecuali `resource` hadir dalam permintaan token. Validator harus menolak token dengan `aud` yang tidak ada, bukan menganggap ketiadaan sebagai wildcard.
- **Mix-up karena pemeriksaan `iss` tidak ada.** Klien yang tidak memvalidasi parameter respons otorisasi `iss` RFC 9207 terhadap penerbit yang tercatat sebelum redirect dapat diarahkan untuk menebus kode AS jujur di endpoint token penyerang. Ini kegagalan sisi klien; server sumber daya tidak dapat mengompensasinya.
- **Balapan peningkatan cakupan.** Dua alur step-up bersamaan untuk pengguna yang sama dapat sama-sama berhasil dan menghasilkan dua token akses dengan cakupan berbeda. Validator harus memakai token yang disajikan pada permintaan, bukan mencari "cakupan terkini pengguna" — itu menciptakan jendela TOCTOU.
- **Pencurian token pendaftaran.** `registration_access_token` yang bocor memungkinkan penyerang menulis ulang URI redirect. Hashkan saat disimpan; wajibkan klien menyajikan teks jernih pada setiap pembaruan; putar jika ada kecurigaan.
- **`iss` tidak dikunci.** Validator yang menerima `iss` apa pun memungkinkan penyerang mendirikan server otorisasinya sendiri, mendaftarkan klien untuk audiens target, dan menerbitkan token. Daftar `authorization_servers` dalam metadata sumber daya terlindungi adalah daftar yang diizinkan; tegakkan itu.
- **Tabrakan tembolok kredensial atau token.** Klien yang mengunci pendaftaran hanya berdasarkan sumber daya dapat menyajikan identitas satu server otorisasi ke server lain. Klien yang mengunci token akses hanya berdasarkan penerbit dapat memutar ulang token di audiens yang salah. Kunci pendaftaran berdasarkan penerbit yang tervalidasi, kunci token akses berdasarkan `(issuer, resource)`, dan daftar ulang setiap kali penerbit berubah.

```figure
t3-jwks-rotate
```

## Gunakanlah

`code/main.py` menelusuri alur produksi lengkap dengan Python stdlib dan tiga peran: `AuthorizationServer`, `ResourceServer`, dan `Client`. Alurnya:

Dari akar repositori, jalankan:

```bash
cd phases/13-tools-and-protocols/18-mcp-auth-production
python3 code/main.py
npx tsx code/main.ts
python3 -m unittest discover -s code/tests -v
```

Perintah pertama mencetak transkrip pendaftaran terikat penerbit dan validasi
token. Perintah kedua melaporkan delapan belas pemeriksaan yang lulus. Keduanya
tidak membuka pendengar jaringan maupun menulis kredensial.

1. Server otorisasi menerbitkan metadata RFC 8414 di `/.well-known/oauth-authorization-server`.
2. Klien MCP memanggil endpoint metadata dan memeriksa opsi pendaftarannya (`client_id_metadata_document_supported` untuk CIMD, `registration_endpoint` untuk DCR) serta dukungan PKCE `S256`.
3. Klien memeriksa pra-pendaftaran yang dibatasi penerbit; jika tidak ada, klien mendaftar dengan Client ID Metadata Document HTTPS-nya. DCR yang tidak digunakan tetap menjadi metode kompatibilitas yang dapat diuji terpisah.
4. Klien merekam penerbit yang tervalidasi, membuat tantangan S256, menerima kode otorisasi sekali pakai plus `iss`, memvalidasi penerbit yang dikembalikan, dan menebus kode dengan verifier asli serta indikator `resource` RFC 8707.
5. Klien MCP memanggil alat di server MCP dengan `Authorization: Bearer ...`.
6. Server MCP menjalankan `validate`, menyelesaikan kunci penandatanganan dari tembolok JWKS.
7. IdP memutar kunci; penyegaran terjadwal menarik ulang JWKS ke dalam tembolok.
8. Panggilan berikutnya tervalidasi terhadap kunci yang telah disegarkan tanpa restart, dan token sebelumnya tetap tervalidasi selama jendela tumpang tindih.
9. Upaya replay audiens terhadap sumber daya MCP lain mendapat 401 dengan `audience mismatch` dan penunjuk `resource_metadata`.

JWT di sini memakai HS256 dengan rahasia bersama (agar pelajaran berjalan hanya dengan stdlib). Produksi memakai RS256 atau EdDSA dengan pola JWKS di atas; logika validasinya tetap sama. Karena IdP dan server sumber daya berada dalam satu proses, `refresh_jwks` membaca daftar kunci server otorisasi secara langsung; melalui kawat, itu adalah `GET` HTTP ke `jwks_uri`.

## Kirimkan

Pelajaran ini menghasilkan `outputs/skill-mcp-auth.md`. Dengan konfigurasi server MCP dan himpunan kapabilitas IdP, skill ini mengeluarkan permukaan auth yang perlu disiapkan — metadata sumber daya terlindungi, jalur pendaftaran yang dipakai (CIMD, pra-pendaftaran, atau fallback DCR), jadwal penyegaran JWKS, pemetaan cakupan, dan aturan penolakan yang diterapkan ketika IdP tidak mendukung profil RFC lengkap.

## Latihan

1. Jalankan `code/main.py`. Telusuri alurnya. Perhatikan bagaimana IdP memutar kunci pada langkah 6 dari keluaran skrip (langkah 7 dari alur di atas), `refresh_jwks` terjadwal menarik ulang kumpulan yang diterbitkan, dan baik token lama (jendela tumpang tindih) maupun token segar tervalidasi tanpa restart.

2. Tambahkan IdP baru ke daftar `authorization_servers` metadata sumber daya terlindungi. Terbitkan token yang ditandatangani IdP baru dan pastikan validator menerimanya. Terbitkan token yang ditandatangani IdP yang tidak terdaftar dan pastikan validator menolaknya dengan `WWW-Authenticate: Bearer error="invalid_token", error_description="iss not allowed"`.

3. Tambahkan pemeriksaan batas laju pada `register_client` yang berjalan sebelum registrar menerima permintaan. Gunakan token-bucket per IP sumber yang disimpan dalam dict kecil berbasis kunci IP.

4. Baca RFC 7591 dan identifikasi dua field yang tidak divalidasi oleh handler `/register` pelajaran ini. Tambahkan validasinya. (Petunjuk: `software_statement` dan skema URI `redirect_uris`.)

5. Tambahkan server otorisasi kedua. Pastikan klien menyimpan pendaftaran terpisah berbasis penerbit dan menolak menggunakan ulang token atau `client_id` penerbit pertama.

6. Buktikan perbaikan DoS. Kirim ke validator sebuah token dengan `kid` acak dan pastikan `refresh_jwks` berjalan paling banyak sekali serta jumlah kunci server otorisasi tidak bertambah. Lalu sengaja sambung ulang fallback ke rotate-and-mint dan lihat jumlah kunci naik per token palsu — kembalikan ke pengambilan ulang setelahnya.

7. Uji DCR yang tidak digunakan lagi dengan klien `native` maupun `web`. Pastikan klien web dengan URI redirect HTTP dan klien native tanpa redirect loopback yang tepat ditolak.

## Istilah Kunci

| Istilah | Yang sering dikatakan | Makna sebenarnya |
|---------|------------------------|------------------|
| ASM | "dokumen metadata OAuth" | JSON RFC 8414 di `/.well-known/oauth-authorization-server` |
| CIMD | "URL metadata klien" | Client ID Metadata Document: URL HTTPS yang dipakai sebagai `client_id`; AS menarik JSON-nya. Pendaftaran yang diutamakan dalam MCP 2026-07-28 |
| DCR | "pendaftaran klien swalayan" | RFC 7591 `POST /register`; tidak digunakan lagi untuk MCP terkini dan hanya dipertahankan untuk kompatibilitas |
| JWKS | "kunci publik untuk validasi JWT" | JSON Web Key Set, diambil dari `jwks_uri`, diindeks berdasarkan `kid` |
| Rotasi vs penyegaran | "memperbarui kunci" | *Rotasi* = AS menerbitkan/mensiunkan kunci penandatanganan; *penyegaran* = server sumber daya mengambil ulang kumpulan yang diterbitkan. Server sumber daya hanya pernah melakukan penyegaran |
| Indikator sumber daya | "parameter audiens" | Parameter `resource` RFC 8707 yang mengunci token ke satu server |
| Klaim `aud` | "Audiens" | Klaim JWT yang dibandingkan validator dengan URL sumber daya kanonik |
| Replay audiens | "Pemutaran ulang token" | Token yang diterbitkan untuk Server A disajikan ke Server B; dicegah oleh validasi audiens (spesifikasi: pembatasan hak token akses) |
| Confused deputy | "Penyalahgunaan token proxy" | Proxy MCP dengan client id statis yang meneruskan token tanpa persetujuan per klien; berbeda dari replay audiens |
| Serangan mix-up | "Endpoint token yang salah" | Klien diarahkan menebus kode AS jujur di endpoint penyerang; dicegah di sisi klien lewat `iss` RFC 9207 |
| Daftar `iss` yang diizinkan | "Server otorisasi tepercaya" | Himpunan yang disebut dalam `authorization_servers` metadata sumber daya terlindungi |
| `resource_metadata` | "Tempat mencari dokumen PRM" | Parameter `WWW-Authenticate` yang menyebut URL metadata RFC 9728 pada 401/403 |
| Klien publik | "Klien native atau peramban" | Klien OAuth tanpa `client_secret`; PKCE menggantikannya |
| `WWW-Authenticate` | "Header respons 401/403" | Membawa direktif `Bearer error=...` yang mendorong pemulihan klien |

## Bacaan Lanjutan

- [Spesifikasi otorisasi MCP (2026-07-28)](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization) — profil otorisasi MCP terkini
- [Catatan perubahan MCP 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/changelog) — CIMD, validasi penerbit, penghapusan DCR, dan perubahan kredensial berbasis penerbit
- [OAuth Client ID Metadata Document (draft-ietf-oauth-client-id-metadata-document-00)](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-client-id-metadata-document-00) — CIMD
- [RFC 8414 — OAuth 2.0 Authorization Server Metadata](https://datatracker.ietf.org/doc/html/rfc8414) — kontrak penemuan
- [RFC 7591 — OAuth 2.0 Dynamic Client Registration Protocol](https://datatracker.ietf.org/doc/html/rfc7591) — DCR (jalur fallback)
- [RFC 7636 — Proof Key for Code Exchange (PKCE)](https://datatracker.ietf.org/doc/html/rfc7636) — bukti kepemilikan klien publik
- [RFC 8707 — Resource Indicators for OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc8707) — penguncian audiens
- [RFC 9728 — OAuth 2.0 Protected Resource Metadata](https://datatracker.ietf.org/doc/html/rfc9728) — penemuan server sumber daya
- [RFC 9207 — OAuth 2.0 Authorization Server Issuer Identification](https://datatracker.ietf.org/doc/html/rfc9207) — parameter `iss` yang menahan serangan mix-up
- [RFC 7662: OAuth 2.0 Token Introspection](https://datatracker.ietf.org/doc/html/rfc7662)
- [RFC 7009: OAuth 2.0 Token Revocation](https://datatracker.ietf.org/doc/html/rfc7009)
